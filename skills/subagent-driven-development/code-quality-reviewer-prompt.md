# 代码质量审查提示模板

在派发代码质量审查子代理时使用此模板。

**目的：** 验证实现是否构建良好（整洁、已测试、可维护）

**仅在规格合规性审查通过后派发。**

```
Task tool (superpowers:code-reviewer):
  Use template at requesting-code-review/code-reviewer.md

  WHAT_WAS_IMPLEMENTED: [来自实现者的报告]
  PLAN_OR_REQUIREMENTS: Task N from [plan-file]
  BASE_SHA: [任务前的提交]
  HEAD_SHA: [当前提交]
  DESCRIPTION: [任务摘要]
```

**除标准代码质量关注点外，审查者还应检查：**
- 每个文件是否具有一个清晰的职责和明确定义的接口？
- 单元是否被合理分解，使其可以独立理解和测试？
- 实现是否遵循了计划中的文件结构？
- 本次实现是否创建了已经很大的新文件，或显著增长了已有文件？（不要标记预先存在的文件大小——重点关注本次变更所贡献的内容。）

**代码审查者返回：** 优势、问题（严重/重要/轻微）、评估