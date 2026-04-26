---
name: verification-before-completion
description: 在声称工作完成、已修复或通过测试之前，在提交或创建 PR 之前使用——要求先运行验证命令并确认输出，然后才能做出任何成功声明；始终先有证据再有断言
---

# 完成前验证

## 概述

未经验证就声称工作已完成，是不诚实，不是高效。

**核心原则：** 先有证据，再做声明，始终如此。

**违反这条规则的字面意思就是违反这条规则的精神。**

## 铁律

```
NO COMPLETION CLAIMS WITHOUT FRESH VERIFICATION EVIDENCE
```

如果你没有在本条消息中运行验证命令，你就不能声称它通过了。

## 门控函数

```
BEFORE claiming any status or expressing satisfaction:

1. IDENTIFY: What command proves this claim?
2. RUN: Execute the FULL command (fresh, complete)
3. READ: Full output, check exit code, count failures
4. VERIFY: Does output confirm the claim?
   - If NO: State actual status with evidence
   - If YES: State claim WITH evidence
5. ONLY THEN: Make the claim

Skip any step = lying, not verifying
```

## 常见失败

| 声明 | 需要的证据 | 不充分的证据 |
|------|-----------|-------------|
| 测试通过 | 测试命令输出：0 个失败 | 之前的运行结果、"应该通过" |
| Linter 无错误 | Linter 输出：0 个错误 | 部分检查、推断 |
| 构建成功 | 构建命令：exit 0 | Linter 通过、日志看起来正常 |
| Bug 已修复 | 原始症状测试：通过 | 代码已修改、假设已修复 |
| 回归测试有效 | 红-绿循环已验证 | 测试只通过一次 |
| Agent 已完成 | VCS diff 显示变更 | Agent 报告"成功" |
| 需求已满足 | 逐项对照检查清单 | 测试通过 |

## Red Flag - 停下

- 使用"应该"、"大概"、"看起来"
- 在验证之前表达满意（"太好了！"、"完美！"、"完成！"等）
- 准备提交/推送/创建 PR 但未验证
- 相信 agent 的成功报告
- 依赖部分验证
- 想"就这一次"
- 疲惫了想让工作结束
- **任何在没有运行验证的情况下暗示成功的措辞**

## 自我辩解防范

| 借口 | 现实 |
|------|------|
| "应该能用了" | 运行验证命令 |
| "我很确定" | 信心 ≠ 证据 |
| "就这一次" | 没有例外 |
| "Linter 通过了" | Linter ≠ 编译器 |
| "Agent 说成功了" | 独立验证 |
| "我累了" | 疲惫 ≠ 借口 |
| "部分检查够了" | 部分检查证明不了什么 |
| "换了个说法所以规则不适用" | 精神高于字面 |

## 关键模式

**测试：**
```
✅ [Run test command] [See: 34/34 pass] "All tests pass"
❌ "Should pass now" / "Looks correct"
```

**回归测试（TDD 红-绿）：**
```
✅ Write → Run (pass) → Revert fix → Run (MUST FAIL) → Restore → Run (pass)
❌ "I've written a regression test" (without red-green verification)
```

**构建：**
```
✅ [Run build] [See: exit 0] "Build passes"
❌ "Linter passed" (linter doesn't check compilation)
```

**需求：**
```
✅ Re-read plan → Create checklist → Verify each → Report gaps or completion
❌ "Tests pass, phase complete"
```

**Agent 委托：**
```
✅ Agent reports success → Check VCS diff → Verify changes → Report actual state
❌ Trust agent report
```

## 为什么这很重要

来自 24 次失败记忆：
- 你的人工伙伴说"我不相信你"——信任已破裂
- 未定义的函数被提交——会导致崩溃
- 缺失的需求被提交——功能不完整
- 时间浪费在虚假完成 → 返工方向调整 → 重新工作
- 违反了："诚实是核心价值观。如果你撒谎，你将被替换。"

## 何时应用

**始终在以下操作之前：**
- 任何形式的成功/完成声明
- 任何表达满意的行为
- 任何关于工作状态的正面陈述
- 提交、创建 PR、完成任务
- 进入下一个任务
- 委托给 agent

**规则适用于：**
- 精确措辞
- 转述和同义词
- 成功的暗示
- 任何暗示完成/正确性的沟通

## 底线

**验证没有捷径。**

运行命令。阅读输出。然后再声称结果。

这是不可协商的。
