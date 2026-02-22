# Interview Questions & Answers - Muhammad Awais Shah

> Comprehensive Q&A covering MERN, AWS, Web3, AI, System Design & More

---

## Table of Contents

1. [JavaScript & TypeScript](#1-javascript--typescript)
2. [React & Next.js](#2-react--nextjs)
3. [Node.js, NestJS & Express](#3-nodejs-nestjs--express)
4. [Databases (MongoDB & PostgreSQL)](#4-databases-mongodb--postgresql)
5. [System Design & Architecture](#5-system-design--architecture)
6. [AWS Cloud Services](#6-aws-cloud-services)
7. [DevOps, Docker & CI/CD](#7-devops-docker--cicd)
8. [Web3 & Blockchain](#8-web3--blockchain)
9. [AI Integration](#9-ai-integration)
10. [Security & Authentication](#10-security--authentication)
11. [Behavioral Questions](#11-behavioral-questions)
12. [Coding Patterns Cheat Sheet](#12-coding-patterns-cheat-sheet)

---

## 1. JavaScript & TypeScript

### Q1: Explain the JavaScript Event Loop with microtasks and macrotasks.

**Answer:**
The Event Loop is the mechanism that allows JavaScript to be non-blocking despite being single-threaded.

**Execution Order:**
1. **Call Stack** - Executes synchronous code
2. **Microtask Queue** - Processes ALL microtasks (Promises, queueMicrotask, MutationObserver)
3. **Macrotask Queue** - Processes ONE macrotask (setTimeout, setInterval, I/O, UI rendering)
4. Repeat: After each macrotask, drain all microtasks again

```javascript
console.log('1'); // sync

setTimeout(() => console.log('2'), 0); // macrotask

Promise.resolve().then(() => console.log('3')); // microtask

Promise.resolve().then(() => {
  console.log('4'); // microtask
  setTimeout(() => console.log('5'), 0); // macrotask (queued from microtask)
});

console.log('6'); // sync

// Output: 1, 6, 3, 4, 2, 5
```

**Why it matters:** Understanding this is critical for debugging async code, avoiding race conditions, and optimizing performance in Node.js and browser environments.

---

### Q2: What are closures? Give a practical example.

**Answer:**
A closure is a function that retains access to its lexical scope even when executed outside that scope.

```javascript
// Practical: Creating a rate limiter
function createRateLimiter(maxCalls, timeWindow) {
  let calls = []; // closed over

  return function () {
    const now = Date.now();
    calls = calls.filter(time => now - time < timeWindow);

    if (calls.length >= maxCalls) {
      throw new Error('Rate limit exceeded');
    }

    calls.push(now);
    return true;
  };
}

const limiter = createRateLimiter(5, 60000); // 5 calls per minute
limiter(); // true
```

**Common interview trap:**
```javascript
// Classic loop problem
for (var i = 0; i < 3; i++) {
  setTimeout(() => console.log(i), 100); // prints 3, 3, 3
}

// Fix with closure
for (var i = 0; i < 3; i++) {
  ((j) => setTimeout(() => console.log(j), 100))(i); // prints 0, 1, 2
}

// Modern fix: use let (block-scoped)
for (let i = 0; i < 3; i++) {
  setTimeout(() => console.log(i), 100); // prints 0, 1, 2
}
```

---

### Q3: Explain Prototypal Inheritance vs Class Inheritance.

**Answer:**
JavaScript uses **prototypal inheritance** - objects inherit directly from other objects via the prototype chain.

```javascript
// Prototypal
const animal = {
  speak() { return `${this.name} makes a sound`; }
};

const dog = Object.create(animal);
dog.name = 'Rex';
dog.speak(); // "Rex makes a sound"

// Class syntax (syntactic sugar over prototypes)
class Animal {
  constructor(name) { this.name = name; }
  speak() { return `${this.name} makes a sound`; }
}

class Dog extends Animal {
  bark() { return `${this.name} barks`; }
}

// Under the hood, Dog.prototype.__proto__ === Animal.prototype
```

**Key difference:** Classical inheritance (Java/C++) copies properties. Prototypal inheritance creates a chain of references - changes to the prototype affect all instances.

---

### Q4: Explain `this` in different contexts.

**Answer:**

| Context | `this` refers to |
|---|---|
| Global scope | `window` (browser) / `global` (Node) / `undefined` (strict) |
| Object method | The object calling the method |
| Arrow function | Inherits `this` from enclosing lexical scope |
| `new` keyword | The newly created instance |
| `call/apply/bind` | The explicitly passed object |
| Event handler | The DOM element (unless arrow function) |
| Class method | The class instance |

```javascript
const obj = {
  name: 'Awais',
  regular: function() { return this.name; },    // 'Awais'
  arrow: () => this.name,                         // undefined (inherits outer this)
  nested: function() {
    const inner = () => this.name;                // 'Awais' (arrow captures obj's this)
    return inner();
  }
};

// Explicit binding
function greet() { return `Hi, ${this.name}`; }
greet.call({ name: 'Awais' });  // "Hi, Awais"
greet.apply({ name: 'Awais' }); // "Hi, Awais"
const bound = greet.bind({ name: 'Awais' });
bound(); // "Hi, Awais"
```

---

### Q5: Promise.all vs Promise.allSettled vs Promise.race vs Promise.any

**Answer:**

```javascript
const p1 = Promise.resolve(1);
const p2 = Promise.reject('error');
const p3 = Promise.resolve(3);

// Promise.all - fails fast on ANY rejection
Promise.all([p1, p2, p3]).catch(e => e); // 'error'

// Promise.allSettled - waits for ALL, never rejects
Promise.allSettled([p1, p2, p3]);
// [{status:'fulfilled', value:1}, {status:'rejected', reason:'error'}, {status:'fulfilled', value:3}]

// Promise.race - first to settle (resolve OR reject) wins
Promise.race([p1, p2, p3]); // 1 (p1 resolves first)

// Promise.any - first to RESOLVE wins, ignores rejections
Promise.any([p2, p1, p3]); // 1 (first successful)
```

**When to use:**
- `all` - Parallel requests where ALL must succeed (e.g., loading dashboard data)
- `allSettled` - Independent operations, handle each result (e.g., sending notifications)
- `race` - Timeout patterns, first-response caching
- `any` - Fallback APIs, fastest mirror selection

---

### Q6: TypeScript Generics - Explain with practical examples.

**Answer:**

```typescript
// Basic generic
function identity<T>(arg: T): T { return arg; }

// Constrained generic
function getProperty<T, K extends keyof T>(obj: T, key: K): T[K] {
  return obj[key];
}

// Generic interface for API responses
interface ApiResponse<T> {
  data: T;
  status: number;
  message: string;
}

// Generic with default
interface PaginatedResponse<T, M = Record<string, unknown>> {
  items: T[];
  total: number;
  page: number;
  meta?: M;
}

// Conditional types
type IsString<T> = T extends string ? 'yes' : 'no';
type A = IsString<string>;  // 'yes'
type B = IsString<number>;  // 'no'

// infer keyword - extract return type
type ReturnOf<T> = T extends (...args: any[]) => infer R ? R : never;
type Result = ReturnOf<() => string>; // string

// Mapped types
type Readonly<T> = { readonly [K in keyof T]: T[K] };
type Optional<T> = { [K in keyof T]?: T[K] };

// Template literal types
type HttpMethod = 'GET' | 'POST' | 'PUT' | 'DELETE';
type Endpoint = `/api/${string}`;
type Route = `${HttpMethod} ${Endpoint}`; // "GET /api/users" etc.
```

---

### Q7: What is the `satisfies` operator in TypeScript?

**Answer:**

```typescript
// satisfies validates the type WITHOUT widening it
type Colors = Record<string, string | string[]>;

// With type annotation - loses specificity
const colors1: Colors = {
  red: '#ff0000',
  blue: ['#0000ff', '#0000cc'],
};
colors1.red.toUpperCase(); // ERROR: string | string[] has no toUpperCase

// With satisfies - keeps narrow type AND validates
const colors2 = {
  red: '#ff0000',
  blue: ['#0000ff', '#0000cc'],
} satisfies Colors;
colors2.red.toUpperCase(); // OK: TypeScript knows it's a string
colors2.blue.map(c => c);  // OK: TypeScript knows it's string[]
```

---

### Q8: Explain WeakMap and WeakSet. When would you use them?

**Answer:**
WeakMap and WeakSet hold "weak" references - they don't prevent garbage collection of their keys/values.

```javascript
// WeakMap - keys must be objects, keys are weakly held
const cache = new WeakMap();

function processUser(user) {
  if (cache.has(user)) return cache.get(user);

  const result = expensiveComputation(user);
  cache.set(user, result);
  return result;
}

let user = { name: 'Awais' };
processUser(user);
user = null; // user object AND cache entry can be garbage collected

// Use cases:
// 1. Caching without memory leaks
// 2. Storing private data for objects
// 3. DOM element metadata (auto-cleanup when element removed)
// 4. Tracking object references without preventing GC
```

**Key difference from Map:** WeakMap is NOT iterable, has no `.size`, keys must be objects. This is by design - you can't enumerate weak references.

---

## 2. React & Next.js

### Q9: Explain React Fiber Architecture and Reconciliation.

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
// Without keys: React re-renders ALL items on reorder
// With keys: React moves DOM nodes efficiently
{items.map(item => <Item key={item.id} {...item} />)}
```

---

### Q10: When to use useMemo, useCallback, and React.memo?

**Answer:**

```jsx
// React.memo - Memoize a COMPONENT (skip re-render if props unchanged)
const ExpensiveList = React.memo(({ items, onSelect }) => {
  return items.map(item => (
    <div key={item.id} onClick={() => onSelect(item.id)}>
      {item.name}
    </div>
  ));
});

// useCallback - Memoize a FUNCTION (stable reference between renders)
function Parent() {
  const [count, setCount] = useState(0);

  // Without useCallback: new function every render → ExpensiveList re-renders
  const handleSelect = useCallback((id) => {
    console.log(id);
  }, []); // stable reference

  return <ExpensiveList items={items} onSelect={handleSelect} />;
}

// useMemo - Memoize a VALUE (skip expensive computation)
function Dashboard({ trades }) {
  const totalPnL = useMemo(() => {
    return trades.reduce((sum, t) => sum + t.pnl, 0); // expensive
  }, [trades]);

  return <div>PnL: {totalPnL}</div>;
}
```

**When NOT to use:**
- Don't wrap everything - memoization has a cost (memory + comparison)
- Don't use for primitive props (cheap to compare)
- Don't use if the component is already fast
- Profile first, optimize second

---

### Q11: Explain Next.js App Router - Server Components vs Client Components.

**Answer:**

```
Server Components (default)          Client Components ('use client')
─────────────────────────────        ────────────────────────────────
✅ Access backend resources directly  ✅ useState, useEffect, hooks
✅ Keep secrets on server             ✅ Browser APIs (window, document)
✅ Reduce client bundle size          ✅ Event handlers (onClick, onChange)
✅ Async/await in component body      ✅ Interactive UI
❌ No hooks (useState, useEffect)     ❌ Larger bundle size
❌ No browser APIs                    ❌ No direct DB/file access
❌ No event handlers
```

```jsx
// Server Component (default in App Router)
async function UserProfile({ id }) {
  const user = await db.user.findUnique({ where: { id } }); // Direct DB access!
  return <div>{user.name}</div>;
}

// Client Component
'use client';
import { useState } from 'react';

function Counter() {
  const [count, setCount] = useState(0);
  return <button onClick={() => setCount(c => c + 1)}>{count}</button>;
}

// Composition pattern - Server wraps Client
async function Page() {
  const data = await fetchData(); // server
  return <InteractiveChart data={data} />; // client component receives server data
}
```

**Rendering strategies:**
- **SSR (Dynamic)**: `export const dynamic = 'force-dynamic'` - rendered per request
- **SSG (Static)**: Default for no dynamic data - rendered at build time
- **ISR**: `export const revalidate = 60` - rebuild every 60 seconds

---

### Q12: How does Next.js caching work in App Router?

**Answer:**

There are **4 caching layers**:

| Cache | What | Where | Duration |
|---|---|---|---|
| **Request Memoization** | Deduplicates same fetch calls in a single render | Server | Per request |
| **Data Cache** | Caches fetch() responses | Server | Persistent (revalidate) |
| **Full Route Cache** | Caches rendered HTML + RSC payload | Server | Persistent (revalidate) |
| **Router Cache** | Caches RSC payload in browser | Client | Session (30s dynamic, 5min static) |

```jsx
// Opt out of caching
fetch(url, { cache: 'no-store' }); // no data cache

// Revalidate every 60 seconds (ISR)
fetch(url, { next: { revalidate: 60 } });

// Revalidate on demand
import { revalidatePath, revalidateTag } from 'next/cache';
revalidatePath('/dashboard');
revalidateTag('user-data');

// Tag a fetch for on-demand revalidation
fetch(url, { next: { tags: ['user-data'] } });
```

---

### Q13: Explain Server Actions in Next.js.

**Answer:**
Server Actions are async functions that execute on the server, callable from client components.

```jsx
// app/actions.ts
'use server';

import { revalidatePath } from 'next/cache';

export async function createTrade(formData: FormData) {
  const symbol = formData.get('symbol');
  const amount = formData.get('amount');

  await db.trade.create({ data: { symbol, amount: Number(amount) } });
  revalidatePath('/trades');
}

// app/trades/page.tsx (Client Component using Server Action)
'use client';
import { createTrade } from '../actions';

function TradeForm() {
  return (
    <form action={createTrade}>
      <input name="symbol" />
      <input name="amount" type="number" />
      <button type="submit">Place Trade</button>
    </form>
  );
}

// Can also call programmatically
async function handleClick() {
  const formData = new FormData();
  formData.set('symbol', 'SOL');
  await createTrade(formData);
}
```

**Benefits:** No API route needed, automatic form handling, works without JS, progressive enhancement.

---

### Q14: How does Zustand work and why use it over Redux?

**Answer:**

```typescript
import { create } from 'zustand';
import { persist, devtools } from 'zustand/middleware';

interface TradeStore {
  trades: Trade[];
  isLoading: boolean;
  addTrade: (trade: Trade) => void;
  fetchTrades: () => Promise<void>;
}

const useTradeStore = create<TradeStore>()(
  devtools(
    persist(
      (set, get) => ({
        trades: [],
        isLoading: false,

        addTrade: (trade) =>
          set((state) => ({ trades: [...state.trades, trade] })),

        fetchTrades: async () => {
          set({ isLoading: true });
          const trades = await api.getTrades();
          set({ trades, isLoading: false });
        },
      }),
      { name: 'trade-store' } // localStorage key
    )
  )
);

// Usage in component - auto re-renders on change
function TradeList() {
  const trades = useTradeStore((state) => state.trades); // selector
  const fetchTrades = useTradeStore((state) => state.fetchTrades);
  // ...
}
```

**Zustand vs Redux:**

| Feature | Zustand | Redux |
|---|---|---|
| Boilerplate | Minimal | Heavy (actions, reducers, types) |
| Bundle size | ~1KB | ~7KB + toolkit |
| Learning curve | Low | High |
| Middleware | Simple composition | Complex middleware chain |
| DevTools | Optional plugin | Built-in |
| TypeScript | Natural | Verbose type definitions |
| React dependency | Optional | Required |

---

## 3. Node.js, NestJS & Express

### Q15: Explain the Node.js Event Loop phases.

**Answer:**

```
   ┌───────────────────────────┐
┌─>│           timers          │  ← setTimeout, setInterval callbacks
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
│  │     pending callbacks     │  ← I/O callbacks deferred to next iteration
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
│  │       idle, prepare       │  ← internal use only
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
│  │           poll            │  ← retrieve new I/O events, execute I/O callbacks
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
│  │           check           │  ← setImmediate callbacks
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
│  │      close callbacks      │  ← socket.on('close'), etc.
│  └───────────────────────────┘

Between each phase: process ALL microtasks (Promise callbacks, process.nextTick)
process.nextTick runs BEFORE Promise microtasks
```

```javascript
// Demonstrate execution order
setTimeout(() => console.log('timeout'), 0);       // timers phase
setImmediate(() => console.log('immediate'));        // check phase
process.nextTick(() => console.log('nextTick'));     // microtask (highest priority)
Promise.resolve().then(() => console.log('promise')); // microtask

// Output: nextTick, promise, timeout (or immediate), immediate (or timeout)
// Note: timeout vs immediate order is non-deterministic in main module
// But inside an I/O callback, setImmediate ALWAYS fires first
```

---

### Q16: Explain NestJS Dependency Injection and Module System.

**Answer:**

```typescript
// NestJS uses an IoC (Inversion of Control) container

// 1. Define a service (Provider)
@Injectable()
export class TradeService {
  constructor(
    @InjectRepository(Trade) private tradeRepo: Repository<Trade>,
    private readonly priceService: PriceService,  // auto-injected
  ) {}

  async executeTrade(dto: CreateTradeDto) {
    const price = await this.priceService.getPrice(dto.symbol);
    return this.tradeRepo.save({ ...dto, price });
  }
}

// 2. Define a module
@Module({
  imports: [TypeOrmModule.forFeature([Trade]), PriceModule],
  controllers: [TradeController],
  providers: [TradeService],
  exports: [TradeService], // available to other modules that import TradeModule
})
export class TradeModule {}

// 3. Custom providers
@Module({
  providers: [
    // Value provider
    { provide: 'API_KEY', useValue: process.env.API_KEY },

    // Factory provider
    {
      provide: 'REDIS_CLIENT',
      useFactory: async (config: ConfigService) => {
        return createRedisClient(config.get('REDIS_URL'));
      },
      inject: [ConfigService],
    },

    // Class provider (useful for testing)
    { provide: TradeService, useClass: MockTradeService },
  ],
})
export class AppModule {}
```

**Provider scopes:**
- `DEFAULT` - Singleton (one instance for entire app)
- `REQUEST` - New instance per request
- `TRANSIENT` - New instance every time it's injected

---

### Q17: Explain NestJS Guards, Interceptors, Pipes, and Filters execution order.

**Answer:**

```
Request → Middleware → Guards → Interceptors (before) → Pipes → Handler → Interceptors (after) → Filters (on error)
```

```typescript
// Guard - Authentication/Authorization (returns boolean)
@Injectable()
export class JwtAuthGuard implements CanActivate {
  canActivate(context: ExecutionContext): boolean {
    const request = context.switchToHttp().getRequest();
    const token = request.headers.authorization?.split(' ')[1];
    if (!token) throw new UnauthorizedException();
    return this.jwtService.verify(token);
  }
}

// Interceptor - Transform response, logging, caching
@Injectable()
export class TransformInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    const now = Date.now();
    return next.handle().pipe(
      map(data => ({
        data,
        statusCode: 200,
        timestamp: new Date().toISOString(),
        duration: `${Date.now() - now}ms`,
      })),
    );
  }
}

// Pipe - Validation and transformation
@Injectable()
export class ValidationPipe implements PipeTransform {
  transform(value: any, metadata: ArgumentMetadata) {
    // validate and transform incoming data
    return plainToInstance(metadata.metatype, value);
  }
}

// Exception Filter - Error handling
@Catch(HttpException)
export class HttpExceptionFilter implements ExceptionFilter {
  catch(exception: HttpException, host: ArgumentHost) {
    const ctx = host.switchToHttp();
    const response = ctx.getResponse();
    response.status(exception.getStatus()).json({
      statusCode: exception.getStatus(),
      message: exception.message,
    });
  }
}

// Apply them
@Controller('trades')
@UseGuards(JwtAuthGuard)
@UseInterceptors(TransformInterceptor)
export class TradeController {
  @Post()
  @UsePipes(new ValidationPipe())
  create(@Body() dto: CreateTradeDto) { ... }
}
```

---

### Q18: How do you handle Microservices in NestJS?

**Answer:**

```typescript
// 1. Create a microservice (TCP transport)
// trade-service/main.ts
const app = await NestFactory.createMicroservice(TradeModule, {
  transport: Transport.TCP,
  options: { host: '0.0.0.0', port: 3001 },
});

// 2. Message patterns
@Controller()
export class TradeController {
  // Request-Response pattern
  @MessagePattern({ cmd: 'get_trade' })
  getTrade(@Payload() data: { id: string }) {
    return this.tradeService.findById(data.id);
  }

  // Event-based pattern (fire and forget)
  @EventPattern('trade_executed')
  handleTradeExecuted(@Payload() data: TradeEvent) {
    this.analyticsService.record(data);
  }
}

// 3. Client side (API Gateway)
@Module({
  imports: [
    ClientsModule.register([
      {
        name: 'TRADE_SERVICE',
        transport: Transport.TCP,
        options: { host: 'trade-service', port: 3001 },
      },
    ]),
  ],
})
export class GatewayModule {}

@Controller('trades')
export class GatewayController {
  constructor(@Inject('TRADE_SERVICE') private client: ClientProxy) {}

  @Get(':id')
  getTrade(@Param('id') id: string) {
    return this.client.send({ cmd: 'get_trade' }, { id });
  }
}

// Transport options: TCP, Redis, NATS, MQTT, gRPC, Kafka, RabbitMQ
```

---

### Q19: Explain Node.js Streams with a practical example.

**Answer:**

```javascript
import { createReadStream, createWriteStream } from 'fs';
import { Transform, pipeline } from 'stream';
import { promisify } from 'util';

const pipelineAsync = promisify(pipeline);

// Transform stream - process CSV line by line
const csvParser = new Transform({
  objectMode: true,
  transform(chunk, encoding, callback) {
    const lines = chunk.toString().split('\n');
    for (const line of lines) {
      const [symbol, price, volume] = line.split(',');
      this.push({ symbol, price: Number(price), volume: Number(volume) });
    }
    callback();
  },
});

// Process a large trade file without loading it all into memory
await pipelineAsync(
  createReadStream('trades.csv'),        // Readable
  csvParser,                              // Transform
  async function* (source) {             // Async generator transform
    for await (const trade of source) {
      if (trade.volume > 1000) {
        yield JSON.stringify(trade) + '\n';
      }
    }
  },
  createWriteStream('large-trades.json'), // Writable
);

// Stream types:
// Readable  - fs.createReadStream, http.IncomingMessage
// Writable  - fs.createWriteStream, http.ServerResponse
// Transform - zlib.createGzip, crypto.createCipheriv
// Duplex    - net.Socket, WebSocket
```

**Why streams matter:** Process GBs of data with constant memory (~16KB buffer). Critical for file processing, real-time data, and API proxying.

---

## 4. Databases (MongoDB & PostgreSQL)

### Q20: MongoDB Aggregation Pipeline - Explain with a trading example.

**Answer:**

```javascript
// Calculate daily PnL per trader with moving average
db.trades.aggregate([
  // Stage 1: Filter recent trades
  { $match: {
    executedAt: { $gte: new Date('2024-01-01') },
    status: 'completed'
  }},

  // Stage 2: Group by trader and day
  { $group: {
    _id: {
      trader: '$traderId',
      day: { $dateToString: { format: '%Y-%m-%d', date: '$executedAt' } }
    },
    dailyPnL: { $sum: '$pnl' },
    tradeCount: { $sum: 1 },
    avgTradeSize: { $avg: '$amount' },
    symbols: { $addToSet: '$symbol' }
  }},

  // Stage 3: Sort by date
  { $sort: { '_id.day': 1 } },

  // Stage 4: Window function (moving average) - MongoDB 5.0+
  { $setWindowFields: {
    partitionBy: '$_id.trader',
    sortBy: { '_id.day': 1 },
    output: {
      movingAvgPnL: {
        $avg: '$dailyPnL',
        window: { documents: [-6, 0] } // 7-day moving average
      }
    }
  }},

  // Stage 5: Reshape output
  { $project: {
    trader: '$_id.trader',
    date: '$_id.day',
    dailyPnL: 1,
    tradeCount: 1,
    movingAvgPnL: { $round: ['$movingAvgPnL', 2] },
    _id: 0
  }}
]);
```

**Performance tips:**
- Put `$match` and `$project` early to reduce data flowing through pipeline
- Create indexes that support your `$match` and `$sort` stages
- Use `$lookup` sparingly (it's essentially a LEFT JOIN)
- Consider `allowDiskUse: true` for large aggregations

---

### Q21: MongoDB Schema Design - Embedding vs Referencing.

**Answer:**

```javascript
// EMBEDDING (Denormalized) - when data is accessed together
// Good for: 1:1, 1:few relationships, read-heavy
{
  _id: ObjectId("..."),
  name: "Awais",
  // Embedded - always fetched with user
  address: {
    street: "123 Main St",
    city: "Islamabad"
  },
  // Embedded array - bounded, small
  recentTrades: [
    { symbol: "SOL", amount: 100, date: new Date() },
    { symbol: "ETH", amount: 50, date: new Date() }
  ]
}

// REFERENCING (Normalized) - when data is large or frequently updated independently
// Good for: 1:many, many:many, unbounded arrays
{
  _id: ObjectId("..."),
  name: "Awais",
  tradeIds: [ObjectId("..."), ObjectId("...")] // references
}

// HYBRID - embed summary, reference details
{
  _id: ObjectId("..."),
  name: "Awais",
  tradeStats: { totalTrades: 150, winRate: 0.65 }, // embedded summary
  // Full trade history in separate collection
}
```

**Decision matrix:**

| Factor | Embed | Reference |
|---|---|---|
| Read together? | Yes → Embed | No → Reference |
| Array size | Bounded (<100) → Embed | Unbounded → Reference |
| Update frequency | Rarely → Embed | Often → Reference |
| Document size | < 16MB → Embed | Large → Reference |
| Duplication OK? | Yes → Embed | No → Reference |

---

### Q22: PostgreSQL Query Optimization - EXPLAIN ANALYZE.

**Answer:**

```sql
-- Slow query
EXPLAIN ANALYZE
SELECT t.*, u.name
FROM trades t
JOIN users u ON t.user_id = u.id
WHERE t.symbol = 'SOL'
AND t.executed_at > '2024-01-01'
ORDER BY t.executed_at DESC
LIMIT 50;

-- Reading EXPLAIN output:
-- Seq Scan → BAD for large tables (needs index)
-- Index Scan → GOOD (using index)
-- Bitmap Index Scan → OK (multiple conditions)
-- Sort → expensive if no index supports ORDER BY
-- Nested Loop → OK for small result sets
-- Hash Join → better for large result sets

-- Fix: Create composite index
CREATE INDEX idx_trades_symbol_date
ON trades (symbol, executed_at DESC)
INCLUDE (user_id, amount, price); -- covering index

-- Common optimization techniques:
-- 1. Composite indexes matching WHERE + ORDER BY
-- 2. Partial indexes for filtered queries
CREATE INDEX idx_active_trades ON trades (executed_at)
WHERE status = 'active';

-- 3. Expression indexes
CREATE INDEX idx_trades_lower_symbol ON trades (LOWER(symbol));

-- 4. Connection pooling (PgBouncer or built-in pool)
-- 5. Partitioning for very large tables
CREATE TABLE trades (
  id SERIAL,
  executed_at TIMESTAMPTZ,
  ...
) PARTITION BY RANGE (executed_at);

CREATE TABLE trades_2024_q1 PARTITION OF trades
FOR VALUES FROM ('2024-01-01') TO ('2024-04-01');
```

---

### Q23: PostgreSQL Window Functions - Practical examples.

**Answer:**

```sql
-- 1. Running total of trades per user
SELECT
  user_id,
  executed_at,
  amount,
  SUM(amount) OVER (
    PARTITION BY user_id
    ORDER BY executed_at
    ROWS UNBOUNDED PRECEDING
  ) AS running_total
FROM trades;

-- 2. Rank traders by PnL
SELECT
  user_id,
  total_pnl,
  RANK() OVER (ORDER BY total_pnl DESC) AS rank,
  DENSE_RANK() OVER (ORDER BY total_pnl DESC) AS dense_rank,
  ROW_NUMBER() OVER (ORDER BY total_pnl DESC) AS row_num
FROM trader_stats;

-- 3. Moving average (7-day)
SELECT
  date,
  daily_pnl,
  AVG(daily_pnl) OVER (
    ORDER BY date
    ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
  ) AS moving_avg_7d
FROM daily_stats;

-- 4. Lead/Lag - compare with previous trade
SELECT
  symbol,
  price,
  LAG(price) OVER (PARTITION BY symbol ORDER BY executed_at) AS prev_price,
  price - LAG(price) OVER (PARTITION BY symbol ORDER BY executed_at) AS price_change
FROM trades;

-- 5. First/Last value in group
SELECT DISTINCT
  symbol,
  FIRST_VALUE(price) OVER w AS open_price,
  LAST_VALUE(price) OVER w AS close_price,
  MAX(price) OVER w AS high,
  MIN(price) OVER w AS low
FROM trades
WINDOW w AS (
  PARTITION BY symbol, DATE(executed_at)
  ORDER BY executed_at
  ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
);
```

---

## 5. System Design & Architecture

### Q24: Design a Real-time Trading Platform (based on your experience).

**Answer:**

```
Architecture Overview:
═══════════════════════════════════════════════════════════════

┌──────────┐     ┌──────────────┐     ┌──────────────────┐
│  Client   │────▶│  CloudFront  │────▶│  Next.js (SSR)   │
│ (React)   │◀────│     CDN      │◀────│  + API Routes    │
└──────────┘     └──────────────┘     └──────────────────┘
     │                                         │
     │ WebSocket/SSE                           │ REST/gRPC
     ▼                                         ▼
┌──────────────┐                    ┌──────────────────┐
│  WebSocket   │                    │   API Gateway    │
│   Server     │                    │   (NestJS)       │
│  (Pusher/    │                    │                  │
│   custom)    │                    └──────┬───────────┘
└──────────────┘                           │
                                    ┌──────┴───────────┐
                              ┌─────┤  Trade Service   ├─────┐
                              │     │  (NestJS)        │     │
                              │     └──────────────────┘     │
                              ▼                               ▼
                    ┌──────────────┐              ┌──────────────┐
                    │  PostgreSQL  │              │    Redis      │
                    │  (Trades,    │              │  (Cache,      │
                    │   Orders)    │              │   Sessions,   │
                    └──────────────┘              │   Rate Limit) │
                              │                   └──────────────┘
                              ▼
                    ┌──────────────┐     ┌──────────────────┐
                    │  SNS Topic   │────▶│  SQS Queues      │
                    │  (Fan-out)   │     │  ├─ Analytics     │
                    └──────────────┘     │  ├─ Notifications │
                                         │  ├─ Risk Check    │
                                         │  └─ Audit Log     │
                                         └──────────────────┘

Key Design Decisions:
1. PostgreSQL for trades (ACID, complex queries, window functions for analytics)
2. Redis for real-time price cache, session management, rate limiting
3. SNS + SQS fan-out for async processing (your proven pattern)
4. WebSocket/SSE for real-time price updates to clients
5. NestJS microservices for trade execution, risk management, analytics
6. Docker + ECS for deployment, auto-scaling on trade volume
```

**Scaling considerations:**
- Horizontal scaling of API servers behind ALB
- Read replicas for PostgreSQL reporting queries
- Redis Cluster for high-throughput price caching
- SQS for backpressure handling during high-volume periods
- Circuit breaker pattern for external price feed APIs

---

### Q25: Explain the SNS + SQS Fan-out Pattern (your specialty).

**Answer:**

```
                         ┌─────────────────┐
  Trade Executed ───────▶│   SNS Topic     │
                         │ "trade-events"  │
                         └────────┬────────┘
                                  │
                    ┌─────────────┼─────────────┐
                    ▼             ▼              ▼
             ┌───────────┐ ┌───────────┐ ┌───────────┐
             │ SQS Queue │ │ SQS Queue │ │ SQS Queue │
             │ Analytics │ │  Notify   │ │  Audit    │
             └─────┬─────┘ └─────┬─────┘ └─────┬─────┘
                   ▼             ▼              ▼
             ┌───────────┐ ┌───────────┐ ┌───────────┐
             │ Analytics │ │ Notif.    │ │ Audit     │
             │ Service   │ │ Service   │ │ Service   │
             └───────────┘ └───────────┘ └───────────┘
```

```typescript
// Publisher (NestJS)
@Injectable()
export class TradeEventPublisher {
  constructor(private readonly snsClient: SNSClient) {}

  async publishTradeEvent(trade: Trade) {
    await this.snsClient.send(new PublishCommand({
      TopicArn: process.env.TRADE_EVENTS_TOPIC_ARN,
      Message: JSON.stringify({
        tradeId: trade.id,
        symbol: trade.symbol,
        amount: trade.amount,
        type: trade.type,
        timestamp: new Date().toISOString(),
      }),
      MessageAttributes: {
        eventType: { DataType: 'String', StringValue: 'TRADE_EXECUTED' },
        symbol: { DataType: 'String', StringValue: trade.symbol },
      },
    }));
  }
}

// Consumer (Analytics Service)
@Injectable()
export class AnalyticsConsumer {
  @SqsMessageHandler('analytics-queue', false)
  async handleMessage(message: SQSMessage) {
    const trade = JSON.parse(JSON.parse(message.Body).Message);
    await this.analyticsService.recordTrade(trade);
  }
}
```

**Benefits:**
- **Decoupling** - Publisher doesn't know about consumers
- **Independent scaling** - Each consumer scales independently
- **Fault tolerance** - DLQ catches failures, messages retry automatically
- **Filtering** - SNS subscription filters reduce unnecessary processing
- **30% efficiency gain** - Your proven result from implementing this pattern

---

### Q26: Explain Microservices vs Monolith. When to use each?

**Answer:**

**Start Monolith, Migrate to Microservices:**

| Factor | Monolith | Microservices |
|---|---|---|
| Team size | < 10 devs | > 10 devs, multiple teams |
| Deployment | Simple, single unit | Complex, independent deploy |
| Data consistency | Easy (single DB, ACID) | Hard (eventual consistency) |
| Development speed | Fast initially | Slower initially, faster later |
| Debugging | Easy (single process) | Hard (distributed tracing) |
| Scaling | Scale entire app | Scale individual services |
| Tech diversity | Single stack | Mixed stacks possible |

**Microservices patterns you should know:**
1. **API Gateway** - Single entry point, routing, auth
2. **Service Discovery** - Services find each other (Consul, Kubernetes DNS)
3. **Circuit Breaker** - Prevent cascade failures
4. **Saga** - Distributed transactions (choreography vs orchestration)
5. **CQRS** - Separate read/write models
6. **Event Sourcing** - Store events, not state
7. **Sidecar** - Cross-cutting concerns (logging, mesh)

---

### Q27: Design a Notification System.

**Answer:**

```
Requirements:
- Multi-channel: Email, Push, SMS, In-app, WebSocket
- User preferences (opt-in/opt-out per channel)
- Template-based messages
- Rate limiting per user
- Delivery tracking

Architecture:
═══════════════════════════════════════════════════════

┌──────────┐     ┌──────────────┐     ┌────────────────┐
│ Services │────▶│  SNS Topic   │────▶│  SQS Priority  │
│ (trade,  │     │ "notify"     │     │  Queues        │
│  auth)   │     └──────────────┘     │  ├─ Critical   │
└──────────┘                           │  ├─ High       │
                                       │  └─ Normal     │
                                       └───────┬────────┘
                                               ▼
                                    ┌──────────────────┐
                                    │ Notification     │
                                    │ Orchestrator     │
                                    │ (NestJS)         │
                                    └──────┬───────────┘
                                           │
                          ┌────────────────┼────────────────┐
                          ▼                ▼                ▼
                  ┌──────────────┐ ┌──────────────┐ ┌──────────────┐
                  │ Email Worker │ │ Push Worker  │ │ SMS Worker   │
                  │ (SES/SMTP)  │ │ (FCM/APNS)  │ │ (Twilio)     │
                  └──────────────┘ └──────────────┘ └──────────────┘

Data Model:
- notifications: id, user_id, type, channel, template_id, payload, status, sent_at
- notification_preferences: user_id, channel, enabled, quiet_hours
- notification_templates: id, name, subject, body (with {{variables}})
```

---

## 6. AWS Cloud Services

### Q28: Explain SQS Standard vs FIFO. When to use DLQ?

**Answer:**

| Feature | Standard | FIFO |
|---|---|---|
| Throughput | Unlimited | 300 msg/s (3000 with batching) |
| Ordering | Best-effort | Strict (within message group) |
| Delivery | At-least-once (may duplicate) | Exactly-once |
| Use case | High throughput, order not critical | Financial transactions, sequential processing |

```typescript
// DLQ (Dead Letter Queue) - catches messages that fail processing
// After maxReceiveCount failures, message moves to DLQ

// SQS config with DLQ
{
  QueueName: 'trade-processing',
  RedrivePolicy: JSON.stringify({
    deadLetterTargetArn: 'arn:aws:sqs:us-east-1:123:trade-processing-dlq',
    maxReceiveCount: 3  // after 3 failures → DLQ
  }),
  VisibilityTimeout: 30,  // seconds before message becomes visible again
}

// DLQ Alarm - alert when messages land in DLQ
// CloudWatch alarm on ApproximateNumberOfMessagesVisible > 0
```

**DLQ use cases:**
- Malformed messages that always fail
- Downstream service permanently unavailable
- Debugging failed message processing
- Audit trail for failed operations

---

### Q29: ECS Fargate vs EC2 Launch Type.

**Answer:**

| Feature | Fargate | EC2 |
|---|---|---|
| Server management | Serverless (no EC2 to manage) | You manage EC2 instances |
| Pricing | Per vCPU + memory per second | EC2 instance pricing |
| Scaling | Per task | Per instance + per task |
| GPU support | Limited | Full support |
| Cost (steady) | Higher | Lower (reserved instances) |
| Cost (bursty) | Lower (pay per use) | Higher (idle resources) |
| Startup time | ~30-60s | Faster if instances running |
| Control | Less (no SSH, limited networking) | Full (SSH, custom AMI) |

```typescript
// ECS Task Definition (Fargate)
{
  family: 'trade-api',
  networkMode: 'awsvpc',        // required for Fargate
  requiresCompatibilities: ['FARGATE'],
  cpu: '512',                    // 0.5 vCPU
  memory: '1024',               // 1 GB
  containerDefinitions: [{
    name: 'trade-api',
    image: '123456.dkr.ecr.us-east-1.amazonaws.com/trade-api:latest',
    portMappings: [{ containerPort: 3000 }],
    logConfiguration: {
      logDriver: 'awslogs',
      options: {
        'awslogs-group': '/ecs/trade-api',
        'awslogs-region': 'us-east-1',
      }
    },
    healthCheck: {
      command: ['CMD-SHELL', 'curl -f http://localhost:3000/health || exit 1'],
      interval: 30,
      timeout: 5,
      retries: 3,
    }
  }]
}
```

**Recommendation:** Use Fargate for most workloads (simpler ops). Use EC2 launch type for GPU workloads (ML/AI), cost optimization with reserved instances, or when you need specific EC2 features.

---

### Q30: Explain DynamoDB design patterns.

**Answer:**

```
DynamoDB is a NoSQL key-value/document store. Design for access patterns, NOT normalization.

Single Table Design:
PK              SK                  Data
USER#123        PROFILE             { name, email, ... }
USER#123        TRADE#2024-01-15    { symbol: SOL, amount: 100 }
USER#123        TRADE#2024-01-16    { symbol: ETH, amount: 50 }
SYMBOL#SOL      PRICE               { price: 95.50, updated: ... }
SYMBOL#SOL      TRADE#2024-01-15    { userId: 123, amount: 100 }

Access Patterns:
1. Get user profile    → PK = USER#123, SK = PROFILE
2. Get user's trades   → PK = USER#123, SK begins_with TRADE#
3. Get trades by date  → PK = USER#123, SK between TRADE#2024-01-01 and TRADE#2024-01-31
4. Get trades by symbol → GSI: PK = SYMBOL#SOL, SK begins_with TRADE#
```

```typescript
// Capacity modes:
// On-Demand: pay per request (unpredictable, bursty)
// Provisioned: set RCU/WCU (predictable, cheaper)

// GSI (Global Secondary Index) - different PK+SK, eventually consistent
// LSI (Local Secondary Index) - same PK, different SK, strongly consistent

// DynamoDB Streams - CDC (Change Data Capture)
// Use for: syncing to Elasticsearch, triggering Lambda, event sourcing
```

---

## 7. DevOps, Docker & CI/CD

### Q31: Docker Multi-stage Build - Optimize a Node.js image.

**Answer:**

```dockerfile
# Stage 1: Install dependencies
FROM node:20-alpine AS deps
WORKDIR /app
COPY package.json pnpm-lock.yaml ./
RUN corepack enable && pnpm install --frozen-lockfile

# Stage 2: Build
FROM node:20-alpine AS builder
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .
RUN pnpm build

# Stage 3: Production image
FROM node:20-alpine AS runner
WORKDIR /app

# Security: non-root user
RUN addgroup --system --gid 1001 nodejs && \
    adduser --system --uid 1001 nestjs
USER nestjs

# Only copy production artifacts
COPY --from=builder --chown=nestjs:nodejs /app/dist ./dist
COPY --from=builder --chown=nestjs:nodejs /app/node_modules ./node_modules
COPY --from=builder --chown=nestjs:nodejs /app/package.json ./

EXPOSE 3000
HEALTHCHECK --interval=30s --timeout=3s \
  CMD wget --no-verbose --tries=1 --spider http://localhost:3000/health || exit 1

CMD ["node", "dist/main.js"]

# Result: ~150MB vs ~1.2GB unoptimized
```

**Optimization tips:**
- Use Alpine base images (~5MB vs ~900MB)
- Multi-stage builds (don't ship dev dependencies)
- Order layers by change frequency (package.json before source code)
- Use `.dockerignore` (node_modules, .git, tests)
- Pin exact versions for reproducibility

---

### Q32: GitHub Actions CI/CD Pipeline for a NestJS app.

**Answer:**

```yaml
# .github/workflows/deploy.yml
name: Deploy

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

env:
  AWS_REGION: us-east-1
  ECR_REPOSITORY: trade-api
  ECS_CLUSTER: production
  ECS_SERVICE: trade-api-service

jobs:
  test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:15
        env:
          POSTGRES_DB: test
          POSTGRES_PASSWORD: test
        ports: [5432:5432]
        options: --health-cmd pg_isready --health-interval 10s
      redis:
        image: redis:7
        ports: [6379:6379]

    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v2
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: 'pnpm'

      - run: pnpm install --frozen-lockfile
      - run: pnpm lint
      - run: pnpm test
      - run: pnpm test:e2e

  deploy:
    needs: test
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ env.AWS_REGION }}

      - uses: aws-actions/amazon-ecr-login@v2
        id: ecr

      - name: Build and push Docker image
        run: |
          docker build -t ${{ steps.ecr.outputs.registry }}/${{ env.ECR_REPOSITORY }}:${{ github.sha }} .
          docker push ${{ steps.ecr.outputs.registry }}/${{ env.ECR_REPOSITORY }}:${{ github.sha }}

      - name: Deploy to ECS
        run: |
          aws ecs update-service \
            --cluster ${{ env.ECS_CLUSTER }} \
            --service ${{ env.ECS_SERVICE }} \
            --force-new-deployment
```

---

### Q33: Explain deployment strategies.

**Answer:**

```
1. ROLLING UPDATE (Default in ECS/K8s)
   ├─ Gradually replace old instances with new
   ├─ [v1][v1][v1][v1] → [v2][v1][v1][v1] → [v2][v2][v1][v1] → [v2][v2][v2][v2]
   ├─ Pro: Zero downtime, gradual rollout
   └─ Con: Two versions running simultaneously

2. BLUE/GREEN
   ├─ Run two identical environments
   ├─ Blue (current) ← traffic
   ├─ Green (new) ← deploy here, test
   ├─ Switch ALB/Route53 to Green
   ├─ Pro: Instant rollback (switch back to Blue)
   └─ Con: 2x infrastructure cost during deploy

3. CANARY
   ├─ Route small % of traffic to new version
   ├─ [v1 95%] [v2 5%] → monitor → [v1 50%] [v2 50%] → [v2 100%]
   ├─ Pro: Minimizes blast radius
   └─ Con: Complex routing, need good monitoring

4. RECREATE (simple, has downtime)
   ├─ Stop all v1 → Start all v2
   ├─ Pro: Simple, no version conflicts
   └─ Con: Downtime
```

---

## 8. Web3 & Blockchain

### Q34: Explain Solana's Architecture - Accounts, Programs, Instructions.

**Answer:**

```
Solana Architecture:
══════════════════════════════════════

Everything is an ACCOUNT:
┌────────────────────────────────────┐
│ Account                            │
│ ├─ lamports (SOL balance)          │
│ ├─ data (byte array)              │
│ ├─ owner (program that owns it)   │
│ ├─ executable (is it a program?)  │
│ └─ rent_epoch                      │
└────────────────────────────────────┘

Programs are STATELESS:
- Programs (smart contracts) don't store data themselves
- Data is stored in separate accounts
- Programs read/write to accounts passed as instruction arguments

Transaction Structure:
┌─────────────────────────────────────┐
│ Transaction                          │
│ ├─ Signatures[]                      │
│ └─ Message                           │
│     ├─ Header (signers, readonly)    │
│     ├─ Account Keys[]                │
│     └─ Instructions[]                │
│         ├─ program_id (which program)│
│         ├─ accounts[] (to read/write)│
│         └─ data (serialized args)    │
└─────────────────────────────────────┘
```

**Key concepts:**
- **Rent**: Accounts must maintain minimum SOL balance (or be rent-exempt with ~2 years worth)
- **PDA (Program Derived Address)**: Deterministic addresses that only programs can sign for
- **CPI (Cross-Program Invocation)**: Programs calling other programs
- **Parallel execution**: Transactions on different accounts run in parallel (Sealevel runtime)

---

### Q35: Explain Anchor Framework for Solana development.

**Answer:**

```rust
// Anchor program - Token Staking example
use anchor_lang::prelude::*;

declare_id!("Stake11111111111111111111111111111111111111");

#[program]
pub mod staking {
    use super::*;

    pub fn initialize(ctx: Context<Initialize>, reward_rate: u64) -> Result<()> {
        let pool = &mut ctx.accounts.pool;
        pool.authority = ctx.accounts.authority.key();
        pool.reward_rate = reward_rate;
        pool.total_staked = 0;
        Ok(())
    }

    pub fn stake(ctx: Context<Stake>, amount: u64) -> Result<()> {
        // Transfer tokens from user to pool vault
        let cpi_ctx = CpiContext::new(
            ctx.accounts.token_program.to_account_info(),
            Transfer {
                from: ctx.accounts.user_token_account.to_account_info(),
                to: ctx.accounts.pool_vault.to_account_info(),
                authority: ctx.accounts.user.to_account_info(),
            },
        );
        token::transfer(cpi_ctx, amount)?;

        // Update stake account
        let stake_account = &mut ctx.accounts.stake_account;
        stake_account.amount += amount;
        stake_account.last_stake_time = Clock::get()?.unix_timestamp;

        Ok(())
    }
}

// Account validation with Anchor constraints
#[derive(Accounts)]
pub struct Stake<'info> {
    #[account(mut)]
    pub user: Signer<'info>,

    #[account(
        init_if_needed,
        payer = user,
        space = 8 + StakeAccount::INIT_SPACE,
        seeds = [b"stake", user.key().as_ref(), pool.key().as_ref()],
        bump, // PDA
    )]
    pub stake_account: Account<'info, StakeAccount>,

    #[account(mut)]
    pub pool: Account<'info, StakePool>,

    #[account(mut)]
    pub user_token_account: Account<'info, TokenAccount>,

    #[account(mut)]
    pub pool_vault: Account<'info, TokenAccount>,

    pub token_program: Program<'info, Token>,
    pub system_program: Program<'info, System>,
}

#[account]
#[derive(InitSpace)]
pub struct StakeAccount {
    pub owner: Pubkey,
    pub amount: u64,
    pub last_stake_time: i64,
}
```

**Anchor benefits:**
- Account validation via derive macros (no manual checks)
- Automatic serialization/deserialization (Borsh)
- IDL generation for client SDK
- PDA derivation helpers
- Error handling with custom error codes

---

### Q36: Explain PDAs (Program Derived Addresses).

**Answer:**

```typescript
// PDA = Deterministic address derived from seeds + program_id
// PDAs are NOT on the ed25519 curve → only the program can "sign" for them

// Deriving a PDA (client-side)
import { PublicKey } from '@solana/web3.js';

const [stakePda, bump] = PublicKey.findProgramAddressSync(
  [
    Buffer.from('stake'),           // seed 1: string
    userPublicKey.toBuffer(),       // seed 2: user's pubkey
    poolPublicKey.toBuffer(),       // seed 3: pool's pubkey
  ],
  programId
);

// Use cases for PDAs:
// 1. Deterministic account addresses (find account without storing address)
// 2. Program-owned accounts (program can sign on behalf of PDA)
// 3. Token vaults (PDA owns tokens, only program can transfer)
// 4. Authority delegation (PDA as mint authority)

// Example: PDA as token vault authority
// The program can transfer tokens FROM the vault using CPI with PDA signer
let seeds = &[b"vault", pool.key().as_ref(), &[bump]];
let signer = &[&seeds[..]];

let cpi_ctx = CpiContext::new_with_signer(
    token_program,
    Transfer { from: vault, to: user_ata, authority: vault_authority },
    signer, // PDA signs!
);
token::transfer(cpi_ctx, amount)?;
```

---

### Q37: How do you integrate wallets in a frontend app?

**Answer:**

```typescript
// Using Privy (your experience) - supports email + wallet login
import { PrivyProvider, usePrivy } from '@privy-io/react-auth';

// Setup
function App() {
  return (
    <PrivyProvider
      appId={process.env.NEXT_PUBLIC_PRIVY_APP_ID}
      config={{
        loginMethods: ['email', 'wallet', 'google'],
        appearance: { theme: 'dark' },
        embeddedWallets: {
          createOnLogin: 'users-without-wallets', // auto-create for email users
        },
      }}
    >
      <MyApp />
    </PrivyProvider>
  );
}

// Usage
function WalletButton() {
  const { login, logout, user, authenticated, sendTransaction } = usePrivy();

  const handleSendSOL = async () => {
    const tx = new Transaction().add(
      SystemProgram.transfer({
        fromPubkey: new PublicKey(user.wallet.address),
        toPubkey: recipientPubkey,
        lamports: LAMPORTS_PER_SOL * 0.1,
      })
    );

    const signedTx = await sendTransaction(tx);
    console.log('TX:', signedTx);
  };

  if (!authenticated) {
    return <button onClick={login}>Connect Wallet</button>;
  }

  return (
    <div>
      <p>{user.wallet.address}</p>
      <button onClick={handleSendSOL}>Send 0.1 SOL</button>
      <button onClick={logout}>Disconnect</button>
    </div>
  );
}
```

---

### Q38: Explain blockchain indexing (Geyser gRPC, Helius).

**Answer:**

```typescript
// Problem: Querying on-chain data is slow and expensive (RPC calls)
// Solution: Index on-chain data into your own database

// 1. Helius Webhooks - Simple, managed
// Subscribe to program events, get HTTP callbacks
const webhook = await helius.createWebhook({
  webhookURL: 'https://api.myapp.com/webhook/trades',
  transactionTypes: ['SWAP'],
  accountAddresses: ['YourProgramId...'],
  webhookType: 'enhanced', // parsed transaction data
});

// 2. Helius DAS API - Digital Asset Standard
const assets = await helius.rpc.getAssetsByOwner({
  ownerAddress: walletAddress,
  page: 1,
});

// 3. Geyser gRPC - Real-time streaming (lowest latency)
// Streams account updates, transactions, slots directly from validator
import { Client } from '@triton-one/yellowstone-grpc';

const client = new Client('https://grpc.myvalidator.com', token);
const stream = await client.subscribe();

// Subscribe to specific account changes
stream.write({
  accounts: {
    'trade-accounts': {
      account: [],
      owner: ['YourProgramId...'],
      filters: [],
    },
  },
  transactions: {},
  slots: {},
});

stream.on('data', (update) => {
  if (update.account) {
    const accountData = update.account.account.data;
    // Deserialize and store in PostgreSQL
    const trade = borsh.deserialize(TradeSchema, accountData);
    await db.trades.upsert(trade);
  }
});

// 4. Custom RPC polling (simplest but slowest)
setInterval(async () => {
  const signatures = await connection.getSignaturesForAddress(programId);
  for (const sig of signatures) {
    const tx = await connection.getParsedTransaction(sig.signature);
    // Process and store
  }
}, 5000);

// Comparison:
// Geyser gRPC: ~100ms latency, highest throughput, complex setup
// Helius Webhooks: ~1-2s latency, easy setup, managed
// RPC Polling: ~5-10s latency, simplest, rate limited
```

---

## 9. AI Integration

### Q39: Explain RAG (Retrieval-Augmented Generation) architecture.

**Answer:**

```
RAG Architecture:
═══════════════════════════════════════════

Document Ingestion Pipeline:
┌──────────┐    ┌──────────┐    ┌───────────┐    ┌──────────────┐
│ Documents │───▶│ Chunking │───▶│ Embedding │───▶│ Vector DB    │
│ (PDF,     │    │ (1000    │    │ (OpenAI   │    │ (Pinecone/   │
│  Markdown)│    │  tokens) │    │  ada-002) │    │  Weaviate)   │
└──────────┘    └──────────┘    └───────────┘    └──────────────┘

Query Pipeline:
┌──────────┐    ┌───────────┐    ┌──────────────┐    ┌───────────┐
│ User     │───▶│ Embed     │───▶│ Vector       │───▶│ Top K     │
│ Question │    │ Question  │    │ Similarity   │    │ Results   │
└──────────┘    └───────────┘    │ Search       │    └─────┬─────┘
                                  └──────────────┘          │
                                                            ▼
                                  ┌──────────────┐    ┌───────────┐
                                  │ LLM Response │◀───│ Prompt +  │
                                  │ to User      │    │ Context   │
                                  └──────────────┘    └───────────┘
```

```typescript
// LangChain RAG implementation
import { ChatOpenAI } from '@langchain/openai';
import { PineconeStore } from '@langchain/pinecone';
import { OpenAIEmbeddings } from '@langchain/openai';
import { RecursiveCharacterTextSplitter } from 'langchain/text_splitter';
import { RetrievalQAChain } from 'langchain/chains';

// 1. Ingest documents
const splitter = new RecursiveCharacterTextSplitter({
  chunkSize: 1000,
  chunkOverlap: 200,
});
const docs = await splitter.splitDocuments(rawDocs);

const vectorStore = await PineconeStore.fromDocuments(docs, new OpenAIEmbeddings(), {
  pineconeIndex,
  namespace: 'trading-docs',
});

// 2. Query
const retriever = vectorStore.asRetriever({ k: 5 });
const llm = new ChatOpenAI({ model: 'gpt-4', temperature: 0 });

const chain = RetrievalQAChain.fromLLM(llm, retriever, {
  returnSourceDocuments: true,
});

const result = await chain.call({
  query: 'What is the maximum position size for SOL trades?',
});
// Returns answer with source documents for verification
```

**RAG best practices:**
- Chunk size matters: too small = missing context, too large = noise
- Overlap chunks by 10-20% to preserve context at boundaries
- Use metadata filtering to narrow search scope
- Implement re-ranking for better relevance
- Cache embeddings for frequently asked questions

---

### Q40: How do you integrate AI into a production application?

**Answer:**

```typescript
// Production AI service with streaming, caching, and error handling
@Injectable()
export class AIService {
  private readonly openai: OpenAI;
  private readonly redis: Redis;

  // 1. Streaming response (for real-time UX)
  async *streamAnalysis(tradeData: TradeData): AsyncGenerator<string> {
    const cacheKey = `analysis:${hash(tradeData)}`;
    const cached = await this.redis.get(cacheKey);
    if (cached) { yield cached; return; }

    const stream = await this.openai.chat.completions.create({
      model: 'gpt-4',
      messages: [
        { role: 'system', content: TRADE_ANALYST_PROMPT },
        { role: 'user', content: JSON.stringify(tradeData) },
      ],
      stream: true,
      max_tokens: 1000,
    });

    let fullResponse = '';
    for await (const chunk of stream) {
      const content = chunk.choices[0]?.delta?.content || '';
      fullResponse += content;
      yield content;
    }

    // Cache for 1 hour
    await this.redis.setex(cacheKey, 3600, fullResponse);
  }

  // 2. Function calling (structured output)
  async extractTradeSignals(marketData: string): Promise<TradeSignal[]> {
    const response = await this.openai.chat.completions.create({
      model: 'gpt-4',
      messages: [{ role: 'user', content: marketData }],
      tools: [{
        type: 'function',
        function: {
          name: 'create_trade_signals',
          parameters: {
            type: 'object',
            properties: {
              signals: {
                type: 'array',
                items: {
                  type: 'object',
                  properties: {
                    symbol: { type: 'string' },
                    action: { enum: ['BUY', 'SELL', 'HOLD'] },
                    confidence: { type: 'number', minimum: 0, maximum: 1 },
                    reasoning: { type: 'string' },
                  },
                },
              },
            },
          },
        },
      }],
      tool_choice: { type: 'function', function: { name: 'create_trade_signals' } },
    });

    return JSON.parse(
      response.choices[0].message.tool_calls[0].function.arguments
    ).signals;
  }

  // 3. Rate limiting and cost management
  @Throttle({ default: { limit: 10, ttl: 60000 } }) // 10 req/min per user
  async analyze(userId: string, prompt: string) {
    // Track token usage per user for billing
    const usage = await this.getMonthlyUsage(userId);
    if (usage > MAX_TOKENS_PER_MONTH) {
      throw new ForbiddenException('Monthly AI quota exceeded');
    }
    // ... make API call
  }
}
```

---

## 10. Security & Authentication

### Q41: Explain JWT-based authentication with refresh tokens.

**Answer:**

```typescript
// Auth flow with access + refresh tokens
@Injectable()
export class AuthService {
  async login(email: string, password: string) {
    const user = await this.validateUser(email, password);

    // Short-lived access token (15 min)
    const accessToken = this.jwtService.sign(
      { sub: user.id, email: user.email, role: user.role },
      { secret: process.env.JWT_SECRET, expiresIn: '15m' }
    );

    // Long-lived refresh token (7 days)
    const refreshToken = this.jwtService.sign(
      { sub: user.id, tokenVersion: user.tokenVersion },
      { secret: process.env.JWT_REFRESH_SECRET, expiresIn: '7d' }
    );

    // Store refresh token hash in DB (for revocation)
    await this.saveRefreshToken(user.id, refreshToken);

    return {
      accessToken,
      refreshToken, // send as httpOnly cookie
    };
  }

  async refreshTokens(refreshToken: string) {
    const payload = this.jwtService.verify(refreshToken, {
      secret: process.env.JWT_REFRESH_SECRET,
    });

    const user = await this.userRepo.findOne({ where: { id: payload.sub } });

    // Check token version (invalidated on password change/logout)
    if (user.tokenVersion !== payload.tokenVersion) {
      throw new UnauthorizedException('Token revoked');
    }

    // Rotate refresh token (one-time use)
    return this.login(user.email, null); // generate new pair
  }

  async logout(userId: string) {
    // Increment token version → all existing refresh tokens become invalid
    await this.userRepo.increment({ id: userId }, 'tokenVersion', 1);
  }
}

// Token storage best practices:
// Access Token → Memory (JS variable) or short-lived cookie
// Refresh Token → httpOnly, Secure, SameSite=Strict cookie
// NEVER store tokens in localStorage (XSS vulnerable)
```

---

### Q42: Explain OAuth 2.0 / OpenID Connect flow.

**Answer:**

```
Authorization Code Flow (most secure for web apps):
════════════════════════════════════════════════════

┌──────┐     ┌───────────┐     ┌──────────────┐     ┌──────────┐
│ User │     │ Your App  │     │ Auth Server  │     │ Resource │
│      │     │ (Client)  │     │ (Google/Auth0)│    │ Server   │
└──┬───┘     └─────┬─────┘     └──────┬───────┘     └────┬─────┘
   │               │                   │                   │
   │ 1. Click      │                   │                   │
   │ "Login"       │                   │                   │
   │──────────────▶│                   │                   │
   │               │ 2. Redirect to    │                   │
   │               │ auth server       │                   │
   │◀──────────────│──────────────────▶│                   │
   │               │                   │                   │
   │ 3. User logs in & consents        │                   │
   │──────────────────────────────────▶│                   │
   │               │                   │                   │
   │ 4. Redirect back with AUTH CODE   │                   │
   │◀──────────────────────────────────│                   │
   │──────────────▶│                   │                   │
   │               │                   │                   │
   │               │ 5. Exchange code  │                   │
   │               │ for tokens        │                   │
   │               │ (server-to-server)│                   │
   │               │──────────────────▶│                   │
   │               │◀──────────────────│                   │
   │               │ (access_token,    │                   │
   │               │  refresh_token,   │                   │
   │               │  id_token)        │                   │
   │               │                   │                   │
   │               │ 6. Call API with  │                   │
   │               │ access_token      │                   │
   │               │─────────────────────────────────────▶│
   │               │◀─────────────────────────────────────│
   │               │                   │                   │
```

**Key tokens in OIDC:**
- **Access Token** - Grants access to APIs (short-lived)
- **Refresh Token** - Get new access tokens (long-lived)
- **ID Token** - JWT with user identity info (name, email)

---

## 11. Behavioral Questions

### Q43: Tell me about yourself (2-minute pitch).

**Answer:**
> "I'm Awais, a Senior Software Engineer with 5+ years of experience specializing in full-stack development with MERN stack, cloud architecture on AWS, and Web3 development on Solana.
>
> Currently at a stealth startup, I've been building AI-powered trading platforms with auto-strategy execution, deploying CVAT annotation workflows with YOLO integration, and architecting scalable systems using AWS SNS+SQS fan-out patterns — which improved our system efficiency by 30%.
>
> At Codora, I led blockchain integration across Ethereum and Solana ecosystems, working with everything from SPL tokens to DeFi protocols, while building the frontend with Next.js and the backend with NestJS microservices.
>
> What excites me most is the intersection of AI, blockchain, and cloud-native architecture. I'm looking for a role where I can leverage this unique combination to build impactful products at scale."

---

### Q44: Tell me about a challenging technical problem you solved.

**Answer (STAR Method):**

> **Situation:** At my current startup, our trading platform was experiencing message processing delays during high-volume trading periods. Notifications, analytics, and audit logs were all competing for the same processing pipeline.
>
> **Task:** I needed to redesign the event processing architecture to handle 10x throughput without losing messages or adding significant latency.
>
> **Action:** I implemented an SNS + SQS fan-out pattern. Instead of a single queue processing all events sequentially, I published trade events to an SNS topic that fanned out to dedicated SQS queues — one for analytics, one for notifications, one for risk checks, and one for audit logging. Each consumer could scale independently, and I added DLQs for fault tolerance.
>
> **Result:** System efficiency improved by 30%, message processing became parallel instead of sequential, and we could independently scale each consumer based on its specific load. The system now handles 10x the original volume with lower latency.

---

### Q45: How do you handle disagreements with team members on technical decisions?

**Answer:**
> "I approach technical disagreements as opportunities to find the best solution. Here's my process:
>
> 1. **Listen first** — Understand their perspective fully before responding
> 2. **Data over opinions** — I suggest running benchmarks, creating POCs, or looking at real-world examples
> 3. **Document trade-offs** — I write down pros/cons of each approach so the discussion is objective
> 4. **Align on goals** — Often disagreements happen because we're optimizing for different things
> 5. **Commit and support** — Once a decision is made, I fully commit even if it wasn't my preferred approach
>
> For example, at Codora, a colleague wanted to use a custom WebSocket implementation while I preferred Pusher for real-time features. We listed trade-offs: custom gave us more control but Pusher reduced maintenance. Given our timeline and team size, we went with Pusher, and it was the right call."

---

### Q46: How do you mentor junior developers?

**Answer:**
> "I focus on three things:
>
> 1. **Code reviews as teaching moments** — I don't just say 'fix this'. I explain WHY a pattern is better, link to docs, and sometimes pair-program to walk through the change
>
> 2. **Incremental responsibility** — Start with small features, gradually increase scope. I break down complex tasks into smaller tickets they can handle independently
>
> 3. **Create safe learning environments** — I share my own mistakes and what I learned. When they make errors, I focus on the learning, not the blame
>
> At Merik Solutions, I helped two junior devs grow from basic CRUD to building microservices independently within 6 months."

---

## 12. Coding Patterns Cheat Sheet

### Pattern 1: Two Pointers

```javascript
// Remove duplicates from sorted array
function removeDuplicates(nums) {
  let slow = 0;
  for (let fast = 1; fast < nums.length; fast++) {
    if (nums[fast] !== nums[slow]) {
      slow++;
      nums[slow] = nums[fast];
    }
  }
  return slow + 1;
}
```

### Pattern 2: Sliding Window

```javascript
// Maximum sum subarray of size k
function maxSubarraySum(arr, k) {
  let windowSum = arr.slice(0, k).reduce((a, b) => a + b);
  let maxSum = windowSum;

  for (let i = k; i < arr.length; i++) {
    windowSum += arr[i] - arr[i - k]; // slide: add right, remove left
    maxSum = Math.max(maxSum, windowSum);
  }
  return maxSum;
}
```

### Pattern 3: BFS/DFS

```javascript
// BFS - Level order traversal
function bfs(root) {
  const result = [];
  const queue = [root];

  while (queue.length) {
    const level = [];
    const size = queue.length;
    for (let i = 0; i < size; i++) {
      const node = queue.shift();
      level.push(node.val);
      if (node.left) queue.push(node.left);
      if (node.right) queue.push(node.right);
    }
    result.push(level);
  }
  return result;
}

// DFS - Inorder traversal (iterative)
function dfs(root) {
  const result = [];
  const stack = [];
  let current = root;

  while (current || stack.length) {
    while (current) {
      stack.push(current);
      current = current.left;
    }
    current = stack.pop();
    result.push(current.val);
    current = current.right;
  }
  return result;
}
```

### Pattern 4: Dynamic Programming

```javascript
// Longest Common Subsequence
function lcs(text1, text2) {
  const dp = Array(text1.length + 1)
    .fill(null)
    .map(() => Array(text2.length + 1).fill(0));

  for (let i = 1; i <= text1.length; i++) {
    for (let j = 1; j <= text2.length; j++) {
      if (text1[i - 1] === text2[j - 1]) {
        dp[i][j] = dp[i - 1][j - 1] + 1;
      } else {
        dp[i][j] = Math.max(dp[i - 1][j], dp[i][j - 1]);
      }
    }
  }
  return dp[text1.length][text2.length];
}
```

### Pattern 5: Binary Search

```javascript
// Find first/last occurrence
function binarySearch(nums, target) {
  let left = 0, right = nums.length - 1;

  while (left <= right) {
    const mid = Math.floor((left + right) / 2);
    if (nums[mid] === target) return mid;
    if (nums[mid] < target) left = mid + 1;
    else right = mid - 1;
  }
  return -1;
}

// Find first position >= target
function lowerBound(nums, target) {
  let left = 0, right = nums.length;
  while (left < right) {
    const mid = Math.floor((left + right) / 2);
    if (nums[mid] < target) left = mid + 1;
    else right = mid;
  }
  return left;
}
```

### Pattern 6: Graph - Topological Sort

```javascript
// Course schedule (detect cycle + ordering)
function canFinish(numCourses, prerequisites) {
  const graph = Array.from({ length: numCourses }, () => []);
  const inDegree = new Array(numCourses).fill(0);

  for (const [course, prereq] of prerequisites) {
    graph[prereq].push(course);
    inDegree[course]++;
  }

  const queue = [];
  for (let i = 0; i < numCourses; i++) {
    if (inDegree[i] === 0) queue.push(i);
  }

  const order = [];
  while (queue.length) {
    const node = queue.shift();
    order.push(node);
    for (const neighbor of graph[node]) {
      inDegree[neighbor]--;
      if (inDegree[neighbor] === 0) queue.push(neighbor);
    }
  }

  return order.length === numCourses; // false if cycle exists
}
```

---

## Quick Reference Card

### Your Key Metrics (Memorize These)
- 30% system efficiency improvement (SNS + SQS fan-out)
- 20% transaction speed improvement (AI trading platform)
- 15% faster development time (blockchain smart contracts)
- 99.9% uptime (cloud-native deployments on GCP & AWS)
- 5 blockchain projects delivered

### Tech Stack Quick Recall
| Layer | Technologies |
|---|---|
| Frontend | React, Next.js, Tailwind, MUI, Zustand |
| Backend | NestJS, Node.js, Express |
| Database | PostgreSQL, MongoDB, DynamoDB, Redis |
| Cloud | AWS (ECS, SQS, SNS, RDS, S3), GCP |
| Web3 | Solana (Anchor), Ethereum, SPL Tokens |
| AI | LangChain, YOLO, CVAT, OpenAI |
| DevOps | Docker, GitHub Actions, Terraform, Prometheus |
| Testing | Cypress, Jest, React Testing Library |

---

> **Study order:** JS/TS fundamentals → React/Next.js → NestJS → System Design → AWS → Web3 → AI → DSA
>
> Good luck with your interviews! 🎯
