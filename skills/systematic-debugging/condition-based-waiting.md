# 基于条件的等待

## 概述

不稳定的测试通常通过任意延时来猜测时机。这会导致竞态条件——测试在快速机器上通过，但在高负载或 CI 环境下失败。

**核心原则：** 等待你真正关心的条件，而不是猜测需要多长时间。

## 何时使用

```dot
digraph when_to_use {
    "Test uses setTimeout/sleep?" [shape=diamond];
    "Testing timing behavior?" [shape=diamond];
    "Document WHY timeout needed" [shape=box];
    "Use condition-based waiting" [shape=box];

    "Test uses setTimeout/sleep?" -> "Testing timing behavior?" [label="yes"];
    "Testing timing behavior?" -> "Document WHY timeout needed" [label="yes"];
    "Testing timing behavior?" -> "Use condition-based waiting" [label="no"];
}
```

**适用场景：**
- 测试包含任意延时（`setTimeout`、`sleep`、`time.sleep()`）
- 测试不稳定（有时通过，高负载时失败）
- 测试在并行运行时超时
- 等待异步操作完成

**不适用场景：**
- 测试实际的定时行为（防抖、节流间隔）
- 如果使用了任意超时，必须记录原因

## 核心模式

```typescript
// ❌ 之前：猜测时机
await new Promise(r => setTimeout(r, 50));
const result = getResult();
expect(result).toBeDefined();

// ✅ 之后：等待条件
await waitFor(() => getResult() !== undefined);
const result = getResult();
expect(result).toBeDefined();
```

## 快速模式

| 场景 | 模式 |
|------|------|
| 等待事件 | `waitFor(() => events.find(e => e.type === 'DONE'))` |
| 等待状态 | `waitFor(() => machine.state === 'ready')` |
| 等待数量 | `waitFor(() => items.length >= 5)` |
| 等待文件 | `waitFor(() => fs.existsSync(path))` |
| 复合条件 | `waitFor(() => obj.ready && obj.value > 10)` |

## 实现

通用轮询函数：
```typescript
async function waitFor<T>(
  condition: () => T | undefined | null | false,
  description: string,
  timeoutMs = 5000
): Promise<T> {
  const startTime = Date.now();

  while (true) {
    const result = condition();
    if (result) return result;

    if (Date.now() - startTime > timeoutMs) {
      throw new Error(`Timeout waiting for ${description} after ${timeoutMs}ms`);
    }

    await new Promise(r => setTimeout(r, 10)); // Poll every 10ms
  }
}
```

完整实现及领域特定辅助函数（`waitForEvent`、`waitForEventCount`、`waitForEventMatch`）来自实际调试会话，请参见本目录下的 `condition-based-waiting-example.ts`。

## 常见错误

**❌ 轮询过快：** `setTimeout(check, 1)` — 浪费 CPU
**✅ 修正：** 每 10ms 轮询一次

**❌ 没有超时：** 条件未满足时无限循环
**✅ 修正：** 始终包含超时并附带清晰的错误信息

**❌ 过时数据：** 在循环前缓存状态
**✅ 修正：** 在循环内调用 getter 获取最新数据

## 何时任意超时是正确的

```typescript
// 工具每 100ms 触发一次 - 需要 2 次触发来验证部分输出
await waitForEvent(manager, 'TOOL_STARTED'); // 首先：等待触发条件
await new Promise(r => setTimeout(r, 200));   // 然后：等待定时行为
// 200ms = 以 100ms 间隔触发 2 次 - 有记录且有依据
```

**要求：**
1. 首先等待触发条件
2. 基于已知时序（而非猜测）
3. 注释说明原因

## 实际影响

来自调试会话（2025-10-03）：
- 修复了 3 个文件中的 15 个不稳定测试
- 通过率：60% → 100%
- 执行时间：加快 40%
- 不再出现竞态条件