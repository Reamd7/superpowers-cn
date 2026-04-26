---
name: writing-skills
description: 在创建新 skill、编辑现有 skill 或部署前验证 skill 是否有效时使用
---

# 编写 Skills

## 概述

**编写 skills，本质上就是把测试驱动开发应用到流程文档上。**

**个人 skills 存放在 agent 专属目录中（Claude Code 为 `~/.claude/skills`，Codex 为 `~/.agents/skills/`）**

你编写测试用例（带有 subagent 的压力场景），观察它们失败（基线行为），编写 skill（文档），观察测试通过（agent 遵守规则），然后重构（堵住漏洞）。

**核心原则：** 如果你没有观察过 agent 在没有 skill 时如何失败，你就不知道 skill 是否教了正确的东西。

**必读背景知识：** 在使用本 skill 之前，你必须理解 superpowers:test-driven-development。那个 skill 定义了基本的 RED-GREEN-REFACTOR 循环。本 skill 将 TDD 适配到文档领域。

**官方指南：** 关于 Anthropic 官方的 skill 编写最佳实践，请参阅 anthropic-best-practices.md。该文档提供了额外的模式和指南，是对本 skill 中以 TDD 为核心的方法的补充。

## 什么是 Skill？

**Skill** 是经过验证的技术、模式或工具的参考指南。Skills 帮助未来的 Claude 实例找到并应用有效的方法。

**Skills 是：** 可复用的技术、模式、工具、参考指南

**Skills 不是：** 关于你某次如何解决问题的叙述

## Skills 的 TDD 映射

| TDD 概念 | Skill 创建 |
|-------------|----------------|
| **测试用例** | 带 subagent 的压力场景 |
| **生产代码** | Skill 文档（SKILL.md） |
| **测试失败（RED）** | 没有 skill 时 agent 违反规则（基线） |
| **测试通过（GREEN）** | 有 skill 时 agent 遵守规则 |
| **重构** | 在保持合规的同时堵住漏洞 |
| **先写测试** | 在编写 skill 之前先运行基线场景 |
| **观察失败** | 记录 agent 使用的确切合理化借口 |
| **最小化代码** | 编写针对那些特定违规行为的 skill |
| **观察通过** | 验证 agent 现在遵守了 |
| **重构循环** | 发现新的合理化借口 → 堵住 → 重新验证 |

整个 skill 创建过程遵循 RED-GREEN-REFACTOR。

## 何时创建 Skill

**在以下情况创建：**
- 这项技术对你来说并非直觉上显而易见
- 你会在不同项目中再次引用它
- 模式具有广泛适用性（非特定项目）
- 其他人会从中受益

**不要为以下情况创建：**
- 一次性的解决方案
- 在别处已有充分文档记录的标准实践
- 项目特定的约定（放在 CLAUDE.md 中）
- 机械性约束（如果可以通过 regex/验证来强制执行，就自动化它——文档留给需要判断力的场景）

## Skill 类型

### 技术型
具有可遵循步骤的具体方法（condition-based-waiting、root-cause-tracing）

### 模式型
思考问题的方式（flatten-with-flags、test-invariants）

### 参考型
API 文档、语法指南、工具文档（office docs）

## 目录结构


```
skills/
  skill-name/
    SKILL.md              # 主要参考文件（必需）
    supporting-file.*     # 仅在需要时添加
```

**扁平命名空间** - 所有 skill 在一个可搜索的命名空间中

**在以下情况使用独立文件：**
1. **大量参考内容**（100+ 行）- API 文档、综合语法
2. **可复用工具** - 脚本、实用程序、模板

**保持内联的内容：**
- 原则和概念
- 代码模式（< 50 行）
- 其他所有内容

## SKILL.md 结构

**Frontmatter（YAML）：**
- 两个必填字段：`name` 和 `description`（参见 [agentskills.io/specification](https://agentskills.io/specification) 了解所有支持的字段）
- 总计最多 1024 个字符
- `name`：仅使用字母、数字和连字符（不要括号、特殊字符）
- `description`：第三人称，仅描述何时使用（不描述它做什么）
  - 以 "Use when..." 开头，聚焦于触发条件
  - 包含具体的症状、情境和上下文
  - **永远不要概括 skill 的流程或工作流**（原因见 CSO 章节）
  - 尽量控制在 500 个字符以内

```markdown
---
name: Skill-Name-With-Hyphens
description: Use when [specific triggering conditions and symptoms]
---

# Skill Name

## Overview
这是什么？用 1-2 句话阐述核心原则。

## When to Use
[如果决策不明显，使用小型内联流程图]

带症状和用例的项目符号列表
何时不使用

## Core Pattern（适用于技术型/模式型）
前后代码对比

## Quick Reference
表格或项目符号，方便快速浏览常见操作

## Implementation
简单模式使用内联代码
大量参考内容或可复用工具使用链接指向独立文件

## Common Mistakes
哪里会出错 + 修复方法

## Real-World Impact（可选）
具体结果
```


## Claude 搜索优化（CSO）

**对发现至关重要：** 未来的 Claude 需要找到你的 skill

### 1. 丰富的描述字段

**目的：** Claude 通过阅读 description 来决定为给定任务加载哪些 skill。让它回答："我现在应该读这个 skill 吗？"

**格式：** 以 "Use when..." 开头，聚焦于触发条件

**关键：Description = 何时使用，不是 Skill 做什么**

description 应该只描述触发条件。不要在 description 中概括 skill 的流程或工作流。

**为什么这很重要：** 测试表明，当 description 概括了 skill 的工作流时，Claude 可能会按照 description 行事而不是阅读完整的 skill 内容。一个 description 说"在任务之间进行代码审查"，导致 Claude 只做了一次审查，尽管 skill 的流程图清楚地显示应该有两次审查（规范合规性然后代码质量）。

当 description 改为只写"Use when executing implementation plans with independent tasks"（没有工作流概括）时，Claude 正确地阅读了流程图并遵循了两阶段审查流程。

**陷阱：** 概括工作流的 description 会创建一条捷径，Claude 会走这条路。Skill 正文变成了 Claude 跳过的文档。

```yaml
# ❌ 错误：概括了工作流 - Claude 可能按这个执行而不去读 skill
description: Use when executing plans - dispatches subagent per task with code review between tasks

# ❌ 错误：太多流程细节
description: Use for TDD - write test first, watch it fail, write minimal code, refactor

# ✅ 正确：只有触发条件，没有工作流概括
description: Use when executing implementation plans with independent tasks in the current session

# ✅ 正确：仅触发条件
description: Use when implementing any feature or bugfix, before writing implementation code
```

**内容：**
- 使用具体的触发条件、症状和情境来表明此 skill 适用
- 描述*问题*（竞态条件、不一致行为）而非*特定语言的症状*（setTimeout、sleep）
- 保持触发条件与技术无关，除非 skill 本身就是技术特定的
- 如果 skill 是技术特定的，在触发条件中明确说明
- 使用第三人称（会被注入到系统提示中）
- **永远不要概括 skill 的流程或工作流**

```yaml
# ❌ 错误：太抽象、模糊，没有包含何时使用
description: For async testing

# ❌ 错误：第一人称
description: I can help you with async tests when they're flaky

# ❌ 错误：提到了技术但 skill 并不特定于该技术
description: Use when tests use setTimeout/sleep and are flaky

# ✅ 正确：以 "Use when" 开头，描述问题，没有工作流
description: Use when tests have race conditions, timing dependencies, or pass/fail inconsistently

# ✅ 正确：技术特定 skill 带有明确的触发条件
description: Use when using React Router and handling authentication redirects
```

### 2. 关键词覆盖

使用 Claude 会搜索的词语：
- 错误信息："Hook timed out"、"ENOTEMPTY"、"race condition"
- 症状："flaky"、"hanging"、"zombie"、"pollution"
- 同义词："timeout/hang/freeze"、"cleanup/teardown/afterEach"
- 工具：实际命令、库名、文件类型

### 3. 描述性命名

**使用主动语态，动词开头：**
- ✅ `creating-skills` 而非 `skill-creation`
- ✅ `condition-based-waiting` 而非 `async-test-helpers`

### 4. Token 效率（关键）

**问题：** getting-started 和频繁引用的 skills 会被加载到每个对话中。每个 token 都很重要。

**目标字数：**
- getting-started 工作流：每个 < 150 词
- 频繁加载的 skills：总计 < 200 词
- 其他 skills：< 500 词（仍然要简洁）

**技巧：**

**将细节移到工具帮助中：**
```bash
# ❌ 错误：在 SKILL.md 中记录所有标志
search-conversations supports --text, --both, --after DATE, --before DATE, --limit N

# ✅ 正确：引用 --help
search-conversations supports multiple modes and filters. Run --help for details.
```

**使用交叉引用：**
```markdown
# ❌ 错误：重复工作流细节
When searching, dispatch subagent with template...
[20 lines of repeated instructions]

# ✅ 正确：引用其他 skill
Always use subagents (50-100x context savings). REQUIRED: Use [other-skill-name] for workflow.
```

**压缩示例：**
```markdown
# ❌ 错误：冗长的示例（42 词）
your human partner: "How did we handle authentication errors in React Router before?"
You: I'll search past conversations for React Router authentication patterns.
[Dispatch subagent with search query: "React Router authentication error handling 401"]

# ✅ 正确：精简的示例（20 词）
Partner: "How did we handle auth errors in React Router?"
You: Searching...
[Dispatch subagent → synthesis]
```

**消除冗余：**
- 不要重复交叉引用的 skill 中已有的内容
- 不要解释从命令中就能看出的东西
- 不要为同一模式提供多个示例

**验证：**
```bash
wc -w skills/path/SKILL.md
# getting-started 工作流：目标每个 < 150 词
# 其他频繁加载的：目标总计 < 200 词
```

**以你做的事或核心洞见命名：**
- ✅ `condition-based-waiting` > `async-test-helpers`
- ✅ `using-skills` 而非 `skill-usage`
- ✅ `flatten-with-flags` > `data-structure-refactoring`
- ✅ `root-cause-tracing` > `debugging-techniques`

**动名词（-ing）适合流程：**
- `creating-skills`、`testing-skills`、`debugging-with-logs`
- 主动式，描述你正在做的动作

### 4. 交叉引用其他 Skills

**在编写引用其他 skill 的文档时：**

仅使用 skill 名称，并带有明确的要求标记：
- ✅ 好的做法：`**REQUIRED SUB-SKILL:** Use superpowers:test-driven-development`
- ✅ 好的做法：`**REQUIRED BACKGROUND:** You MUST understand superpowers:systematic-debugging`
- ❌ 不好的做法：`See skills/testing/test-driven-development`（不清楚是否必需）
- ❌ 不好的做法：`@skills/testing/test-driven-development/SKILL.md`（强制加载，浪费 context）

**为什么不用 @ 链接：** `@` 语法会立即强制加载文件，在你需要之前就消耗 200k+ 的 context。

## 流程图使用

```dot
digraph when_flowchart {
    "需要展示信息？" [shape=diamond];
    "可能会做错的决策？" [shape=diamond];
    "使用 markdown" [shape=box];
    "小型内联流程图" [shape=box];

    "需要展示信息？" -> "可能会做错的决策？" [label="yes"];
    "可能会做错的决策？" -> "小型内联流程图" [label="yes"];
    "可能会做错的决策？" -> "使用 markdown" [label="no"];
}
```

**仅在以下情况使用流程图：**
- 不明显的决策点
- 你可能会过早停止的流程循环
- "何时使用 A vs B" 的决策

**永远不要在以下情况使用流程图：**
- 参考材料 → 表格、列表
- 代码示例 → Markdown 代码块
- 线性指令 → 编号列表
- 没有语义含义的标签（step1、helper2）

关于 graphviz 样式规则，请参阅 @graphviz-conventions.dot。

**为你的 human partner 可视化：** 使用此目录中的 `render-graphs.js` 将 skill 的流程图渲染为 SVG：
```bash
./render-graphs.js ../some-skill           # 每个图单独渲染
./render-graphs.js ../some-skill --combine # 所有图合并到一个 SVG
```

## 代码示例

**一个优秀的示例胜过多个平庸的示例**

选择最相关的语言：
- 测试技术 → TypeScript/JavaScript
- 系统调试 → Shell/Python
- 数据处理 → Python

**好的示例：**
- 完整且可运行
- 注释充分，解释原因
- 来自真实场景
- 清晰地展示模式
- 可直接改编（不是通用模板）

**不要：**
- 用 5+ 种语言实现
- 创建填空式模板
- 编写人为构造的示例

你擅长移植代码——一个出色的示例就够了。

## 文件组织

### 自包含 Skill
```
defense-in-depth/
  SKILL.md    # 所有内容内联
```
适用情况：所有内容都能放下，不需要大量参考

### 带可复用工具的 Skill
```
condition-based-waiting/
  SKILL.md    # 概述 + 模式
  example.ts  # 可改编的可用辅助工具
```
适用情况：工具是可复用的代码，而不仅仅是叙述

### 带大量参考的 Skill
```
pptx/
  SKILL.md       # 概述 + 工作流
  pptxgenjs.md   # 600 行 API 参考
  ooxml.md       # 500 行 XML 结构
  scripts/       # 可执行工具
```
适用情况：参考材料太大无法内联

## 铁律（与 TDD 相同）

```
NO SKILL WITHOUT A FAILING TEST FIRST
```

这适用于新 skill 和对现有 skill 的编辑。

先写 skill 再测试？删掉它。从头开始。
编辑 skill 却不测试？同样的违规。

**没有例外：**
- 不适用于"简单的添加"
- 不适用于"只是加一个章节"
- 不适用于"文档更新"
- 不要保留未测试的更改作为"参考"
- 不要在运行测试时"顺便调整"
- 删除就是删除

**必读背景知识：** superpowers:test-driven-development skill 解释了为什么这很重要。相同的原则适用于文档。

## 测试所有类型的 Skills

不同类型的 skill 需要不同的测试方法：

### 纪律执行型 Skills（规则/要求）

**示例：** TDD、完成前验证、编码前设计

**测试方式：**
- 学术问题：它们理解规则吗？
- 压力场景：在压力下它们会遵守吗？
- 多重压力组合：时间 + 沉没成本 + 疲劳
- 识别合理化借口并添加明确的反驳

**成功标准：** Agent 在最大压力下仍然遵循规则

### 技术型 Skills（操作指南）

**示例：** condition-based-waiting、root-cause-tracing、defensive-programming

**测试方式：**
- 应用场景：它们能正确应用技术吗？
- 变体场景：它们能处理边界情况吗？
- 信息缺失测试：指令是否有空白？

**成功标准：** Agent 成功将技术应用到新场景

### 模式型 Skills（心智模型）

**示例：** reducing-complexity、信息隐藏概念

**测试方式：**
- 识别场景：它们能识别模式何时适用吗？
- 应用场景：它们能使用该心智模型吗？
- 反例：它们知道何时不该应用吗？

**成功标准：** Agent 正确识别何时/如何应用模式

### 参考型 Skills（文档/API）

**示例：** API 文档、命令参考、库指南

**测试方式：**
- 检索场景：它们能找到正确的信息吗？
- 应用场景：它们能正确使用找到的信息吗？
- 缺口测试：常见用例是否被覆盖？

**成功标准：** Agent 找到并正确应用参考信息

## 跳过测试的常见合理化借口

| 借口 | 现实 |
|--------|---------|
| "Skill 显然很清晰" | 对你清晰 ≠ 对其他 agent 清晰。测试它。 |
| "这只是参考资料" | 参考资料可能有空白、不清楚的部分。测试检索。 |
| "测试是小题大做" | 未测试的 skill 总是有问题。15 分钟测试节省数小时。 |
| "有问题出现时再测" | 问题 = agent 无法使用 skill。在部署之前测试。 |
| "测试太繁琐" | 测试比在生产环境中调试有问题的 skill 更不繁琐。 |
| "我有信心它没问题" | 过度自信保证会有问题。无论如何都要测试。 |
| "学术审查就够了" | 阅读 ≠ 使用。测试应用场景。 |
| "没时间测试" | 部署未测试的 skill 之后修复它浪费的时间更多。 |

**以上所有都意味着：在部署之前测试。没有例外。**

## 让 Skills 抵御合理化借口

执行纪律的 skills（如 TDD）需要抵御合理化借口。Agent 很聪明，在压力下会找到漏洞。

**心理学注释：** 理解说服技巧为什么有效有助于你系统性地应用它们。关于权威、承诺、稀缺、社会认同和团结原则的研究基础，请参阅 persuasion-principles.md（Cialdini, 2021; Meincke et al., 2025）。

### 明确堵住每个漏洞

不要只是陈述规则——禁止特定的变通方法：

<Bad>
```markdown
在测试之前写了代码？删掉它。
```
</Bad>

<Good>
```markdown
在测试之前写了代码？删掉它。从头开始。

**没有例外：**
- 不要保留它作为"参考"
- 不要在写测试时"顺便调整"它
- 不要看它
- 删除就是删除
```
</Good>

### 应对"精神 vs 字面"的论点

尽早添加基础原则：

```markdown
**违反规则的字面意思就是违反规则的精神。**
```

这可以切断一整类"我在遵循精神"的合理化借口。

### 建立合理化借口表

从基线测试中收集合理化借口（参见下面的测试部分）。Agent 做出的每个借口都放入表中：

```markdown
| 借口 | 现实 |
|--------|---------|
| "太简单不需要测试" | 简单的代码也会出错。测试只需 30 秒。 |
| "我之后再测" | 之后立即通过的测试证明不了什么。 |
| "之后的测试能达到同样的目的" | 之后测试 = "这做了什么？" 先测试 = "这应该做什么？" |
```

### 建立 Red Flag 列表

让 agent 在进行合理化时容易自我检查：

```markdown
## Red Flag - 停下来，从头开始

- 先写代码再写测试
- "我已经手动测试过了"
- "之后的测试能达到同样的目的"
- "关键在于精神而不是仪式"
- "这次不同因为..."

**以上所有都意味着：删除代码。用 TDD 从头开始。**
```

### 为违规症状更新 CSO

在 description 中添加：你即将违反规则时的症状：

```yaml
description: use when implementing any feature or bugfix, before writing implementation code
```

## Skills 的 RED-GREEN-REFACTOR

遵循 TDD 循环：

### RED：编写失败的测试（基线）

在没有 skill 的情况下用 subagent 运行压力场景。记录确切行为：
- 它们做了什么选择？
- 它们使用了什么合理化借口（逐字记录）？
- 哪些压力触发了违规？

这就是"观察测试失败"——你必须先看到 agent 在没有 skill 时的自然行为，然后再编写 skill。

### GREEN：编写最小化的 Skill

编写针对那些特定合理化借口的 skill。不要为假设的情况添加额外内容。

用相同的场景运行，但这次有 skill。Agent 现在应该遵守规则。

### REFACTOR：堵住漏洞

Agent 找到了新的合理化借口？添加明确的反驳。重新测试直到无懈可击。

**测试方法论：** 完整的测试方法论请参阅 @testing-skills-with-subagents.md：
- 如何编写压力场景
- 压力类型（时间、沉没成本、权威、疲劳）
- 系统性地堵住漏洞
- 元测试技术

## 反模式

### ❌ 叙述式示例
"在 2025-10-03 的会话中，我们发现空的 projectDir 导致了……"
**为什么不好：** 太具体，不可复用

### ❌ 多语言稀释
example-js.js, example-py.py, example-go.go
**为什么不好：** 质量平庸，维护负担

### ❌ 流程图中的代码
```dot
step1 [label="import fs"];
step2 [label="read file"];
```
**为什么不好：** 无法复制粘贴，难以阅读

### ❌ 通用标签
helper1, helper2, step3, pattern4
**为什么不好：** 标签应该有语义含义

## 停下来：在进入下一个 Skill 之前

**编写完任何 skill 之后，你必须停下来并完成部署流程。**

**不要：**
- 批量创建多个 skill 而不逐个测试
- 在当前 skill 验证之前就进入下一个
- 因为"批量处理更高效"而跳过测试

**下面的部署检查清单对每个 skill 都是强制性的。**

部署未测试的 skill = 部署未测试的代码。这是对质量标准的违反。

## Skill 创建检查清单（TDD 适配版）

**重要：使用 TodoWrite 为以下每个检查项创建待办事项。**

**RED 阶段 - 编写失败的测试：**
- [ ] 创建压力场景（纪律型 skill 需要 3+ 个组合压力）
- [ ] 在没有 skill 的情况下运行场景 - 逐字记录基线行为
- [ ] 识别合理化借口/失败中的模式

**GREEN 阶段 - 编写最小化的 Skill：**
- [ ] 名称仅使用字母、数字、连字符（不要括号/特殊字符）
- [ ] YAML frontmatter 包含必需的 `name` 和 `description` 字段（最多 1024 字符；参见 [spec](https://agentskills.io/specification)）
- [ ] Description 以 "Use when..." 开头并包含具体的触发条件/症状
- [ ] Description 使用第三人称
- [ ] 全文包含搜索关键词（错误、症状、工具）
- [ ] 清晰的概述，包含核心原则
- [ ] 针对 RED 阶段识别的具体基线失败进行处理
- [ ] 代码内联或链接到独立文件
- [ ] 一个优秀的示例（不要多语言版本）
- [ ] 有 skill 的情况下运行场景 - 验证 agent 现在遵守规则

**REFACTOR 阶段 - 堵住漏洞：**
- [ ] 识别测试中出现的新的合理化借口
- [ ] 添加明确的反驳（如果是纪律型 skill）
- [ ] 从所有测试迭代中建立合理化借口表
- [ ] 建立 red flags 列表
- [ ] 重新测试直到无懈可击

**质量检查：**
- [ ] 仅在决策不明显时使用小型流程图
- [ ] 快速参考表
- [ ] 常见错误章节
- [ ] 没有叙述式讲故事
- [ ] 支持文件仅用于工具或大量参考

**部署：**
- [ ] 将 skill 提交到 git 并推送到你的 fork（如果已配置）
- [ ] 考虑通过 PR 贡献回去（如果具有广泛适用性）

## 发现工作流

未来的 Claude 如何找到你的 skill：

1. **遇到问题**（"测试不稳定"）
3. **找到 SKILL**（description 匹配）
4. **浏览概述**（这相关吗？）
5. **阅读模式**（快速参考表）
6. **加载示例**（仅在实现时）

**为此流程优化** - 尽早且频繁地放置可搜索的术语。

## 总结

**创建 skills 就是流程文档的 TDD。**

同样的铁律：没有失败的测试就没有 skill。
同样的循环：RED（基线）→ GREEN（编写 skill）→ REFACTOR（堵住漏洞）。
同样的好处：更高质量、更少意外、无懈可击的结果。

如果你在编码时遵循 TDD，那在编写 skill 时也请遵循。这是同样的纪律，只是应用到了文档上。
