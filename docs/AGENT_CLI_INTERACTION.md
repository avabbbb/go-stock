# Agent CLI 交互与机器协议

本文定义 go-stock CLI 面向本地 Agent 与高级用户的交互契约。目标是让 Codex、Claude Code、OpenCode、Oh My Pi、WorkBuddy 等客户端无需理解 go-stock 内部实现即可稳定发现、描述和调用能力。

## 命令面

第一阶段只提供少量稳定命令：

```text
gostock
├── tools
│   ├── list
│   ├── search
│   ├── describe
│   └── run
├── proposal
│   ├── show
│   ├── execute
│   └── reject
├── config
│   ├── path
│   └── show
├── doctor
├── mcp
│   └── serve
└── version
```

其中 Proposal 与 MCP 可以分阶段实现，但命名和语义应保持兼容。

## Agent 发现流程

推荐 Agent 使用以下流程：

1. `gostock doctor --json` 确认运行环境；
2. `gostock tools search "<intent>" --json` 搜索相关工具；
3. `gostock tools describe <name> --json` 获取参数 Schema；
4. `gostock tools run <name> --args '<json>' --json` 执行；
5. 对副作用操作处理 `confirmation_required`，不得绕过 Proposal。

Agent 不应解析帮助文本来推断参数，也不应依赖人类可读 Markdown 作为协议字段。

## 稳定 JSON Envelope

所有 `--json` 模式均应输出单一 JSON 文档：

```json
{
  "version": "1",
  "ok": true,
  "operation": "GetStockInfo",
  "result": {
    "format": "markdown",
    "content": "..."
  },
  "meta": {
    "duration_ms": 238,
    "side_effect": "none"
  },
  "error": null
}
```

失败时：

```json
{
  "version": "1",
  "ok": false,
  "operation": "GetStockInfo",
  "result": null,
  "meta": {},
  "error": {
    "code": "invalid_arguments",
    "message": "stockCode is required"
  }
}
```

### 兼容策略

- `version` 是协议版本，不是应用版本；
- 新增可选字段不得破坏旧客户端；
- 删除或重命名字段需要提升协议 major；
- JSON mode 不得混入日志、进度条、ANSI 控制字符或提示语。

## stdout / stderr

这是硬约束：

- stdout：结果协议或 MCP JSON-RPC；
- stderr：日志、debug、warning、诊断信息。

在 `--json` 和 `mcp serve --stdio` 模式下，stdout 必须可被机器直接解析。

## tools list

```bash
gostock tools list --json
gostock tools list --group money_flow --json
gostock tools list --read-only --json
```

返回最小信息：

```json
{
  "tools": [
    {
      "name": "GetStockInfo",
      "group": "stock_analysis",
      "description": "获取股票基本行情",
      "side_effect": "none",
      "risk": "read",
      "available": true
    }
  ]
}
```

## tools search

```bash
gostock tools search "贵州茅台资金流" --json
```

搜索只负责能力发现，不执行工具。结果按名称、描述、Group 与关键词匹配。

## tools describe

```bash
gostock tools describe GetEastMoneyKLine --json
```

返回从真实 Tool Schema 生成的参数契约，不维护第二份手工 Schema。

## tools run

```bash
gostock tools run GetEastMoneyKLine \
  --args '{"stockCode":"600519.SH","kLineType":"day","limit":60}' \
  --json
```

基础参数：

- `--args`：JSON object；
- `--json`：机器输出；
- `--timeout`：单次调用超时；
- `--root`：可选工作根；
- `--read-only`：强制拒绝所有有副作用工具。

### Result format

现有 Tool 大量返回 Markdown。第一阶段允许：

```json
{
  "result": {
    "format": "markdown",
    "content": "..."
  }
}
```

后续工具可逐步升级为：

```json
{
  "result": {
    "format": "structured",
    "data": {}
  }
}
```

但 CLI 不应为了结构化输出重写现有业务工具。

## doctor

`doctor` 是 Agent 的启动探针：

```bash
gostock doctor --json
```

建议检查：

- data/config path；
- database；
- network；
- 已配置的数据源 Key；
- 当前可用/不可用工具数量；
- 只读模式状态。

不得输出 API Key、Token 或 OAuth secret。

## 权限交互

Capability 至少区分：

- `none`；
- `local_write`；
- `external_write`。

读取操作直接执行。

副作用操作默认返回：

```json
{
  "ok": false,
  "status": "confirmation_required",
  "proposal": {
    "id": "prop_xxx",
    "operation": "FollowStock",
    "arguments": {
      "stockCode": "600519.SH"
    },
    "effects": [
      "modify local watchlist"
    ]
  }
}
```

用户批准后再执行：

```bash
gostock proposal execute prop_xxx
```

第一阶段若尚未实现 Proposal，则所有副作用工具必须直接拒绝，而不是临时允许 `--yes` 绕过。

## 退出码

建议约定：

- `0`：成功；
- `2`：参数或 Schema 错误；
- `3`：工具不存在/不可用；
- `4`：权限拒绝或需要确认；
- `5`：上游数据源失败；
- `10`：内部错误。

JSON envelope 与 exit code 应一致。

## 人类模式

不带 `--json` 时可以保留 Markdown、表格与友好提示，但不得改变命令语义。

## MCP 交互

未来：

```bash
gostock mcp serve --stdio --read-only
```

MCP Server 必须从同一个 Capability Registry 注册工具，并复用同一权限元数据；不得出现“CLI 可调用但 MCP 使用另一套实现”的分叉。

## 兼容性验收

任何 CLI 改动至少验证：

1. JSON stdout 可被 `json.Unmarshal` 直接解析；
2. stderr 日志不会污染 stdout；
3. 未知工具返回稳定错误码；
4. 缺少必填参数返回 Schema 错误；
5. 默认拒绝副作用工具；
6. 同一 Tool 的 CLI Schema 与 `BaseTool.Info()` 一致；
7. Windows PowerShell、macOS/Linux shell 的 JSON 参数调用均有文档示例。
