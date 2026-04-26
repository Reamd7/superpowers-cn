---
name: executing-plans
description: 当你有一个已写好的实施计划需要在单独的会话中执行，并包含审查检查点时使用
---

# 执行计划

## 概述

加载计划，批判性审查，执行所有任务，完成后报告。

**开始时宣布：** "I'm using the executing-plans skill to implement this plan."

**注意：** 告诉你的人工伙伴，Superpowers 在有子代理访问权限时效果更好。如果在支持子代理的平台（如 Claude Code 或 Codex）上运行，工作质量会显著提高。如果子代理可用，请使用 `superpowers:subagent-driven-development` 而非本 skill。

## 流程

### 步骤 1：加载并审查计划
1. 读取计划文件
2. 批判性审查 — 找出关于计划的任何疑问或顾虑
3. 如有顾虑：在开始前向你的人工伙伴提出
4. 如无顾虑：创建 TodoWrite 并继续

### 步骤 2：执行任务

对每个任务：
1. 标记为 in_progress
2. 严格遵循每个步骤（计划中包含细分的小步骤）
3. 按要求运行验证
4. 标记为已完成

### 步骤 3：完成开发

所有任务完成并验证后：
- 宣布："I'm using the finishing-a-development-branch skill to complete this work."
- **必需的子 skill：** 使用 `superpowers:finishing-a-development-branch`
- 按该 skill 验证测试、展示选项、执行选择

## 何时停下来寻求帮助

**在以下情况立即停止执行：**
- 遇到阻碍（缺少依赖、测试失败、指令不明确）
- 计划存在严重缺口，无法开始
- 你不理解某条指令
- 验证反复失败

**宁可请求澄清，也不要猜测。**

## 何时回顾之前的步骤

**在以下情况返回审查（步骤 1）：**
- 伙伴根据你的反馈更新了计划
- 基本方法需要重新思考

**不要强行突破阻碍** — 停下来问。

## 记住
- 先批判性审查计划
- 严格按计划步骤执行
- 不要跳过验证
- 计划中要求引用 skill 时要引用
- 遇到阻碍就停下，不要猜测
- 未经用户明确同意，绝不在 main/master 分支上开始实施

## 集成

**必需的工作流 skill：**
- **`superpowers:using-git-worktrees`** — 必需：开始前设置隔离工作区
- **`superpowers:writing-plans`** — 创建本 skill 所执行的计划
- **`superpowers:finishing-a-development-branch`** — 所有任务完成后完成开发
