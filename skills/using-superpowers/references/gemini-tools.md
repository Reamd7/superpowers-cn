# Gemini CLI 工具映射

Skills 使用 Claude Code 的工具名称。当你在 skill 中遇到这些名称时，请使用你所处平台的对应工具：

| Skill 中的引用 | Gemini CLI 对应工具 |
|-----------------|----------------------|
| `Read`（读取文件） | `read_file` |
| `Write`（创建文件） | `write_file` |
| `Edit`（编辑文件） | `replace` |
| `Bash`（运行命令） | `run_shell_command` |
| `Grep`（搜索文件内容） | `grep_search` |
| `Glob`（按名称搜索文件） | `glob` |
| `TodoWrite`（任务跟踪） | `write_todos` |
| `Skill` 工具（调用 skill） | `activate_skill` |
| `WebSearch` | `google_web_search` |
| `WebFetch` | `web_fetch` |
| `Task` 工具（派发子代理） | 无对应工具 — Gemini CLI 不支持子代理 |

## 不支持子代理

Gemini CLI 没有与 Claude Code 的 `Task` 工具对等的工具。依赖子代理派发的 skills（`subagent-driven-development`、`dispatching-parallel-agents`）将通过 `executing-plans` 回退为单会话执行。

## Gemini CLI 额外工具

这些工具在 Gemini CLI 中可用，但在 Claude Code 中没有对应工具：

| 工具 | 用途 |
|------|------|
| `list_directory` | 列出文件和子目录 |
| `save_memory` | 将信息持久化到 GEMINI.md，跨会话保留 |
| `ask_user` | 向用户请求结构化输入 |
| `tracker_create_task` | 丰富的任务管理（创建、更新、列表、可视化） |
| `enter_plan_mode` / `exit_plan_mode` | 在进行更改前切换到只读研究模式 |
