# 基于条件的等待

## 概述

不稳定的测试通常通过随意的延迟来猜测时序。这会产生竞态条件——在快速的机器上测试通过，但在负载较高或 CI 中失败。

**核心原则：** 等待你真正关心的条件，而不是猜测它需要多长时间。

## 何时使用

```dot
digraph when_to_use {
    "测试使用了 setTimeout/sleep？" [shape=diamond];
    "是在测试时序行为？" [shape=diamond];
    "记录为什么需要超时" [shape=box];
    "使用基于条件的等待" [shape=box];

    "测试使用了 setTimeout/sleep？" -> "是在测试时序行为？" [label="是"];
    "是在测试时序行为？" -> "记录为什么需要超时" [label="是"];
    "是在测试时序行为？" -> "使用基于条件的等待" [label="否"];
}
```

**使用场景：**
- 测试中有随意的延迟（`setTimeout`、`sleep`、`time.sleep()`）
- 测试不稳定（有时通过，负载高时失败）
- 并行运行测试时超时
- 等待异步操作完成

**不适用场景：**
- 测试真正的时序行为（防抖、节流间隔）
- 如果使用随意超时，务必记录原因

## 核心模式

```typescript
// ❌ 之前：猜测时序
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
|----------|---------|
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
      throw new Error(`等待 ${description} 超时，已超过 ${timeoutMs}ms`);
    }

    await new Promise(r => setTimeout(r, 10)); // 每 10ms 轮询一次
  }
}
```

参见本目录下的 `condition-based-waiting-example.ts` 获取完整实现，包括领域特定的辅助函数（`waitForEvent`、`waitForEventCount`、`waitForEventMatch`），来自实际调试过程。

## 常见错误

**❌ 轮询太快：** `setTimeout(check, 1)` —— 浪费 CPU
**✅ 修复：** 每 10ms 轮询一次

**❌ 没有超时：** 条件永远不满足时无限循环
**✅ 修复：** 始终包含超时并给出清晰的错误信息

**❌ 使用过期数据：** 在循环前缓存了状态
**✅ 修复：** 在循环内调用 getter 获取最新数据

## 什么时候使用随意超时是正确的

```typescript
// 工具每 100ms tick 一次——需要 2 次 tick 来验证部分输出
await waitForEvent(manager, 'TOOL_STARTED'); // 首先：等待条件
await new Promise(r => setTimeout(r, 200));   // 然后：等待定时行为
// 200ms = 100ms 间隔的 2 次 tick —— 有文档记录且有依据
```

**要求：**
1. 先等待触发条件
2. 基于已知的时序（不是猜测）
3. 注释说明原因

## 实际效果

来自调试实践（2025-10-03）的数据：
- 修复了 3 个文件中的 15 个不稳定测试
- 通过率：60% → 100%
- 执行时间：快了 40%
- 不再有竞态条件
