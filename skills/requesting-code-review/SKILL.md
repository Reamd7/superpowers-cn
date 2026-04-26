---
name: requesting-code-review
description: 在完成任务、实现主要功能或合并前使用，以验证工作是否符合要求
---

# 请求代码审查

派遣 `superpowers:code-reviewer` 子代理，在问题扩散之前捕获它们。审查者会收到精心构建的评估上下文——绝不是你的会话历史。这让审查者专注于工作产出，而不是你的思考过程，同时也为你保留上下文以便继续工作。

**核心原则：** 尽早审查，频繁审查。

## 何时请求审查

**必须审查：**
- 在子代理驱动开发中每完成一个任务后
- 完成主要功能后
- 合并到 main 分支前

**可选但推荐：**
- 遇到困难时（获取新视角）
- 重构前（基线检查）
- 修复复杂 bug 后

## 如何请求

**1. 获取 git SHA：**
```bash
BASE_SHA=$(git rev-parse HEAD~1)  # or origin/main
HEAD_SHA=$(git rev-parse HEAD)
```

**2. 派遣 `code-reviewer` 子代理：**

使用 Task 工具，类型为 `superpowers:code-reviewer`，填写 `code-reviewer.md` 中的模板

**占位符：**
- `{WHAT_WAS_IMPLEMENTED}` - 你刚刚构建的内容
- `{PLAN_OR_REQUIREMENTS}` - 它应该实现的功能
- `{BASE_SHA}` - 起始提交
- `{HEAD_SHA}` - 结束提交
- `{DESCRIPTION}` - 简要摘要

**3. 根据反馈行动：**
- 立即修复 Critical 级别问题
- 在继续之前修复 Important 级别问题
- 记录 Minor 级别问题，留待后续处理
- 如果审查者判断有误，可以反驳（需附上理由）

## 示例

```
[刚完成任务 2：添加验证函数]

你：让我在继续之前请求代码审查。

BASE_SHA=$(git log --oneline | grep "Task 1" | head -1 | awk '{print $1}')
HEAD_SHA=$(git rev-parse HEAD)

[派遣 superpowers:code-reviewer 子代理]
  WHAT_WAS_IMPLEMENTED: Verification and repair functions for conversation index
  PLAN_OR_REQUIREMENTS: Task 2 from docs/superpowers/plans/deployment-plan.md
  BASE_SHA: a7981ec
  HEAD_SHA: 3df7661
  DESCRIPTION: Added verifyIndex() and repairIndex() with 4 issue types

[子代理返回]：
  Strengths: Clean architecture, real tests
  Issues:
    Important: Missing progress indicators
    Minor: Magic number (100) for reporting interval
  Assessment: Ready to proceed

你：[修复进度指示器]
[继续任务 3]
```

## 与工作流集成

**子代理驱动开发：**
- 每个任务后都进行审查
- 在问题累积之前捕获它们
- 在进入下一个任务之前修复

**执行计划：**
- 每完成一批（3 个任务）后审查
- 获取反馈，应用修改，继续

**临时开发：**
- 合并前审查
- 遇到困难时审查

## Red Flag

**绝不要：**
- 因为"很简单"就跳过审查
- 忽略 Critical 级别问题
- 在 Important 级别问题未修复的情况下继续
- 对合理的技术反馈争辩

**如果审查者判断有误：**
- 用技术理由反驳
- 展示证明其正常工作的代码/测试
- 请求澄清

模板见：`requesting-code-review/code-reviewer.md`
