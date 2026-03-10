---
name: finishing-a-development-branch
description: 当实现已经完成、所有测试都通过，并且你需要决定如何集成这项工作时使用；它会通过结构化选项来引导你选择合并、PR 或清理路径
---

# 完成开发分支

## 概述

通过清晰选项来收尾开发工作，并按用户选择执行对应流程。

**核心原则：** 验证测试 -> 展示选项 -> 执行选择 -> 清理。

**开始时要明确说明：** "I'm using the finishing-a-development-branch skill to complete this work."

## 流程

### 第一步：验证测试

**在展示选项之前，必须先确认测试通过：**

```bash
npm test / cargo test / pytest / go test ./...
```

**如果测试失败：**

```text
Tests failing (<N> failures). Must fix before completing:

[Show failures]

Cannot proceed with merge/PR until tests pass.
```

停止。不要进入第二步。

**如果测试通过：** 继续第二步。

### 第二步：确定基线分支

```bash
git merge-base HEAD main 2>/dev/null || git merge-base HEAD master 2>/dev/null
```

或者直接询问："This branch split from main - is that correct?"

### 第三步：展示选项

**必须原样提供下面这 4 个选项：**

```text
Implementation complete. What would you like to do?

1. Merge back to <base-branch> locally
2. Push and create a Pull Request
3. Keep the branch as-is (I'll handle it later)
4. Discard this work

Which option?
```

**不要额外解释。** 选项必须保持简洁。

### 第四步：执行用户选择

#### 选项 1：本地合并

```bash
git checkout <base-branch>
git pull
git merge <feature-branch>
<test command>
git branch -d <feature-branch>
```

然后执行：工作树清理（第五步）

#### 选项 2：推送并创建 PR

```bash
git push -u origin <feature-branch>
gh pr create --title "<title>" --body "$(cat <<'EOF'
## Summary
<2-3 bullets of what changed>

## Test Plan
- [ ] <verification steps>
EOF
)"
```

然后执行：工作树清理（第五步）

#### 选项 3：保持现状

报告："Keeping branch <name>. Worktree preserved at <path>."

**不要清理 worktree。**

#### 选项 4：丢弃

**必须先确认：**

```text
This will permanently delete:
- Branch <name>
- All commits: <commit-list>
- Worktree at <path>

Type 'discard' to confirm.
```

等待用户输入精确的 `discard`。

如果确认：

```bash
git checkout <base-branch>
git branch -D <feature-branch>
```

然后执行：工作树清理（第五步）

### 第五步：清理 worktree

**针对选项 1、2、4：**

```bash
git worktree list | grep $(git branch --show-current)
```

如果当前分支是 worktree：

```bash
git worktree remove <worktree-path>
```

**针对选项 3：** 保留 worktree。

## 快速参考

| 选项 | 合并 | 推送 | 保留 Worktree | 清理分支 |
|------|------|------|---------------|----------|
| 1. 本地合并 | ✓ | - | - | ✓ |
| 2. 创建 PR | - | ✓ | ✓ | - |
| 3. 保持现状 | - | - | ✓ | - |
| 4. 丢弃 | - | - | - | ✓（强制） |

## 常见错误

**跳过测试验证**
- **问题：** 合并坏代码，或创建一个会失败的 PR
- **修复：** 展示选项前必须先验证测试

**提开放式问题**
- **问题：** “接下来怎么做？” -> 太模糊
- **修复：** 必须只给出结构化的 4 个选项

**自动清理 worktree**
- **问题：** 在仍可能需要 worktree 时把它删掉
- **修复：** 只有选项 1 和 4 才清理

**丢弃前不确认**
- **问题：** 误删工作成果
- **修复：** 必须要求用户手动输入 `discard`

## 红旗

**绝不要：**
- 在测试失败时继续往下走
- 不验证合并结果就合并
- 未确认就删除工作成果
- 未经明确请求强推 `force-push`

**始终要：**
- 在展示选项前先验证测试
- 只展示这 4 个选项
- 对选项 4 获取用户手动确认
- 只在选项 1 和 4 时清理 worktree

## 集成关系

**调用方：**
- **`subagent-driven-development`**(第 7 步)- 所有任务完成后调用
- **`executing-plans`** (第 5 步) - 所有批次完成后调用

**配套 skill：**
- **`using-git-worktrees`** - 用于清理该 skill 创建的 worktree
