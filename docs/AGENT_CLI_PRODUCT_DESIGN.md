# Agent-first CLI 产品与架构设计

## 背景

go-stock 已经具备 Wails GUI、150+ AI 数据工具、React / PlanExecute / DeepAgents、Skill 与 MCP Client 能力。下一阶段需要让 Codex、Claude Code、OpenCode、Oh My Pi、WorkBuddy 等本地 Agent 能在不依赖 GUI 的情况下稳定调用 go-stock 的股票能力。

目标不是再复制一套股票 API，而是把现有工具能力抽象成唯一的 Capability Registry，并通过多个 Surface 暴露。

## 产品原则

> One capability registry, multiple surfaces; read by default, propose before write.

### 1. 单一能力源

GUI、CLI、MCP Server 与内置 Agent 必须复用同一套工具实现、参数 Schema 与权限元数据。禁止为了 CLI 再实现一套 `GetStockInfo` / `GetKLine` 等业务逻辑。

现有 `tools.GetAllDataTools()`、`tool.BaseTool.Info()` 与 `InvokableRun()` 应成为第一阶段能力注册表的基础。

### 2. Agent-first，而不是 command-first

第一阶段 CLI 不为 150+ 工具手写一套人类命令树，而提供通用协议：

- `gostock tools list`
- `gostock tools search <query>`
- `gostock tools describe <tool>`
- `gostock tools run <tool> --args '<json>'`

这保证新增 Tool 后无需同时维护新的 CLI command。

### 3. 默认只读

外部 Agent 默认只能调用无副作用能力，例如行情、K 线、财务、资金流、新闻、公告、研报与筛选。

本地写入、配置变更与外部消息发送必须显式标记副作用与风险等级。

### 4. Proposal before write

写操作不应仅依赖 `--yes`。目标模型为：

1. Agent 生成操作 Proposal；
2. CLI 返回结构化影响说明；
3. 用户确认后执行；
4. 写入 Audit Log。

第一阶段可只实现只读边界，但 Capability Metadata 必须为后续 Proposal/HITL 预留字段。

### 5. Headless 与 GUI 解耦

Agent 调一次行情不应启动 Wails 窗口、菜单、WebView、SingleInstanceLock 或独立 Web 服务。

CLI 应使用独立 headless bootstrap，只初始化其真正需要的数据、配置和日志设施。

## Capability 模型

建议统一元数据：

```go
type CapabilityMeta struct {
    Name            string
    Group           ToolGroup
    SideEffect      SideEffect
    Risk            RiskLevel
    RequiresConfirm bool
    AgentVisible    bool
}
```

建议副作用分类：

- `none`：纯读取；
- `local_write`：修改本地数据库/配置；
- `external_write`：向飞书、钉钉、Webhook 等外部系统发送数据。

## Surface

```text
                    ┌──────────── Wails GUI
                    │
Capability Registry ├──────────── CLI
                    │
                    ├──────────── built-in Agent
                    │
                    └──────────── MCP stdio Server
                                      │
                     Codex / CC / OMP / WorkBuddy
```

CLI 是所有本地 Agent 都能使用的 fallback；MCP 是支持 MCP 的客户端的 first-class integration。

## 非目标

第一阶段不做：

- 自动下单或券商交易接口；
- 未确认的写操作；
- 为每个工具设计独立 CLI 子命令；
- 重写现有数据抓取和 Tool handler；
- 改变 GUI 的现有交互；
- 把 CLI 与桌面 Wails 生命周期绑定。

## 成功标准

当该方向完成后，本地 Agent 可以：

1. 自发现 go-stock 可用能力；
2. 获取机器可读参数 Schema；
3. 以稳定 JSON 协议调用只读工具；
4. 在无 GUI 环境运行；
5. MCP 客户端从同一 Capability Registry 获得同样能力；
6. 未来写操作统一经过 Proposal / Approval / Audit，而不是绕过权限层。
