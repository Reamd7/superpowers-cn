---
name: finishing-a-development-branch
description: 当实现完成、所有测试通过，需要决定如何集成这些工作时使用——通过提供合并、PR 或清理的结构化选项来指导完成开发工作
---

# 完成开发分支

## 概述

通过提供清晰的选项并处理所选工作流，指导完成开发工作。

**核心原则：** 验证测试 → 展示选项 → 执行选择 → 清理。

**开始时宣布：** "I'm using the finishing-a-development-branch skill to complete this work."

## 流程

### 步骤 1：验证测试

**在展示选项之前，验证测试通过：**

```bash
# Run project's test suite
npm test / cargo test / pytest / go test ./...
```

**如果测试失败：**
```
Tests failing (<N> failures). Must fix before completing:

[Show failures]

Cannot proceed with merge/PR until tests pass.
```

停止。不要进入步骤 2。

**如果测试通过：** 继续步骤 2。

### 步骤 2：确定基准分支

```bash
# Try common base branches
git merge-base HEAD main 2>/dev/null || git merge-base HEAD master 2>/dev/null
```

或者询问："This branch split from main - is that correct?"

### 步骤 3：展示选项

展示恰好这 4 个选项：

```
Implementation complete. What would you like to do?

1. Merge back to <base-branch> locally
2. Push and create a Pull Request
3. Keep the branch as-is (I'll handle it later)
4. Discard this work

Which option?
```

**不要添加解释** - 保持选项简洁。

### 步骤 4：执行选择

#### 选项 1：本地合并

```bash
# Switch to base branch
git checkout <base-branch>

# Pull latest
git pull

# Merge feature branch
git merge <feature-branch>

# Verify tests on merged result
<test command>

# If tests pass
git branch -d <feature-branch>
```

然后：清理工作树（步骤 5）

#### 选项 2：推送并创建 PR

```bash
# Push branch
git push -u origin <feature-branch>

# Create PR
gh pr create --title "<title>" --body "$(cat <<'EOF'
## Summary
<2-3 bullets of what changed>

## Test Plan
- [ ] <verification steps>
EOF
)"
```

然后：清理工作树（步骤 5）

#### 选项 3：保持现状

报告："Keeping branch <name>. Worktree preserved at <path>."

**不要清理工作树。**

#### 选项 4：丢弃

**先确认：**
```
This will permanently delete:
- Branch <name>
- All commits: <commit-list>
- Worktree at <path>

Type 'discard' to confirm.
```

等待精确确认。

确认后：
```bash
git checkout <base-branch>
git branch -D <feature-branch>
```

然后：清理工作树（步骤 5）

### 步骤 5：清理工作树

**适用于选项 1、2、4：**

检查是否在工作树中：
```bash
git worktree list | grep $(git branch --show-current)
```

如果是：
```bash
git worktree remove <worktree-path>
```

**对于选项 3：** 保留工作树。

## 快速参考

| 选项 | 合并 | 推送 | 保留工作树 | 清理分支 |
|------|------|------|-----------|---------|
| 1. 本地合并 | ✓ | - | - | ✓ |
| 2. 创建 PR | - | ✓ | ✓ | - |
| 3. 保持现状 | - | - | ✓ | - |
| 4. 丢弃 | - | - | - | ✓ (强制) |

## 常见错误

**跳过测试验证**
- **问题：** 合并损坏的代码，创建失败的 PR
- **修复：** 始终在展示选项前验证测试

**开放式提问**
- **问题：** "What should I do next?" → 含糊不清
- **修复：** 展示恰好 4 个结构化选项

**自动清理工作树**
- **问题：** 在可能还需要工作树时将其删除（选项 2、3）
- **修复：** 仅对选项 1 和 4 进行清理

**丢弃操作无确认**
- **问题：** 意外删除工作成果
- **修复：** 要求输入 "discard" 确认

## Red Flag

**绝不要：**
- 在测试失败时继续操作
- 在未验证合并结果测试的情况下合并
- 未经确认删除工作成果
- 未经明确请求进行 force-push

**始终：**
- 在展示选项前验证测试
- 展示恰好 4 个选项
- 对选项 4 获取输入确认
- 仅对选项 1 和 4 清理工作树

## 集成

**被以下 skill 调用：**
- **subagent-driven-development**（步骤 7）- 所有任务完成后
- **executing-plans**（步骤 5）- 所有批次完成后

**配合使用：**
- **using-git-worktrees** - 清理由该 skill 创建的工作树
