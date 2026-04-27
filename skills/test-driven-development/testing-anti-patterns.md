# 测试反模式

**在以下场景加载此参考：** 编写或修改测试、添加 mock，或 tempted 在生产代码中添加仅供测试使用的方法时。

## 概述

测试必须验证真实行为，而非 mock 行为。Mock 是隔离的手段，不是被测试的对象。

**核心原则：** 测试代码做了什么，而不是 mock 做了什么。

**严格遵循 TDD 可以避免这些反模式。**

## 铁律

```
1. 永远不要测试 mock 行为
2. 永远不要在生产类中添加仅供测试使用的方法
3. 永远不要在不理解依赖的情况下使用 mock
```

## 反模式 1：测试 Mock 行为

**违规示例：**
```typescript
// ❌ 错误：测试 mock 是否存在
test('renders sidebar', () => {
  render(<Page />);
  expect(screen.getByTestId('sidebar-mock')).toBeInTheDocument();
});
```

**为什么这是错的：**
- 你在验证 mock 能工作，而不是组件能工作
- mock 存在时测试通过，不存在时测试失败
- 对真实行为没有任何说明作用

**你的 human partner 的纠正：** "我们在测试 mock 的行为吗？"

**修复方式：**
```typescript
// ✅ 正确：测试真实组件，或者不要 mock 它
test('renders sidebar', () => {
  render(<Page />);  // 不要 mock sidebar
  expect(screen.getByRole('navigation')).toBeInTheDocument();
});

// 或者如果 sidebar 必须被 mock 以实现隔离：
// 不要对 mock 做断言 - 测试 Page 在 sidebar 存在时的行为
```

### 门控函数

```
在对任何 mock 元素做断言之前：
  问："我在测试真实组件的行为，还是仅仅在测试 mock 的存在？"

  如果是测试 mock 的存在：
    停止 - 删除断言或取消 mock 该组件

  转而测试真实行为
```

## 反模式 2：生产代码中的仅供测试方法

**违规示例：**
```typescript
// ❌ 错误：destroy() 仅在测试中使用
class Session {
  async destroy() {  // 看起来像生产 API！
    await this._workspaceManager?.destroyWorkspace(this.id);
    // ... cleanup
  }
}

// 在测试中
afterEach(() => session.destroy());
```

**为什么这是错的：**
- 生产类被仅供测试的代码污染
- 如果在生产环境中被意外调用会很危险
- 违反 YAGNI 原则和关注点分离
- 混淆了对象生命周期与实体生命周期

**修复方式：**
```typescript
// ✅ 正确：测试工具负责测试清理
// Session 没有 destroy() - 它在生产中是无状态的

// 在 test-utils/ 中
export async function cleanupSession(session: Session) {
  const workspace = session.getWorkspaceInfo();
  if (workspace) {
    await workspaceManager.destroyWorkspace(workspace.id);
  }
}

// 在测试中
afterEach(() => cleanupSession(session));
```

### 门控函数

```
在生产类中添加任何方法之前：
  问："这个方法是否只在测试中使用？"

  如果是：
    停止 - 不要添加它
    把它放到测试工具中

  问："这个类是否拥有此资源的生命周期？"

  如果不是：
    停止 - 这个方法不属于这个类
```

## 反模式 3：不理解就 Mock

**违规示例：**
```typescript
// ❌ 错误：Mock 破坏了测试逻辑
test('detects duplicate server', () => {
  // Mock 阻止了测试所依赖的 config 写入！
  vi.mock('ToolCatalog', () => ({
    discoverAndCacheTools: vi.fn().mockResolvedValue(undefined)
  }));

  await addServer(config);
  await addServer(config);  // 应该抛出异常 - 但不会！
});
```

**为什么这是错的：**
- 被 mock 的方法有测试依赖的副作用（写入 config）
- 过度 mock 以"安全起见"破坏了实际行为
- 测试因错误的原因通过，或莫名其妙地失败

**修复方式：**
```typescript
// ✅ 正确：在正确的层级进行 mock
test('detects duplicate server', () => {
  // 只 mock 慢的部分，保留测试需要的行为
  vi.mock('MCPServerManager'); // 只 mock 慢的服务器启动

  await addServer(config);  // Config 已写入
  await addServer(config);  // 检测到重复 ✓
});
```

### 门控函数

```
在 mock 任何方法之前：
  停止 - 先不要 mock

  1. 问："真实方法有什么副作用？"
  2. 问："这个测试是否依赖其中任何副作用？"
  3. 问："我是否完全理解这个测试需要什么？"

  如果依赖副作用：
    在更低的层级进行 mock（实际的慢速/外部操作）
    或使用保留必要行为的 test doubles
    而非测试所依赖的高层方法

  如果不确定测试依赖什么：
    先用真实实现运行测试
    观察实际需要发生什么
    然后在正确的层级添加最小化的 mock

  红旗：
    - "为了安全起见我 mock 一下"
    - "这可能很慢，还是 mock 吧"
    - 在不理解依赖链的情况下 mock
```

## 反模式 4：不完整的 Mock

**违规示例：**
```typescript
// ❌ 错误：部分 mock - 只有你认为需要的字段
const mockResponse = {
  status: 'success',
  data: { userId: '123', name: 'Alice' }
  // 缺少：下游代码使用的 metadata
};

// 后续：当代码访问 response.metadata.requestId 时会出错
```

**为什么这是错的：**
- **部分 mock 隐藏了结构性假设** - 你只 mock 了你知道的字段
- **下游代码可能依赖你未包含的字段** - 静默失败
- **测试通过但集成失败** - Mock 不完整，真实 API 完整
- **虚假的信心** - 测试对真实行为毫无证明作用

**铁律：** Mock 完整的数据结构，与现实存在的完全一致，而不仅仅是当前测试使用的字段。

**修复方式：**
```typescript
// ✅ 正确：镜像真实 API 的完整性
const mockResponse = {
  status: 'success',
  data: { userId: '123', name: 'Alice' },
  metadata: { requestId: 'req-789', timestamp: 1234567890 }
  // 真实 API 返回的所有字段
};
```

### 门控函数

```
在创建 mock 响应之前：
  检查："真实 API 响应包含哪些字段？"

  操作：
    1. 从文档/示例中检查实际的 API 响应
    2. 包含系统可能在下游消费的所有字段
    3. 验证 mock 完全匹配真实响应的 schema

  关键：
    如果你创建 mock，你必须理解整个结构
    当代码依赖被省略的字段时，部分 mock 会静默失败

  如果不确定：包含所有有文档记录的字段
```

## 反模式 5：集成测试作为事后补充

**违规示例：**
```
✅ 实现已完成
❌ 没有编写测试
"准备好测试了"
```

**为什么这是错的：**
- 测试是实现的一部分，不是可选的后续步骤
- TDD 本可以捕获这些问题
- 没有测试就不能声称完成

**修复方式：**
```
TDD 循环：
1. 编写失败的测试
2. 实现使其通过
3. 重构
4. 然后再声称完成
```

## 当 Mock 变得过于复杂

**警告信号：**
- Mock 设置比测试逻辑还长
- 为了让测试通过而 mock 一切
- Mock 缺少真实组件拥有的方法
- 当 mock 变化时测试就失败

**你的 human partner 的问题：** "我们这里需要使用 mock 吗？"

**考虑：** 使用真实组件的集成测试通常比复杂的 mock 更简单

## TDD 防止这些反模式

**为什么 TDD 有帮助：**
1. **先写测试** → 迫使你思考你实际在测试什么
2. **看着它失败** → 确认测试的是真实行为，而非 mock
3. **最小化实现** → 不会悄悄混入仅供测试的方法
4. **真实依赖** → 在 mock 之前你就能看到测试实际需要什么

**如果你在测试 mock 行为，说明你违反了 TDD** - 你在没有先看着测试对真实代码失败的情况下就添加了 mock。

## 快速参考

| 反模式 | 修复方式 |
|--------|----------|
| 对 mock 元素做断言 | 测试真实组件或取消 mock |
| 生产代码中的仅供测试方法 | 移到测试工具中 |
| 不理解就 mock | 先理解依赖，最小化 mock |
| 不完整的 mock | 完整镜像真实 API |
| 测试作为事后补充 | TDD - 测试先行 |
| 过度复杂的 mock | 考虑集成测试 |

## 红旗

- 断言检查 `*-mock` 测试 ID
- 只在测试文件中被调用的方法
- Mock 设置占测试的 >50%
- 移除 mock 后测试就失败
- 无法解释为什么需要 mock
- "为了安全起见"而 mock

## 总结

**Mock 是隔离的工具，不是被测试的对象。**

如果 TDD 揭示你在测试 mock 行为，那你走偏了。

修复：测试真实行为，或质疑你为什么需要 mock。
