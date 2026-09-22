# Agent CLI 实现方案与分阶段计划

本文把产品与交互契约落到具体代码结构、迁移步骤和验收标准。原则是最小侵入：复用现有 Tool，不重写业务数据层，不影响 Wails GUI。

## 总体结构

建议新增：

```text
cmd/
└── gostock/
    └── main.go

backend/
├── capability/
│   ├── registry.go
│   ├── metadata.go
│   ├── executor.go
│   ├── errors.go
│   └── registry_test.go
├── headless/
│   ├── bootstrap.go
│   └── bootstrap_test.go
├── cli/
│   ├── root.go
│   ├── tools.go
│   ├── doctor.go
│   ├── output.go
│   └── cli_test.go
└── mcpserver/
    ├── server.go
    └── server_test.go
```

若实际实现中现有包依赖导致循环，应优先调整包边界，而不是复制 Tool handler。

## Phase 0：能力盘点与只读边界

### 目标

建立第一版 Agent-visible allowlist，并给 Tool 补充权限元数据。

### 工作

1. 从 `tools.GetAllDataTools()` 获取现有 `tool.BaseTool`；
2. 建立 `CapabilityMeta`：
   - group
   - side effect
   - risk
   - requires confirmation
   - agent visible
3. 第一版只开放读取能力；
4. 将 Operations 中的读取工具与写入工具区分，不能直接用 ToolGroup 推断权限。

### 第一批建议能力

至少覆盖：

- QueryStockCodeInfo
- GetStockInfo
- GetStockKLine
- GetEastMoneyKLine
- GetEastMoneyKLineWithMA
- GetStockMinuteData
- GetStockFinancialInfo
- GetStockHolderNum
- GetStockRZRQInfo
- GetStockConceptInfo
- GetStockMoneyData
- GetStockHistoryMoneyData
- GetMarketData
- GlobalStockIndexesReadable
- SearchNews
- GetNewsListData
- GetStockResearchReport
- StockNotice
- GetInvestCalendar
- FilterStocks
- SearchStockByIndicators

### 验收

- 任何 Agent-visible Tool 都有 metadata；
- 未分类的有副作用 Tool 默认不可执行；
- 读取工具无需确认；
- 写工具不能通过 `tools run` 直接执行。

## Phase 1：Headless Bootstrap

### 问题

当前桌面 `main.go` 启动时包含 Wails、WebView、菜单、窗口、SingleInstanceLock、assistant-web 等生命周期。CLI 不能复用完整 GUI 启动路径。

### 目标

提取 CLI 真正需要的最小初始化。

建议：

```go
func InitHeadless(opts Options) (*Runtime, error)
```

职责仅包括：

- 确定 root/data/config path；
- 初始化日志；
- 初始化 machine/build key 依赖（若业务工具需要）；
- 初始化数据库；
- 必要的数据源初始化；
- 返回 Close/cleanup。

明确不启动：

- Wails；
- browser/WebView；
- menu/tray；
- assistant-web；
- GUI event loop；
- SingleInstanceLock。

### 验收

- `gostock version` 不打开任何窗口；
- `gostock doctor --json` 可在无 GUI 环境运行；
- headless 启动失败返回错误，不 panic；
- 初始化日志只进入 stderr。

## Phase 2：Capability Registry

### API 草案

```go
type Registry interface {
    List(ctx context.Context, filter Filter) []Capability
    Search(ctx context.Context, query string) []Capability
    Describe(ctx context.Context, name string) (Capability, error)
    Run(ctx context.Context, name string, args json.RawMessage) (Result, error)
}
```

`Capability` 的 Schema 必须来自真实 `BaseTool.Info()`。

`Run` 最终调用真实 `BaseTool.InvokableRun()`。

### 禁止

- 手写第二份参数 Schema；
- 在 CLI 包直接调用 `NewStockDataApi().Xxx()` 绕过 Tool；
- CLI 与 MCP 分别维护 Tool allowlist。

### 验收

- Describe 与 `BaseTool.Info()` 一致；
- Run 与内置 Agent 调同一个 Tool handler；
- unknown tool 返回 typed error；
- invalid JSON / invalid args 返回 typed error；
- timeout 通过 context 控制。

## Phase 3：CLI

### 命令

先实现：

```text
gostock version
gostock doctor
gostock tools list
gostock tools search
gostock tools describe
gostock tools run
```

不要第一版就引入 150+ command。

### CLI 库

项目当前未引入 Cobra。第一版可优先使用标准库 `flag` 或轻量自定义 parser，避免仅为 6 个命令增加大型依赖。若后续命令增长再评估 Cobra。

### JSON 输出

统一 `Envelope`：

```go
type Envelope struct {
    Version   string      `json:"version"`
    OK        bool        `json:"ok"`
    Operation string      `json:"operation,omitempty"`
    Result    any         `json:"result,omitempty"`
    Meta      Meta        `json:"meta,omitempty"`
    Error     *ErrorBody  `json:"error,omitempty"`
}
```

### 验收

- 所有 `--json` 成功与错误路径均为单个合法 JSON；
- stdout 无日志；
- stderr 无协议数据；
- exit code 与错误类型一致；
- PowerShell / Bash/Zsh 示例均能调用 JSON args。

## Phase 4：Doctor

### 检查项

- root/data/config/database 路径可访问；
- DB 初始化状态；
- network 基本状态；
- IwencaiApiKey / EmApiKey / QgqpBId 等只检查“已配置/未配置”，绝不输出值；
- Feishu/DingTalk 只输出 enabled/disabled；
- Agent-visible capabilities 总数；
- 当前 available/unavailable 数量。

### 验收

任何 doctor 输出都不得包含 secret 原文。

## Phase 5：MCP stdio Server

仓库已经依赖 `github.com/mark3labs/mcp-go`，优先复用现有依赖。

新增：

```bash
gostock mcp serve --stdio --read-only
```

MCP tool list 从 Capability Registry 动态生成：

```text
Capability Registry
      ├── CLI Adapter
      └── MCP Adapter
```

### 验收

- MCP tools/list 与 CLI `tools list --read-only` 的能力集合一致；
- 参数 Schema 来源一致；
- MCP tools/call 最终走同一个 executor；
- stdio stdout 只包含 JSON-RPC；
- Ctrl+C / parent cancel 能正常释放资源。

## Phase 6：Proposal / Approval / Audit

仅在只读链稳定后实现。

### Proposal

```go
type Proposal struct {
    ID        string
    Operation string
    Arguments json.RawMessage
    Effects   []string
    CreatedAt time.Time
    ExpiresAt time.Time
}
```

写操作流程：

```text
Agent request
   ↓
validate
   ↓
permission check
   ↓
proposal
   ↓
human approval
   ↓
execute
   ↓
audit log
```

禁止使用隐藏 flag 让 Agent 直接绕过确认。

### 验收

- local_write / external_write 默认不会直接执行；
- proposal 可查看、拒绝、执行；
- 执行前重新校验参数与权限；
- 审计日志记录 operation、时间、结果，不记录 secret。

## 测试策略

### 单元测试

- registry discover/describe/search；
- metadata 权限默认值；
- envelope serialization；
- exit code mapping；
- stdout/stderr separation；
- unknown tool；
- invalid args；
- write rejection。

### 集成测试

选不需要付费 Key 的稳定读取工具做 smoke test。网络数据源测试应显式标记 integration，避免普通 CI 因第三方波动失败。

### 建议命令

```bash
go test ./backend/capability/...
go test ./backend/headless/...
go test ./backend/cli/...
go test ./backend/mcpserver/...
go test ./...
```

如全量测试包含现有网络/环境依赖失败，PR 必须列出失败项并说明是否与本次改动相关。

## PR 拆分建议

后续代码按以下顺序独立提交，保持每个 PR 一个可验证目标：

1. `feat: add capability metadata and read-only registry`
2. `refactor: extract headless runtime bootstrap`
3. `feat: add agent-first CLI discovery and run commands`
4. `feat: add CLI doctor and stable machine output`
5. `feat: expose capabilities through MCP stdio server`
6. `feat: add proposal approval and audit for side effects`

每个 PR 必须基于最新 `dev`，避免提前堆叠大分支。

## Definition of Done

Agent CLI 第一阶段完成的最低标准：

- CLI 是独立 headless binary；
- 新 Tool 无需手写新 CLI command 即可被 Registry 发现；
- 至少 20 个核心只读股票能力可被 Agent 调用；
- machine mode stdout 可直接解析；
- 默认拒绝所有副作用能力；
- MCP 与 CLI 共享 Registry；
- GUI 原行为不受影响；
- 有自动化测试覆盖权限边界和机器协议。
