---
name: writing-plans
description: 当你拥有多步骤任务的规格说明或需求时，在动手写代码之前使用此技能
---

# 编写计划

## 概述

编写详尽的实现计划，假设工程师对我们的代码库零了解，品味也有待商榷。记录他们需要知道的一切：每个任务要触碰哪些文件、代码、测试、可能需要查阅的文档，以及如何测试。以小粒度任务的形式提供完整计划。DRY。YAGNI。TDD。频繁提交。

假设他们是有经验的开发者，但对我们使用的工具集和问题领域几乎一无所知。假设他们不太了解优秀的测试设计。

**开始时宣布：** "我正在使用 writing-plans 技能来创建实现计划。"

**上下文：** 这应该在一个专用的工作树中运行（由 brainstorming 技能创建）。

**计划保存到：** `docs/superpowers/plans/YYYY-MM-DD-<feature-name>.md`
- （用户对计划位置的个人偏好优先于此默认值）

## 范围检查

如果规格说明涵盖多个独立子系统，它应该在头脑风暴阶段就被拆分为子项目规格。如果没有，建议将其拆分为多个独立计划——每个子系统一个。每个计划都应该能独立产出可工作的、可测试的软件。

## 文件结构

在定义任务之前，先梳理出需要创建或修改哪些文件，以及每个文件的职责。分解决策在此确定。

- 设计具有清晰边界和明确定义接口的单元。每个文件应该有一个明确的职责。
- 你对能一次性容纳在上下文中的代码推理能力最好，当文件聚焦时编辑也更可靠。优先选择小而专注的文件，而非大而臃肿的文件。
- 一起变更的文件应该放在一起。按职责拆分，而非按技术层拆分。
- 在现有代码库中，遵循已有的模式。如果代码库使用大文件，不要擅自重构——但如果你正在修改的文件已经变得臃肿，在计划中包含拆分是合理的。

此结构指导任务分解。每个任务应产出独立有意义的、自包含的变更。

## 小粒度任务

**每个步骤是一个动作（2-5 分钟）：**
- "编写失败的测试" - 一个步骤
- "运行它以确保它失败" - 一个步骤
- "实现使测试通过的最小代码" - 一个步骤
- "运行测试并确保它们通过" - 一个步骤
- "提交" - 一个步骤

## 计划文档头部

**每个计划必须以此头部开始：**

```markdown
# [Feature Name] Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** [One sentence describing what this builds]

**Architecture:** [2-3 sentences about approach]

**Tech Stack:** [Key technologies/libraries]

---
```

## 任务结构

````markdown
### Task N: [Component Name]

**Files:**
- Create: `exact/path/to/file.py`
- Modify: `exact/path/to/existing.py:123-145`
- Test: `tests/exact/path/to/test.py`

- [ ] **Step 1: Write the failing test**

```python
def test_specific_behavior():
    result = function(input)
    assert result == expected
```

- [ ] **Step 2: Run test to verify it fails**

Run: `pytest tests/path/test.py::test_name -v`
Expected: FAIL with "function not defined"

- [ ] **Step 3: Write minimal implementation**

```python
def function(input):
    return expected
```

- [ ] **Step 4: Run test to verify it passes**

Run: `pytest tests/path/test.py::test_name -v`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add tests/path/test.py src/path/file.py
git commit -m "feat: add specific feature"
```
````

## 不要占位符

每个步骤必须包含工程师所需的实际内容。以下都是**计划失败**——绝对不要写：
- "TBD"、"TODO"、"implement later"、"fill in details"
- "Add appropriate error handling" / "add validation" / "handle edge cases"
- "Write tests for the above"（没有实际测试代码）
- "Similar to Task N"（重复代码——工程师可能不按顺序阅读任务）
- 只描述做什么而不展示怎么做的步骤（代码步骤必须有代码块）
- 引用未在任何任务中定义的类型、函数或方法

## 记住
- 始终提供精确的文件路径
- 每个步骤都包含完整代码——如果某个步骤修改代码，展示该代码
- 精确的命令及预期输出
- DRY、YAGNI、TDD、频繁提交

## 自检

编写完整个计划后，以全新的眼光审视规格说明，并对照它检查计划。这是你自己运行的检查清单——不是派发子代理。

**1. 规格覆盖：** 浏览规格中的每个章节/需求。你能指出哪个任务实现了它吗？列出任何遗漏。

**2. 占位符扫描：** 在计划中搜索危险信号——即"不要占位符"部分中的任何模式。修复它们。

**3. 类型一致性：** 你在后续任务中使用的类型、方法签名和属性名称是否与先前任务中定义的一致？Task 3 中叫 `clearLayers()` 但 Task 7 中变成 `clearFullLayers()` 就是一个 bug。

如果发现问题，直接内联修复。不需要重新审查——修复后继续。如果发现规格需求没有对应任务，添加该任务。

## 执行交接

保存计划后，提供执行方式选择：

**"计划已完成并保存到 `docs/superpowers/plans/<filename>.md`。两种执行方式：**

**1. 子代理驱动（推荐）** - 我为每个任务派发一个新子代理，在任务之间进行审查，快速迭代

**2. 内联执行** - 在当前会话中使用 executing-plans 执行任务，批量执行并设置检查点

**选择哪种方式？"**

**如果选择了子代理驱动：**
- **必需技能：** 使用 superpowers:subagent-driven-development
- 每个任务一个新子代理 + 两阶段审查

**如果选择了内联执行：**
- **必需技能：** 使用 superpowers:executing-plans
- 批量执行并设置审查检查点
