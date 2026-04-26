---
name: receiving-code-review
description: 在收到代码审查反馈时使用，在实施建议之前使用，尤其是当反馈看起来不清楚或在技术上有疑问时——需要技术严谨性和验证，而非表演性的同意或盲目实施
---

# 接收代码审查

## 概述

代码审查需要技术评估，而非情感表演。

**核心原则：** 先验证再实施。先提问再假设。技术正确性优先于社交舒适感。

## 回应模式

```
WHEN receiving code review feedback:

1. READ: Complete feedback without reacting
2. UNDERSTAND: Restate requirement in own words (or ask)
3. VERIFY: Check against codebase reality
4. EVALUATE: Technically sound for THIS codebase?
5. RESPOND: Technical acknowledgment or reasoned pushback
6. IMPLEMENT: One item at a time, test each
```

## 禁止的回应

**绝对不要：**
- "You're absolutely right!"（明确违反 `CLAUDE.md`）
- "Great point!" / "Excellent feedback!"（表演性的）
- "Let me implement that now"（在验证之前）

**应该这样做：**
- 重述技术要求
- 提出澄清问题
- 如果有误，用技术理由反驳
- 直接开始工作（行动胜于言语）

## 处理不明确的反馈

```
IF any item is unclear:
  STOP - do not implement anything yet
  ASK for clarification on unclear items

WHY: Items may be related. Partial understanding = wrong implementation.
```

**示例：**
```
your human partner: "Fix 1-6"
You understand 1,2,3,6. Unclear on 4,5.

❌ WRONG: Implement 1,2,3,6 now, ask about 4,5 later
✅ RIGHT: "I understand items 1,2,3,6. Need clarification on 4 and 5 before proceeding."
```

## 按来源分别处理

### 来自你的人工伙伴
- **可信的**——理解后即可实施
- **仍然要提问**，如果范围不明确
- **不要表演性同意**
- **直接行动**或作技术性确认

### 来自外部审查者
```
BEFORE implementing:
  1. Check: Technically correct for THIS codebase?
  2. Check: Breaks existing functionality?
  3. Check: Reason for current implementation?
  4. Check: Works on all platforms/versions?
  5. Check: Does reviewer understand full context?

IF suggestion seems wrong:
  Push back with technical reasoning

IF can't easily verify:
  Say so: "I can't verify this without [X]. Should I [investigate/ask/proceed]?"

IF conflicts with your human partner's prior decisions:
  Stop and discuss with your human partner first
```

**你的人工伙伴的规则：** "外部反馈——要持怀疑态度，但仔细核实"

## 对"专业化"功能的 YAGNI 检查

```
IF reviewer suggests "implementing properly":
  grep codebase for actual usage

  IF unused: "This endpoint isn't called. Remove it (YAGNI)?"
  IF used: Then implement properly
```

**你的人工伙伴的规则：** "你和审查者都向我汇报。如果我们不需要这个功能，就别加。"

## 实施顺序

```
FOR multi-item feedback:
  1. Clarify anything unclear FIRST
  2. Then implement in this order:
     - Blocking issues (breaks, security)
     - Simple fixes (typos, imports)
     - Complex fixes (refactoring, logic)
  3. Test each fix individually
  4. Verify no regressions
```

## 何时应该反驳

以下情况应该反驳：
- 建议会破坏现有功能
- 审查者缺乏完整上下文
- 违反 YAGNI 原则（未使用的功能）
- 对当前技术栈来说技术上不正确
- 存在遗留/兼容性原因
- 与你的人工伙伴的架构决策冲突

**如何反驳：**
- 使用技术理由，而非防御性态度
- 提出具体问题
- 引用能正常工作的测试/代码
- 如果涉及架构问题，让你的人工伙伴参与

**如果你对公开反驳感到不舒服，发信号：** "Strange things are afoot at the Circle K"

## 确认正确的反馈

当反馈确实正确时：
```
✅ "Fixed. [Brief description of what changed]"
✅ "Good catch - [specific issue]. Fixed in [location]."
✅ [Just fix it and show in the code]

❌ "You're absolutely right!"
❌ "Great point!"
❌ "Thanks for catching that!"
❌ "Thanks for [anything]"
❌ ANY gratitude expression
```

**为什么不感谢：** 行动胜于言辞。直接修复。代码本身就说明你听到了反馈。

**如果你发现自己正要写 "Thanks"：** 删掉它。改为说明你做了什么修复。

## 得体地纠正自己的反驳

如果你反驳了但自己是错的：
```
✅ "You were right - I checked [X] and it does [Y]. Implementing now."
✅ "Verified this and you're correct. My initial understanding was wrong because [reason]. Fixing."

❌ Long apology
❌ Defending why you pushed back
❌ Over-explaining
```

实事求是地陈述纠正，然后继续。

## 常见错误

| 错误 | 正确做法 |
|------|----------|
| 表演性同意 | 陈述要求或直接行动 |
| 盲目实施 | 先对照代码库验证 |
| 不测试就批量实施 | 逐个修复，逐一测试 |
| 假设审查者是对的 | 检查是否会破坏现有功能 |
| 回避反驳 | 技术正确性 > 舒适感 |
| 部分实施 | 先澄清所有条目 |
| 无法验证却继续推进 | 说明局限，请求方向指引 |

## 真实示例

**表演性同意（不好）：**
```
Reviewer: "Remove legacy code"
❌ "You're absolutely right! Let me remove that..."
```

**技术验证（好）：**
```
Reviewer: "Remove legacy code"
✅ "Checking... build target is 10.15+, this API needs 13+. Need legacy for backward compat. Current impl has wrong bundle ID - fix it or drop pre-13 support?"
```

**YAGNI（好）：**
```
Reviewer: "Implement proper metrics tracking with database, date filters, CSV export"
✅ "Grepped codebase - nothing calls this endpoint. Remove it (YAGNI)? Or is there usage I'm missing?"
```

**不明确的条目（好）：**
```
your human partner: "Fix items 1-6"
You understand 1,2,3,6. Unclear on 4,5.
✅ "Understand 1,2,3,6. Need clarification on 4 and 5 before implementing."
```

## GitHub 线程回复

在回复 GitHub 上的行内审查评论时，请在评论线程中回复（`gh api repos/{owner}/{repo}/pulls/{pr}/comments/{id}/replies`），而不是作为顶级 PR 评论。

## 底线

**外部反馈 = 需要评估的建议，而非必须服从的命令。**

验证。质疑。然后再实施。

不要表演性同意。始终保持技术严谨。
