# Codex 工具映射

Skills 使用 Claude Code 的工具名称。当你在 skill 中遇到这些名称时，请使用你所在平台的对应工具：

| Skill 引用 | Codex 对应工具 |
|-----------------|------------------|
| `Task` 工具（派发子代理） | `spawn_agent`（参见[命名代理派发](#named-agent-dispatch)） |
| 多个 `Task` 调用（并行） | 多个 `spawn_agent` 调用 |
| Task 返回结果 | `wait` |
| Task 自动完成 | `close_agent` 释放槽位 |
| `TodoWrite`（任务追踪） | `update_plan` |
| `Skill` 工具（调用 skill） | Skills 原生加载——直接按指令执行 |
| `Read`、`Write`、`Edit`（文件操作） | 使用你的原生文件工具 |
| `Bash`（运行命令） | 使用你的原生 shell 工具 |

## 子代理派发需要多代理支持

在你的 Codex 配置（`~/.codex/config.toml`）中添加：

```toml
[features]
multi_agent = true
```

这将启用 `spawn_agent`、`wait` 和 `close_agent`，以支持 `dispatching-parallel-agents` 和 `subagent-driven-development` 等 skills。

## 命名代理派发

Claude Code skills 引用命名代理类型，如 `superpowers:code-reviewer`。
Codex 没有命名代理注册表——`spawn_agent` 从内置角色（`default`、`explorer`、`worker`）创建通用代理。

当 skill 要求派发命名代理类型时：

1. 找到代理的 prompt 文件（例如 `agents/code-reviewer.md`，或 skill 的本地 prompt 模板如 `code-quality-reviewer-prompt.md`）
2. 读取 prompt 内容
3. 填充模板占位符（`{BASE_SHA}`、`{WHAT_WAS_IMPLEMENTED}` 等）
4. 以填充后的内容作为 `message`，派发一个 `worker` 代理

| Skill 指令 | Codex 对应方式 |
|-------------------|------------------|
| `Task tool (superpowers:code-reviewer)` | 使用 `code-reviewer.md` 内容执行 `spawn_agent(agent_type="worker", message=...)` |
| `Task tool (general-purpose)` 带内联 prompt | 使用相同 prompt 执行 `spawn_agent(message=...)` |

### 消息框架

`message` 参数是用户级输入，而非系统 prompt。请按以下结构组织以确保最大化指令遵从：

```
Your task is to perform the following. Follow the instructions below exactly.

<agent-instructions>
[从代理的 .md 文件中填充的 prompt 内容]
</agent-instructions>

Execute this now. Output ONLY the structured response following the format
specified in the instructions above.
```

- 使用任务委派框架（"Your task is..."）而非角色框架（"You are..."）
- 用 XML 标签包裹指令——模型会将标记块视为权威内容
- 以明确的执行指令结尾，防止模型对指令进行摘要

### 此变通方案何时可以移除

此方法补偿的是 Codex 的插件系统尚未在 `plugin.json` 中支持 `agents` 字段的问题。当 `RawPluginManifest` 获得 `agents` 字段后，插件可以符号链接到 `agents/`（镜像现有的 `skills/` 符号链接），skills 就可以直接派发命名代理类型了。

## 环境检测

创建 worktree 或完成分支的 skills 应在继续操作之前通过只读 git 命令检测其环境：

```bash
GIT_DIR=$(cd "$(git rev-parse --git-dir)" 2>/dev/null && pwd -P)
GIT_COMMON=$(cd "$(git rev-parse --git-common-dir)" 2>/dev/null && pwd -P)
BRANCH=$(git branch --show-current)
```

- `GIT_DIR != GIT_COMMON` → 已在链接的 worktree 中（跳过创建）
- `BRANCH` 为空 → 分离的 HEAD（无法在沙箱中创建分支/推送/PR）

关于每个 skill 如何使用这些信号，参见 `using-git-worktrees` 的步骤 0 和 `finishing-a-development-branch` 的步骤 1。

## Codex App 完成流程

当沙箱阻止分支/推送操作时（在外部管理的 worktree 中处于分离 HEAD 状态），代理会提交所有工作并告知用户使用 App 的原生控件：

- **"Create branch"**——命名分支，然后通过 App UI 提交/推送/创建 PR
- **"Hand off to local"**——将工作转移到用户的本地检出目录

代理仍可运行测试、暂存文件，并输出建议的分支名称、提交信息和 PR 描述供用户复制。