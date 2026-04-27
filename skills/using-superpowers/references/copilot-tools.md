# Copilot CLI 工具映射

Skill 使用 Claude Code 的工具名。当你在 skill 中遇到这些工具时，请使用你所在平台的等价工具：

| Skill 中引用的工具 | Copilot CLI 等价工具 |
|-----------------|----------------------|
| `Read`（读取文件） | `view` |
| `Write`（创建文件） | `create` |
| `Edit`（编辑文件） | `edit` |
| `Bash`（运行命令） | `bash` |
| `Grep`（搜索文件内容） | `grep` |
| `Glob`（按名称搜索文件） | `glob` |
| `Skill` 工具（调用 skill） | `skill` |
| `WebFetch` | `web_fetch` |
| `Task` 工具（派发子代理） | `task`（参见[代理类型](#agent-types)） |
| 多个 `Task` 调用（并行） | 多个 `task` 调用 |
| 任务状态/输出 | `read_agent`、`list_agents` |
| `TodoWrite`（任务追踪） | `sql` 配合内置 `todos` 表 |
| `WebSearch` | 无等价工具 — 使用 `web_fetch` 加搜索引擎 URL |
| `EnterPlanMode` / `ExitPlanMode` | 无等价工具 — 留在主会话中 |

## 代理类型

Copilot CLI 的 `task` 工具接受 `agent_type` 参数：

| Claude Code 代理 | Copilot CLI 等价工具 |
|-------------------|----------------------|
| `general-purpose` | `"general-purpose"` |
| `Explore` | `"explore"` |
| 命名插件代理（例如 `superpowers:code-reviewer`） | 从已安装的插件中自动发现 |

## 异步 Shell 会话

Copilot CLI 支持持久化的异步 Shell 会话，Claude Code 中没有直接等价的功能：

| 工具 | 用途 |
|------|---------|
| `bash` 加 `async: true` | 在后台启动长时间运行的命令 |
| `write_bash` | 向运行中的异步会话发送输入 |
| `read_bash` | 读取异步会话的输出 |
| `stop_bash` | 终止异步会话 |
| `list_bash` | 列出所有活跃的 Shell 会话 |

## 其他 Copilot CLI 工具

| 工具 | 用途 |
|------|---------|
| `store_memory` | 持久化保存关于代码库的事实，供未来会话使用 |
| `report_intent` | 更新 UI 状态行，显示当前意图 |
| `sql` | 查询会话的 SQLite 数据库（待办事项、元数据） |
| `fetch_copilot_cli_documentation` | 查阅 Copilot CLI 文档 |
| GitHub MCP 工具（`github-mcp-server-*`） | 原生 GitHub API 访问（issues、PR、代码搜索） |