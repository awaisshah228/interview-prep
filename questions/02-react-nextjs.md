# React & Next.js - Interview Q&A

> 25+ questions covering React internals, hooks, patterns, Next.js App Router, and state management

---

## Table of Contents

- [React Core](#react-core)
- [React Hooks Deep Dive](#react-hooks-deep-dive)
- [React Patterns & Performance](#react-patterns--performance)
- [Next.js App Router](#nextjs-app-router)
- [State Management (Zustand)](#state-management-zustand)
- [Testing](#testing)

---

## React Core

### Q1: Explain React Fiber Architecture and Reconciliation.

**Answer:**
**Fiber** is React's internal reconciliation engine (introduced in React 16).

**Key concepts:**
- Each React element gets a corresponding Fiber node
- Fibers form a linked list tree (child, sibling, return pointers)
- Work is split into units that can be paused, resumed, or aborted
- Enables **concurrent rendering** (time-slicing)

**Reconciliation process:**
1. **Render Phase** (can be interrupted): Creates new Fiber tree ("work in progress"), diffs with current tree
2. **Commit Phase** (synchronous, cannot be interrupted): Applies DOM changes

**Diffing Algorithm (O(n) heuristic):**
- Different element types → tear down old tree, build new
- Same type → update props, recurse on children
- Keys help identify which items changed in lists

```jsx
// Why keys matter
// BAD: Without stable keys, React re-renders ALL items on reorder
{items.map((item, index) => <Item key={index} {...item} />)} // ❌ index as key

// GOOD: Stable keys let React move DOM nodes efficiently
{items.map(item => <Item key={item.id} {...item} />)} // ✅ unique id
```

**Why index as key is bad:**
- Inserting/removing items causes wrong elements to re-render
- State gets associated with wrong items
- Causes bugs with controlled inputs, animations

---

### Q2: What is Virtual DOM and how does React update the real DOM?

**Answer:**

```
1. State changes → React creates new Virtual DOM tree (JS objects)
2. React diffs new VDOM vs old VDOM (reconciliation)
3. React computes minimal set of DOM operations needed
4. React batches and applies DOM updates in commit phase

Virtual DOM node (simplified):
{
  type: 'div',
  props: {
    className: 'container',
    children: [
      { type: 'h1', props: { children: 'Hello' } },
      { type: 'p', props: { children: 'World' } }
    ]
  }
}
```

**Why not update DOM directly?**
- DOM operations are expensive (reflow, repaint)
- Batching multiple changes into one update is faster
- Cross-platform abstraction (React Native, etc.)

---

### Q3: Explain React's Concurrent Features.

**Answer:**

```jsx
// 1. useTransition - mark updates as non-urgent
function SearchPage() {
  const [query, setQuery] = useState('');
  const [isPending, startTransition] = useTransition();

  function handleChange(e) {
    setQuery(e.target.value);              // urgent: update input
    startTransition(() => {
      setSearchResults(e.target.value);     // non-urgent: can be interrupted
    });
  }

  return (
    <>
      <input value={query} onChange={handleChange} />
      {isPending ? <Spinner /> : <Results />}
    </>
  );
}

// 2. useDeferredValue - defer a value
function SearchResults({ query }) {
  const deferredQuery = useDeferredValue(query);
  // deferredQuery updates with lower priority
  // UI stays responsive while results are computing

  const results = useMemo(() =>
    expensiveSearch(deferredQuery), [deferredQuery]
  );

  return <ResultList results={results} />;
}

// 3. Suspense - declarative loading states
function App() {
  return (
    <Suspense fallback={<Loading />}>
      <UserProfile />   {/* can "suspend" while loading data */}
    </Suspense>
  );
}

// 4. Streaming SSR (Next.js)
// Components wrapped in Suspense can stream HTML progressively
// User sees content as it becomes available, not all at once
```

---

### Q4: Explain React Error Boundaries.

**Answer:**

```jsx
// Error Boundary - class component that catches JS errors in child tree
class ErrorBoundary extends React.Component {
  constructor(props) {
    super(props);
    this.state = { hasError: false, error: null };
  }

  // Update state when error occurs
  static getDerivedStateFromError(error) {
    return { hasError: true, error };
  }

  // Log error (send to monitoring service)
  componentDidCatch(error, errorInfo) {
    console.error('Error caught:', error, errorInfo.componentStack);
    // Send to New Relic, Sentry, etc.
  }

  render() {
    if (this.state.hasError) {
      return this.props.fallback || <h1>Something went wrong</h1>;
    }
    return this.props.children;
  }
}

// Usage
<ErrorBoundary fallback={<ErrorPage />}>
  <TradingDashboard />
</ErrorBoundary>

// Granular error boundaries
<ErrorBoundary fallback={<ChartError />}>
  <PriceChart />
</ErrorBoundary>
<ErrorBoundary fallback={<OrderError />}>
  <OrderBook />
</ErrorBoundary>

// NOTE: Error boundaries do NOT catch:
// - Event handlers (use try/catch)
// - Async code (Promises)
// - SSR errors
// - Errors in the boundary itself
```

---

## React Hooks Deep Dive

### Q5: Explain useEffect cleanup and dependency array.

**Answer:**

```jsx
useEffect(() => {
  // SETUP: runs after render

  const subscription = api.subscribe(channel, onMessage);
  const timer = setInterval(fetchPrices, 5000);

  // CLEANUP: runs before next effect AND on unmount
  return () => {
    subscription.unsubscribe();
    clearInterval(timer);
  };
}, [channel]); // DEPENDENCIES: re-run when channel changes

// Dependency rules:
// []           → run once on mount, cleanup on unmount
// [dep1, dep2] → run when dep1 or dep2 changes
// (no array)   → run after every render (usually wrong)

// Common mistakes:
// 1. Missing dependency
useEffect(() => {
  fetchData(userId);  // userId should be in deps!
}, []); // ❌ stale closure

// 2. Object/array in deps (creates new reference each render)
useEffect(() => {
  // runs every render because {} !== {}
}, [{ id: 1 }]); // ❌

// Fix: use primitive values or useMemo
useEffect(() => {
  fetchUser(userId);
}, [userId]); // ✅ primitive

// 3. Don't lie about dependencies
// If linter says add a dep, add it or restructure
```

---

### Q6: useRef - What is it and when to use it?

**Answer:**

```jsx
// useRef returns a mutable object that persists across renders
// Changes to .current do NOT trigger re-render

// Use case 1: DOM access
function VideoPlayer() {
  const videoRef = useRef(null);

  const handlePlay = () => videoRef.current.play();
  const handlePause = () => videoRef.current.pause();

  return <video ref={videoRef} src="video.mp4" />;
}

// Use case 2: Store mutable value without re-render
function StopWatch() {
  const [time, setTime] = useState(0);
  const intervalRef = useRef(null);

  const start = () => {
    intervalRef.current = setInterval(() => {
      setTime(t => t + 1);
    }, 1000);
  };

  const stop = () => clearInterval(intervalRef.current);

  return <div>{time}s</div>;
}

// Use case 3: Track previous value
function usePrevious(value) {
  const ref = useRef();
  useEffect(() => {
    ref.current = value;
  });
  return ref.current;
}

function PriceDisplay({ price }) {
  const prevPrice = usePrevious(price);
  const direction = price > prevPrice ? 'up' : 'down';
  return <span className={direction}>{price}</span>;
}

// Use case 4: Avoid stale closures in callbacks
function SearchComponent() {
  const [query, setQuery] = useState('');
  const queryRef = useRef(query);
  queryRef.current = query; // always up to date

  const handleSubmit = useCallback(() => {
    // queryRef.current is always fresh, no dependency needed
    search(queryRef.current);
  }, []); // stable reference
}
```

---

### Q7: useReducer - When to use it over useState?

**Answer:**

```jsx
// useReducer is better when:
// 1. State logic is complex
// 2. Next state depends on previous state
// 3. Multiple related state values
// 4. Actions need to be testable

interface TradeState {
  trades: Trade[];
  isLoading: boolean;
  error: string | null;
  filter: 'all' | 'buy' | 'sell';
}

type TradeAction =
  | { type: 'FETCH_START' }
  | { type: 'FETCH_SUCCESS'; payload: Trade[] }
  | { type: 'FETCH_ERROR'; payload: string }
  | { type: 'SET_FILTER'; payload: 'all' | 'buy' | 'sell' }
  | { type: 'ADD_TRADE'; payload: Trade };

function tradeReducer(state: TradeState, action: TradeAction): TradeState {
  switch (action.type) {
    case 'FETCH_START':
      return { ...state, isLoading: true, error: null };
    case 'FETCH_SUCCESS':
      return { ...state, isLoading: false, trades: action.payload };
    case 'FETCH_ERROR':
      return { ...state, isLoading: false, error: action.payload };
    case 'SET_FILTER':
      return { ...state, filter: action.payload };
    case 'ADD_TRADE':
      return { ...state, trades: [...state.trades, action.payload] };
    default:
      return state;
  }
}

function TradeDashboard() {
  const [state, dispatch] = useReducer(tradeReducer, {
    trades: [],
    isLoading: false,
    error: null,
    filter: 'all',
  });

  useEffect(() => {
    dispatch({ type: 'FETCH_START' });
    fetchTrades()
      .then(trades => dispatch({ type: 'FETCH_SUCCESS', payload: trades }))
      .catch(err => dispatch({ type: 'FETCH_ERROR', payload: err.message }));
  }, []);
}

// Reducer is pure function → easy to test
test('FETCH_SUCCESS updates trades', () => {
  const state = tradeReducer(initialState, {
    type: 'FETCH_SUCCESS',
    payload: [{ id: '1', symbol: 'SOL' }],
  });
  expect(state.trades).toHaveLength(1);
  expect(state.isLoading).toBe(false);
});
```

---

### Q8: Custom Hooks - Patterns and best practices.

**Answer:**

```jsx
// 1. Data fetching hook
function useFetch<T>(url: string) {
  const [data, setData] = useState<T | null>(null);
  const [error, setError] = useState<Error | null>(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    const controller = new AbortController();

    async function fetchData() {
      try {
        setLoading(true);
        const response = await fetch(url, { signal: controller.signal });
        if (!response.ok) throw new Error(`HTTP ${response.status}`);
        const json = await response.json();
        setData(json);
      } catch (err) {
        if (err.name !== 'AbortError') setError(err);
      } finally {
        setLoading(false);
      }
    }

    fetchData();
    return () => controller.abort(); // cleanup on unmount or url change
  }, [url]);

  return { data, error, loading };
}

// 2. WebSocket hook
function useWebSocket(url: string) {
  const [messages, setMessages] = useState([]);
  const [status, setStatus] = useState<'connecting' | 'open' | 'closed'>('connecting');
  const wsRef = useRef<WebSocket | null>(null);

  useEffect(() => {
    const ws = new WebSocket(url);
    wsRef.current = ws;

    ws.onopen = () => setStatus('open');
    ws.onclose = () => setStatus('closed');
    ws.onmessage = (event) => {
      setMessages(prev => [...prev, JSON.parse(event.data)]);
    };

    return () => ws.close();
  }, [url]);

  const send = useCallback((data: any) => {
    wsRef.current?.send(JSON.stringify(data));
  }, []);

  return { messages, status, send };
}

// 3. Local storage hook
function useLocalStorage<T>(key: string, initialValue: T) {
  const [stored, setStored] = useState<T>(() => {
    try {
      const item = window.localStorage.getItem(key);
      return item ? JSON.parse(item) : initialValue;
    } catch {
      return initialValue;
    }
  });

  const setValue = (value: T | ((val: T) => T)) => {
    const newValue = value instanceof Function ? value(stored) : value;
    setStored(newValue);
    window.localStorage.setItem(key, JSON.stringify(newValue));
  };

  return [stored, setValue] as const;
}

// 4. Debounce hook
function useDebounce<T>(value: T, delay: number): T {
  const [debouncedValue, setDebouncedValue] = useState(value);

  useEffect(() => {
    const timer = setTimeout(() => setDebouncedValue(value), delay);
    return () => clearTimeout(timer);
  }, [value, delay]);

  return debouncedValue;
}

// Usage
function Search() {
  const [query, setQuery] = useState('');
  const debouncedQuery = useDebounce(query, 300);
  const { data } = useFetch(`/api/search?q=${debouncedQuery}`);
  // fetch only fires 300ms after user stops typing
}
```

---

## React Patterns & Performance

### Q9: When to use useMemo, useCallback, and React.memo?

**Answer:**

```jsx
// React.memo - Memoize a COMPONENT (skip re-render if props unchanged)
const ExpensiveList = React.memo(({ items, onSelect }) => {
  console.log('Rendering list...'); // only when items or onSelect changes
  return items.map(item => (
    <div key={item.id} onClick={() => onSelect(item.id)}>
      {item.name}
    </div>
  ));
});

// useCallback - Memoize a FUNCTION reference
function Parent() {
  const [count, setCount] = useState(0);

  // ❌ Without useCallback: new function every render → child re-renders
  const handleSelect = (id) => console.log(id);

  // ✅ With useCallback: stable reference
  const handleSelect = useCallback((id) => {
    console.log(id);
  }, []); // stable reference

  return <ExpensiveList items={items} onSelect={handleSelect} />;
}

// useMemo - Memoize a VALUE (skip expensive computation)
function Dashboard({ trades }) {
  // ❌ Recomputes every render
  const totalPnL = trades.reduce((sum, t) => sum + t.pnl, 0);

  // ✅ Only recomputes when trades changes
  const totalPnL = useMemo(() => {
    return trades.reduce((sum, t) => sum + t.pnl, 0);
  }, [trades]);

  return <div>PnL: {totalPnL}</div>;
}
```

**Decision guide:**

| Situation | Tool |
|---|---|
| Child receives function prop + is memoized | `useCallback` |
| Expensive calculation | `useMemo` |
| Child re-renders too often | `React.memo` |
| Primitive props | Don't memoize (cheap comparison) |
| Component renders fast already | Don't memoize (overhead not worth it) |

**Rule of thumb:** Profile first, optimize second. React DevTools Profiler shows what's slow.

---

### Q10: Explain React rendering behavior. When does a component re-render?

**Answer:**

A component re-renders when:
1. **Its state changes** (setState/dispatch)
2. **Its parent re-renders** (even if props didn't change!)
3. **Context value it consumes changes**

```jsx
// Problem: Child re-renders even though its props didn't change
function Parent() {
  const [count, setCount] = useState(0);
  return (
    <div>
      <button onClick={() => setCount(c => c + 1)}>{count}</button>
      <ExpensiveChild />  {/* re-renders every time count changes! */}
    </div>
  );
}

// Fix 1: React.memo
const ExpensiveChild = React.memo(() => {
  console.log('render'); // only on first render
  return <div>I'm expensive</div>;
});

// Fix 2: Composition (children pattern)
function Parent({ children }) {
  const [count, setCount] = useState(0);
  return (
    <div>
      <button onClick={() => setCount(c => c + 1)}>{count}</button>
      {children} {/* doesn't re-render because it's created by grandparent */}
    </div>
  );
}

// Usage:
<Parent>
  <ExpensiveChild />
</Parent>

// Fix 3: Move state down
function Parent() {
  return (
    <div>
      <Counter />  {/* only Counter re-renders */}
      <ExpensiveChild />
    </div>
  );
}
```

---

### Q11: Context API - Performance pitfalls and solutions.

**Answer:**

```jsx
// Problem: ALL consumers re-render when ANY context value changes
const AppContext = createContext({ user: null, theme: 'light', locale: 'en' });

function App() {
  const [state, setState] = useState({ user: null, theme: 'light', locale: 'en' });
  return (
    <AppContext.Provider value={state}>
      {/* ALL consumers re-render when ANY part of state changes */}
      <Header />   {/* only needs user */}
      <Sidebar />  {/* only needs theme */}
      <Content />  {/* only needs locale */}
    </AppContext.Provider>
  );
}

// Solution 1: Split contexts
const UserContext = createContext(null);
const ThemeContext = createContext('light');
const LocaleContext = createContext('en');

// Solution 2: Memoize context value
function App() {
  const [user, setUser] = useState(null);
  const [theme, setTheme] = useState('light');

  const value = useMemo(() => ({ user, theme }), [user, theme]);

  return (
    <AppContext.Provider value={value}>
      <Children />
    </AppContext.Provider>
  );
}

// Solution 3: Use Zustand instead (selector-based, no re-render on unrelated changes)
const useStore = create((set) => ({
  user: null,
  theme: 'light',
  setUser: (user) => set({ user }),
}));

function Header() {
  const user = useStore((state) => state.user); // only re-renders when user changes
}
```

---

### Q12: Explain React Server Components (RSC).

**Answer:**

```
Server Components vs Client Components:

Server Components (default in Next.js App Router):
✅ Run on the server only
✅ Can access databases, file system, secrets directly
✅ Zero client JS bundle impact
✅ Can use async/await at component level
✅ Can import Server Components AND Client Components
❌ Cannot use hooks (useState, useEffect)
❌ Cannot use browser APIs
❌ Cannot use event handlers

Client Components ('use client'):
✅ Full React interactivity (hooks, events)
✅ Access to browser APIs
✅ Can import other Client Components
❌ Cannot import Server Components (but can accept them as children)
❌ Add to client JS bundle
```

```jsx
// Server Component (default)
async function TradeHistory({ userId }) {
  // Direct database access - no API needed!
  const trades = await db.trade.findMany({ where: { userId } });
  const summary = await calculatePnL(trades);

  return (
    <div>
      <h2>Trade History</h2>
      <PnLSummary data={summary} />           {/* Server Component */}
      <InteractiveChart data={trades} />       {/* Client Component */}
    </div>
  );
}

// Client Component
'use client';
import { useState } from 'react';

function InteractiveChart({ data }) {
  const [timeframe, setTimeframe] = useState('1D');
  return (
    <div>
      <TimeframeSelector value={timeframe} onChange={setTimeframe} />
      <Chart data={filterByTimeframe(data, timeframe)} />
    </div>
  );
}

// Composition pattern: Server Component wraps Client Component
// The data flows from server → client as serialized props
```

---

## Next.js App Router

### Q13: Explain Next.js rendering strategies.

**Answer:**

```jsx
// 1. STATIC (default) - rendered at build time
// Good for: marketing pages, blog posts, docs
export default function About() {
  return <h1>About Us</h1>;
}

// 2. DYNAMIC - rendered per request
// Triggered by: cookies(), headers(), searchParams, uncached fetch
export default async function Dashboard() {
  const user = await getCurrentUser(); // uses cookies → dynamic
  return <h1>Hi {user.name}</h1>;
}

// Force dynamic
export const dynamic = 'force-dynamic';

// 3. ISR (Incremental Static Regeneration) - static + revalidation
export const revalidate = 60; // rebuild every 60 seconds

export default async function Prices() {
  const prices = await fetch('https://api.prices.com/sol', {
    next: { revalidate: 60 },
  });
  return <PriceTable data={prices} />;
}

// 4. On-demand revalidation
import { revalidatePath, revalidateTag } from 'next/cache';

// In a Server Action or Route Handler:
export async function updateTrade() {
  await db.trade.update(...);
  revalidatePath('/trades');           // revalidate specific path
  revalidateTag('trade-data');         // revalidate by tag
}

// Tag a fetch
fetch(url, { next: { tags: ['trade-data'] } });

// 5. STREAMING - progressive rendering
export default function Page() {
  return (
    <div>
      <Header />  {/* renders immediately */}
      <Suspense fallback={<ChartSkeleton />}>
        <SlowChart />  {/* streams in when ready */}
      </Suspense>
      <Suspense fallback={<TableSkeleton />}>
        <SlowTable />  {/* streams in independently */}
      </Suspense>
    </div>
  );
}
```

---

### Q14: Next.js Caching - Explain all 4 layers.

**Answer:**

| Layer | What | Where | Opt Out |
|---|---|---|---|
| **Request Memoization** | Dedupes same `fetch` in one render tree | Server | N/A (auto) |
| **Data Cache** | Caches `fetch()` responses persistently | Server | `cache: 'no-store'` |
| **Full Route Cache** | Caches rendered HTML + RSC payload | Server | `dynamic = 'force-dynamic'` |
| **Router Cache** | Caches RSC payload in browser | Client | `router.refresh()` |

```jsx
// Request Memoization - automatic deduplication
// Both components fetch same URL → only 1 network request
async function Layout() {
  const user = await fetch('/api/user'); // request 1 → actual fetch
  return <Sidebar user={user}><Page /></Sidebar>;
}
async function Page() {
  const user = await fetch('/api/user'); // request 2 → reuses result
  return <h1>{user.name}</h1>;
}

// Data Cache control
fetch(url);                                 // cached forever (default)
fetch(url, { cache: 'no-store' });         // never cache
fetch(url, { next: { revalidate: 60 } });  // cache for 60s
fetch(url, { next: { tags: ['users'] } }); // tag-based invalidation

// Full Route Cache
// Static routes → cached at build time
// Dynamic routes → not cached
// ISR routes → cached with revalidation

// Router Cache (client-side)
// Prefetched routes cached for 30s (dynamic) or 5min (static)
// Clear with: router.refresh()
```

---

### Q15: Server Actions in Next.js.

**Answer:**

```tsx
// app/actions.ts
'use server';

import { revalidatePath } from 'next/cache';
import { redirect } from 'next/navigation';
import { z } from 'zod';

const TradeSchema = z.object({
  symbol: z.string().min(1),
  amount: z.number().positive(),
  type: z.enum(['buy', 'sell']),
});

export async function createTrade(formData: FormData) {
  // Validate
  const parsed = TradeSchema.safeParse({
    symbol: formData.get('symbol'),
    amount: Number(formData.get('amount')),
    type: formData.get('type'),
  });

  if (!parsed.success) {
    return { error: parsed.error.flatten().fieldErrors };
  }

  // Create in DB
  await db.trade.create({ data: parsed.data });

  // Revalidate and redirect
  revalidatePath('/trades');
  redirect('/trades');
}

// Usage 1: Form action (works without JS!)
export default function TradeForm() {
  return (
    <form action={createTrade}>
      <input name="symbol" placeholder="SOL" required />
      <input name="amount" type="number" required />
      <select name="type">
        <option value="buy">Buy</option>
        <option value="sell">Sell</option>
      </select>
      <SubmitButton />
    </form>
  );
}

// Usage 2: Programmatic with useActionState
'use client';
import { useActionState } from 'react';

function TradeForm() {
  const [state, formAction, isPending] = useActionState(createTrade, null);

  return (
    <form action={formAction}>
      <input name="symbol" />
      {state?.error?.symbol && <p className="error">{state.error.symbol}</p>}
      <button disabled={isPending}>
        {isPending ? 'Placing...' : 'Place Trade'}
      </button>
    </form>
  );
}

// Usage 3: Direct call (not in form)
'use client';
async function handleClick() {
  const result = await createTrade(formData);
  if (result?.error) showToast(result.error);
}
```

---

### Q16: Next.js Middleware.

**Answer:**

```typescript
// middleware.ts (at project root)
import { NextResponse } from 'next/server';
import type { NextRequest } from 'next/server';

export function middleware(request: NextRequest) {
  const { pathname } = request.nextUrl;

  // 1. Auth check
  const token = request.cookies.get('auth-token');
  if (pathname.startsWith('/dashboard') && !token) {
    return NextResponse.redirect(new URL('/login', request.url));
  }

  // 2. Geolocation-based routing
  const country = request.geo?.country || 'US';
  if (pathname === '/' && country === 'PK') {
    return NextResponse.rewrite(new URL('/pk', request.url));
  }

  // 3. Add headers
  const response = NextResponse.next();
  response.headers.set('x-request-id', crypto.randomUUID());
  return response;
}

// Only run on specific paths
export const config = {
  matcher: [
    '/dashboard/:path*',
    '/api/:path*',
    '/((?!_next/static|favicon.ico).*)',
  ],
};

// Middleware runs on EVERY matching request at the Edge
// Use for: auth, redirects, headers, A/B testing, geolocation
// Don't use for: heavy computation, database queries
```

---

### Q17: Parallel Routes and Intercepting Routes.

**Answer:**

```
Parallel Routes (@slot):
─────────────────────────
app/
├── layout.tsx          ← receives @analytics and @trades as props
├── page.tsx
├── @analytics/
│   ├── page.tsx        ← renders in analytics slot
│   └── loading.tsx     ← independent loading state
└── @trades/
    ├── page.tsx        ← renders in trades slot
    └── error.tsx       ← independent error boundary
```

```tsx
// layout.tsx
export default function DashboardLayout({
  children,
  analytics,
  trades,
}: {
  children: React.ReactNode;
  analytics: React.ReactNode;
  trades: React.ReactNode;
}) {
  return (
    <div className="grid grid-cols-2">
      <div>{children}</div>
      <div>{analytics}</div>  {/* loads independently */}
      <div>{trades}</div>     {/* loads independently */}
    </div>
  );
}

// Each slot has its own loading, error, and not-found states
// Slots load in parallel → faster page loads
```

```
Intercepting Routes (.):
─────────────────────────
app/
├── trades/
│   └── [id]/
│       └── page.tsx       ← full page (direct navigation / refresh)
└── @modal/
    └── (.)trades/
        └── [id]/
            └── page.tsx   ← modal (intercepted soft navigation)

(.)  → intercept same level
(..) → intercept one level up
(..)(..)) → two levels up
(...) → intercept from root
```

Use case: Click a trade in a list → opens as modal (intercepted). Refresh or share URL → shows full page.

---

## State Management (Zustand)

### Q18: Zustand - Full setup with middleware.

**Answer:**

```typescript
import { create } from 'zustand';
import { devtools, persist, subscribeWithSelector } from 'zustand/middleware';
import { immer } from 'zustand/middleware/immer';

interface Trade {
  id: string;
  symbol: string;
  amount: number;
  price: number;
  type: 'buy' | 'sell';
}

interface TradeStore {
  // State
  trades: Trade[];
  isLoading: boolean;
  filter: 'all' | 'buy' | 'sell';
  selectedTradeId: string | null;

  // Actions
  addTrade: (trade: Trade) => void;
  removeTrade: (id: string) => void;
  setFilter: (filter: TradeStore['filter']) => void;
  fetchTrades: () => Promise<void>;

  // Computed (using get)
  getFilteredTrades: () => Trade[];
  getTotalValue: () => number;
}

const useTradeStore = create<TradeStore>()(
  devtools(
    persist(
      subscribeWithSelector(
        immer((set, get) => ({
          trades: [],
          isLoading: false,
          filter: 'all',
          selectedTradeId: null,

          addTrade: (trade) =>
            set((state) => {
              state.trades.push(trade); // immer allows mutation
            }),

          removeTrade: (id) =>
            set((state) => {
              state.trades = state.trades.filter((t) => t.id !== id);
            }),

          setFilter: (filter) => set({ filter }),

          fetchTrades: async () => {
            set({ isLoading: true });
            try {
              const response = await fetch('/api/trades');
              const trades = await response.json();
              set({ trades, isLoading: false });
            } catch {
              set({ isLoading: false });
            }
          },

          getFilteredTrades: () => {
            const { trades, filter } = get();
            return filter === 'all'
              ? trades
              : trades.filter((t) => t.type === filter);
          },

          getTotalValue: () => {
            return get().trades.reduce((sum, t) => sum + t.amount * t.price, 0);
          },
        }))
      ),
      {
        name: 'trade-store',
        partialize: (state) => ({
          trades: state.trades,
          filter: state.filter,
        }), // only persist these fields
      }
    ),
    { name: 'TradeStore' }
  )
);

// Usage with selectors (only re-render when selected data changes)
function TradeList() {
  const trades = useTradeStore((s) => s.getFilteredTrades());
  const isLoading = useTradeStore((s) => s.isLoading);
  // ...
}

// Subscribe to changes outside React
useTradeStore.subscribe(
  (state) => state.trades.length,
  (count, prevCount) => {
    if (count > prevCount) showNotification('New trade added!');
  }
);

// Zustand vs Redux:
// ✅ ~1KB vs ~7KB
// ✅ No Provider wrapper needed
// ✅ No boilerplate (actions, reducers, types)
// ✅ Works outside React
// ✅ Selector-based re-renders by default
// ✅ Simple middleware composition
```

---

## Testing

### Q19: Testing React components with React Testing Library.

**Answer:**

```tsx
import { render, screen, fireEvent, waitFor } from '@testing-library/react';
import userEvent from '@testing-library/user-event';

// Test 1: Render and assert
test('renders trade form', () => {
  render(<TradeForm />);

  expect(screen.getByLabelText('Symbol')).toBeInTheDocument();
  expect(screen.getByRole('button', { name: /place trade/i })).toBeInTheDocument();
});

// Test 2: User interaction
test('submits trade form', async () => {
  const onSubmit = jest.fn();
  render(<TradeForm onSubmit={onSubmit} />);

  await userEvent.type(screen.getByLabelText('Symbol'), 'SOL');
  await userEvent.type(screen.getByLabelText('Amount'), '100');
  await userEvent.click(screen.getByRole('button', { name: /place trade/i }));

  expect(onSubmit).toHaveBeenCalledWith({
    symbol: 'SOL',
    amount: 100,
  });
});

// Test 3: Async data loading
test('loads and displays trades', async () => {
  // Mock API
  jest.spyOn(global, 'fetch').mockResolvedValue({
    json: () => Promise.resolve([{ id: '1', symbol: 'SOL', amount: 100 }]),
  });

  render(<TradeList />);

  expect(screen.getByText('Loading...')).toBeInTheDocument();

  await waitFor(() => {
    expect(screen.getByText('SOL')).toBeInTheDocument();
    expect(screen.getByText('100')).toBeInTheDocument();
  });
});

// Test 4: Testing hooks
import { renderHook, act } from '@testing-library/react';

test('useCounter hook', () => {
  const { result } = renderHook(() => useCounter(0));

  expect(result.current.count).toBe(0);

  act(() => result.current.increment());
  expect(result.current.count).toBe(1);
});

// Testing philosophy:
// ✅ Test behavior, not implementation
// ✅ Use accessible queries (getByRole, getByLabelText)
// ❌ Don't test implementation details (state, refs)
// ❌ Don't snapshot test everything
```

---

### Q20: Cypress E2E testing patterns.

**Answer:**

```javascript
// cypress/e2e/trading.cy.ts
describe('Trading Dashboard', () => {
  beforeEach(() => {
    cy.intercept('GET', '/api/trades', { fixture: 'trades.json' }).as('getTrades');
    cy.visit('/dashboard');
    cy.wait('@getTrades');
  });

  it('displays trade list', () => {
    cy.get('[data-testid="trade-list"]').should('be.visible');
    cy.get('[data-testid="trade-row"]').should('have.length.greaterThan', 0);
  });

  it('places a new trade', () => {
    cy.intercept('POST', '/api/trades', { statusCode: 201 }).as('createTrade');

    cy.get('[data-testid="symbol-input"]').type('SOL');
    cy.get('[data-testid="amount-input"]').type('100');
    cy.get('[data-testid="buy-button"]').click();

    cy.wait('@createTrade').its('request.body').should('deep.include', {
      symbol: 'SOL',
      amount: 100,
    });

    cy.get('[data-testid="toast"]').should('contain', 'Trade placed');
  });

  it('handles error gracefully', () => {
    cy.intercept('POST', '/api/trades', { statusCode: 500 }).as('failedTrade');

    cy.get('[data-testid="symbol-input"]').type('SOL');
    cy.get('[data-testid="amount-input"]').type('100');
    cy.get('[data-testid="buy-button"]').click();

    cy.wait('@failedTrade');
    cy.get('[data-testid="error-message"]').should('be.visible');
  });
});

// Custom commands
// cypress/support/commands.ts
Cypress.Commands.add('login', (email, password) => {
  cy.session([email, password], () => {
    cy.visit('/login');
    cy.get('#email').type(email);
    cy.get('#password').type(password);
    cy.get('button[type="submit"]').click();
    cy.url().should('include', '/dashboard');
  });
});
```

---

## References & Deep Dive Resources

### React Core
| Topic | Resource |
|---|---|
| React Official Docs | [react.dev](https://react.dev/) — New official docs (App Router era) |
| Fiber Architecture | [React Fiber Architecture (GitHub)](https://github.com/acdlite/react-fiber-architecture) — Andrew Clark's deep dive |
| Reconciliation | [react.dev - Preserving and Resetting State](https://react.dev/learn/preserving-and-resetting-state) |
| Virtual DOM Explained | [React.js: The Documentary (YouTube)](https://www.youtube.com/watch?v=8pDqJVdNa44) |
| React Rendering Behavior | [Mark Erikson - React Rendering Behavior](https://blog.isquaredsoftware.com/2020/05/blogged-answers-a-mostly-complete-guide-to-react-rendering-behavior/) |

### React Hooks
| Topic | Resource |
|---|---|
| Hooks Reference | [react.dev - Hooks](https://react.dev/reference/react/hooks) |
| useEffect Deep Dive | [Dan Abramov - A Complete Guide to useEffect](https://overreacted.io/a-complete-guide-to-useeffect/) |
| useMemo/useCallback | [Kent C. Dodds - When to useMemo and useCallback](https://kentcdodds.com/blog/usememo-and-usecallback) |
| useRef | [react.dev - useRef](https://react.dev/reference/react/useRef) |
| useReducer | [react.dev - useReducer](https://react.dev/reference/react/useReducer) |
| Custom Hooks | [react.dev - Reusing Logic with Custom Hooks](https://react.dev/learn/reusing-logic-with-custom-hooks) |

### React Performance
| Topic | Resource |
|---|---|
| React.memo | [react.dev - memo](https://react.dev/reference/react/memo) |
| React DevTools Profiler | [react.dev - Profiler](https://react.dev/reference/react/Profiler) |
| React Compiler | [react.dev - React Compiler](https://react.dev/learn/react-compiler) — Auto-memoization |
| Million.js | [million.dev](https://million.dev/) — React performance optimization |

### React Concurrent Features
| Topic | Resource |
|---|---|
| useTransition | [react.dev - useTransition](https://react.dev/reference/react/useTransition) |
| useDeferredValue | [react.dev - useDeferredValue](https://react.dev/reference/react/useDeferredValue) |
| Suspense | [react.dev - Suspense](https://react.dev/reference/react/Suspense) |
| Error Boundaries | [react.dev - Error Boundary](https://react.dev/reference/react/Component#catching-rendering-errors-with-an-error-boundary) |

### Next.js
| Topic | Resource |
|---|---|
| Next.js Docs | [nextjs.org/docs](https://nextjs.org/docs) — Official documentation |
| App Router | [nextjs.org - App Router](https://nextjs.org/docs/app) |
| Server Components | [nextjs.org - Server Components](https://nextjs.org/docs/app/building-your-application/rendering/server-components) |
| Caching | [nextjs.org - Caching](https://nextjs.org/docs/app/building-your-application/caching) |
| Server Actions | [nextjs.org - Server Actions](https://nextjs.org/docs/app/building-your-application/data-fetching/server-actions-and-mutations) |
| Middleware | [nextjs.org - Middleware](https://nextjs.org/docs/app/building-your-application/routing/middleware) |
| Parallel Routes | [nextjs.org - Parallel Routes](https://nextjs.org/docs/app/building-your-application/routing/parallel-routes) |
| Lee Robinson (YouTube) | [Lee Robinson](https://www.youtube.com/@laborb) — Next.js core team tutorials |
| Vercel Blog | [vercel.com/blog](https://vercel.com/blog) — Latest Next.js features |

### State Management
| Topic | Resource |
|---|---|
| Zustand Docs | [zustand GitHub](https://github.com/pmndrs/zustand) |
| Zustand Tutorial | [TkDodo - Working with Zustand](https://tkdodo.eu/blog/working-with-zustand) |
| Context vs Zustand | [react.dev - Scaling Up with Context](https://react.dev/learn/scaling-up-with-reducer-and-context) |

### Testing
| Topic | Resource |
|---|---|
| Testing Library | [testing-library.com](https://testing-library.com/docs/react-testing-library/intro/) |
| Kent C. Dodds Testing | [Testing JavaScript](https://testingjavascript.com/) |
| Cypress Docs | [docs.cypress.io](https://docs.cypress.io/) |
| Jest Docs | [jestjs.io](https://jestjs.io/docs/getting-started) |

---

> **Back to main**: [INTERVIEW_ROADMAP.md](../INTERVIEW_ROADMAP.md)
> **Prev**: [JavaScript & TypeScript](./01-javascript-typescript.md) | **Next**: [Node.js, NestJS & Express](./03-nodejs-nestjs.md)
