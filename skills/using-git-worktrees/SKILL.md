---
name: using-git-worktrees
description: 当开始需要与当前工作区隔离的功能开发，或在执行实现计划之前，需要使用这个 skill；它会通过智能目录选择和安全校验来创建隔离的 git worktree
---

# 使用 Git Worktrees

## 概述

Git worktree 可以在共享同一个仓库的前提下创建隔离工作区，让你在多个分支上同时工作，而不需要来回切换。

**核心原则：** 系统化地选择目录 + 做安全校验 = 可靠隔离。

**开始时要明确说明：** "I'm using the using-git-worktrees skill to set up an isolated workspace."

## 目录选择流程

按下面的优先级顺序执行：

### 1. 检查已有目录

```bash
ls -d .worktrees 2>/dev/null
ls -d worktrees 2>/dev/null
```

**如果找到了：** 用这个目录。如果两个都存在，优先 `.worktrees`。

### 2. 检查 `CLAUDE.md`

```bash
grep -i "worktree.*director" CLAUDE.md 2>/dev/null
```

**如果指定了偏好：** 直接使用，不必再问。

### 3. 询问用户

如果既没有已有目录，也没有 `CLAUDE.md` 偏好：

```text
No worktree directory found. Where should I create worktrees?

1. .worktrees/ (project-local, hidden)
2. ~/.config/superpowers/worktrees/<project-name>/ (global location)

Which would you prefer?
```

## 安全校验

### 对项目内目录（`.worktrees` 或 `worktrees`）

**在创建 worktree 之前，必须确认目录被 git 忽略：**

```bash
git check-ignore -q .worktrees 2>/dev/null || git check-ignore -q worktrees 2>/dev/null
```

**如果没有被忽略：**

根据 Jesse 的规则“坏掉的东西要立刻修”：
1. 在 `.gitignore` 里加上正确的条目
2. 提交这个变更
3. 再继续创建 worktree

**为什么重要：** 防止把 worktree 目录内容误提交进仓库。

### 对全局目录（`~/.config/superpowers/worktrees`）

不需要 `.gitignore` 校验，因为它在项目外部。

## 创建步骤

### 1. 检测项目名

```bash
project=$(basename "$(git rev-parse --show-toplevel)")
```

### 2. 创建 worktree

```bash
case $LOCATION in
  .worktrees|worktrees)
    path="$LOCATION/$BRANCH_NAME"
    ;;
  ~/.config/superpowers/worktrees/*)
    path="~/.config/superpowers/worktrees/$project/$BRANCH_NAME"
    ;;
esac

git worktree add "$path" -b "$BRANCH_NAME"
cd "$path"
```

### 3. 执行项目初始化

自动探测并运行合适的初始化命令：

```bash
if [ -f package.json ]; then npm install; fi
if [ -f Cargo.toml ]; then cargo build; fi
if [ -f requirements.txt ]; then pip install -r requirements.txt; fi
if [ -f pyproject.toml ]; then poetry install; fi
if [ -f go.mod ]; then go mod download; fi
```

### 4. 验证干净基线

运行测试，确保这个 worktree 一开始就是干净的：

```bash
npm test
cargo test
pytest
go test ./...
```

**如果测试失败：** 报告失败情况，并询问要继续还是先调查。

**如果测试通过：** 报告工作区已就绪。

### 5. 报告位置

```text
Worktree ready at <full-path>
Tests passing (<N> tests, 0 failures)
Ready to implement <feature-name>
```

## 快速参考

| 情况 | 行动 |
|------|------|
| `.worktrees/` 存在 | 使用它，并验证被忽略 |
| `worktrees/` 存在 | 使用它，并验证被忽略 |
| 两者都存在 | 使用 `.worktrees/` |
| 两者都不存在 | 先查 `CLAUDE.md`，再问用户 |
| 目录未被忽略 | 添加到 `.gitignore` 并提交 |
| 基线测试失败 | 报告失败并询问 |
| 没有 `package.json`/`Cargo.toml` | 跳过依赖安装 |

## 常见错误

### 跳过 ignore 校验

- **问题：** worktree 内容被 git 跟踪，污染状态
- **修复：** 创建项目内 worktree 前，始终先用 `git check-ignore`

### 擅自假定目录位置

- **问题：** 破坏项目约定，制造不一致
- **修复：** 遵循优先级：已有目录 > `CLAUDE.md` > 询问用户

### 带着失败的测试继续

- **问题：** 无法区分新 bug 和已有问题
- **修复：** 报告失败，并取得继续的明确许可

### 硬编码初始化命令

- **问题：** 在不同工具链项目中容易失效
- **修复：** 根据项目文件自动探测

## 示例工作流

```text
You: I'm using the using-git-worktrees skill to set up an isolated workspace.

[Check .worktrees/ - exists]
[Verify ignored - git check-ignore confirms .worktrees/ is ignored]
[Create worktree: git worktree add .worktrees/auth -b feature/auth]
[Run npm install]
[Run npm test - 47 passing]

Worktree ready at /Users/jesse/myproject/.worktrees/auth
Tests passing (47 tests, 0 failures)
Ready to implement auth feature
```

## Red Flag

**绝不要：**
- 在未确认目录被忽略的情况下创建项目内 worktree
- 跳过基线测试验证
- 在测试失败时不问就继续往下走
- 目录位置有歧义时擅自决定
- 跳过 `CLAUDE.md` 检查

**始终要：**
- 遵循目录优先级：已有目录 > `CLAUDE.md` > 询问用户
- 对项目内目录先验证已被忽略
- 自动探测并执行项目初始化
- 验证干净测试基线

## 集成关系

**会由以下流程调用：**
- **`brainstorming`** - 在设计获批后、开始实现前必须调用
- **`subagent-driven-development`** - 执行任何任务前必须调用
- **`executing-plans`** - 执行任何任务前必须调用
- 任何需要隔离工作区的 skill

**与之配套：**
- **`finishing-a-development-branch`** - 工作完成后负责清理 worktree
