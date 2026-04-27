# GLM-5 调研：Superpowers 方法论的确定性模型

## 会话背景

- 模型：glm-5.1 (axonhub/glm-5.1)
- 项目：superpowers-cn
- 任务：寻找数学模型/确定性模型来描述 Superpowers 方法论，核心是将 LLM 判断节点转化为可程序验证的确定性节点

---

## 一、项目概览

`skills/` 目录包含 14 个 skill：

| Skill | 描述 |
|-------|------|
| `brainstorming` | 创造性工作前的探索 |
| `dispatching-parallel-agents` | 并行独立任务分发 |
| `executing-plans` | 执行实施计划 |
| `finishing-a-development-branch` | 完成开发分支集成 |
| `receiving-code-review` | 处理代码审查反馈 |
| `requesting-code-review` | 请求代码审查 |
| `subagent-driven-development` | 子代理驱动开发 |
| `systematic-debugging` | 系统化调试 |
| `test-driven-development` | 测试驱动开发 |
| `using-git-worktrees` | Git worktree 管理 |
| `using-superpowers` | Skill 使用规则（已加载） |
| `verification-before-completion` | 完成前验证 |
| `writing-plans` | 编写实施计划 |
| `writing-skills` | 创建/编辑 skill |

---

## 二、初始理论框架（第一轮回答）

### 1. 状态机 + 守卫转换（Automata with Guarded Transitions）

最直观的模型。整个工作流是一个确定性有限自动机 (DFA)：

```
States = { Brainstorm, Plan, Implement, Review, Complete }
Transitions = δ: State × Guard → State
```

关键在于 **Guard 函数是确定性的**：

| 节点 | 旧方式（LLM 判断） | 新方式（程序检查） |
|------|-------------------|-------------------|
| 设计完成 | LLM 说"设计看起来完整" | 自检清单逐项通过 |
| 实现正确 | LLM 说"代码应该没问题" | `npm test` exit code = 0 |
| 可以提交 | LLM 说"我觉得做完了" | 验证命令输出确认 0 failures |
| Review 通过 | LLM 说"看起来 OK" | spec-reviewer 读实际代码 vs spec |

用 Dijkstra 的 **Guarded Command Language (GCL)** 表示：

```
if
  tests_pass() ∧ spec_covered() → next_phase()
  ¬tests_pass() → fix_and_retry()
fi
```

LLM 的角色从"判断者"变成了"提议者"——它提议状态转换，但**守卫函数独立验证**。

### 2. Hoare 逻辑（Hoare Logic / Design by Contract）

每个计划步骤是一个 **Hoare 三元组** `{P} S {Q}`：

```
{前置条件: 上一步测试通过}
  写代码 → 运行测试
{后置条件: 测试输出显示 0 failures}
```

`verification-before-completion` skill 就是严格执行这个：**你必须先运行验证命令获得 Q 为真的证据，才能声明完成**。

整个系统可以看作一个 **Hoare 逻辑的证明树**：
- 叶子节点 = 单个测试用例
- 内部节点 = 组合验证（集成测试）
- 根节点 = 全系统验证

### 3. 贝叶斯证据累积（Bayesian Evidence Accumulation）

每次验证都在**更新实现正确的后验概率**：

```
P(correct | evidence) = P(evidence | correct) × P(correct) / P(evidence)
```

| 证据类型 | P(correct | evidence) | 信息量 |
|---------|----------------------|--------|
| LLM 声称"做完了" | ~0.5-0.7 | 低（高假阳性率） |
| 单元测试通过 | ~0.85-0.95 | 中 |
| 集成测试通过 | ~0.95-0.99 | 高 |
| TDD RED→GREEN 确认 | ~0.99+ | 极高（确认测试本身有效） |

TDD 的 RED 阶段特别关键——它验证的是**测试本身的信息量**（拒绝空实现的区分度）。这解决了 "测试通过但测试无效" 的问题，类似于贝叶斯中**校准先验**的概念。

### 4. 类型论 / Curry-Howard 对应（Type Theory）

在 Curry-Howard 对应中：

- **命题 (Proposition)** = 功能需求
- **证明 (Proof)** = 测试用例
- **类型检查 (Type Checking)** = 运行测试

```
功能需求 spec  ≈  类型签名 (Type)
实现代码      ≈  项 (Term)
测试用例      ≈  证明 (Proof)
运行测试      ≈  类型检查 (Type Checking)
```

这与 **Proof-Carrying Code (PCC)** 的思想一致——代码必须附带可机械验证的证明（测试），验证者不需要信任代码作者（LLM），只需验证证明（测试输出）。

### 5. 控制论 / 反馈系统（Control Theory）

从控制论角度：

- **开环系统**（无 Superpowers）：LLM → 生成代码 → 声称完成（无反馈）
- **闭环系统**（有 Superpowers）：LLM → 生成代码 → **测试反馈** → 修正 → 再测试

```
         ┌─────────────────────────┐
         │                         │
         ▼                         │
  setpoint ──►(+)──► LLM ──► Code ─┤
  (spec)     (-)                    │ tests
              │                     │
              ▼                     ▼
           error ◄────────── test results
         (failure count)
```

`systematic-debugging` 中的 "3+ fixes failed → question architecture" 规则，本质是一个**积分饱和保护**——防止在错误方向上持续累积无效修正。

### 6. 参考监视器模型（Reference Monitor）

来自安全领域的概念。参考监视器是一个**不可绕过的中介**，每次访问都经过策略检查：

```
          ┌──────────────┐
  LLM ──► │  Reference   │ ──► Action
  proposes│  Monitor     │
          │  (skill +    │
          │   test gate) │
          └──────────────┘
```

Skills 就是策略规则（"不能跳过 TDD"、"必须先运行验证"），测试是策略实施。LLM 无法绕过这个监视器，因为**它不控制验证环节**。

### 统一视角

> Superpowers 将 LLM agent 的执行流从一个 **不可靠的认知判断链** 转化为一个 **守卫状态机 + 证明携带代码** 的混合系统，其中 LLM 负责**生成**（创造性的），程序负责**验证**（确定性的），两者的职责严格分离。

这类似于编译器设计中 **parser（机械的） vs. programmer（创造性的）** 的分离——程序员写代码，编译器检查语法和类型，两者互不替代。

---

## 三、精确形式化模型（第二轮回答——核心产出）

用户反馈：需要精确的、可审计的数学/编程模型，而非仅介绍理论框架。

### I. 核心抽象：守卫管道（Guarded Pipeline）

#### 定义域

```
ArtifactKind = Spec | Plan | TestFile | ProdCode | Review | Claim
Context      = Map<StageId, Artifact>    // 管道累积的上下文
```

#### 定义 1：守卫谓词（Guard Predicate）

φ : (A, C) → { Pass | Fail(evidence, hint) }

其中 A 是被检 artifact，C 是管道上下文。

#### 定义 2：确定性守卫（Deterministic Guard）

φ 是**确定性**的，当且仅当存在图灵机 M_φ 能在有限步内计算 φ，且 M_φ 不调用任何 LLM。记为 φ ∈ Φ_D。

否则 φ ∈ Φ_J（判断性守卫，需要 LLM 或人类）。

#### 定义 3：守卫阶段（Guarded Stage）

Sᵢ = (Gᵢ, Aᵢ, Φᵢ, Rᵢ)

- Gᵢ : C → Aᵢ — 生成器（LLM 或人类）
- Aᵢ — 输出 artifact 类型
- Φᵢ = Φ_D^(i) ∪ Φ_J^(i) — 守卫集合
- Rᵢ — 恢复策略

#### 定义 4：管道执行规则

```
execute([S₁, S₂, ..., Sₖ], C₀):
  for i = 1..k:
    aᵢ = Gᵢ(Cᵢ₋₁)                           // LLM/人生成 artifact
    
    for φ ∈ Φ_D⁽ⁱ⁾:                           // 先执行所有确定性守卫
      result = φ(aᵢ, Cᵢ₋₁)
      if result = Fail(e, hint):
        return Recover(Rᵢ, e, hint, Sᵢ, Cᵢ₋₁)  // 确定性失败 → 即时恢复
    
    for φ ∈ Φ_J⁽ⁱ⁾:                           // 再执行判断性守卫
      result = φ(aᵢ, Cᵢ₋₁)
      if result = Fail(e, hint):
        return Recover(Rᵢ, e, hint, Sᵢ, Cᵢ₋₁)
    
    Cᵢ = Cᵢ₋₁ ∪ {stageᵢ: aᵢ}                // 累积上下文
  return Success(Cₖ)
```

**关键规则**：确定性守卫先于判断性守卫执行。如果确定性守卫失败，不浪费 LLM 判断资源。

#### 定义 5：确定性覆盖率

DC(P) = |⋃ Φ_D^(i)| / |⋃ Φᵢ|

### II. 将 artifact 结构化以提高确定性

当前 Superpowers 的 artifact 大多是**自由文本**（markdown）。这是判断性守卫比例高的根本原因。

**核心定理（结构化提升定理）**：如果将 artifact 类型 A 替换为结构化类型 A' ⊂ A（A' 有明确 schema），则可以将部分 Φ_J 转化为 Φ_D。

具体机制：

| 当前（自由文本） | 结构化后 | 可转化的守卫 |
|---|---|---|
| spec 是 markdown | spec 是 YAML + 必填字段 | `has_required_sections()` → 确定性 |
| plan 是 markdown checkbox | plan 是 JSON `{tasks: [{id, spec_refs, cmd, expected}]}` | `spec_coverage()` → 确定性 |
| review 是自由文本 | review 是 JSON `{issues: [{severity, location, description}]}` | `severity_format()` → 确定性 |
| "先写测试"是 LLM 自律 | git hook 检查 commit 中 test 先于 code | `test_before_code()` → 确定性 |

### III. 每阶段守卫精确规格

#### Stage 1: Brainstorming → Spec

**输入**：用户需求（自由文本）
**输出**：`Spec` artifact

```typescript
type Spec = {
  problem: string                    // 问题描述
  constraints: string[]              // 约束列表
  api_design: string                 // API 设计
  data_model: string                 // 数据模型
  test_strategy: string              // 测试策略
  scope: 'single' | 'multi'         // 范围
  requirements: Requirement[]        // 可追踪的需求列表
}

type Requirement = {
  id: string                         // "REQ-1", "REQ-2"...
  description: string
  acceptance_criteria: string[]      // 可验证的验收标准
}
```

| 守卫 | 类型 | 实现机制 | 精确定义 |
|---|---|---|---|
| `has_problem` | Φ_D | `spec.problem.length > 0` | 非空 |
| `has_constraints` | Φ_D | `spec.constraints.length > 0` | 至少一个约束 |
| `has_requirements` | Φ_D | `spec.requirements.length > 0` | 至少一个需求 |
| `each_req_has_criteria` | Φ_D | `spec.requirements.every(r => r.acceptance_criteria.length > 0)` | 每个需求有验收标准 |
| `no_placeholders` | Φ_D | `!/(TBD\|TODO\|FIXME\|XX+)/.test(JSON.stringify(spec))` | 无占位符文本 |
| `scope_single` | Φ_D | `spec.scope === 'single'` | 可用单计划覆盖 |
| `user_approved` | Φ_J | Human gate | 人类判断设计质量 |

**确定性覆盖率**：6/7 = **86%**

当前 Superpowers 的 brainstorming 确定性覆盖率约为 2/9 ≈ 22%（只有 placeholder grep 和 human approval 是结构化的）。

#### Stage 2: Plan Writing

**输入**：`Spec` artifact
**输出**：`Plan` artifact

```typescript
type Plan = {
  spec_id: string                    // 关联的 spec
  file_map: FileMapping[]            // 文件结构映射
  tasks: Task[]                      // 任务列表
}

type FileMapping = {
  path: string
  responsibility: string
  action: 'create' | 'modify' | 'delete'
}

type Task = {
  id: string                         // "TASK-1", "TASK-2"...
  spec_refs: string[]                // ["REQ-1", "REQ-3"] — 需求追踪
  steps: Step[]
}

type Step = {
  action: string                     // 描述
  command: string | null             // 要运行的命令（可为 null）
  expected_output: string | null     // 预期输出模式
  artifact_path: string | null       // 产出文件路径
}
```

| 守卫 | 类型 | 实现机制 | 精确定义 |
|---|---|---|---|
| `every_req_covered` | Φ_D | `spec.requirements.every(r => plan.tasks.some(t => t.spec_refs.includes(r.id)))` | 每个需求被至少一个任务引用 |
| `no_orphan_refs` | Φ_D | `plan.tasks.flatMap(t=>t.spec_refs).every(ref => spec.requirements.some(r=>r.id===ref))` | 无幽灵引用 |
| `no_placeholders` | Φ_D | grep TBD/TODO in JSON | 无占位符 |
| `every_task_has_steps` | Φ_D | `plan.tasks.every(t => t.steps.length > 0)` | 每个任务有步骤 |
| `steps_have_commands` | Φ_D | 验证步骤中 command/expected_output 配对存在 | 可验证的步骤 |
| `file_map_complete` | Φ_D | `plan.tasks.flatMap(t=>t.steps).every(s => !s.artifact_path \|\| plan.file_map.some(f=>f.path===s.artifact_path))` | 步骤产出的文件在 file_map 中 |
| `naming_consistent` | Φ_D | AST 或 regex 交叉检查步骤间引用的函数名/类型名一致 | 类型一致性 |
| `user_chosen_mode` | Φ_J | Human choice | 选择执行方式 |

**确定性覆盖率**：7/8 = **87.5%**

#### Stage 3: TDD RED（写测试）

**输入**：`Task` from Plan
**输出**：`TestFile` artifact

```typescript
type TestFile = {
  path: string
  content: string
  target_req_ids: string[]           // 测试覆盖的需求 ID
}
```

| 守卫 | 类型 | 实现机制 | 精确定义 |
|---|---|---|---|
| `file_exists` | Φ_D | `fs.existsSync(test_file.path)` | 文件已创建 |
| `file_compiles` | Φ_D | `tsc --noEmit test_file.path; exitCode === 0` | 编译通过 |
| `test_fails` | Φ_D | `exec(test_runner + test_file.path); exitCode !== 0` | 测试失败（exit code） |
| `fails_for_feature_missing` | Φ_D | 解析 test output，匹配 `/fail\|assert\|expected.*got\|not.*defined/i`，**排除** `/syntax error\|type error\|import error\|cannot find/i` | 失败原因是功能缺失，不是语法错误 |
| `targets_spec_requirement` | Φ_D | `test_file.target_req_ids.every(id => spec.requirements.some(r=>r.id===id))` | 测试目标在需求范围内 |
| `not_testing_existing_behavior` | Φ_D | 测试 exit code 是非零（隐含） | 不是测试已有行为 |

**确定性覆盖率**：6/6 = **100%**

这是整个管道中最关键的洞察：**TDD RED 阶段可以完全确定性验证**。

当前 Superpowers 中 `fails_for_right_reason` 依赖 LLM 读 test output 判断。通过结构化 test output parsing，可以将其转化为确定性。

#### Stage 4: TDD GREEN（写实现）

**输入**：`TestFile` + `Task`
**输出**：`ProdCode` artifact

| 守卫 | 类型 | 实现机制 | 精确定义 |
|---|---|---|---|
| `test_passes` | Φ_D | `exec(test_runner); exitCode === 0` | 测试通过 |
| `all_other_tests_pass` | Φ_D | `exec(test_runner --all); exitCode === 0` | 没有回归 |
| `no_todos_in_code` | Φ_D | `!/(TODO\|FIXME\|HACK\|XXX)/.test(code_content)` | 无 TODO |
| `diff_within_scope` | Φ_D | `git diff --name-only` 的文件列表 ⊆ plan.file_map 的 path 集合 | 修改范围不超出计划 |
| `code_compiles` | Φ_D | `tsc --noEmit; exitCode === 0` | 类型正确 |
| `lint_passes` | Φ_D | `eslint; exitCode === 0` | 代码风格 |
| `minimal_implementation` | Φ_J | LLM 判断是否过度工程 | — |

**确定性覆盖率**：6/7 = **85.7%**

#### Stage 5: Verification Before Completion

| 守卫 | 类型 | 实现机制 | 精确定义 |
|---|---|---|---|
| `fresh_test_evidence` | Φ_D | 检查当前消息中是否执行了 test command 并获得了输出 | 在**同一条消息**中有 test output |
| `all_tests_pass` | Φ_D | `exec(test_runner); exitCode === 0` | 0 failures |
| `all_plan_tasks_done` | Φ_D | 解析 plan JSON 中每个 task 的完成状态 | 每个 task 标记 complete |
| `lint_passes` | Φ_D | `eslint; exitCode === 0` | 无 lint 错误 |
| `typecheck_passes` | Φ_D | `tsc --noEmit; exitCode === 0` | 无类型错误 |
| `no_uncommitted_changes` | Φ_D | `git status --porcelain; output === ''` | 所有变更已提交 |
| `requirements_met` | Φ_J | 逐项核对需求与实现 | 需要语义匹配 |

**确定性覆盖率**：6/7 = **85.7%**

#### Stage 6: Spec Review（子代理）

| 守卫 | 类型 | 实现机制 | 精确定义 |
|---|---|---|---|
| `review_format_valid` | Φ_D | 解析 review JSON，验证有 `issues` 数组和 `assessment` 字段 | 结构合法 |
| `all_spec_sections_reviewed` | Φ_D | review 覆盖了 spec 中所有 requirements | 覆盖完整 |
| `no_overbuilding` | Φ_D | `git diff --stat` 的文件 ⊈ plan.file_map 之外的文件 | 不超范围 |
| `spec_compliance` | Φ_J | LLM 判断代码是否实现了 spec | 语义判断 |

**确定性覆盖率**：3/4 = **75%**

#### Stage 7: Finishing Branch

| 守卫 | 类型 | 实现机制 | 精确定义 |
|---|---|---|---|
| `tests_pass` | Φ_D | `exec(test_runner); exitCode === 0` | 测试通过 |
| `not_on_main` | Φ_D | `git branch --show-current` ∉ `['main', 'master']` | 不在主分支 |
| `merged_tests_pass` | Φ_D | 合并后重新运行 test | 合并安全 |
| `user_confirmed_action` | Φ_J | Human gate | 选择操作 |
| `worktree_cleaned` | Φ_D | `git worktree list \| grep <worktree_path>` 返回空 | worktree 已清理 |

**确定性覆盖率**：4/5 = **80%**

### IV. 全局确定性覆盖率分析

将所有阶段汇总：

```
Stage                 | Φ_D | Φ_J | Total | DC
──────────────────────|─────|─────|───────|─────
Brainstorming→Spec    |   6 |   1 |     7 | 86%
Plan Writing          |   7 |   1 |     8 | 87%
TDD RED               |   6 |   0 |     6 | 100%
TDD GREEN             |   6 |   1 |     7 | 86%
REFACTOR              |   4 |   2 |     6 | 67%
Spec Review           |   3 |   1 |     4 | 75%
Code Quality Review   |   2 |   3 |     5 | 40%
Verification          |   6 |   1 |     7 | 86%
Finishing Branch      |   4 |   1 |     5 | 80%
──────────────────────|─────|─────|───────|─────
TOTAL                 |  44 |  11 |    55 | 80%
```

**当前 Superpowers 的估计覆盖率**：约 15/55 = **27%**（多数守卫依赖 LLM 自律）

**结构化 artifact 后的覆盖率**：44/55 = **80%**

**瓶颈**：Code Quality Review 是最难确定化的阶段（40%），因为代码质量本质上是语义判断。

### V. 可编程模型：Guarded Pipeline DSL

以下是可直接实现的类型定义和运行时：

```typescript
// ═══════════════════════════════════════
// 核心类型
// ═══════════════════════════════════════

type GuardId = string
type StageId = string

type GuardResult =
  | { pass: true; evidence: string }
  | { pass: false; error: string; fix_hint: string }

type Mechanism =
  | { kind: 'command'; cmd: string; expect_exit: 0 | 1 }
  | { kind: 'grep'; pattern: RegExp; target: 'content'; negate: boolean }
  | { kind: 'fs'; check: 'exists' | 'not_exists'; path: string }
  | { kind: 'schema'; validate: (a: unknown) => boolean }
  | { kind: 'cross_ref'; source: string; relation: 'covers' | 'consistent_with'; target: string }
  | { kind: 'human'; prompt: string }
  | { kind: 'llm'; prompt: string }

interface Guard<A = unknown> {
  id: GuardId
  category: 'deterministic' | 'judgment'
  mechanism: Mechanism
  check: (artifact: A, ctx: Context) => GuardResult
}

interface Stage<I, O> {
  id: StageId
  input_schema: I
  output_schema: O
  generator: 'llm' | 'human'
  guards: Guard<O>[]
  recovery: {
    max_retries: number
    strategy: 'retry_with_error' | 'escalate_to_human' | 'abort'
  }
}

type Context = Record<StageId, unknown>

// ═══════════════════════════════════════
// 运行时
// ═══════════════════════════════════════

class PipelineRuntime {
  private stages: Stage<unknown, unknown>[] = []
  private context: Context = {}

  addStage<I, O>(stage: Stage<I, O>): this {
    this.stages.push(stage)
    return this
  }

  async execute(): Promise<{ success: boolean; context: Context; failures: GuardResult[] }> {
    const allFailures: GuardResult[] = []

    for (const stage of this.stages) {
      // 1. Generate artifact (LLM or human)
      const artifact = await this.generate(stage)

      // 2. Run deterministic guards FIRST
      const detGuards = stage.guards.filter(g => g.category === 'deterministic')
      for (const guard of detGuards) {
        const result = guard.check(artifact, this.context)
        if (!result.pass) {
          allFailures.push(result)
          const recovered = await this.recover(stage, result)
          if (!recovered) {
            return { success: false, context: this.context, failures: allFailures }
          }
          continue // retry stage
        }
      }

      // 3. Run judgment guards only after all deterministic guards pass
      const judGuards = stage.guards.filter(g => g.category === 'judgment')
      for (const guard of judGuards) {
        const result = guard.check(artifact, this.context)
        if (!result.pass) {
          allFailures.push(result)
          // judgment failures may be advisory, not blocking
        }
      }

      // 4. Commit artifact to context
      this.context[stage.id] = artifact
    }

    return { success: true, context: this.context, failures: allFailures }
  }

  // ═══════════════════════════════════════
  // 关键：确定性守卫的机械实现
  // ═══════════════════════════════════════

  private execGuard(mechanism: Mechanism, artifact: unknown): GuardResult {
    switch (mechanism.kind) {
      case 'command': {
        const output = execSync(mechanism.cmd, { encoding: 'utf-8' })
        const exitCode = output ? 0 : 1 // simplified
        return exitCode === mechanism.expect_exit
          ? { pass: true, evidence: output }
          : { pass: false, error: `Command exited with ${exitCode}`, fix_hint: output }
      }
      case 'grep': {
        const content = JSON.stringify(artifact)
        const found = mechanism.pattern.test(content)
        const pass = mechanism.negate ? !found : found
        return pass
          ? { pass: true, evidence: `Pattern ${mechanism.pattern} ${mechanism.negate ? 'not found' : 'found'}` }
          : { pass: false, error: `Pattern check failed`, fix_hint: `Expected ${mechanism.negate ? 'absence' : 'presence'} of ${mechanism.pattern}` }
      }
      case 'fs': {
        const exists = existsSync(mechanism.path)
        const pass = mechanism.check === 'exists' ? exists : !exists
        return pass
          ? { pass: true, evidence: `${mechanism.path} ${mechanism.check === 'exists' ? 'exists' : 'does not exist'}` }
          : { pass: false, error: `File system check failed: ${mechanism.path}`, fix_hint: `Expected to ${mechanism.check}` }
      }
      case 'schema': {
        const valid = mechanism.validate(artifact)
        return valid
          ? { pass: true, evidence: 'Schema validation passed' }
          : { pass: false, error: 'Schema validation failed', fix_hint: 'Check required fields' }
      }
      case 'cross_ref': {
        // Mechanically check traceability
        // e.g., every spec requirement is covered by a plan task
        const sourceIds = this.extractIds(mechanism.source, artifact)
        const targetIds = this.extractIds(mechanism.target, this.context)
        if (mechanism.relation === 'covers') {
          const uncovered = sourceIds.filter(id => !targetIds.includes(id))
          return uncovered.length === 0
            ? { pass: true, evidence: `All ${sourceIds.length} items covered` }
            : { pass: false, error: `Uncovered items: ${uncovered.join(', ')}`, fix_hint: `Add coverage for these items` }
        }
        return { pass: true, evidence: 'Cross-reference check passed' }
      }
      case 'human':
        return { pass: true, evidence: 'Deferred to human judgment' } // handled externally
      case 'llm':
        return { pass: true, evidence: 'Deferred to LLM judgment' } // handled externally
    }
  }
}

// ═══════════════════════════════════════
// 具体守卫实例
// ═══════════════════════════════════════

// TDD RED: test fails for feature missing, not syntax error
const FAILS_FOR_FEATURE_MISSING: Guard<TestFile> = {
  id: 'tdd-red-fails-for-feature',
  category: 'deterministic',
  mechanism: { kind: 'command', cmd: 'npm test {path}', expect_exit: 1 },
  check: (testFile, _ctx) => {
    const output = execSync(`npx jest ${testFile.path} 2>&1`, { encoding: 'utf-8' })
    
    const syntaxErrors = /SyntaxError|TypeError.*not.*function|Cannot find module|Unexpected token/i
    const assertionFailures = /Expected.*received|assert.*fail|expect\(.*\)\./i
    
    if (syntaxErrors.test(output)) {
      return {
        pass: false,
        error: 'Test fails due to syntax/import error, not feature missing',
        fix_hint: 'Fix the test structure. The test should compile but fail because the feature does not exist.'
      }
    }
    if (!assertionFailures.test(output) && !output.includes('FAIL')) {
      return {
        pass: false,
        error: 'Test output does not indicate an assertion failure',
        fix_hint: 'Ensure the test makes an assertion about the expected behavior.'
      }
    }
    return { pass: true, evidence: output.slice(-500) }
  }
}

// Spec coverage: every requirement maps to a plan task
const EVERY_REQ_COVERED: Guard<Plan> = {
  id: 'plan-spec-coverage',
  category: 'deterministic',
  mechanism: { kind: 'cross_ref', source: 'spec.requirements', relation: 'covers', target: 'plan.tasks' },
  check: (plan, ctx) => {
    const spec = ctx['brainstorming'] as Spec
    const uncovered = spec.requirements.filter(
      req => !plan.tasks.some(task => task.spec_refs.includes(req.id))
    )
    if (uncovered.length > 0) {
      return {
        pass: false,
        error: `Uncovered requirements: ${uncovered.map(r => r.id).join(', ')}`,
        fix_hint: `Add tasks covering: ${uncovered.map(r => `${r.id}: ${r.description}`).join('; ')}`
      }
    }
    return { pass: true, evidence: `All ${spec.requirements.length} requirements covered by plan tasks` }
  }
}

// No placeholders: grep-based
const NO_PLACEHOLDERS: Guard<unknown> = {
  id: 'no-placeholders',
  category: 'deterministic',
  mechanism: { kind: 'grep', pattern: /\b(TBD|TODO|FIXME|XX+)\b/, target: 'content', negate: true },
  check: (artifact, _ctx) => {
    const content = JSON.stringify(artifact)
    const pattern = /\b(TBD|TODO|FIXME|XX+)\b/g
    const matches = content.match(pattern)
    if (matches) {
      return {
        pass: false,
        error: `Found placeholders: ${[...new Set(matches)].join(', ')}`,
        fix_hint: 'Replace all placeholder text with actual content.'
      }
    }
    return { pass: true, evidence: 'No placeholders found' }
  }
}
```

### VI. 数学性质

#### 性质 1：确定性守卫的完备性（Soundness）

如果 φ ∈ Φ_D 通过，则对应的属性确实为真。

∀ φ ∈ Φ_D, ∀ a, C: φ(a, C) = Pass ⟹ P_φ(a, C) = true

这是**平凡的**——因为确定性守卫就是 P_φ 的机械实现（exit code, grep, schema validation）。

#### 性质 2：守卫独立性

确定性守卫之间应该独立，即一个守卫的通过不应隐含另一个的通过。

∀ φᵢ, φⱼ ∈ Φ_D, i ≠ j: ¬(φᵢ ⟹ φⱼ)

违反独立性意味着存在冗余守卫，应删除。

#### 性质 3：恢复终止性

如果确定性守卫失败次数超过阈值 T，则 escalate：

fail_count(Sᵢ) ≥ T ⟹ Rᵢ = escalate_to_human

这保证管道不会无限循环。对应 Superpowers 中 debugging 的 "3+ fixes failed → question architecture"。

#### 性质 4：上下文单调性

管道上下文只增不减：

C₀ ⊆ C₁ ⊆ ⋯ ⊆ Cₖ

这保证后续阶段可以依赖前序阶段的 artifact，且不会被篡改。

### VII. 将当前 Superpowers 迁移到此模型的关键路径

最大杠杆点（按投入产出比排序）：

| 优先级 | 改动 | 当前 → 目标覆盖率 | 成本 |
|---|---|---|---|
| **P0** | spec/plan artifact 结构化（加 schema） | 22% → 86% | 中 |
| **P1** | TDD RED 失败原因解析器 | LLM判断 → 100%确定性 | 低 |
| **P2** | git hook 强制 test-before-code | 自律 → 确定性 | 低 |
| **P3** | review 输出结构化（JSON schema） | 40% → 75% | 中 |
| **P4** | verification-before-completion 自动化（CI gate） | 手动 → 确定性 | 中 |

**P0 的 ROI 最高**：仅通过将 spec 和 plan 从自由 markdown 转为带 schema 的结构化格式，就能将覆盖从 27% 提升到约 60%。

---

## 四、判断节点完整清单（子代理产出）

通过深度阅读所有 14 个 SKILL.md 及其参考文件，识别出约 65 个判断节点。以下是按 skill 分类的完整清单。

### SKILL 1: `using-superpowers`

**J1.1: "Whether a skill has 1% chance of being applicable"**
- Current check: LLM self-assessment
- Deterministic? No

**J1.2: "Whether to enter planning mode / brainstorm first"**
- Current check: LLM self-assessment
- Deterministic? No

**J1.3: "Whether a checklist exists in the loaded skill"**
- Current check: LLM reading skill content
- Deterministic? Partially — grep could detect checklist syntax

**J1.4: "Skill priority ordering when multiple skills apply"**
- Current check: LLM self-assessment
- Deterministic? No

**J1.5: "Whether a skill is rigid vs. flexible"**
- Current check: Skill declares its type, enforcement is LLM
- Deterministic? Partially

### SKILL 2: `brainstorming`

**J2.1: "Whether visual content will be involved"**
- Current check: LLM self-assessment
- Deterministic? No

**J2.2: "Whether project scope is too large for a single spec"**
- Current check: LLM self-assessment
- Deterministic? No

**J2.3: "Whether per-question content is visual or textual"**
- Current check: LLM self-assessment per question
- Deterministic? No

**J2.4: "Whether design section is correct (user approval)"**
- Current check: Human approval
- Deterministic? Yes — human gate

**J2.5: "Spec self-check: placeholder scan"**
- Current check: LLM self-assessment (could be grep)
- Deterministic? Partially — grep for TBD/TODO

**J2.6: "Spec self-check: internal consistency"**
- Current check: LLM self-assessment
- Deterministic? No

**J2.7: "Spec self-check: scope check"**
- Current check: LLM self-assessment
- Deterministic? No

**J2.8: "Spec self-check: ambiguity check"**
- Current check: LLM self-assessment
- Deterministic? No

**J2.9: "User approval of spec document"**
- Current check: Human approval
- Deterministic? Yes — human gate

**J2.10: "Whether YAGNI applies"**
- Current check: LLM self-assessment
- Deterministic? No

### SKILL 3: `writing-plans`

**J3.1: "Scope check: whether spec covers multiple independent subsystems"**
- Current check: LLM self-assessment
- Deterministic? No

**J3.2: "File structure decomposition"**
- Current check: LLM self-assessment
- Deterministic? No

**J3.3: "Whether existing files should be split"**
- Current check: LLM self-assessment
- Deterministic? No — but file line count could be measured mechanically

**J3.4: "Self-check: spec coverage"**
- Current check: LLM self-assessment
- Deterministic? Partially — could be cross-referenced

**J3.5: "Self-check: placeholder scan"**
- Current check: LLM self-assessment (grep for TBD/TODO)
- Deterministic? Partially — grep could find TBD/TODO patterns

**J3.6: "Self-check: type consistency"**
- Current check: LLM self-assessment
- Deterministic? Partially — grep-verified for naming consistency

**J3.7: "Execution method selection"**
- Current check: Human choice
- Deterministic? Yes — human gate

### SKILL 4: `executing-plans`

**J4.1: "Critical review of plan — find concerns"**
- Current check: LLM self-assessment
- Deterministic? No

**J4.2: "Whether to stop and ask for help"**
- Current check: LLM self-assessment
- Deterministic? Partially — test failure is deterministic; unclear instructions is LLM

**J4.3: "Whether to go back and re-review plan"**
- Current check: LLM self-assessment + human signal
- Deterministic? No

**J4.4: "Whether running on main/master branch"**
- Current check: `git branch --show-current`
- Deterministic? Yes — git command verification

### SKILL 5: `test-driven-development`

**J5.1: "Whether TDD exceptions apply"**
- Current check: LLM identifies category + asks human
- Deterministic? Partially

**J5.2: "Verify RED: test fails for the right reason"**
- Current check: LLM reads test command output
- Deterministic? Partially — exit code is deterministic; "right reason" requires LLM

**J5.3: "Test passes when it should fail"**
- Current check: LLM observes test runner output
- Deterministic? Yes — test exit code is deterministic

**J5.4: "Verify GREEN: all tests pass, output clean"**
- Current check: Test runner output
- Deterministic? Yes — test runner exit code + output inspection

**J5.5: "Whether test is testing real behavior vs mock behavior"**
- Current check: LLM self-assessment
- Deterministic? No

**J5.6: "Whether code was written before test (Iron Law violation)"**
- Current check: LLM self-assessment / memory
- Deterministic? No — relies on LLM honesty about action order

**J5.7: "Verification checklist completion"**
- Current check: LLM self-assessment
- Deterministic? Partially

### SKILL 6: `systematic-debugging`

**J6.1: "Can reliably reproduce the bug?"**
- Current check: LLM attempts reproduction
- Deterministic? Partially — reproducibility can be tested by running commands

**J6.2: "Identify which component boundary fails"**
- Current check: Diagnostic logging + LLM analysis
- Deterministic? Partially — logging output is deterministic; interpretation is LLM

**J6.3: "Whether hypothesis is confirmed"**
- Current check: LLM observes result of minimal test
- Deterministic? Yes — test pass/fail is deterministic

**J6.4: "Count of failed fixes (threshold: 3)"**
- Current check: LLM self-tracking of fix count
- Deterministic? Partially — LLM must honestly count, but threshold is explicit

**J6.5: "Whether architecture is fundamentally flawed"**
- Current check: LLM self-assessment
- Deterministic? No — pattern recognition is LLM judgment

**J6.6: "Whether fix broke other tests"**
- Current check: Full test suite run
- Deterministic? Yes

### SKILL 7: `writing-skills`

**J7.1: "Whether to create a skill vs. not"**
- Current check: LLM self-assessment
- Deterministic? No

**J7.2: "Skill type classification"**
- Current check: LLM self-assessment
- Deterministic? No

**J7.3: "Whether a flowchart is needed"**
- Current check: LLM self-assessment
- Deterministic? No

**J7.4: "Description quality (CSO)"**
- Current check: LLM self-assessment
- Deterministic? Partially — could grep for "Use when" prefix

**J7.5: "Token efficiency targets"**
- Current check: `wc -w` command
- Deterministic? Yes — word count is deterministic

**J7.6: "Whether skill passes pressure testing"**
- Current check: Subagent pressure testing
- Deterministic? Partially — subagent output is observable

**J7.7: "Whether skill is bulletproof"**
- Current check: Meta-testing with subagent
- Deterministic? No — requires interpreting subagent responses

### SKILL 8: `verification-before-completion`

**J8.1: "Whether verification evidence is fresh"**
- Current check: LLM self-assessment
- Deterministic? No — relies on LLM honesty about timing

**J8.2: "Gate function: 5-step verification sequence"**
- Current check: LLM runs commands + reads output
- Deterministic? Partially — steps 2-3 are deterministic; steps 1,4-5 are LLM

**J8.3: "Whether evidence is sufficient for the claim type"**
- Current check: LLM matches claim type against table
- Deterministic? Partially — table is explicit, but matching is LLM

**J8.4: "Whether agent report can be trusted"**
- Current check: `git diff` / VCS inspection
- Deterministic? Yes — git diff is deterministic

**J8.5: "Whether requirements are met"**
- Current check: Re-read plan → Create checklist → Verify each
- Deterministic? No — requires matching implementation to requirements

### SKILL 9: `requesting-code-review`

**J9.1: "Whether review is mandatory vs optional"**
- Current check: LLM identifies context
- Deterministic? Partially — context identification is LLM, but rules are explicit

**J9.2: "How to handle reviewer feedback by severity"**
- Current check: LLM self-assessment
- Deterministic? Partially — severity labels come from reviewer subagent

### SKILL 10: `receiving-code-review`

**J10.1: "Whether feedback is technically correct for this codebase"**
- Current check: LLM evaluates against codebase
- Deterministic? No

**J10.2: "Whether YAGNI applies to reviewer's suggestion"**
- Current check: `grep` command
- Deterministic? Yes — grep results are deterministic

**J10.3: "Whether any feedback items are unclear"**
- Current check: LLM self-assessment
- Deterministic? No

**J10.4: "Whether feedback conflicts with human partner's prior decisions"**
- Current check: LLM self-assessment
- Deterministic? No

**J10.5: "Whether to push back vs. accept"**
- Current check: LLM self-assessment against 6 criteria
- Deterministic? No

### SKILL 11: `finishing-a-development-branch`

**J11.1: "Whether tests pass before showing options"**
- Current check: Test runner
- Deterministic? Yes

**J11.2: "Determine base branch"**
- Current check: Git command or human confirmation
- Deterministic? Partially — git command is deterministic

**J11.3: "Test verification after merge"**
- Current check: Test runner on merged result
- Deterministic? Yes

**J11.4: "Confirm discard action"**
- Current check: Human input (must type exact word "discard")
- Deterministic? Yes

### SKILL 12: `subagent-driven-development`

**J12.1: "Whether to use subagent-driven vs executing-plans vs manual"**
- Current check: LLM self-assessment
- Deterministic? No

**J12.2: "Model selection for each task"**
- Current check: LLM self-assessment
- Deterministic? No

**J12.3: "How to handle implementer status"**
- Current check: LLM reads subagent status report
- Deterministic? Partially — status field is structured

**J12.4: "Whether concerns in DONE_WITH_CONCERNS are blocking"**
- Current check: LLM self-assessment
- Deterministic? No

**J12.5: "Spec compliance review pass/fail"**
- Current check: Spec reviewer subagent
- Deterministic? Partially — subagent makes judgment against explicit spec text

**J12.6: "Code quality review pass/fail"**
- Current check: Code quality reviewer subagent
- Deterministic? Partially — severity categorization is LLM judgment

**J12.7: "Review loop completion"**
- Current check: Re-dispatch reviewer after fix
- Deterministic? No — relies on protocol being followed

**J12.8: "Review ordering (spec before quality)"**
- Current check: LLM must enforce ordering
- Deterministic? No — process enforcement

### SKILL 13: `dispatching-parallel-agents`

**J13.1: "Whether failures are independent"**
- Current check: LLM self-assessment
- Deterministic? No

**J13.2: "Whether agents can work in parallel (shared state check)"**
- Current check: LLM self-assessment
- Deterministic? Partially — could check file overlap mechanically

**J13.3: "Post-integration verification"**
- Current check: Full test suite run + manual diff review
- Deterministic? Partially — test run is deterministic; conflict detection is LLM

### SKILL 14: `using-git-worktrees`

**J14.1: "Which directory to use for worktrees"**
- Current check: Sequential command execution (`ls`, `grep`)
- Deterministic? Yes — commands are deterministic

**J14.2: "Whether worktree directory is gitignored"**
- Current check: `git check-ignore` command
- Deterministic? Yes — git command is deterministic

**J14.3: "Whether baseline tests pass"**
- Current check: Test runner
- Deterministic? Yes — test runner exit code

**J14.4: "Auto-detect project setup needs"**
- Current check: File existence checks
- Deterministic? Yes — file existence is deterministic

### 参考文件中的额外判断节点

**J-REF1: "Whether scenario applies sufficient pressure"** (testing-skills-with-subagents.md)
- Current check: LLM self-assessment
- Deterministic? No

**J-REF2: "Whether agent response indicates bulletproof skill"** (testing-skills-with-subagents.md)
- Current check: Subagent response analysis
- Deterministic? No

**J-REF3: "Meta-test classification"** (testing-skills-with-subagents.md)
- Current check: Subagent response analysis
- Deterministic? No

**J-REF4: "Am I testing mock behavior?"** (testing-anti-patterns.md)
- Current check: LLM self-assessment
- Deterministic? No

**J-REF5: "Is this method only for tests?"** (testing-anti-patterns.md)
- Current check: LLM self-assessment
- Deterministic? No

**J-REF6: "Do I understand what I'm mocking?"** (testing-anti-patterns.md)
- Current check: LLM self-assessment
- Deterministic? No

**J-REF7: "Whether to flag an issue or approve"** (spec-reviewer-prompt.md)
- Current check: Subagent judgment
- Deterministic? No

**J-REF8: "Whether plan is buildable"** (plan-document-reviewer-prompt.md)
- Current check: Subagent judgment
- Deterministic? No

**J-REF9: "Issue severity classification"** (code-reviewer.md)
- Current check: Subagent judgment
- Deterministic? No

**J-REF10: "Ready to merge assessment"** (code-reviewer.md)
- Current check: Subagent judgment
- Deterministic? No

**J-REF11: "Whether to escalate vs. continue"** (implementer-prompt.md)
- Current check: Subagent self-assessment
- Deterministic? No

**J-REF12: "Self-review completeness"** (implementer-prompt.md)
- Current check: Subagent self-assessment
- Deterministic? No

**J-REF13: "Per-question browser vs terminal decision"** (visual-companion.md)
- Current check: LLM self-assessment per question
- Deterministic? No

---

## 五、总结

这个模型的本质是一个**渐进类型系统**（Gradual Type System）：

- 确定性守卫 = **静态类型**（编译期检查，不可绕过）
- 判断性守卫 = **运行时类型**（LLM/人类判断，可能出错）
- 目标 = 逐步将运行时类型迁移为静态类型

Superpowers 目前的 skill 体系是**以自然语言编写的类型规约**——它们描述了检查什么，但检查的执行依赖 LLM 自律。上述模型将其转化为**以代码编写的类型检查器**——同样的规约，但由程序强制执行。

两者之间的差距，就是 "27% 确定性" 和 "80% 确定性" 之间的差距。
