# React 进阶开发核心知识点全景图

> 整理版本：2024-2025 | 覆盖 React 18/19 | 适合有基础的开发者进阶收藏

---

## 目录

1. [React 18+ 并发特性](#一react-18-并发特性)
2. [高级 Hooks 与自定义模式](#二高级-hooks-与自定义模式)
3. [性能优化体系](#三性能优化体系)
4. [组件模式与架构设计](#四组件模式与架构设计)
5. [状态管理进阶](#五状态管理进阶)
6. [数据获取与服务端集成](#六数据获取与服务端集成)
7. [TypeScript 与 React 深度结合](#七typescript-与-react-深度结合)
8. [测试策略](#八测试策略)
9. [工程化与构建优化](#九工程化与构建优化)
10. [React 19 新特性前瞻](#十react-19-新特性前瞻)
11. [实战架构决策](#十一实战架构决策)

---

## 一、React 18+ 并发特性

### 1.1 并发模式核心概念

并发模式（Concurrent Mode）是 React 18 的根本性变化，允许渲染工作可被**中断、恢复、丢弃**。

```jsx
// React 18：使用 createRoot 开启并发模式
import { createRoot } from 'react-dom/client';
const root = createRoot(document.getElementById('root'));
root.render(<App />);
```

**核心机制：**
- **时间切片（Time Slicing）**：将长渲染任务拆分为多个小任务，避免阻塞主线程
- **优先级调度（Priority Scheduling）**：不同更新有不同优先级（用户输入 > 数据加载 > 后台渲染）
- **可中断渲染**：高优先级更新可打断低优先级渲染

### 1.2 Transitions —— 区分紧急与非紧急更新

```jsx
import { useTransition, startTransition } from 'react';

function SearchPage() {
  const [query, setQuery] = useState('');
  const [results, setResults] = useState([]);
  const [isPending, startTransition] = useTransition();

  const handleChange = (e) => {
    // 紧急更新：立即响应输入
    setQuery(e.target.value);

    // 非紧急更新：可以被中断
    startTransition(() => {
      setResults(searchData(e.target.value));
    });
  };

  return (
    <>
      <input value={query} onChange={handleChange} />
      {isPending ? <Spinner /> : <ResultList data={results} />}
    </>
  );
}
```

**使用场景：**
- 搜索过滤、Tab 切换、路由跳转、大列表渲染

### 1.3 useDeferredValue —— 延迟派生状态

```jsx
import { useDeferredValue } from 'react';

function App() {
  const [text, setText] = useState('');
  const deferredText = useDeferredValue(text); // 低优先级的"滞后"副本

  return (
    <>
      <input value={text} onChange={e => setText(e.target.value)} />
      {/* 使用 deferredText 渲染重量级列表，不阻塞输入 */}
      <HeavyList filter={deferredText} />
    </>
  );
}
```

> **`useTransition` vs `useDeferredValue`**：前者控制状态更新的优先级，后者控制值的使用优先级。当你无法访问状态设置函数时（如来自第三方库），用 `useDeferredValue`。

### 1.4 Suspense 全面升级

```jsx
// React 18：Suspense 支持数据获取（配合框架或 use()）
<Suspense fallback={<Loading />}>
  <UserProfile userId={id} />  {/* 组件内部可以"挂起" */}
</Suspense>

// 并发 Suspense 支持嵌套与流式渲染
<Suspense fallback={<PageSkeleton />}>
  <Header />
  <Suspense fallback={<FeedSkeleton />}>
    <Feed />
  </Suspense>
  <Suspense fallback={<SidebarSkeleton />}>
    <Sidebar />
  </Suspense>
</Suspense>
```

**流式 SSR（Streaming SSR）：**
- React 18 SSR 中，`renderToPipeableStream` 支持流式传输 HTML
- 已准备好的部分立即发送，挂起的部分稍后流式补充

### 1.5 Automatic Batching —— 自动批处理

```jsx
// React 17：只有 React 事件处理函数内才批处理
// React 18：所有更新默认批处理（包括 setTimeout、Promise、原生事件）

setTimeout(() => {
  setCount(c => c + 1);
  setFlag(f => !f);
  // React 18：只触发一次重渲染 ✅
}, 1000);

// 需要退出批处理时：
import { flushSync } from 'react-dom';
flushSync(() => setCount(c => c + 1)); // 立即渲染
flushSync(() => setFlag(f => !f));     // 再次立即渲染
```

---

## 二、高级 Hooks 与自定义模式

### 2.1 useReducer —— 复杂状态逻辑

```typescript
type State = { count: number; status: 'idle' | 'loading' | 'error' };
type Action =
  | { type: 'INCREMENT' }
  | { type: 'SET_LOADING' }
  | { type: 'SET_ERROR'; payload: string };

function reducer(state: State, action: Action): State {
  switch (action.type) {
    case 'INCREMENT':
      return { ...state, count: state.count + 1 };
    case 'SET_LOADING':
      return { ...state, status: 'loading' };
    default:
      return state;
  }
}

const [state, dispatch] = useReducer(reducer, { count: 0, status: 'idle' });
```

### 2.2 useCallback 与 useMemo 的正确使用姿势

```jsx
// ❌ 过度优化：简单值不需要 useMemo
const value = useMemo(() => a + b, [a, b]); // 不值得

// ✅ 正确场景：昂贵计算
const sortedList = useMemo(
  () => heavySort(largeArray),
  [largeArray]
);

// ✅ 正确场景：稳定引用（传给子组件或 useEffect 依赖）
const handleSubmit = useCallback((data) => {
  submitForm(data);
}, [submitForm]);

// 关键原则：先写正确代码，再用 Profiler 找瓶颈，再优化
```

### 2.3 useRef 的多种用法

```jsx
// 1. DOM 引用
const inputRef = useRef<HTMLInputElement>(null);
useEffect(() => { inputRef.current?.focus(); }, []);

// 2. 保存不触发重渲染的可变值（如定时器 ID）
const timerRef = useRef<NodeJS.Timeout | null>(null);

// 3. 保存上一次的值
function usePrevious<T>(value: T): T | undefined {
  const ref = useRef<T>();
  useEffect(() => { ref.current = value; });
  return ref.current;
}

// 4. 回调引用（避免 stale closure）
function useEvent<T extends (...args: any[]) => any>(fn: T): T {
  const ref = useRef(fn);
  useLayoutEffect(() => { ref.current = fn; });
  return useCallback((...args) => ref.current(...args), []) as T;
}
```

### 2.4 自定义 Hooks 设计模式

**原则：** 抽离逻辑，而不是抽离 UI；以 `use` 开头；返回值语义清晰。

```typescript
// 模式一：数据获取 Hook
function useFetch<T>(url: string) {
  const [data, setData] = useState<T | null>(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState<Error | null>(null);

  useEffect(() => {
    const controller = new AbortController();
    fetch(url, { signal: controller.signal })
      .then(r => r.json())
      .then(setData)
      .catch(e => !e.name.includes('Abort') && setError(e))
      .finally(() => setLoading(false));
    return () => controller.abort();
  }, [url]);

  return { data, loading, error };
}

// 模式二：本地存储同步
function useLocalStorage<T>(key: string, initialValue: T) {
  const [value, setValue] = useState<T>(() => {
    try {
      const item = window.localStorage.getItem(key);
      return item ? JSON.parse(item) : initialValue;
    } catch { return initialValue; }
  });

  const setStoredValue = useCallback((newValue: T | ((prev: T) => T)) => {
    setValue(prev => {
      const next = typeof newValue === 'function'
        ? (newValue as (p: T) => T)(prev)
        : newValue;
      window.localStorage.setItem(key, JSON.stringify(next));
      return next;
    });
  }, [key]);

  return [value, setStoredValue] as const;
}

// 模式三：防抖 Hook
function useDebounce<T>(value: T, delay: number): T {
  const [debounced, setDebounced] = useState(value);
  useEffect(() => {
    const timer = setTimeout(() => setDebounced(value), delay);
    return () => clearTimeout(timer);
  }, [value, delay]);
  return debounced;
}
```

### 2.5 useImperativeHandle —— 精控暴露给父组件的接口

```typescript
interface VideoPlayerRef {
  play: () => void;
  pause: () => void;
  seek: (time: number) => void;
}

const VideoPlayer = forwardRef<VideoPlayerRef, Props>((props, ref) => {
  const videoRef = useRef<HTMLVideoElement>(null);

  useImperativeHandle(ref, () => ({
    play: () => videoRef.current?.play(),
    pause: () => videoRef.current?.pause(),
    seek: (time) => { if (videoRef.current) videoRef.current.currentTime = time; },
  }), []);

  return <video ref={videoRef} {...props} />;
});
```

### 2.6 useSyncExternalStore —— 订阅外部数据源

```typescript
// 适用于：与非 React 状态管理（Redux、Zustand、浏览器 API）集成
function useWindowSize() {
  return useSyncExternalStore(
    (callback) => {
      window.addEventListener('resize', callback);
      return () => window.removeEventListener('resize', callback);
    },
    () => ({ width: window.innerWidth, height: window.innerHeight }),
    () => ({ width: 0, height: 0 }) // SSR snapshot
  );
}
```

---

## 三、性能优化体系

### 3.1 渲染优化三件套

```jsx
// 1. React.memo —— 避免不必要的子组件重渲染
const ExpensiveComponent = memo(({ data, onAction }) => {
  return <div>{/* 复杂渲染 */}</div>;
}, (prevProps, nextProps) => {
  // 自定义比较（返回 true 表示相同，跳过渲染）
  return prevProps.data.id === nextProps.data.id;
});

// 2. useMemo —— 缓存计算结果
const processedData = useMemo(() =>
  data.filter(item => item.active).sort((a, b) => b.score - a.score),
  [data]
);

// 3. useCallback —— 稳定函数引用
const handleDelete = useCallback((id: string) => {
  dispatch({ type: 'DELETE', payload: id });
}, [dispatch]);
```

### 3.2 虚拟列表（Virtualization）

```jsx
// 使用 @tanstack/react-virtual 或 react-window
import { useVirtualizer } from '@tanstack/react-virtual';

function VirtualList({ items }) {
  const parentRef = useRef(null);
  const virtualizer = useVirtualizer({
    count: items.length,
    getScrollElement: () => parentRef.current,
    estimateSize: () => 50,
    overscan: 5,
  });

  return (
    <div ref={parentRef} style={{ height: '400px', overflow: 'auto' }}>
      <div style={{ height: virtualizer.getTotalSize() }}>
        {virtualizer.getVirtualItems().map(row => (
          <div
            key={row.key}
            style={{ transform: `translateY(${row.start}px)`, position: 'absolute' }}
          >
            {items[row.index].name}
          </div>
        ))}
      </div>
    </div>
  );
}
```

### 3.3 代码分割与懒加载

```jsx
// 路由级别懒加载
const Dashboard = lazy(() => import('./pages/Dashboard'));
const Profile = lazy(() => import('./pages/Profile'));

// 组件级别懒加载（按需）
const HeavyChart = lazy(() =>
  import('./components/HeavyChart').then(m => ({ default: m.HeavyChart }))
);

// 预加载策略
const prefetchDashboard = () => import('./pages/Dashboard');
<Link onMouseEnter={prefetchDashboard} to="/dashboard">Dashboard</Link>
```

### 3.4 Context 性能陷阱与优化

```jsx
// ❌ 问题：Context value 每次渲染都是新对象，导致所有消费者重渲染
function Provider({ children }) {
  const [user, setUser] = useState(null);
  return (
    <UserContext.Provider value={{ user, setUser }}> {/* 每次都是新对象！ */}
      {children}
    </UserContext.Provider>
  );
}

// ✅ 方案一：拆分 Context（状态与 dispatch 分离）
const UserStateContext = createContext(null);
const UserDispatchContext = createContext(null);

// ✅ 方案二：useMemo 稳定 value
function Provider({ children }) {
  const [user, setUser] = useState(null);
  const value = useMemo(() => ({ user, setUser }), [user]);
  return <UserContext.Provider value={value}>{children}</UserContext.Provider>;
}

// ✅ 方案三：使用 Zustand / Jotai 替代全局 Context
```

### 3.5 性能分析工具

```jsx
// React DevTools Profiler
// 1. 录制渲染 → 找出渲染耗时最长的组件
// 2. "Highlight updates" 可视化哪些组件在重渲染

// Profiler API（编程式）
<Profiler id="Navigation" onRender={(id, phase, actualDuration) => {
  console.log(`${id} ${phase}: ${actualDuration}ms`);
}}>
  <Navigation />
</Profiler>

// why-did-you-render（开发环境）
import whyDidYouRender from '@welldone-software/why-did-you-render';
whyDidYouRender(React, { trackAllPureComponents: true });
```

### 3.6 其他优化手段

| 技术 | 场景 | 工具/方法 |
|------|------|-----------|
| 图片懒加载 | 长页面图片 | `loading="lazy"` / Intersection Observer |
| 防抖/节流 | 高频输入、滚动 | `useDebounce` / `lodash.throttle` |
| Web Workers | CPU 密集计算 | `comlink` 库 |
| 服务端渲染 | 首屏性能、SEO | Next.js / Remix |
| 静态生成 | 内容不变的页面 | Next.js SSG |
| Bundle 分析 | 包体积过大 | `webpack-bundle-analyzer` |

---

## 四、组件模式与架构设计

### 4.1 复合组件模式（Compound Components）

```typescript
// 使用 Context 在组件族间隐式共享状态
interface TabsContextValue {
  activeTab: string;
  setActiveTab: (tab: string) => void;
}

const TabsContext = createContext<TabsContextValue | null>(null);

function Tabs({ children, defaultTab }: { children: ReactNode; defaultTab: string }) {
  const [activeTab, setActiveTab] = useState(defaultTab);
  return (
    <TabsContext.Provider value={{ activeTab, setActiveTab }}>
      <div className="tabs">{children}</div>
    </TabsContext.Provider>
  );
}

function TabList({ children }: { children: ReactNode }) {
  return <div role="tablist">{children}</div>;
}

function Tab({ id, children }: { id: string; children: ReactNode }) {
  const { activeTab, setActiveTab } = useContext(TabsContext)!;
  return (
    <button
      role="tab"
      aria-selected={activeTab === id}
      onClick={() => setActiveTab(id)}
    >
      {children}
    </button>
  );
}

function TabPanel({ id, children }: { id: string; children: ReactNode }) {
  const { activeTab } = useContext(TabsContext)!;
  return activeTab === id ? <div role="tabpanel">{children}</div> : null;
}

// 挂载子组件
Tabs.List = TabList;
Tabs.Tab = Tab;
Tabs.Panel = TabPanel;

// 使用方式（API 干净、高度灵活）
<Tabs defaultTab="profile">
  <Tabs.List>
    <Tabs.Tab id="profile">个人信息</Tabs.Tab>
    <Tabs.Tab id="settings">设置</Tabs.Tab>
  </Tabs.List>
  <Tabs.Panel id="profile"><ProfileForm /></Tabs.Panel>
  <Tabs.Panel id="settings"><SettingsForm /></Tabs.Panel>
</Tabs>
```

### 4.2 Render Props 与控制反转

```typescript
// 现代写法：children as function
interface DataProviderProps<T> {
  url: string;
  children: (state: { data: T | null; loading: boolean; error: Error | null }) => ReactNode;
}

function DataProvider<T>({ url, children }: DataProviderProps<T>) {
  const state = useFetch<T>(url);
  return <>{children(state)}</>;
}

// 使用
<DataProvider<User[]> url="/api/users">
  {({ data, loading }) =>
    loading ? <Spinner /> : <UserList users={data ?? []} />
  }
</DataProvider>
```

### 4.3 高阶组件（HOC）的现代用法

```typescript
// HOC 适合横切关注点：权限、日志、错误边界
function withAuth<P extends object>(
  WrappedComponent: ComponentType<P>,
  requiredRole?: string
) {
  return function AuthenticatedComponent(props: P) {
    const { user, isLoading } = useAuth();
    if (isLoading) return <Spinner />;
    if (!user) return <Navigate to="/login" />;
    if (requiredRole && !user.roles.includes(requiredRole)) return <Forbidden />;
    return <WrappedComponent {...props} />;
  };
}

const AdminDashboard = withAuth(Dashboard, 'admin');
```

### 4.4 错误边界（Error Boundary）

```typescript
// 类组件（目前唯一实现方式，React 19 引入 use() 有所改善）
class ErrorBoundary extends Component<
  { children: ReactNode; fallback: ComponentType<{ error: Error; reset: () => void }> },
  { error: Error | null }
> {
  state = { error: null };

  static getDerivedStateFromError(error: Error) {
    return { error };
  }

  componentDidCatch(error: Error, info: ErrorInfo) {
    reportError(error, info); // 上报错误监控
  }

  render() {
    if (this.state.error) {
      const Fallback = this.props.fallback;
      return <Fallback error={this.state.error} reset={() => this.setState({ error: null })} />;
    }
    return this.props.children;
  }
}

// 推荐使用 react-error-boundary 库
import { ErrorBoundary } from 'react-error-boundary';
<ErrorBoundary FallbackComponent={ErrorFallback} onReset={resetApp}>
  <App />
</ErrorBoundary>
```

### 4.5 受控 vs 非受控组件的设计决策

```typescript
// 无头组件（Headless Component）：逻辑与 UI 完全分离
// 参考：Radix UI、Headless UI、React Aria

function useDisclosure(defaultOpen = false) {
  const [isOpen, setIsOpen] = useState(defaultOpen);
  const open = useCallback(() => setIsOpen(true), []);
  const close = useCallback(() => setIsOpen(false), []);
  const toggle = useCallback(() => setIsOpen(o => !o), []);
  return { isOpen, open, close, toggle };
}

// 使用者完全控制样式
const { isOpen, open, close } = useDisclosure();
<button onClick={open}>打开</button>
{isOpen && <Modal onClose={close}><YourContent /></Modal>}
```

### 4.6 组件设计原则

- **单一职责**：一个组件只做一件事
- **最小 API 表面**：只暴露必要的 props
- **默认可用**：提供合理的默认值
- **可组合性**：避免深层 prop drilling，优先 composition
- **关注点分离**：UI 组件 / 容器组件 / 业务 Hooks 各司其职

---

## 五、状态管理进阶

### 5.1 状态分类与选型原则

```
状态类型               推荐方案
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
本地 UI 状态          useState / useReducer
跨组件共享状态         Context（小规模）/ Zustand（中大规模）
服务端数据（异步）      TanStack Query / SWR
表单状态               React Hook Form / Formik
URL 状态               React Router / nuqs
全局复杂状态           Redux Toolkit / Zustand / Jotai
```

### 5.2 Zustand —— 轻量高效

```typescript
import { create } from 'zustand';
import { immer } from 'zustand/middleware/immer';
import { persist } from 'zustand/middleware';

interface BearStore {
  bears: number;
  todos: Todo[];
  addBear: () => void;
  addTodo: (text: string) => void;
  removeTodo: (id: string) => void;
}

const useBearStore = create<BearStore>()(
  persist(
    immer((set) => ({
      bears: 0,
      todos: [],
      addBear: () => set(state => { state.bears += 1; }),
      addTodo: (text) => set(state => {
        state.todos.push({ id: crypto.randomUUID(), text, done: false });
      }),
      removeTodo: (id) => set(state => {
        state.todos = state.todos.filter(t => t.id !== id);
      }),
    })),
    { name: 'bear-storage' }
  )
);

// 使用：选择性订阅（避免无关更新导致重渲染）
const bears = useBearStore(state => state.bears);
const addTodo = useBearStore(state => state.addTodo);
```

### 5.3 Jotai —— 原子化状态

```typescript
import { atom, useAtom, useAtomValue, useSetAtom } from 'jotai';

// 原子状态（天然细粒度，无需手动订阅优化）
const countAtom = atom(0);
const doubledAtom = atom(get => get(countAtom) * 2); // 派生 atom

// 异步 atom
const userAtom = atom(async () => {
  const res = await fetch('/api/user');
  return res.json();
});

// 可写异步 atom
const todosAtom = atom<Todo[]>([]);
const addTodoAtom = atom(null, async (get, set, text: string) => {
  const todo = await createTodo(text);
  set(todosAtom, [...get(todosAtom), todo]);
});
```

### 5.4 Redux Toolkit —— 企业级复杂场景

```typescript
import { createSlice, createAsyncThunk, configureStore } from '@reduxjs/toolkit';

// createAsyncThunk 处理异步
export const fetchUser = createAsyncThunk('user/fetch', async (id: string) => {
  const res = await fetch(`/api/users/${id}`);
  return res.json();
});

const userSlice = createSlice({
  name: 'user',
  initialState: { data: null, status: 'idle' } as UserState,
  reducers: {
    logout: state => { state.data = null; },
  },
  extraReducers: builder => {
    builder
      .addCase(fetchUser.pending, state => { state.status = 'loading'; })
      .addCase(fetchUser.fulfilled, (state, action) => {
        state.status = 'succeeded';
        state.data = action.payload;
      })
      .addCase(fetchUser.rejected, state => { state.status = 'failed'; });
  },
});

// RTK Query（内置数据获取与缓存）
import { createApi, fetchBaseQuery } from '@reduxjs/toolkit/query/react';

export const api = createApi({
  baseQuery: fetchBaseQuery({ baseUrl: '/api' }),
  endpoints: build => ({
    getUser: build.query<User, string>({ query: id => `/users/${id}` }),
    updateUser: build.mutation<User, Partial<User>>({
      query: ({ id, ...patch }) => ({ url: `/users/${id}`, method: 'PATCH', body: patch }),
    }),
  }),
});

export const { useGetUserQuery, useUpdateUserMutation } = api;
```

### 5.5 TanStack Query —— 服务端状态的最佳实践

```typescript
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query';

// 查询
function UserProfile({ id }: { id: string }) {
  const { data, isLoading, error } = useQuery({
    queryKey: ['user', id],
    queryFn: () => fetchUser(id),
    staleTime: 5 * 60 * 1000, // 5 分钟内不重新请求
    gcTime: 10 * 60 * 1000,   // 10 分钟后清除缓存
  });
}

// 变更（乐观更新）
function useUpdateUser() {
  const queryClient = useQueryClient();
  return useMutation({
    mutationFn: updateUser,
    onMutate: async (newUser) => {
      await queryClient.cancelQueries({ queryKey: ['user', newUser.id] });
      const previous = queryClient.getQueryData(['user', newUser.id]);
      queryClient.setQueryData(['user', newUser.id], newUser); // 乐观更新
      return { previous };
    },
    onError: (_, newUser, context) => {
      queryClient.setQueryData(['user', newUser.id], context?.previous); // 回滚
    },
    onSettled: (_, __, newUser) => {
      queryClient.invalidateQueries({ queryKey: ['user', newUser.id] }); // 重新验证
    },
  });
}
```

---

## 六、数据获取与服务端集成

### 6.1 数据获取模式

```typescript
// 模式一：并行请求（避免瀑布流）
const [users, posts] = await Promise.all([fetchUsers(), fetchPosts()]);

// 模式二：预取（Prefetching）
// 在路由跳转前开始请求
queryClient.prefetchQuery({ queryKey: ['user', id], queryFn: () => fetchUser(id) });

// 模式三：依赖查询
const { data: user } = useQuery({ queryKey: ['user', userId], queryFn: fetchUser });
const { data: orders } = useQuery({
  queryKey: ['orders', user?.id],
  queryFn: () => fetchOrders(user!.id),
  enabled: !!user?.id, // 等待 user 加载完成
});
```

### 6.2 React Server Components（RSC）

```tsx
// 服务端组件：在服务器运行，不发送 JS 到客户端
// app/page.tsx (Next.js App Router)
async function ProductPage({ params }: { params: { id: string } }) {
  // 直接在服务端查询数据库，无需 API 层
  const product = await db.product.findUnique({ where: { id: params.id } });

  return (
    <div>
      <h1>{product.name}</h1>
      {/* 交互部分标记为客户端组件 */}
      <AddToCartButton productId={product.id} />
    </div>
  );
}

// 客户端组件（需要 useState/useEffect/事件处理）
'use client';
function AddToCartButton({ productId }: { productId: string }) {
  const [added, setAdded] = useState(false);
  return <button onClick={() => setAdded(true)}>{added ? '已添加' : '加入购物车'}</button>;
}
```

**RSC 核心原则：**
- 默认服务端，交互用 `'use client'` 标记
- 服务端组件可以 `async/await`，客户端不行
- 服务端组件可以直接访问数据库、文件系统
- 不能在服务端组件使用 `useState`、`useEffect`、事件处理

---

## 七、TypeScript 与 React 深度结合

### 7.1 组件类型定义

```typescript
// 函数组件
interface ButtonProps {
  variant?: 'primary' | 'secondary' | 'danger';
  size?: 'sm' | 'md' | 'lg';
  children: ReactNode;
  onClick?: (event: MouseEvent<HTMLButtonElement>) => void;
  disabled?: boolean;
}

const Button: FC<ButtonProps> = ({ variant = 'primary', children, ...rest }) => (
  <button className={`btn btn-${variant}`} {...rest}>{children}</button>
);

// 泛型组件
interface ListProps<T> {
  items: T[];
  renderItem: (item: T, index: number) => ReactNode;
  keyExtractor: (item: T) => string;
}

function List<T>({ items, renderItem, keyExtractor }: ListProps<T>) {
  return (
    <ul>
      {items.map((item, i) => (
        <li key={keyExtractor(item)}>{renderItem(item, i)}</li>
      ))}
    </ul>
  );
}
```

### 7.2 常用工具类型

```typescript
// ComponentProps —— 提取组件 props 类型
type InputProps = ComponentPropsWithoutRef<'input'>;
type DivProps = ComponentPropsWithRef<'div'>;

// 扩展原生元素
interface CustomInputProps extends ComponentPropsWithoutRef<'input'> {
  label: string;
  error?: string;
}

// 多态组件（as prop）
type PolymorphicProps<E extends ElementType, P = {}> = P & {
  as?: E;
} & Omit<ComponentPropsWithoutRef<E>, keyof P | 'as'>;

function Text<E extends ElementType = 'span'>({ as, ...props }: PolymorphicProps<E>) {
  const Component = as ?? 'span';
  return <Component {...props} />;
}
// <Text as="h1">标题</Text>
// <Text as="p">段落</Text>
```

### 7.3 Hooks 类型

```typescript
// 联合类型 + 类型收窄
type RequestState<T> =
  | { status: 'idle' }
  | { status: 'loading' }
  | { status: 'success'; data: T }
  | { status: 'error'; error: Error };

function useRequest<T>(fn: () => Promise<T>): RequestState<T> {
  const [state, setState] = useState<RequestState<T>>({ status: 'idle' });
  // ...
  return state;
}

// 使用时可以精准类型收窄
const state = useRequest(fetchUser);
if (state.status === 'success') {
  console.log(state.data); // TypeScript 知道这里是 T
}
```

---

## 八、测试策略

### 8.1 测试金字塔

```
         E2E Tests (Playwright/Cypress)
              少量，慢，高置信度
         ─────────────────────────────
        Integration Tests (React Testing Library)
              适量，测行为而非实现
         ─────────────────────────────────────
       Unit Tests (Vitest/Jest)
           大量，快，测纯函数/Hooks
```

### 8.2 React Testing Library 最佳实践

```typescript
import { render, screen, userEvent } from '@testing-library/react';

// ✅ 按用户可见行为测试，而非内部实现
test('用户输入后点击提交，显示成功消息', async () => {
  const user = userEvent.setup();
  render(<ContactForm />);

  // 按角色查找（无障碍优先）
  await user.type(screen.getByRole('textbox', { name: /邮箱/i }), 'test@example.com');
  await user.click(screen.getByRole('button', { name: /提交/i }));

  expect(await screen.findByText(/提交成功/i)).toBeInTheDocument();
});

// 测试自定义 Hook
import { renderHook, act } from '@testing-library/react';

test('useCounter 应该正确递增', () => {
  const { result } = renderHook(() => useCounter(0));
  act(() => { result.current.increment(); });
  expect(result.current.count).toBe(1);
});
```

### 8.3 Mock 策略

```typescript
// MSW (Mock Service Worker) —— 拦截真实网络请求
import { http, HttpResponse } from 'msw';
import { setupServer } from 'msw/node';

const server = setupServer(
  http.get('/api/users/:id', ({ params }) => {
    return HttpResponse.json({ id: params.id, name: '张三' });
  })
);

beforeAll(() => server.listen());
afterEach(() => server.resetHandlers());
afterAll(() => server.close());
```

---

## 九、工程化与构建优化

### 9.1 现代构建工具选型

| 工具 | 适用场景 | 特点 |
|------|----------|------|
| **Vite** | SPA / 库开发 | 极快冷启动，ESM 原生 |
| **Next.js** | 全栈 / SSR / SSG | 完整框架，App Router |
| **Remix** | 全栈，表单优先 | Web 标准，嵌套路由 |
| **Turbopack** | Webpack 替代 | Vercel 出品，增量计算 |

### 9.2 路由架构（React Router v6）

```typescript
// Data Router（v6.4+）：路由级数据加载
const router = createBrowserRouter([
  {
    path: '/',
    element: <Root />,
    errorElement: <ErrorPage />,
    children: [
      {
        path: 'users',
        element: <UserList />,
        loader: async () => fetch('/api/users').then(r => r.json()), // 并行加载
      },
      {
        path: 'users/:id',
        element: <UserDetail />,
        loader: ({ params }) => loadUser(params.id),
        action: async ({ request }) => {
          const formData = await request.formData();
          return updateUser(Object.fromEntries(formData));
        },
      },
    ],
  },
]);

// 在组件中使用
function UserDetail() {
  const user = useLoaderData() as User;
  const navigation = useNavigation();
  return navigation.state === 'loading' ? <Spinner /> : <UserForm user={user} />;
}
```

### 9.3 微前端架构

```
方案对比：
┌─────────────────┬────────────────────────────────┐
│ Module Federation│ Webpack 5 原生支持，运行时共享  │
│ Single-SPA      │ 框架无关，生态丰富              │
│ qiankun         │ 阿里开源，沙箱隔离              │
│ Nx Monorepo     │ 构建缓存，依赖图分析            │
└─────────────────┴────────────────────────────────┘
```

### 9.4 CI/CD 最佳实践

```yaml
# GitHub Actions 示例
- name: Type Check
  run: tsc --noEmit

- name: Lint
  run: eslint . --ext .ts,.tsx

- name: Test
  run: vitest run --coverage

- name: Build
  run: vite build

- name: Bundle Size Check
  uses: andresz1/size-limit-action@v1
```

---

## 十、React 19 新特性前瞻

### 10.1 `use()` Hook

```typescript
// 在渲染时读取 Promise 或 Context（配合 Suspense）
import { use } from 'react';

function UserProfile({ userPromise }: { userPromise: Promise<User> }) {
  const user = use(userPromise); // 自动与 Suspense 集成
  return <h1>{user.name}</h1>;
}

// 条件使用 Context（打破了 Hooks 规则限制）
function ThemedButton({ showLabel }: { showLabel: boolean }) {
  if (showLabel) {
    const theme = use(ThemeContext); // ✅ React 19 允许在条件中使用
    return <button style={{ color: theme.color }}>按钮</button>;
  }
  return <button>按钮</button>;
}
```

### 10.2 Server Actions

```typescript
// Next.js App Router + React 19 Server Actions
'use server';

async function createPost(formData: FormData) {
  const title = formData.get('title') as string;
  await db.post.create({ data: { title } });
  revalidatePath('/posts');
}

// 客户端直接使用，无需手写 API
<form action={createPost}>
  <input name="title" />
  <button type="submit">发布</button>
</form>
```

### 10.3 其他 React 19 改进

- **`useFormStatus`**：读取父表单提交状态
- **`useOptimistic`**：内置乐观更新 Hook
- **`useActionState`**：管理 Server Action 状态
- **`ref` 作为 prop**：不再需要 `forwardRef`
- **Document Metadata**：`<title>`、`<meta>` 可在组件中声明，自动提升至 `<head>`
- **Resource Preloading API**：`preload()`、`preinit()` 等

---

## 十一、实战架构决策

### 11.1 项目结构推荐（Feature-Sliced Design）

```
src/
├── app/                    # 应用级配置（路由、Provider、全局样式）
├── pages/                  # 页面组件（路由入口）
├── widgets/                # 独立页面块（Header、Sidebar）
├── features/               # 业务功能（auth、cart、search）
│   └── auth/
│       ├── api/            # API 调用
│       ├── model/          # 状态管理（store/hooks）
│       ├── ui/             # UI 组件
│       └── index.ts        # 公开 API（只导出外部需要的）
├── entities/               # 业务实体（User、Product、Order）
└── shared/                 # 通用基础设施
    ├── ui/                 # 基础 UI 组件库
    ├── api/                # API 客户端配置
    ├── lib/                # 工具函数
    └── config/             # 环境变量等配置
```

### 11.2 技术栈组合推荐

```
场景一：中大型 SPA
Vite + React 18 + TypeScript + TanStack Router
+ Zustand（全局状态）+ TanStack Query（服务端状态）
+ React Hook Form + Zod（表单验证）
+ Tailwind CSS + Radix UI（样式与组件）

场景二：全栈应用（SEO 重要）
Next.js 14+ + TypeScript + Prisma + tRPC
+ Zustand + TanStack Query + React Hook Form

场景三：企业级大型项目
Next.js + TypeScript + Redux Toolkit (RTK Query)
+ React Hook Form + Zod + 内部 UI 组件库
```

### 11.3 进阶学习路线图

```
基础巩固（1-2 个月）
  ↓
并发特性 + 高级 Hooks（1 个月）
  ↓
状态管理深度（Zustand/TanStack Query）（1 个月）
  ↓
组件设计模式 + TypeScript 深度（1 个月）
  ↓
SSR/RSC + Next.js App Router（1-2 个月）
  ↓
测试体系 + 工程化 + 性能优化（持续）
  ↓
架构设计 + 微前端 + 技术选型（高级）
```

---

## 参考资源

| 资源 | 地址 |
|------|------|
| React 官方文档 | react.dev |
| TanStack Query | tanstack.com/query |
| Zustand | github.com/pmndrs/zustand |
| Redux Toolkit | redux-toolkit.js.org |
| React Testing Library | testing-library.com |
| Next.js 文档 | nextjs.org/docs |
| Josh W Comeau's Blog | joshwcomeau.com |
| Kent C. Dodds's Blog | kentcdodds.com |

---

> 📌 **学习建议**：不要试图一次性掌握所有内容。先在实际项目中用起来，遇到问题再深挖。**能解决真实问题的知识，才是真正掌握的知识。**

