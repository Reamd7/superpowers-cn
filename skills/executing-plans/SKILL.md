---
name: executing-plans
description: 当你已经有一个书面的实现计划，并需要在单独会话中执行它且包含审查检查点时使用
---

# 执行计划

## 概述

加载计划，进行严格审查，执行所有任务，并在完成时汇报。

**开始时要明确说明：** "I'm using the executing-plans skill to implement this plan."

**注意：** 要告诉用户，Superpowers 在能使用子代理的平台上效果会好得多。如果运行环境支持子代理（例如 Claude Code 或 Codex），工作质量会明显更高。如果子代理可用，应使用 `superpowers:subagent-driven-development`，而不是这个 skill。

## 流程

### 第一步：加载并审查计划
1. 阅读计划文件
2. 进行严格审查，识别你对计划的任何问题或顾虑
3. 如果有顾虑：在开始前先向用户提出
4. 如果没有顾虑：创建 TodoWrite，然后继续

### 第二步：执行任务

对每个任务：
1. 标记为 `in_progress`
2. 严格按每个步骤执行（计划已经拆成小步）
3. 按计划要求运行验证
4. 标记为 `completed`

### 第三步：完成开发

在所有任务都完成并验证后：
- 宣告："I'm using the finishing-a-development-branch skill to complete this work."
- **必需的子 skill：** 使用 `superpowers:finishing-a-development-branch`
- 按该 skill 的要求验证测试、展示选项并执行用户选择

## 什么时候必须停下并求助

**出现以下情况要立刻停止执行：**
- 遇到阻塞项（缺依赖、测试失败、说明不清）
- 计划本身存在关键缺口，导致无法开始
- 你不理解某条指令
- 验证反复失败

**要请求澄清，不要靠猜。**

## 什么时候回到前面的步骤

**以下情况应回到“审查计划”阶段：**
- 用户根据你的反馈更新了计划
- 基本实现思路需要重想

**不要硬顶着 blocker 往前做。** 停下来，提问。

## 记住
- 先严格审查计划
- 严格按计划执行
- 不要跳过验证
- 计划要求引用 skill 时就照做
- 被阻塞就停，不要猜
- 没有用户明确同意，不要在 `main/master` 分支上直接开始实现

## 集成关系

**必需的工作流 skills：**
- **`superpowers:using-git-worktrees`** - 开始前必须先建立隔离工作区
- **`superpowers:writing-plans`** - 这个 skill 所执行的计划就是由它生成的
- **`superpowers:finishing-a-development-branch`** - 所有批次完成后用于收尾
