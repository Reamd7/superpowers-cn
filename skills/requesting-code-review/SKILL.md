---
name: requesting-code-review
description: 当完成任务、实现主要功能或在合并前，需要验证工作是否符合要求时使用
---

# 请求代码审查

派发 `superpowers:code-reviewer` 子代理，在问题扩散之前把它们抓出来。

**核心原则：** 尽早审查，频繁审查。

## 什么时候请求审查

**必须请求：**
- 在子代理驱动开发中，每完成一个任务之后
- 完成一个主要功能之后
- 合并到主分支之前

**可选但很有价值：**
- 卡住时，需要新视角
- 重构前，做基线检查
- 修完复杂 bug 之后

## 如何请求

**1. 获取 git SHA：**
```bash
BASE_SHA=$(git rev-parse HEAD~1)
HEAD_SHA=$(git rev-parse HEAD)
```

**2. 派发 code-reviewer 子代理：**

使用 Task 工具，并套用 `superpowers:code-reviewer` 类型，按 `code-reviewer.md` 模板填充。

**占位符：**
- `{WHAT_WAS_IMPLEMENTED}` - 刚刚实现了什么
- `{PLAN_OR_REQUIREMENTS}` - 它本来应该做到什么
- `{BASE_SHA}` - 起始提交
- `{HEAD_SHA}` - 结束提交
- `{DESCRIPTION}` - 简要摘要

**3. 根据反馈行动：**
- Critical 问题立刻修
- Important 问题在继续前修完
- Minor 问题可以记录到后面
- 如果 reviewer 错了，要给出技术性反驳

## 示例

```text
[刚完成 Task 2: Add verification function]

You: Let me request code review before proceeding.

BASE_SHA=$(git log --oneline | grep "Task 1" | head -1 | awk '{print $1}')
HEAD_SHA=$(git rev-parse HEAD)

[Dispatch superpowers:code-reviewer subagent]
  WHAT_WAS_IMPLEMENTED: Verification and repair functions for conversation index
  PLAN_OR_REQUIREMENTS: Task 2 from docs/superpowers/plans/deployment-plan.md
  BASE_SHA: a7981ec
  HEAD_SHA: 3df7661
  DESCRIPTION: Added verifyIndex() and repairIndex() with 4 issue types

[Subagent returns]:
  Strengths: Clean architecture, real tests
  Issues:
    Important: Missing progress indicators
    Minor: Magic number (100) for reporting interval
  Assessment: Ready to proceed

You: [Fix progress indicators]
[Continue to Task 3]
```

## 集成关系

**子代理驱动开发：**
- 每个任务后都做审查
- 在问题堆积前就把它们拦下来
- 修完再进入下一个任务

**执行计划：**
- 每个批次（例如 3 个任务）后做一次审查
- 获取反馈，修复，再继续

**临时开发：**
- 合并前做审查
- 卡住时做审查

## Red Flag

**绝不要：**
- 因为“这个很简单”就跳过审查
- 忽略 Critical 问题
- 带着未修复的 Important 问题继续往下走
- 对正确的技术反馈硬抬杠

**如果 reviewer 错了：**
- 用技术理由反驳
- 展示代码或测试作为证据
- 必要时请求对方澄清

模板见：`requesting-code-review/code-reviewer.md`
