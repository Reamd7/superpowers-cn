---
name: writing-plans
description: 当你已经有一个多步骤任务的 spec 或需求，并且在动代码之前，需要使用这个 skill
---

# 编写实现计划

## 概述

编写完整的实现计划，并假设执行它的工程师对我们的代码库几乎毫无上下文，而且品味还不太可靠。你要把他们需要知道的一切都写清楚：每个任务改哪些文件、具体代码是什么、要查看哪些文档、如何测试。把整份计划拆成一口一口能吃下去的小任务。DRY。YAGNI。TDD。频繁提交。

假设执行者是一个有经验的开发者，但几乎不了解我们的工具链或业务域；也假设他们并不擅长设计好的测试。

**开始时要明确说明：** "I'm using the writing-plans skill to create the implementation plan."

**上下文要求：** 这个 skill 应该在专门的 worktree 中执行（由 `brainstorming` skill 创建）。

**计划保存位置：** `docs/superpowers/plans/YYYY-MM-DD-<feature-name>.md`
- 如果用户对计划存放位置有偏好，以用户偏好为准

## 范围检查

如果一个 spec 覆盖了多个彼此独立的子系统，它本应在 brainstorming 阶段就被拆成多个子项目 spec。如果没有拆，就建议把它们拆成多个计划，每个子系统一个计划。每份计划都应该独立产出可工作的、可测试的软件。

## 文件结构

在定义任务之前，先画清楚哪些文件会被创建或修改，以及每个文件负责什么。这一步会把拆分边界基本锁定下来。

- 设计边界清晰、接口明确的单元。每个文件只承担一个清晰职责。
- 你最擅长推理的是那些可以完整装进上下文的代码，文件越聚焦，修改越可靠。优先使用更小、更专注的文件，而不是巨大的多职责文件。
- 一起变化的文件应当放在一起。按职责拆分，而不是按技术层拆分。
- 在现有代码库里，要遵循已有模式。如果代码库本来就偏向大文件，不要擅自重构整个结构；但如果你要改的某个文件已经失控，在计划里安排拆分是合理的。

这个结构会直接影响后续任务拆解。每个任务都应形成一组自洽的改动，独立阅读也说得通。

## 小任务粒度

**每一步只做一个动作（2-5 分钟）：**
- “写一个失败的测试” - 一步
- “运行它，确认它失败” - 一步
- “写最小实现使测试通过” - 一步
- “运行测试，确认通过” - 一步
- “提交” - 一步

## 计划文档头部

**每一份计划都必须以这个头部开始：**

```markdown
# [Feature Name] Implementation Plan

> **For agentic workers:** REQUIRED: Use superpowers:subagent-driven-development (if subagents available) or superpowers:executing-plans to implement this plan. Steps use checkbox (`- [ ]`) syntax for tracking.

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

## 记住
- 文件路径必须精确
- 计划里的代码必须完整，不要只写“加校验”这类模糊描述
- 命令必须精确，并写清预期输出
- 用 `@` 语法引用相关 skills
- 保持 DRY、YAGNI、TDD 和频繁提交

## 计划审查循环

每完成一个计划分块之后：

1. 为当前分块派发 `plan-document-reviewer` 子代理（见 `plan-document-reviewer-prompt.md`）
   - 提供：分块内容、spec 文档路径
2. 如果发现 ❌ 问题：
   - 修复这个分块中的问题
   - 重新派发 reviewer 审查该分块
   - 重复直到 ✅ Approved
3. 如果 ✅ Approved：继续到下一个分块（如果已经是最后一个分块，就进入执行交接）

**分块边界：** 使用 `## Chunk N: <name>` 标题来划分块。每个分块应当不超过 1000 行，并且逻辑上自洽。

**审查循环指导：**
- 由写这份计划的同一个代理来修复问题，以保留上下文
- 如果循环超过 5 次，交给用户决定
- reviewer 的意见是咨询性的；如果你认为反馈不正确，应当解释原因

## 执行交接

保存计划后，给出下面这句话：

**"Plan complete and saved to `docs/superpowers/plans/<filename>.md`. Ready to execute?"**

**执行路径取决于当前 harness 的能力：**

**如果 harness 支持子代理（如 Claude Code 等）：**
- **必须**使用 `superpowers:subagent-driven-development`
- 不要把这当成一个可选项。subagent-driven 是标准路径
- 每个任务使用新的子代理 + 两阶段评审

**如果 harness 不支持子代理：**
- 在当前会话里使用 `superpowers:executing-plans` 执行计划
- 分批执行，并设置检查点供复核
