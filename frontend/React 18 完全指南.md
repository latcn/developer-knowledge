
> **从第一性原理出发，涵盖原理、Hooks、高级特性、性能优化、状态管理、路由、数据获取、测试及 Next.js 全栈实践。**  
> 所有代码示例均可直接运行，适合作为开发者案头参考。

---
## 第一部分：React 的本质与核心原理

### 1.1 第一性原理：UI = f(state)

React 最核心的公式：**用户界面是状态的函数**。  
开发者只需描述“UI 应该长什么样”，React 负责高效地将状态映射为 DOM。

### 1.2 声明式 vs 命令式

- **命令式**：直接操作 DOM，如 `document.getElementById('btn').onclick = ...`。
- **声明式**：描述 UI 状态，如 `<button onClick={...}>`，React 处理渲染细节。

### 1.3 React 元素与虚拟 DOM

- **React 元素**：是一个普通的 JavaScript 对象，描述 DOM 节点（或组件）的“蓝图”。
- **虚拟 DOM**：在内存中维护一棵与真实 DOM 对应的树，每次状态变化生成新树，通过 Diff 算法找出最小变更集，批量更新真实 DOM。

### 1.4 协调（Reconciliation）与 Diff 算法

- **Diff 假设**：
  1. 不同类型的元素会产生不同的树。
  2. 同级元素可以通过 `key` 属性稳定标识。
- **Diff 过程**：
  - 根节点类型不同 → 卸载旧树，创建新树。
  - 相同类型 DOM 元素 → 保留 DOM 节点，仅更新属性。
  - 相同类型组件 → 更新组件实例的 props，触发重新渲染。
  - 列表通过 `key` 进行移动/增删。
- **Fiber 架构（React 16+）**：将渲染过程拆分为可中断的工作单元，支持优先级调度和并发渲染。

---

## 第二部分：组件基础

### 2.1 函数组件（推荐）

```jsx
function Greeting({ name }) {
  return <h1>Hello, {name}!</h1>;
}
```

### 2.2 类组件（仅用于 Error Boundaries）

```jsx
class ErrorBoundary extends React.Component {
  state = { hasError: false };
  static getDerivedStateFromError(error) {
    return { hasError: true };
  }
  componentDidCatch(error, info) {
    console.error(error, info);
  }
  render() {
    if (this.state.hasError) return <h2>出错了</h2>;
    return this.props.children;
  }
}
```

### 2.3 Props：只读的数据传递

- 父组件通过属性传递数据给子组件。
- Props 不可变，子组件绝不能修改。

### 2.4 列表与 Key

使用唯一且稳定的 `key`，避免使用数组下标。

```jsx
{todos.map(todo => <TodoItem key={todo.id} todo={todo} />)}
```

---

## 第三部分：Hooks 全面详解

### 3.1 useState —— 内部状态

```jsx
const [count, setCount] = useState(0);
// 函数式更新避免闭包陷阱
setCount(prev => prev + 1);
```

**原理**：每个 Hook 在组件 Fiber 节点上通过链表顺序存储，调用顺序必须稳定。

### 3.2 useEffect —— 副作用管理

```jsx
useEffect(() => {
  // 执行副作用
  const subscription = subscribe();
  return () => { // 清理
    subscription.unsubscribe();
  };
}, [deps]); // 依赖变化时重新执行
```

**执行时机**：布局与绘制之后（异步），适合大部分副作用。

### 3.3 useLayoutEffect —— 同步副作用

```jsx
useLayoutEffect(() => {
  // 在 DOM 变更后、浏览器绘制前同步执行
  const height = ref.current.getBoundingClientRect().height;
}, [deps]);
```

**适用场景**：测量 DOM 尺寸、避免闪烁的动画。

### 3.4 useRef —— 跨渲染的可变数据

- 引用 DOM 元素：`<input ref={myRef} />`
- 存储不触发重绘的值：`ref.current = value`

### 3.5 useMemo —— 缓存计算结果

```jsx
const expensiveValue = useMemo(() => computeExpensive(a, b), [a, b]);
```

### 3.6 useCallback —— 缓存函数引用

```jsx
const handleClick = useCallback(() => doSomething(id), [id]);
```

配合 `React.memo` 避免子组件无效重渲染。

### 3.7 useReducer —— 复杂状态逻辑

```jsx
const [state, dispatch] = useReducer(reducer, initialState);
// reducer 接收当前 state 和 action，返回新 state
```

适用于状态更新涉及多个子值或依赖于先前状态。

### 3.8 useContext —— 跨组件传递数据

```jsx
const ThemeContext = React.createContext('light');
// Provider 提供值，useContext 读取
const theme = useContext(ThemeContext);
```

### 3.9 自定义 Hooks —— 封装逻辑

命名以 `use` 开头，内部可调用其他 Hooks。

```jsx
function useWindowSize() {
  const [size, setSize] = useState({ width: window.innerWidth, height: window.innerHeight });
  useEffect(() => {
    const handleResize = () => setSize({ width: window.innerWidth, height: window.innerHeight });
    window.addEventListener('resize', handleResize);
    return () => window.removeEventListener('resize', handleResize);
  }, []);
  return size;
}
```

---

## 第四部分：高级特性

### 4.1 高阶组件（HOC）

```jsx
function withLogger(WrappedComponent) {
  return function WithLogger(props) {
    useEffect(() => console.log('render'), []);
    return <WrappedComponent {...props} />;
  };
}
```

### 4.2 Render Props

```jsx
<MouseTracker render={({ x, y }) => <div>{x}, {y}</div>} />
```

### 4.3 forwardRef 与 useImperativeHandle

- `forwardRef`：将父组件传入的 `ref` 转发给子组件内的 DOM 元素。
- `useImperativeHandle`：限制暴露的方法。

```jsx
const FancyInput = forwardRef((props, ref) => {
  const inputRef = useRef();
  useImperativeHandle(ref, () => ({
    focus: () => inputRef.current.focus()
  }));
  return <input ref={inputRef} />;
});
```

### 4.4 Portals

```jsx
ReactDOM.createPortal(<div>Modal</div>, document.body);
```

### 4.5 错误边界（Error Boundaries）

仅类组件支持，捕获子组件渲染错误，显示备用 UI。

### 4.6 合成事件（SyntheticEvent）

React 所有事件使用合成事件，统一跨浏览器接口。React 17+ 不再使用事件池。

---

## 第五部分：性能优化

### 5.1 React.memo —— 浅比较 props

```jsx
const MemoizedComponent = React.memo(MyComponent);
```

### 5.2 代码分割与懒加载

```jsx
const LazyComponent = lazy(() => import('./HeavyComponent'));
<Suspense fallback={<Loading />}>
  <LazyComponent />
</Suspense>
```

### 5.3 稳定 Key

### 5.4 避免在渲染中创建新对象/函数（配合 useMemo/useCallback）

### 5.5 虚拟化长列表（react-window, react-virtualized）

---

## 第六部分：状态管理

### 6.1 使用 Context + useReducer 实现轻量级全局状态

```jsx
const AppContext = createContext();
function AppProvider({ children }) {
  const [state, dispatch] = useReducer(reducer, initialState);
  return (
    <AppContext.Provider value={{ state, dispatch }}>
      {children}
    </AppContext.Provider>
  );
}
```

### 6.2 Redux Toolkit（官方推荐）

```jsx
import { configureStore, createSlice } from '@reduxjs/toolkit';
const counterSlice = createSlice({
  name: 'counter',
  initialState: 0,
  reducers: {
    increment: (state) => state + 1,
    decrement: (state) => state - 1,
  },
});
export const store = configureStore({ reducer: { counter: counterSlice.reducer } });
// 在组件中使用 useSelector 和 useDispatch
```

---

## 第七部分：React 18 并发特性

### 7.1 自动批处理

所有 `setState` 在同一个事件循环中合并为一次更新。

### 7.2 useTransition

标记非紧急更新，可被中断，优先响应用户输入。

```jsx
const [isPending, startTransition] = useTransition();
startTransition(() => {
  setFilter(value); // 低优先级更新
});
```

### 7.3 useDeferredValue

延迟某个值的更新，类似防抖但由 React 调度。

```jsx
const deferredSearch = useDeferredValue(searchTerm);
```

### 7.4 Suspense 数据获取（需配合支持 Suspense 的库如 SWR/React Query）

```jsx
<Suspense fallback={<Spinner />}>
  <UserProfile />
</Suspense>
```

---

## 第八部分：路由（React Router v6）

```jsx
import { BrowserRouter, Routes, Route, Link, useParams } from 'react-router-dom';
function App() {
  return (
    <BrowserRouter>
      <Link to="/">Home</Link>
      <Routes>
        <Route path="/" element={<Home />} />
        <Route path="/user/:id" element={<User />} />
      </Routes>
    </BrowserRouter>
  );
}
function User() {
  const { id } = useParams();
  return <div>User {id}</div>;
}
```

---

## 第九部分：数据获取与缓存（TanStack Query）

```jsx
import { useQuery, QueryClient, QueryClientProvider } from '@tanstack/react-query';
const queryClient = new QueryClient();
function App() {
  return (
    <QueryClientProvider client={queryClient}>
      <MyComponent />
    </QueryClientProvider>
  );
}
function MyComponent() {
  const { data, isLoading } = useQuery({
    queryKey: ['todos'],
    queryFn: () => fetch('/api/todos').then(res => res.json()),
  });
  if (isLoading) return <div>Loading...</div>;
  return <div>{/* render */}</div>;
}
```

---

## 第十部分：表单处理（React Hook Form）

```jsx
import { useForm } from 'react-hook-form';
function Form() {
  const { register, handleSubmit } = useForm();
  const onSubmit = data => console.log(data);
  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      <input {...register('email')} />
      <input {...register('password')} />
      <button type="submit">Submit</button>
    </form>
  );
}
```

---

## 第十一部分：测试（Jest + React Testing Library）

```jsx
import { render, screen, fireEvent } from '@testing-library/react';
test('increments counter', () => {
  render(<Counter />);
  const button = screen.getByText('+');
  fireEvent.click(button);
  expect(screen.getByText('1')).toBeInTheDocument();
});
```

---

## 第十二部分：最佳实践汇总

1. **组件粒度**：保持单一职责，组件不宜过大。
2. **避免内联函数/对象**，除非必要，否则使用 useCallback/useMemo。
3. **状态最少化**：能计算得到的状态不存储。
4. **合理使用 Context**：避免巨大 Context，可拆分多个。
5. **使用 TypeScript**：增强类型安全。
6. **代码分割**：路由级懒加载。
7. **服务端状态缓存**：使用 TanStack Query。
8. **环境变量**：通过 `process.env` 管理。
9. **ESLint + Prettier** 统一代码风格。
10. **编写测试**：至少覆盖核心交互。

---

## 第十三部分：完整可运行的综合示例（HTML 单文件）

> 包含：useState, useEffect, useRef, useMemo, useCallback, useReducer, useContext, React.memo, Portals, 错误边界, 并发特性, 自定义 Hook, 列表与 Key。

```html
<!DOCTYPE html>
<html>
<head>
  <meta charset="UTF-8">
  <title>React 18 综合示例</title>
  <script src="https://unpkg.com/react@18/umd/react.production.min.js"></script>
  <script src="https://unpkg.com/react-dom@18/umd/react-dom.production.min.js"></script>
  <script src="https://unpkg.com/@babel/standalone/babel.min.js"></script>
  <style>body { font-family: system-ui; padding: 20px; } .card { border: 1px solid #ccc; padding: 16px; margin: 16px 0; }</style>
</head>
<body>
<div id="root"></div>
<script type="text/babel">
const { useState, useEffect, useRef, useMemo, useCallback, useReducer, useContext, createContext, useTransition, useDeferredValue, Suspense, lazy } = React;

// 1. 自定义 Hook
function useLocalStorage(key, initial) {
  const [value, setValue] = useState(() => {
    const stored = localStorage.getItem(key);
    return stored ? JSON.parse(stored) : initial;
  });
  useEffect(() => localStorage.setItem(key, JSON.stringify(value)), [key, value]);
  return [value, setValue];
}

// 2. Context
const ThemeContext = createContext('light');

// 3. Reducer
const counterReducer = (state, action) => {
  switch (action.type) {
    case 'inc': return state + 1;
    case 'dec': return state - 1;
    default: return state;
  }
};

// 4. 错误边界（类组件）
class ErrorBoundary extends React.Component {
  state = { hasError: false };
  static getDerivedStateFromError() { return { hasError: true }; }
  render() {
    if (this.state.hasError) return <div style={{color:'red'}}>出错了</div>;
    return this.props.children;
  }
}

// 5. 子组件 - memo
const MemoChild = React.memo(({ onClick }) => {
  console.log('MemoChild render');
  return <button onClick={onClick}>Memo Button</button>;
});

// 6. 主应用
function App() {
  const [count, setCount] = useState(0);
  const [text, setText] = useState('');
  const inputRef = useRef(null);
  const renderCount = useRef(0);
  renderCount.current++;

  const [counter, dispatch] = useReducer(counterReducer, 0);
  const [theme, setTheme] = useLocalStorage('theme', 'light');

  const expensive = useMemo(() => count * 2, [count]);
  const handleClick = useCallback(() => setCount(c => c + 1), []);

  const [isPending, startTransition] = useTransition();
  const [filter, setFilter] = useState('');
  const deferredFilter = useDeferredValue(filter);

  // 模拟长列表
  const items = useMemo(() => Array(1000).fill('item'), []);
  const filtered = useMemo(() => items.filter(item => item.includes(deferredFilter)), [items, deferredFilter]);

  useEffect(() => {
    console.log('App mounted');
    return () => console.log('App unmounted');
  }, []);

  return (
    <ThemeContext.Provider value={theme}>
      <div className="card">
        <h2>React 18 综合示例</h2>
        <div>渲染次数: {renderCount.current}</div>
        <button onClick={handleClick}>Count: {count} (memoized)</button>
        <div>Expensive: {expensive}</div>
        <input ref={inputRef} value={text} onChange={e => setText(e.target.value)} />
        <button onClick={() => inputRef.current?.focus()}>Focus input</button>
        <div>Reducer: {counter} <button onClick={() => dispatch({ type: 'inc' })}>+</button></div>
        <div>Theme: {theme} <button onClick={() => setTheme(theme === 'light' ? 'dark' : 'light')}>Toggle</button></div>

        <MemoChild onClick={handleClick} />

        <input placeholder="Search (transition)" onChange={e => startTransition(() => setFilter(e.target.value))} />
        {isPending && <span>Loading...</span>}
        <div>Filtered: {filtered.length} items</div>

        <ErrorBoundary>
          {/* 故意抛出错误测试边界 */}
          {/* <ThrowError /> */}
        </ErrorBoundary>

        <div>
          <button onClick={() => alert(theme)}>Show context via alert</button>
        </div>

        {/* Portal 示例 */}
        <PortalDemo />
      </div>
    </ThemeContext.Provider>
  );
}

function PortalDemo() {
  const [show, setShow] = useState(false);
  return (
    <>
      <button onClick={() => setShow(true)}>Open Portal</button>
      {show && ReactDOM.createPortal(
        <div style={{ position: 'fixed', top: 0, left: 0, right: 0, bottom: 0, backgroundColor: 'rgba(0,0,0,0.5)', display: 'flex', alignItems: 'center', justifyContent: 'center' }}>
          <div style={{ background: 'white', padding: 30 }}>Portal 内容 <button onClick={() => setShow(false)}>关闭</button></div>
        </div>,
        document.body
      )}
    </>
  );
}

// 故意错误组件（取消注释可测试边界）
// function ThrowError() { throw new Error('Test error'); }

ReactDOM.createRoot(document.getElementById('root')).render(<App />);
</script>
</body>
</html>
```

---

## 第十四部分：Next.js 全栈集成（核心知识点）

### 14.1 文件路由（App Router）

- `page.js` → 页面
- `layout.js` → 布局
- `loading.js` → 加载 UI
- `error.js` → 错误边界
- `route.js` → API 路由

### 14.2 服务器组件（默认）

- 在服务端执行，可直接 `async/await`，不发送到客户端。

### 14.3 客户端组件

- `'use client'` 指令，用于交互。

### 14.4 数据获取

```jsx
// 服务端组件
export default async function Page() {
  const data = await fetch('https://api.example.com', { next: { revalidate: 60 } }).then(res => res.json());
  return <div>{data.title}</div>;
}
```

### 14.5 Server Actions

```jsx
'use server';
export async function createPost(formData) {
  // 直接操作数据库
  revalidatePath('/posts');
}
```

### 14.6 中间件（Middleware / Proxy）

```jsx
import { NextResponse } from 'next/server';
export function middleware(request) {
  if (!request.cookies.has('token')) {
    return NextResponse.redirect(new URL('/login', request.url));
  }
}
```

### 14.7 部署与优化

- Vercel 一键部署
- ISR、SSG、SSR 策略选择
- 图片优化（`next/image`）
- 字体优化（`next/font`）

---

## 第十五部分：React 源码原理速览

- **React.createElement**：创建 React 元素。
- **Fiber 架构**：链表结构，可中断渲染。
- **Scheduler**：调度任务优先级。
- **Reconciler**：协调算法（Diff）。
- **Renderer**：渲染到不同环境（DOM、Native）。

---

## 附录：常见问题

**Q: 为什么 Hooks 不能在条件语句中调用？**  
A: 因为 React 依赖调用顺序来管理 Hook 状态，条件语句会破坏顺序。

**Q: setState 是同步还是异步？**  
A: 在合成事件和生命周期钩子中是异步批处理的，在 `setTimeout` 或原生事件中同步。

**Q: React.memo 与 useMemo 的区别？**  
A: `React.memo` 防止整个组件重新渲染；`useMemo` 缓存计算结果。

**Q: 何时使用 useLayoutEffect 而非 useEffect？**  
A: 当需要在绘制前同步读取 DOM 布局并更新时，例如测量滚动位置。

---

## 总结

这份手册涵盖了 React 18 从原理到实战的全链路知识。建议你边读边运行示例代码，加深理解。  
**记住 React 的核心：UI = f(state)**，并善用 Hooks 管理副作用和状态，你就能构建出高性能、可维护的现代 Web 应用。