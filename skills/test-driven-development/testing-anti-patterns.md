# 测试反模式

**在以下情况加载此参考：** 编写或修改测试、添加 mock、或想在生产代码中添加仅供测试使用的方法时。

## 概述

测试必须验证真实行为，而非 mock 行为。Mock 是隔离的手段，不是被测试的对象。

**核心原则：** 测试代码做了什么，而不是 mock 做了什么。

**严格遵循 TDD 可以预防这些反模式。**

## 铁律

```
1. 绝不测试 mock 行为
2. 绝不在生产类中添加仅供测试使用的方法
3. 绝不在不理解依赖关系的情况下使用 mock
```

## 反模式 1：测试 Mock 行为

**错误做法：**
```typescript
// ❌ 坏的：测试 mock 是否存在
test('renders sidebar', () => {
  render(<Page />);
  expect(screen.getByTestId('sidebar-mock')).toBeInTheDocument();
});
```

**为什么这是错误的：**
- 你在验证 mock 是否工作，而不是组件是否工作
- 有 mock 就通过，没有就失败
- 对真实行为毫无意义

**你的用户的纠正：** "我们是在测试 mock 的行为吗？"

**正确做法：**
```typescript
// ✅ 好的：测试真实组件或不使用 mock
test('renders sidebar', () => {
  render(<Page />);  // 不要 mock sidebar
  expect(screen.getByRole('navigation')).toBeInTheDocument();
});

// 或者如果必须 mock sidebar 来做隔离：
// 不要对 mock 做断言——测试 Page 在有 sidebar 存在时的行为
```

### 门控函数

```
在对任何 mock 元素做断言之前：
  问自己："我是在测试真实组件行为，还是仅仅在测试 mock 是否存在？"

  如果是测试 mock 是否存在：
    STOP - 删除该断言或取消 mock

  改为测试真实行为
```

## 反模式 2：在生产代码中添加仅供测试使用的方法

**错误做法：**
```typescript
// ❌ 坏的：destroy() 只在测试中使用
class Session {
  async destroy() {  // 看起来像生产 API！
    await this._workspaceManager?.destroyWorkspace(this.id);
    // ... 清理
  }
}

// 在测试中
afterEach(() => session.destroy());
```

**为什么这是错误的：**
- 生产类被仅供测试使用的代码污染
- 如果在生产环境中被误调用会很危险
- 违反 YAGNI 和关注点分离原则
- 混淆了对象生命周期和实体生命周期

**正确做法：**
```typescript
// ✅ 好的：测试工具处理测试清理
// Session 没有 destroy() —— 它在生产环境中是无状态的

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
在往生产类中添加任何方法之前：
  问自己："这个方法是否只有测试在使用？"

  如果是：
    STOP - 不要添加它
    把它放到测试工具中

  问自己："这个类是否拥有该资源的生命周期？"

  如果没有：
    STOP - 这个方法不应该放在这个类中
```

## 反模式 3：不理解依赖关系就使用 Mock

**错误做法：**
```typescript
// ❌ 坏的：Mock 破坏了测试逻辑
test('detects duplicate server', () => {
  // Mock 阻止了测试所依赖的配置写入！
  vi.mock('ToolCatalog', () => ({
    discoverAndCacheTools: vi.fn().mockResolvedValue(undefined)
  }));

  await addServer(config);
  await addServer(config);  // 应该抛出异常——但不会了！
});
```

**为什么这是错误的：**
- 被 mock 的方法有测试依赖的副作用（写入配置）
- 为了"安全"而过度 mock 破坏了实际行为
- 测试因错误的原因通过，或莫名其妙地失败

**正确做法：**
```typescript
// ✅ 好的：在正确的层级做 Mock
test('detects duplicate server', () => {
  // Mock 慢的部分，保留测试需要的行为
  vi.mock('MCPServerManager'); // 只 mock 慢的服务器启动

  await addServer(config);  // 配置写入
  await addServer(config);  // 检测到重复 ✓
});
```

### 门控函数

```
在 mock 任何方法之前：
  STOP - 先不要 mock

  1. 问自己："真实方法有什么副作用？"
  2. 问自己："这个测试是否依赖其中某些副作用？"
  3. 问自己："我是否完全理解这个测试需要什么？"

  如果依赖副作用：
    在更低的层级做 mock（实际慢的/外部的操作）
    或使用保留必要行为的测试替身
    不要 mock 测试依赖的高层方法

  如果不确定测试依赖什么：
    先用真实实现运行测试
    观察实际需要发生什么
    然后在正确的层级添加最少的 mock

  危险信号：
    - "我 mock 一下以防万一"
    - "这个可能会慢，最好 mock 掉"
    - 在不理解依赖链的情况下做 mock
```

## 反模式 4：不完整的 Mock

**错误做法：**
```typescript
// ❌ 坏的：部分 mock —— 只模拟了你以为需要的字段
const mockResponse = {
  status: 'success',
  data: { userId: '123', name: 'Alice' }
  // 缺少：下游代码使用的 metadata
};

// 后来：代码访问 response.metadata.requestId 时出错
```

**为什么这是错误的：**
- **部分 mock 隐藏了结构假设** —— 你只 mock 了你知道的字段
- **下游代码可能依赖你没有包含的字段** —— 静默失败
- **测试通过但集成失败** —— mock 不完整，真实 API 完整
- **虚假的信心** —— 测试对真实行为什么也没证明

**铁律：** Mock 现实中的完整数据结构，而不是只 mock 你当前测试用到的字段。

**正确做法：**
```typescript
// ✅ 好的：与真实 API 保持一致的完整性
const mockResponse = {
  status: 'success',
  data: { userId: '123', name: 'Alice' },
  metadata: { requestId: 'req-789', timestamp: 1234567890 }
  // 包含真实 API 返回的所有字段
};
```

### 门控函数

```
在创建 mock 响应之前：
  检查："真实 API 响应包含哪些字段？"

  操作：
    1. 从文档/示例中查看实际 API 响应
    2. 包含系统下游可能消费的所有字段
    3. 验证 mock 完全匹配真实响应的 schema

  关键：
    如果你在创建 mock，你必须理解完整的结构
    部分 mock 在代码依赖被省略的字段时会静默失败

  如果不确定：包含所有文档记录的字段
```

## 反模式 5：把集成测试当事后补充

**错误做法：**
```
✅ 实现完成
❌ 没有编写测试
"准备好测试了"
```

**为什么这是错误的：**
- 测试是实现的一部分，不是可选的后续步骤
- TDD 本来就能发现这个问题
- 没有测试就不能声称完成

**正确做法：**
```
TDD 循环：
1. 编写失败的测试
2. 实现使其通过
3. 重构
4. 然后才能声称完成
```

## 当 Mock 变得过于复杂时

**警告信号：**
- Mock 设置比测试逻辑还长
- 为了让测试通过而 mock 了所有东西
- Mock 缺少真实组件拥有的方法
- 修改 mock 就会破坏测试

**你的用户的提问：** "我们真的需要在这里使用 mock 吗？"

**考虑：** 使用真实组件的集成测试通常比复杂的 mock 更简单

## TDD 如何预防这些反模式

**TDD 为什么有帮助：**
1. **先写测试** → 迫使你思考你到底在测试什么
2. **看它失败** → 确认测试测的是真实行为而非 mock
3. **最小实现** → 不会悄悄混入仅供测试使用的方法
4. **真实依赖** → 你在 mock 之前就看到测试实际需要什么

**如果你在测试 mock 行为，说明你违反了 TDD** —— 你在没有先让测试对真实代码失败的情况下就添加了 mock。

## 快速参考

| 反模式 | 修复方法 |
|--------------|-----|
| 对 mock 元素做断言 | 测试真实组件或取消 mock |
| 在生产代码中添加仅供测试使用的方法 | 移到测试工具中 |
| 不理解依赖就做 mock | 先理解依赖关系，最小化 mock |
| 不完整的 mock | 与真实 API 完全保持一致 |
| 测试作为事后补充 | TDD —— 测试先行 |
| 过于复杂的 mock | 考虑使用集成测试 |

## 危险信号

- 断言中检查 `*-mock` test ID
- 某些方法只在测试文件中被调用
- Mock 设置超过测试的 50%
- 移除 mock 后测试就失败
- 无法解释为什么需要 mock
- "mock 一下以防万一"

## 底线

**Mock 是隔离的工具，不是被测试的对象。**

如果 TDD 揭示你在测试 mock 行为，说明你走偏了。

修复方法：测试真实行为，或质疑你为什么需要 mock。
