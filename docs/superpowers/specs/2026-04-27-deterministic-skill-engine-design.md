# Superpowers 确定性 Skill 引擎：统一模型

## 0. 文档元信息

- 日期：2026-04-27
- 来源：Claude Opus 分析（Petri Net 模型）+ GLM-5 分析（Guarded Pipeline 模型）合并
- 目标：一份精确的、可审计的数学/编程模型，将 Superpowers skills 中的关键判断节点从 LLM 自律转化为程序强制执行
- 未来实现目标：OpenCode 插件

---

## 1. 问题陈述

### 1.1 现状

当前 Superpowers 的架构：

```
skill 文档（自然语言）→ 注入 LLM 上下文 → 希望 LLM 遵守
```

skill 描述了"应该做什么"，但执行完全依赖 LLM 的自律性。LLM 在以下条件下系统性退化：
- 上下文膨胀（token 接近上限）
- 沉没成本压力（"已经写了 X 小时的代码"）
- 时间压力（用户催促"快点"）
- 自我合理化（"这太简单了不需要测试"）

`writing-skills` 承认了这个问题——它用 subagent 压力测试来对抗，但这仍然是用一个 LLM 去检查另一个 LLM。

### 1.2 目标

```
skill 定义（形式化）→ 运行时引擎（确定性程序）→ LLM 只在引擎允许时行动
```

将 skills 中约 80% 的关键决策节点从"LLM 自律"变为"程序强制"。剩余 20% 涉及语义判断的节点，仍由 LLM 或人类负责。

### 1.3 不做什么

- 不替代 skill 文档（skill 仍然给 LLM 提供理解上下文）
- 不试图程序化所有判断（"设计是否好"、"代码是否可读"仍是 LLM 的工作）
- 不引入外部依赖（与 Superpowers 零依赖哲学一致）

---

## 2. 核心抽象

### 2.1 基本概念

**Artifact**：skill 流程中产生和消费的工件。

```
ArtifactKind = Spec | Plan | TestFile | ProdCode | Review | Claim | CommitRecord
```

**Context**：管道执行过程中累积的所有 artifact。

```
Context = Map<StageId, Artifact>
```

上下文满足**单调性**：`C₀ ⊆ C₁ ⊆ ... ⊆ Cₖ`（只增不减，后续阶段可依赖前序阶段的 artifact）。

### 2.2 守卫谓词（Guard Predicate）

```
φ : (Artifact, Context) → Pass(evidence) | Fail(error, fix_hint)
```

分为两类：

- **确定性守卫 Φ_D**：存在图灵机 M_φ 能在有限步内计算 φ，且 M_φ 不调用任何 LLM。
- **判断性守卫 Φ_J**：需要 LLM 或人类介入。

**执行顺序规则**：确定性守卫先于判断性守卫执行。如果确定性守卫失败，不浪费 LLM/人类的判断资源。

### 2.3 确定性覆盖率

```
DC(Pipeline) = |⋃ Φ_D⁽ⁱ⁾| / |⋃ Φᵢ|
```

一个可计算的质量指标。当前 Superpowers 估计 DC ≈ 27%，目标 DC ≈ 80%。

### 2.4 结构化提升定理

> 如果将 artifact 类型 A（自由文本）替换为结构化类型 A'⊂A（A' 有明确 schema），则可以将部分 Φ_J 转化为 Φ_D。

这是提升 DC 的主要杠杆。当前 spec/plan 都是自由 markdown，这是判断性守卫比例高的根本原因。将 artifact schema 化是所有其他工作的前置依赖。

---

## 3. 执行模型：带守卫的 Petri Net

### 3.1 为什么选 Petri Net 而非线性 Pipeline

Superpowers 的 skill 流程不是线性的：
- `dispatching-parallel-agents`：并发执行多个子代理
- `brainstorming`：用户拒绝 → 回到设计（循环）
- `systematic-debugging`：阶段 1-4 有条件分支和回退
- `subagent-driven-development`：资源约束（一次只允许一个 in_progress 子代理）

Pipeline 无法自然表达并发、循环、资源约束。Petri Net 可以。

### 3.2 形式定义

```
SkillNet = (P, T, F, W, M₀, G, E)

P = {p₁, p₂, ...}           -- 库所（places）= 状态/条件
T = {t₁, t₂, ...}           -- 变迁（transitions）= 动作
F ⊆ (P × T) ∪ (T × P)      -- 弧（flow relation）
W : F → ℕ                    -- 弧权重（默认 1）
M₀ : P → ℕ                  -- 初始标记（initial marking）
G : T → Guard[]              -- 守卫函数集：变迁的前置断言
E : T → Effect               -- 效果函数：变迁触发后的副作用
```

**G 和 E 不是自然语言，它们是可执行的程序。**

### 3.3 变迁触发规则

变迁 t 可触发（enabled），当且仅当：
1. **结构可行**：对 t 的每个输入库所 p，当前标记 M(p) ≥ W(p,t)
2. **守卫通过**：G(t) 中的所有确定性守卫返回 Pass，且所有判断性守卫返回 Pass

触发 t 后：
1. 消耗输入 token：对每个输入 p，M(p) -= W(p,t)
2. 产生输出 token：对每个输出 p'，M(p') += W(t,p')
3. 执行副作用：E(t)

### 3.4 实例：TDD skill 的 Petri Net

```
库所 P:
  p_no_test        -- 无测试存在
  p_test_written   -- 测试已写入文件系统
  p_test_red       -- 测试运行失败（RED 确认）
  p_code_written   -- 生产代码已写入
  p_test_green     -- 测试运行通过（GREEN 确认）
  p_refactored     -- 重构完成，测试仍通过
  p_committed      -- 已提交

变迁 T（及其守卫 G）:

  t_write_test:
    输入: {p_no_test: 1}
    输出: {p_test_written: 1}
    G_D: [file_diff_contains_test_code, no_prod_code_in_diff]
    G_J: []
    E: snapshot_file_state()

  t_verify_red:
    输入: {p_test_written: 1}
    输出: {p_test_red: 1}
    G_D: [test_exit_code_nonzero, failure_is_assertion_not_syntax]
    G_J: []
    E: record_failure_output()

  t_write_code:
    输入: {p_test_red: 1}
    输出: {p_code_written: 1}
    G_D: [file_diff_contains_prod_code]
    G_J: [diff_is_minimal]   -- "最少代码"需要判断
    E: snapshot_file_state()

  t_verify_green:
    输入: {p_code_written: 1}
    输出: {p_test_green: 1}
    G_D: [test_exit_code_zero, all_other_tests_pass]
    G_J: []
    E: record_pass_output()

  t_refactor:
    输入: {p_test_green: 1}
    输出: {p_refactored: 1}
    G_D: [test_exit_code_zero, no_new_behavior_in_diff]
    G_J: []
    E: []

  t_commit:
    输入: {p_refactored: 1}
    输出: {p_committed: 1}
    G_D: [git_status_clean_after_commit]
    G_J: []
    E: git_commit()

初始标记: M₀ = {p_no_test: 1, 其他: 0}
终止标记: p_committed 有 token
```

DC = 11/12 = **91.7%**（唯一的 Φ_J 是 `diff_is_minimal`）

### 3.5 实例：Brainstorming skill 的 Petri Net（含循环）

```
库所 P:
  p_start
  p_context_explored
  p_questions_done
  p_proposals_shown
  p_design_shown
  p_design_approved       -- 用户批准
  p_spec_written
  p_spec_self_checked
  p_spec_user_approved    -- 终止状态

变迁:

  t_explore_context:
    输入: {p_start: 1}
    输出: {p_context_explored: 1}
    G_D: [project_files_read]  -- 检查 Read 工具是否被调用过
    G_J: []

  t_clarify:
    输入: {p_context_explored: 1}
    输出: {p_questions_done: 1}
    G_D: []
    G_J: [questions_sufficient]  -- 需要判断

  t_propose:
    输入: {p_questions_done: 1}
    输出: {p_proposals_shown: 1}
    G_D: [proposal_count_gte_2]  -- 至少 2 个方案
    G_J: []

  t_show_design:
    输入: {p_proposals_shown: 1}
    输出: {p_design_shown: 1}
    G_D: []
    G_J: [design_covers_required_aspects]

  t_user_approve_design:
    输入: {p_design_shown: 1}
    输出: {p_design_approved: 1}
    G_D: [user_said_yes]  -- human gate（确定性：等待精确输入）
    G_J: []

  t_user_reject_design:             -- 循环弧
    输入: {p_design_shown: 1}
    输出: {p_proposals_shown: 1}    -- 回到方案阶段
    G_D: [user_said_no]
    G_J: []

  t_write_spec:
    输入: {p_design_approved: 1}
    输出: {p_spec_written: 1}
    G_D: [spec_file_exists, spec_committed_to_git]
    G_J: []

  t_self_check:
    输入: {p_spec_written: 1}
    输出: {p_spec_self_checked: 1}
    G_D: [no_placeholders, no_empty_headings, naming_consistent]
    G_J: []   -- 将原本依赖 LLM 的自检全部程序化

  t_user_approve_spec:
    输入: {p_spec_self_checked: 1}
    输出: {p_spec_user_approved: 1}
    G_D: [user_said_yes]
    G_J: []

  t_user_revise_spec:               -- 循环弧
    输入: {p_spec_self_checked: 1}
    输出: {p_spec_written: 1}       -- 回到编写阶段
    G_D: [user_requested_changes]
    G_J: []
```

Petri Net 自然表达了两个循环：设计不通过 → 回到方案，规格不通过 → 回到编写。Pipeline 需要额外的控制流语法才能做到。

---

## 4. 守卫函数库

### 4.1 守卫类型分类

```typescript
type Mechanism =
  | { kind: 'exec';      cmd: string; expect_exit: number }
  | { kind: 'grep';      pattern: RegExp; target: 'file' | 'artifact'; negate: boolean }
  | { kind: 'fs';        check: 'exists' | 'not_exists'; path: string }
  | { kind: 'schema';    validate: (a: unknown) => boolean }
  | { kind: 'cross_ref'; source_field: string; target_field: string; relation: 'covers' | 'subset_of' }
  | { kind: 'git';       subcommand: string; expect: string | RegExp }
  | { kind: 'count';     field: string; min?: number; max?: number }
  | { kind: 'human';     prompt: string }
  | { kind: 'llm';       prompt: string }
```

前 7 种 → Φ_D，后 2 种 → Φ_J。

### 4.2 每阶段守卫精确规格

#### Stage: Brainstorming → Spec

结构化 Spec artifact：

```typescript
type Spec = {
  problem: string
  constraints: string[]
  requirements: { id: string; description: string; acceptance_criteria: string[] }[]
  scope: 'single' | 'multi'
  api_design?: string
  data_model?: string
  test_strategy?: string
}
```

| ID | 守卫 | 类型 | 机制 | 精确定义 |
|----|------|------|------|---------|
| G-BS-1 | `has_problem` | Φ_D | schema | `spec.problem.length > 0` |
| G-BS-2 | `has_constraints` | Φ_D | schema | `spec.constraints.length > 0` |
| G-BS-3 | `has_requirements` | Φ_D | schema | `spec.requirements.length > 0` |
| G-BS-4 | `each_req_has_criteria` | Φ_D | schema | `spec.requirements.every(r => r.acceptance_criteria.length > 0)` |
| G-BS-5 | `no_placeholders` | Φ_D | grep | `!/(TBD|TODO|FIXME|PLACEHOLDER|待定|待补充)\b/.test(content)` |
| G-BS-6 | `no_empty_headings` | Φ_D | grep | markdown heading 后有实质内容 |
| G-BS-7 | `naming_consistent` | Φ_D | cross_ref | 提取所有标识符名称，检查前后一致 |
| G-BS-8 | `scope_single` | Φ_D | schema | `spec.scope === 'single'` |
| G-BS-9 | `user_approved` | Φ_J | human | 等待用户确认 |

DC = 8/9 = **88.9%**

#### Stage: Writing Plans

结构化 Plan artifact：

```typescript
type Plan = {
  spec_id: string
  file_map: { path: string; responsibility: string; action: 'create' | 'modify' | 'delete' }[]
  tasks: {
    id: string
    spec_refs: string[]     // ["REQ-1", "REQ-3"]
    steps: {
      action: string
      command?: string
      expected_output?: string
      artifact_path?: string
    }[]
  }[]
}
```

| ID | 守卫 | 类型 | 机制 | 精确定义 |
|----|------|------|------|---------|
| G-PL-1 | `every_req_covered` | Φ_D | cross_ref | `spec.requirements.every(r => plan.tasks.some(t => t.spec_refs.includes(r.id)))` |
| G-PL-2 | `no_orphan_refs` | Φ_D | cross_ref | `plan.tasks.flatMap(t=>t.spec_refs).every(ref => spec.requirements.some(r=>r.id===ref))` |
| G-PL-3 | `no_placeholders` | Φ_D | grep | 同 G-BS-5 |
| G-PL-4 | `every_task_has_steps` | Φ_D | count | `plan.tasks.every(t => t.steps.length >= 3)` |
| G-PL-5 | `has_tdd_structure` | Φ_D | grep | 每个 task 的 steps 中包含 "test"/"测试" 和 "run"/"运行" 关键词 |
| G-PL-6 | `file_map_complete` | Φ_D | cross_ref | steps 产出的文件路径 ⊆ file_map.paths |
| G-PL-7 | `naming_consistent` | Φ_D | cross_ref | 函数名/类型名在 steps 间一致 |
| G-PL-8 | `user_chosen_mode` | Φ_J | human | 选择 subagent-driven 或 inline |

DC = 7/8 = **87.5%**

#### Stage: TDD RED（写测试 + 验证失败）

| ID | 守卫 | 类型 | 机制 | 精确定义 |
|----|------|------|------|---------|
| G-RED-1 | `test_file_exists` | Φ_D | fs | `fs.existsSync(testPath)` |
| G-RED-2 | `no_prod_code_in_diff` | Φ_D | git | `git diff --cached --name-only` 只含测试文件 |
| G-RED-3 | `test_compiles` | Φ_D | exec | 编译器/解释器对测试文件返回 exit 0（语法正确） |
| G-RED-4 | `test_fails` | Φ_D | exec | `exec(test_runner, testPath) → exitCode !== 0` |
| G-RED-5 | `fails_for_assertion` | Φ_D | grep | stdout/stderr 匹配 `/assert|expect|fail/i` 且不匹配 `/SyntaxError|TypeError|Cannot find module/i` |
| G-RED-6 | `targets_spec_req` | Φ_D | cross_ref | `testFile.target_req_ids ⊆ spec.requirements.map(r=>r.id)` |

DC = 6/6 = **100%**

这是整个管道中最关键的洞察：**TDD RED 阶段可以完全确定性验证。**

#### Stage: TDD GREEN（写实现 + 验证通过）

| ID | 守卫 | 类型 | 机制 | 精确定义 |
|----|------|------|------|---------|
| G-GRN-1 | `test_passes` | Φ_D | exec | `exec(test_runner) → exitCode === 0` |
| G-GRN-2 | `all_tests_pass` | Φ_D | exec | `exec(test_runner, --all) → exitCode === 0` |
| G-GRN-3 | `no_todos_in_code` | Φ_D | grep | `!/(TODO|FIXME|HACK|XXX)/.test(prodCode)` |
| G-GRN-4 | `diff_within_scope` | Φ_D | git+cross_ref | `git diff --name-only ⊆ plan.file_map.paths` |
| G-GRN-5 | `code_compiles` | Φ_D | exec | `tsc --noEmit → exitCode === 0` (或等价命令) |
| G-GRN-6 | `lint_passes` | Φ_D | exec | `eslint → exitCode === 0` (或等价命令) |
| G-GRN-7 | `minimal_impl` | Φ_J | llm | diff 行数/断言数比率可辅助，但最终判断需要 LLM |

DC = 6/7 = **85.7%**

#### Stage: Verification Before Completion

| ID | 守卫 | 类型 | 机制 | 精确定义 |
|----|------|------|------|---------|
| G-VF-1 | `fresh_test_evidence` | Φ_D | exec | 当前回合中执行过 test 命令且有输出记录 |
| G-VF-2 | `all_tests_pass` | Φ_D | exec | `exec(test_runner) → exitCode === 0` |
| G-VF-3 | `all_tasks_done` | Φ_D | cross_ref | plan 中每个 task 标记 completed |
| G-VF-4 | `lint_passes` | Φ_D | exec | lint exit 0 |
| G-VF-5 | `typecheck_passes` | Φ_D | exec | typecheck exit 0 |
| G-VF-6 | `no_uncommitted` | Φ_D | git | `git status --porcelain → 空` |
| G-VF-7 | `requirements_met` | Φ_J | llm | 逐项核对需求与实现（语义匹配） |

DC = 6/7 = **85.7%**

#### Stage: Systematic Debugging

| ID | 守卫 | 类型 | 机制 | 精确定义 |
|----|------|------|------|---------|
| G-DB-1 | `bug_reproduced` | Φ_D | exec | 复现命令 exit code 符合预期（确认 bug 存在） |
| G-DB-2 | `phase1_completed` | Φ_D | cross_ref | TodoList 中阶段1条目全部 completed |
| G-DB-3 | `hypothesis_tested` | Φ_D | exec | 最小测试通过/失败符合假设预期 |
| G-DB-4 | `fix_count_under_3` | Φ_D | git+count | `git log --oneline | grep -c "fix" < 3` |
| G-DB-5 | `no_regression` | Φ_D | exec | 完整测试套件 exit 0 |
| G-DB-6 | `root_cause_identified` | Φ_J | llm | 需要推理，无法程序化 |
| G-DB-7 | `architecture_ok` | Φ_J | llm | 当 fix_count ≥ 3 时触发，需要判断 |

DC = 5/7 = **71.4%**

#### Stage: Spec Review（子代理）

| ID | 守卫 | 类型 | 机制 | 精确定义 |
|----|------|------|------|---------|
| G-SR-1 | `review_format_valid` | Φ_D | schema | review 输出含 issues[] + assessment |
| G-SR-2 | `all_reqs_reviewed` | Φ_D | cross_ref | review 覆盖了 spec 中所有 requirements |
| G-SR-3 | `no_overbuilding` | Φ_D | git+cross_ref | diff 文件 ⊆ plan.file_map |
| G-SR-4 | `spec_compliance` | Φ_J | llm | 代码是否实现了 spec（语义判断） |

DC = 3/4 = **75%**

#### Stage: Code Quality Review（子代理）

| ID | 守卫 | 类型 | 机制 | 精确定义 |
|----|------|------|------|---------|
| G-CQ-1 | `review_format_valid` | Φ_D | schema | 结构化输出 |
| G-CQ-2 | `lint_passes` | Φ_D | exec | lint exit 0 |
| G-CQ-3 | `code_readable` | Φ_J | llm | 可读性是语义判断 |
| G-CQ-4 | `naming_quality` | Φ_J | llm | 命名质量是语义判断 |
| G-CQ-5 | `architecture_sound` | Φ_J | llm | 架构合理性是语义判断 |

DC = 2/5 = **40%**（这是整个管道的瓶颈）

#### Stage: Finishing Branch

| ID | 守卫 | 类型 | 机制 | 精确定义 |
|----|------|------|------|---------|
| G-FN-1 | `tests_pass` | Φ_D | exec | exit 0 |
| G-FN-2 | `not_on_main` | Φ_D | git | `git branch --show-current ∉ ['main','master']` |
| G-FN-3 | `merge_tests_pass` | Φ_D | exec | 合并后 exit 0 |
| G-FN-4 | `worktree_cleaned` | Φ_D | git | worktree 列表中不含当前路径 |
| G-FN-5 | `user_confirmed` | Φ_J | human | 选择操作方式 |

DC = 4/5 = **80%**

#### Stage: Subagent-Driven Development（流程守卫）

| ID | 守卫 | 类型 | 机制 | 精确定义 |
|----|------|------|------|---------|
| G-SD-1 | `spec_review_before_code_review` | Φ_D | cross_ref | 审查日志中 spec-review 在 code-review 之前 |
| G-SD-2 | `one_agent_at_a_time` | Φ_D | count | 活跃子代理数 ≤ 1 |
| G-SD-3 | `implementer_status_valid` | Φ_D | schema | 状态 ∈ {DONE, DONE_WITH_CONCERNS, NEEDS_CONTEXT, BLOCKED} |
| G-SD-4 | `not_on_main` | Φ_D | git | 同 G-FN-2 |
| G-SD-5 | `concerns_assessed` | Φ_J | llm | DONE_WITH_CONCERNS 时需要判断是否阻塞 |

DC = 4/5 = **80%**

---

## 5. 全局确定性覆盖率

| 阶段 | Φ_D | Φ_J | 总计 | DC |
|------|-----|-----|------|-----|
| Brainstorming → Spec | 8 | 1 | 9 | 88.9% |
| Writing Plans | 7 | 1 | 8 | 87.5% |
| TDD RED | 6 | 0 | 6 | **100%** |
| TDD GREEN | 6 | 1 | 7 | 85.7% |
| Verification | 6 | 1 | 7 | 85.7% |
| Debugging | 5 | 2 | 7 | 71.4% |
| Spec Review | 3 | 1 | 4 | 75% |
| Code Quality Review | 2 | 3 | 5 | **40%** |
| Finishing Branch | 4 | 1 | 5 | 80% |
| SDD 流程守卫 | 4 | 1 | 5 | 80% |
| **合计** | **51** | **12** | **63** | **81%** |

**当前 Superpowers 估计 DC ≈ 27%** → **结构化后目标 DC ≈ 81%**

---

## 6. 判断节点穷举清单

以下是通过深度阅读全部 14 个 SKILL.md 识别出的约 65 个判断节点。

标记说明：
- **D** = 可完全确定性化
- **PD** = 部分确定性化（程序可辅助，但最终需判断）
- **J** = 纯判断，无法程序化

### 6.1 using-superpowers

| ID | 节点 | 当前方式 | 可程序化 |
|----|------|---------|---------|
| J1.1 | skill 是否有 1% 可能适用 | LLM 自评 | J |
| J1.2 | 是否进入规划模式 | LLM 自评 | J |
| J1.3 | 已加载 skill 中是否有清单 | LLM 阅读 | PD（grep 检测 checklist 语法） |
| J1.4 | 多 skill 的优先级排序 | LLM 自评 | J |
| J1.5 | skill 是刚性还是柔性 | skill 声明 + LLM 执行 | PD（frontmatter 可标记） |

### 6.2 brainstorming

| ID | 节点 | 当前方式 | 可程序化 |
|----|------|---------|---------|
| J2.1 | 是否涉及视觉内容 | LLM 自评 | J |
| J2.2 | 项目范围是否太大 | LLM 自评 | J |
| J2.3 | 每个问题用浏览器还是终端 | LLM 自评 | J |
| J2.4 | 用户批准设计 | 人类审批 | D（human gate） |
| J2.5 | 规格自检：占位符 | LLM 自评 | D（grep TBD/TODO） |
| J2.6 | 规格自检：内部一致性 | LLM 自评 | PD（结构化后可交叉引用） |
| J2.7 | 规格自检：范围 | LLM 自评 | J |
| J2.8 | 规格自检：歧义 | LLM 自评 | J |
| J2.9 | 用户审核规格 | 人类审批 | D（human gate） |
| J2.10 | YAGNI 判断 | LLM 自评 | PD（grep 使用频率） |

### 6.3 writing-plans

| ID | 节点 | 当前方式 | 可程序化 |
|----|------|---------|---------|
| J3.1 | 规格是否覆盖多个独立子系统 | LLM 自评 | J |
| J3.2 | 文件结构分解 | LLM 自评 | J |
| J3.3 | 现有文件是否需要拆分 | LLM 自评 | PD（行数可测量） |
| J3.4 | 自检：规格覆盖 | LLM 自评 | D（结构化后交叉引用） |
| J3.5 | 自检：占位符 | LLM 自评 | D（grep） |
| J3.6 | 自检：类型一致性 | LLM 自评 | PD（grep 命名） |
| J3.7 | 执行方式选择 | 人类选择 | D（human gate） |

### 6.4 executing-plans

| ID | 节点 | 当前方式 | 可程序化 |
|----|------|---------|---------|
| J4.1 | 批判性审查计划 | LLM 自评 | J |
| J4.2 | 是否停下寻求帮助 | LLM 自评 | PD（测试失败是确定性的） |
| J4.3 | 是否回退重新审查 | LLM + 人类 | J |
| J4.4 | 是否在 main/master 分支 | git 命令 | D |

### 6.5 test-driven-development

| ID | 节点 | 当前方式 | 可程序化 |
|----|------|---------|---------|
| J5.1 | TDD 例外是否适用 | LLM 识别 + 问人类 | PD |
| J5.2 | RED：测试因正确原因失败 | LLM 读输出 | D（解析 test output） |
| J5.3 | 测试在应该失败时通过 | test runner | D |
| J5.4 | GREEN：所有测试通过 | test runner | D |
| J5.5 | 测试是否测真实行为 vs mock | LLM 自评 | J |
| J5.6 | 是否先写了代码（铁律违反） | LLM 自律 / 记忆 | PD（git hook 可辅助） |
| J5.7 | 验证清单完成度 | LLM 自评 | PD |

### 6.6 systematic-debugging

| ID | 节点 | 当前方式 | 可程序化 |
|----|------|---------|---------|
| J6.1 | 能否可靠复现 | LLM 尝试 | PD（命令运行结果确定性） |
| J6.2 | 识别哪个组件边界失败 | 诊断日志 + LLM | PD |
| J6.3 | 假设是否被确认 | 最小测试结果 | D |
| J6.4 | 修复失败次数（阈值 3） | LLM 自计数 | D（git log 可计数） |
| J6.5 | 架构是否根本有问题 | LLM 自评 | J |
| J6.6 | 修复是否破坏其他测试 | 完整测试套件 | D |

### 6.7 verification-before-completion

| ID | 节点 | 当前方式 | 可程序化 |
|----|------|---------|---------|
| J8.1 | 验证证据是否新鲜 | LLM 自评 | PD（执行日志可检查） |
| J8.2 | 5 步验证序列 | LLM 执行 | PD（步骤 2-3 确定性） |
| J8.3 | 证据是否足够支撑声明 | LLM 匹配表格 | PD |
| J8.4 | agent 报告能否信任 | git diff | D |
| J8.5 | 需求是否满足 | 逐项核对 | J |

### 6.8 receiving-code-review

| ID | 节点 | 当前方式 | 可程序化 |
|----|------|---------|---------|
| J10.1 | 反馈对此代码库是否技术正确 | LLM 评估 | J |
| J10.2 | YAGNI 是否适用 | grep | D |
| J10.3 | 反馈条目是否不清楚 | LLM 自评 | J |
| J10.4 | 是否与人工伙伴的决策冲突 | LLM 自评 | J |
| J10.5 | 是否应该反驳 | LLM 依据 6 条标准 | J |

### 6.9 subagent-driven-development

| ID | 节点 | 当前方式 | 可程序化 |
|----|------|---------|---------|
| J12.1 | 使用 subagent-driven vs executing-plans vs 手动 | LLM 自评 | J |
| J12.2 | 每个任务的模型选择 | LLM 自评 | J |
| J12.3 | 处理实现者状态 | LLM 读状态报告 | PD（状态字段结构化） |
| J12.4 | DONE_WITH_CONCERNS 是否阻塞 | LLM 自评 | J |
| J12.5 | 规格合规审查通过/失败 | 子代理 | PD |
| J12.6 | 代码质量审查通过/失败 | 子代理 | PD |
| J12.7 | 审查循环是否完成 | 协议遵守 | PD |
| J12.8 | 审查顺序（规格先于质量） | LLM 自律 | D（Petri Net 弧强制） |

### 6.10 dispatching-parallel-agents

| ID | 节点 | 当前方式 | 可程序化 |
|----|------|---------|---------|
| J13.1 | 故障是否独立 | LLM 自评 | J |
| J13.2 | 代理能否并行（共享状态检查） | LLM 自评 | PD（文件重叠可检查） |
| J13.3 | 集成后验证 | 测试套件 + diff 审查 | PD |

### 6.11 finishing-a-development-branch

| ID | 节点 | 当前方式 | 可程序化 |
|----|------|---------|---------|
| J11.1 | 测试是否通过 | test runner | D |
| J11.2 | 确定基准分支 | git 命令 / 人类确认 | PD |
| J11.3 | 合并后测试验证 | test runner | D |
| J11.4 | 确认丢弃操作 | 人类输入 | D（精确输入 "discard"） |

### 6.12 using-git-worktrees

| ID | 节点 | 当前方式 | 可程序化 |
|----|------|---------|---------|
| J14.1 | worktree 目录选择 | 顺序命令 | D |
| J14.2 | 目录是否被 gitignore | git check-ignore | D |
| J14.3 | 基线测试是否通过 | test runner | D |
| J14.4 | 项目设置需求检测 | 文件存在检查 | D |

### 6.13 writing-skills / requesting-code-review

| ID | 节点 | 当前方式 | 可程序化 |
|----|------|---------|---------|
| J7.5 | token 效率（字数） | wc -w | D |
| J7.4 | description 质量 | LLM 自评 | PD（grep "Use when" 前缀） |
| J9.1 | 审查是否必须 vs 可选 | LLM 识别上下文 | PD |

### 汇总统计

```
总判断节点: ~65
D  (完全可确定性化):  ~21  (32%)
PD (部分可确定性化):  ~22  (34%)
J  (纯判断):          ~22  (34%)
```

**D + PD 的大部分 ≈ 结构化后可达到的 81% 覆盖率。**

---

## 7. 审计日志

每次变迁触发产生一条不可篡改的审计记录：

```json
{
  "id": "tx_00042",
  "timestamp": "2026-04-27T10:23:45.123Z",
  "skill": "test-driven-development",
  "transition": "t_verify_red",
  "guard_results": [
    {
      "guard_id": "G-RED-4",
      "category": "deterministic",
      "mechanism": "exec",
      "inputs": { "command": "npm test src/utils.test.ts", "timeout_ms": 30000 },
      "output": { "exitCode": 1, "stderr": "FAIL src/utils.test.ts\n  Expected: 42\n  Received: undefined" },
      "result": "PASS",
      "duration_ms": 3420
    },
    {
      "guard_id": "G-RED-5",
      "category": "deterministic",
      "mechanism": "grep",
      "inputs": { "pattern": "/assert|expect|fail/i", "content_source": "stderr" },
      "output": { "matched": true, "syntax_error_matched": false },
      "result": "PASS",
      "duration_ms": 2
    }
  ],
  "marking_before": { "p_test_written": 1, "p_test_red": 0 },
  "marking_after": { "p_test_written": 0, "p_test_red": 1 },
  "artifact_snapshot_hash": "sha256:a1b2c3..."
}
```

任何人（或程序）可以事后验证：
1. 这个 transition 被触发时，所有守卫是否确实被执行了
2. 守卫的输入数据是什么
3. 状态转移是否与 Petri Net 定义一致
4. LLM 没有伪造验证结果（因为 exec/git/grep 的输出是独立捕获的）

---

## 8. 数学性质

### 性质 1：确定性守卫的正确性（Soundness）

```
∀ φ ∈ Φ_D, ∀ a, C: φ(a, C) = Pass ⟹ P_φ(a, C) = true
```

确定性守卫就是属性 P_φ 的机械实现，正确性是平凡的。

### 性质 2：守卫独立性

```
∀ φᵢ, φⱼ ∈ Φ_D, i ≠ j: ¬(φᵢ ⟹ φⱼ)
```

如果 φᵢ 通过隐含 φⱼ 一定通过，则 φⱼ 冗余，应删除。

### 性质 3：恢复终止性

```
fail_count(Sᵢ) ≥ T ⟹ recovery = escalate_to_human
```

保证管道不会无限循环。T 的默认值 = 3（与 debugging skill 的"3+ 次修复失败 → 质疑架构"一致）。

### 性质 4：上下文单调性

```
C₀ ⊆ C₁ ⊆ ... ⊆ Cₖ
```

后续阶段可依赖前序阶段的 artifact，且前序 artifact 不会被篡改。

### 性质 5：Petri Net 安全性（1-bounded）

对本模型中的所有库所 p：

```
∀ 可达标记 M: M(p) ≤ 1
```

即每个状态条件同一时刻至多为真一次。这保证了不会出现"同时处于两个阶段"的非法状态。

### 性质 6：确定性守卫先行

```
∀ transition t: 
  exec_order(Φ_D(t)) < exec_order(Φ_J(t))
```

如果确定性守卫就能拒绝，不浪费判断性资源。

---

## 9. 迁移路径

按投入产出比排序：

| 优先级 | 改动 | 当前 DC → 目标 DC | 成本 | 依赖 |
|--------|------|-------------------|------|------|
| **P0** | Spec/Plan artifact 结构化（加 JSON schema） | 22% → ~60% | 中 | 无 |
| **P1** | TDD RED 失败原因解析器 | LLM 判断 → 100% 确定性 | 低 | P0 |
| **P2** | git hook 强制 test-before-code 顺序 | LLM 自律 → 确定性 | 低 | 无 |
| **P3** | Review 输出结构化（JSON schema） | 40% → 75% | 中 | P0 |
| **P4** | verification-before-completion 自动化 | 手动 → 确定性 | 中 | P1 |
| **P5** | 完整 Petri Net 引擎 + 审计日志 | - | 高 | P0-P4 |

**P0 的 ROI 最高**：仅通过将 spec 和 plan 从自由 markdown 转为带 schema 的结构化格式，就能将 DC 从 27% 提升到约 60%。

### OpenCode 插件集成路径

目标运行环境是 OpenCode 插件。集成方式：

1. **Phase 1**（P0-P2）：在现有 skill 旁加 `guards.ts` + `petri-net.json`，通过 OpenCode 的 `config` hook 注入守卫检查
2. **Phase 2**（P3-P4）：实现 PetriNetRuntime 作为 OpenCode plugin，拦截 LLM 的工具调用，在执行前/后运行守卫
3. **Phase 3**（P5）：完整引擎 + 审计日志 + DC 指标仪表盘

文件结构预想：

```
skills/
  test-driven-development/
    SKILL.md              # 不变——仍给 LLM 提供理解上下文
    guards.ts             # 新增：守卫函数实现
    petri-net.json        # 新增：形式化状态转移定义
  verification-before-completion/
    SKILL.md
    guards.ts
    petri-net.json
engine/
  runtime.ts              # Petri Net 执行引擎
  audit.ts                # 审计日志
  types.ts                # 共享类型
.opencode/
  plugin/
    superpowers-engine.ts  # OpenCode 集成入口
```

---

## 10. 两份来源分析的合并决策

| 内容 | 来源 | 采纳理由 |
|------|------|---------|
| Petri Net 作为执行模型 | Opus | 处理并发、循环、资源约束 |
| DC 覆盖率公式 | GLM-5 | 可量化的质量指标 |
| 结构化提升定理 | GLM-5 | artifact schema 化是前置依赖 |
| 确定性守卫先于判断性守卫 | GLM-5 | 节省 LLM 调用资源 |
| 65 个判断节点穷举清单 | GLM-5（补充） | 落地前提 |
| P0-P5 迁移优先级 | GLM-5（调整） | 行动指导 |
| 审计日志格式 | Opus | 可审计性保证 |
| 守卫函数的具体实现 | Opus | 可运行的代码 |
| 多层反馈环（内/中/外） | Opus | 理解系统层级 |
| 数学性质证明 | 合并 | 形式化保证 |

**弃用**：贝叶斯模型（数字无实证基础）、Curry-Howard 对应（过度类比，对落地无帮助）。
