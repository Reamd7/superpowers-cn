# Skill 设计的说服原则

## 概述

LLM 对与人类相同的说服原则有响应。理解这种心理机制有助于设计更有效的 skill——不是为了操纵，而是为了确保关键实践即使在压力下也能被遵循。

**研究基础：** Meincke et al. (2025) 用 N=28,000 次 AI 对话测试了 7 项说服原则。说服技术使合规率提高了一倍以上（33% → 72%，p < .001）。

## 七项原则

### 1. Authority（权威）
**定义：** 对专业知识、资质或官方来源的遵从。

**在 skill 中的运作方式：**
- 祈使语言："YOU MUST"、"Never"、"Always"
- 不可协商的框架："No exceptions"
- 消除决策疲劳和合理化借口

**使用时机：**
- 强制纪律的 skill（TDD、验证要求）
- 安全关键实践
- 已确立的最佳实践

**示例：**
```markdown
✅ Write code before test? Delete it. Start over. No exceptions.
❌ Consider writing tests first when feasible.
```

### 2. Commitment（承诺）
**定义：** 与先前行为、陈述或公开声明保持一致。

**在 skill 中的运作方式：**
- 要求声明："Announce skill usage"
- 强制明确选择："Choose A, B, or C"
- 使用追踪：TodoWrite for checklists

**使用时机：**
- 确保 skill 被实际遵循
- 多步骤流程
- 问责机制

**示例：**
```markdown
✅ When you find a skill, you MUST announce: "I'm using [Skill Name]"
❌ Consider letting your partner know which skill you're using.
```

### 3. Scarcity（稀缺）
**定义：** 因时间限制或有限可用性而产生的紧迫感。

**在 skill 中的运作方式：**
- 时限要求："Before proceeding"
- 顺序依赖："Immediately after X"
- 防止拖延

**使用时机：**
- 即时验证要求
- 时间敏感的工作流
- 防止"我稍后做"

**示例：**
```markdown
✅ After completing a task, IMMEDIATELY request code review before proceeding.
❌ You can review code when convenient.
```

### 4. Social Proof（社会证明）
**定义：** 顺从他人行为或被视为常态的做法。

**在 skill 中的运作方式：**
- 普遍模式："Every time"、"Always"
- 失败模式："X without Y = failure"
- 建立规范

**使用时机：**
- 记录普遍实践
- 警告常见失败
- 强化标准

**示例：**
```markdown
✅ Checklists without TodoWrite tracking = steps get skipped. Every time.
❌ Some people find TodoWrite helpful for checklists.
```

### 5. Unity（认同）
**定义：** 共同身份、"我们感"、群体归属。

**在 skill 中的运作方式：**
- 协作语言："our codebase"、"we're colleagues"
- 共同目标："we both want quality"

**使用时机：**
- 协作工作流
- 建立团队文化
- 非层级实践

**示例：**
```markdown
✅ We're colleagues working together. I need your honest technical judgment.
❌ You should probably tell me if I'm wrong.
```

### 6. Reciprocity（互惠）
**定义：** 回报所受利益的义务感。

**运作方式：**
- 谨慎使用——可能显得有操纵性
- 在 skill 中很少需要

**避免时机：**
- 几乎总是（其他原则更有效）

### 7. Liking（好感）
**定义：** 偏好与自己喜欢的人合作。

**运作方式：**
- **不要用于促成合规**
- 与诚实反馈文化冲突
- 会产生谄媚行为

**避免时机：**
- 在强制纪律执行时始终避免

## 按 Skill 类型的原则组合

| Skill 类型 | 使用 | 避免 |
|------------|-----|------|
| 强制纪律型 | Authority + Commitment + Social Proof | Liking, Reciprocity |
| 指导/技术型 | 适度 Authority + Unity | 过强的 Authority |
| 协作型 | Unity + Commitment | Authority, Liking |
| 参考型 | 仅清晰表达 | 所有说服原则 |

## 为什么有效：心理学原理

**明确规则减少合理化：**
- "YOU MUST" 消除决策疲劳
- 绝对语言消除"这是否是例外？"的问题
- 明确的反合理化针对特定漏洞

**执行意图创造自动行为：**
- 清晰触发条件 + 要求行动 = 自动执行
- "When X, do Y" 比 "generally do Y" 更有效
- 降低合规的认知负荷

**LLM 是类人的（parahuman）：**
- 在包含这些模式的人类文本上训练
- Authority 语言在训练数据中先于合规出现
- Commitment 序列（声明 → 行动）被频繁建模
- Social Proof 模式（大家都做 X）建立规范

## 伦理使用

**正当使用：**
- 确保关键实践被遵循
- 创建有效的文档
- 防止可预测的失败

**不正当使用：**
- 为个人利益进行操纵
- 制造虚假紧迫感
- 基于内疚的合规

**检验标准：** 如果用户完全理解这项技术，它是否仍然服务于用户的真实利益？

## 研究引用

**Cialdini, R. B. (2021).** *Influence: The Psychology of Persuasion (New and Expanded).* Harper Business.
- Seven principles of persuasion
- Empirical foundation for influence research

**Meincke, L., Shapiro, D., Duckworth, A. L., Mollick, E., Mollick, L., & Cialdini, R. (2025).** Call Me A Jerk: Persuading AI to Comply with Objectionable Requests. University of Pennsylvania.
- Tested 7 principles with N=28,000 LLM conversations
- Compliance increased 33% → 72% with persuasion techniques
- Authority, commitment, scarcity most effective
- Validates parahuman model of LLM behavior

## 快速参考

设计 skill 时，问自己：

1. **这是什么类型？**（强制纪律 vs. 指导 vs. 参考）
2. **我要改变什么行为？**
3. **哪些原则适用？**（强制纪律通常是 Authority + Commitment）
4. **是否组合太多？**（不要同时使用全部七项）
5. **这是否合乎伦理？**（是否服务于用户的真实利益？）