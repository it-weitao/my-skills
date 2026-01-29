---
name: testable-state
description: 使用 Reducer 模式编写可测试的状态管理代码。确保所有状态变化都通过单一纯函数入口，提高可测试性和可维护性。
---

# 可测试的状态管理

本 Skill 确保 Hook/组件代码可测试、可预测、可维护，强制执行 Reducer 模式。

## 核心原则

### 1. Reducer 模式（必须）

所有状态变化必须通过单一纯函数入口：

```typescript
// 定义 Action 类型
type Action =
  | { type: 'ACTION_A'; payload: ... }
  | { type: 'ACTION_B'; ... }
  | { type: 'ACTION_C'; ... };

// 核心 reducer：纯函数，输入 state + action → 输出 new state
const reducer = (state: State, action: Action, context: Context): State => {
  switch (action.type) {
    case 'ACTION_A':
      // 完整的状态流转逻辑
      const step1 = processStep1(state, action.payload, context);
      const step2 = processStep2(step1, context);
      return step2;
    case 'ACTION_B':
      return processActionB(state, action.payload, context);
    default:
      return state;
  }
};
```

### 2. 纯函数优先

- 将业务逻辑提取为 Hook/Component 外部的纯函数
- 纯函数**禁止**：
  - 接收 refs
  - 访问外部状态
  - 产生副作用
- 纯函数**必须**：
  - 可独立测试
  - 返回新对象（不可变性）
  - 使用 Readonly 类型

### 3. 分层架构

| 层级 | 类型 | 说明 |
|------|------|------|
| **纯函数** | 纯函数 | 参数计算、数据转换 |
| **Reducer (dispatch)** | 纯函数 | 状态流转：`(state, action) => newState` |
| **业务流程** | 纯函数 + 注入的副作用 | 编排逻辑，接收 `[state, dispatch]` + `deps` |
| **Hook** | 组合层 | 协调各层，管理生命周期 |

### 4. 副作用依赖注入

业务流程函数通过 `deps` 对象接收副作用（不硬编码）：

```typescript
/**
 * 业务流程函数
 * @param stateWithDispatch - 状态管理层（元组）
 * @param deps - 注入的副作用依赖
 */
const fetchBusinessFlow = async (
  [state, dispatch]: [State, (action: Action) => void],
  deps: {
    fetch: (url: string) => Promise<Data>;
    // 未来可扩展更多依赖
    // logger: (msg: string) => void;
    // tracker: (event: string) => void;
  }
): Promise<void> => {
  // 纯函数：计算参数
  const params = calculateParams(state);

  // 副作用：通过 deps 调用
  const result = await deps.fetch(params);

  // 纯函数：处理结果
  const newState = processResult(result, state);

  // Dispatch：纯函数
  dispatch({ type: 'SUCCESS', payload: newState });
};
```

### 5. Hook/组件 职责

- 只负责协调
- Hook/Component 内不包含业务逻辑
- 向业务流程注入 `deps` 对象
- 副作用集中在 `useEffect`

### 6. 显式的状态流转

- 状态变化路径可追踪
- 避免在多处直接调用 `setState`
- 中间状态可观测

## 代码模板

```typescript
// ==================== 类型定义 ====================

type State = Readonly<{ ... }>;
type Action = { type: '...' } | { type: '...' };
type Context = Readonly<{ ... }>;

// ==================== 纯函数 ====================

const calculateParams = (state: State): Params => { ... };
const processResult = (result: Result, state: State): Partial<State> => { ... };

// ==================== Reducer ====================

const reducer = (state: State, action: Action): State => {
  switch (action.type) {
    case 'SUCCESS':
      return { ...state, ...action.payload };
    default:
      return state;
  }
};

// ==================== 业务流程（注入副作用）====================

const fetchBusinessFlow = async (
  [state, dispatch]: [State, (action: Action) => void],
  deps: {
    fetch: (params: Params) => Promise<Result>;
    logger?: (msg: string) => void;
  }
): Promise<void> => {
  const params = calculateParams(state);
  const result = await deps.fetch(params);
  const newState = processResult(result, state);
  dispatch({ type: 'SUCCESS', payload: newState });
};

// ==================== Hook ====================

export const useFeature = (options: Options): Return => {
  const [state, dispatch] = useReducer(reducer, initialState);

  // 依赖对象
  const deps = useMemo(() => ({
    fetch: fetchService.fetch,
    logger: loggerService.log,
  }), []);

  useEffect(() => {
    fetchBusinessFlow([state, dispatch], deps);
  }, [state.id, deps]);

  return { state };
};
```

## 检查清单

生成/接受状态管理代码前确认：

- [ ] 所有状态变化是否通过单一 reducer 函数？
- [ ] reducer 是否为纯函数（相同输入 → 相同输出）？
- [ ] 业务逻辑是否提取到 Hook/Component 外部？
- [ ] 副作用是否通过 `deps` 对象注入（不硬编码）？
- [ ] 业务流程是否接收 `[state, dispatch]` 元组？
- [ ] 业务流程是否可通过注入 mock `deps` 进行测试？
- [ ] 状态类型是否使用 Readonly 确保不可变性？
- [ ] 纯函数是否不接收 refs、不访问外部状态、无副作用？

## 反模式（避免）

```typescript
// ❌ 错误：多个 setState 入口点
const handleActionA = () => {
  const result = calculateSomething();
  setPosition(result);
};

const handleActionB = () => {
  setPosition({ x: 0, y: 0 });
};

// ❌ 错误：Hook 内包含业务逻辑，硬编码副作用
const handleAction = useCallback(() => {
  const step1 = complexCalculation(state);  // 无法独立测试
  const step2 = anotherCalculation(step1);
  setState(step2);
}, [state]);

// ❌ 错误：业务流程中硬编码副作用
const businessFlow = async (
  [state, dispatch]: [State, Dispatch]
) => {
  const data = await fetch('/api/data');  // 硬编码！
  dispatch({ type: 'SUCCESS', payload: data });
};

// ✅ 正确：业务流程注入副作用
const businessFlow = async (
  [state, dispatch]: [State, Dispatch],
  deps: { fetch: (url: string) => Promise<Data> }
) => {
  const data = await deps.fetch('/api/data');  // 注入的
  dispatch({ type: 'SUCCESS', payload: data });
};
```

## 测试优势

使用此模式，可以通过注入 `mock deps` 测试业务流程：

```typescript
describe('fetchBusinessFlow', () => {
  it('应该正确 fetch 并 dispatch', async () => {
    const mockState = { id: 1 };
    const mockDispatch = vi.fn();
    const mockDeps = {
      fetch: vi.fn().mockResolvedValue({ data: 'test' }),
    };

    await fetchBusinessFlow([mockState, mockDispatch], mockDeps);

    expect(mockDeps.fetch).toHaveBeenCalledWith(expectedParams);
    expect(mockDispatch).toHaveBeenCalledWith({ type: 'SUCCESS', payload: expectedState });
  });
});
```

无需 React Testing Library——只需测试纯函数并注入 mock。
