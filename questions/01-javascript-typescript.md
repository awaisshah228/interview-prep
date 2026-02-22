# JavaScript & TypeScript - Interview Q&A

> 30+ questions covering core JS, ES6+, async patterns, and advanced TypeScript

---

## Table of Contents

- [JavaScript Fundamentals](#javascript-fundamentals)
- [Async JavaScript](#async-javascript)
- [ES6+ Features](#es6-features)
- [TypeScript](#typescript)
- [Tricky Output Questions](#tricky-output-questions)

---

## JavaScript Fundamentals

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

// Fix with closure (IIFE)
for (var i = 0; i < 3; i++) {
  ((j) => setTimeout(() => console.log(j), 100))(i); // prints 0, 1, 2
}

// Modern fix: use let (block-scoped)
for (let i = 0; i < 3; i++) {
  setTimeout(() => console.log(i), 100); // prints 0, 1, 2
}
```

**Other practical uses:**
- Data privacy / encapsulation (module pattern)
- Function factories
- Memoization
- Event handlers with preserved state

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

// Prototype chain: dog → animal → Object.prototype → null

// Class syntax (syntactic sugar over prototypes)
class Animal {
  constructor(name) { this.name = name; }
  speak() { return `${this.name} makes a sound`; }
}

class Dog extends Animal {
  bark() { return `${this.name} barks`; }
}

// Under the hood:
// Dog.prototype.__proto__ === Animal.prototype  ✅
// new Dog('Rex') instanceof Animal              ✅
```

**Key difference:** Classical inheritance (Java/C++) copies properties. Prototypal inheritance creates a chain of references - changes to the prototype affect all instances.

```javascript
Animal.prototype.eat = function() { return 'eating'; };
const rex = new Dog('Rex');
rex.eat(); // "eating" — inherited dynamically!
```

---

### Q4: Explain `this` in different contexts.

**Answer:**

| Context | `this` refers to |
|---|---|
| Global scope | `window` (browser) / `global` (Node) / `undefined` (strict mode) |
| Object method | The object calling the method |
| Arrow function | Inherits `this` from enclosing lexical scope |
| `new` keyword | The newly created instance |
| `call/apply/bind` | The explicitly passed object |
| Event handler | The DOM element (unless arrow function) |
| Class method | The class instance |

```javascript
const obj = {
  name: 'Awais',
  regular: function() { return this.name; },     // 'Awais'
  arrow: () => this.name,                          // undefined (inherits outer this)
  nested: function() {
    const inner = () => this.name;                 // 'Awais' (arrow captures obj's this)
    return inner();
  }
};

// Explicit binding
function greet() { return `Hi, ${this.name}`; }
greet.call({ name: 'Awais' });   // "Hi, Awais"
greet.apply({ name: 'Awais' });  // "Hi, Awais"  (same, but args as array)
const bound = greet.bind({ name: 'Awais' });
bound(); // "Hi, Awais"

// bind is permanent — can't re-bind
const reBound = bound.bind({ name: 'Ali' });
reBound(); // "Hi, Awais" — original bind wins
```

---

### Q5: Explain scope chain and hoisting.

**Answer:**

**Scope chain:** When accessing a variable, JS looks in the current scope, then parent scope, then grandparent, all the way to global scope.

```javascript
const global = 'global';

function outer() {
  const outerVar = 'outer';

  function inner() {
    const innerVar = 'inner';
    console.log(innerVar);  // found in current scope
    console.log(outerVar);  // found in parent scope
    console.log(global);    // found in global scope
  }
  inner();
}
```

**Hoisting:** Declarations are moved to the top of their scope during compilation.

```javascript
// What you write:
console.log(a); // undefined (not ReferenceError!)
console.log(b); // ReferenceError: Cannot access 'b' before initialization
console.log(c); // ReferenceError: Cannot access 'c' before initialization

var a = 1;
let b = 2;
const c = 3;

// How JS sees it:
var a;           // hoisted (initialized as undefined)
// let b;        // hoisted to TDZ (Temporal Dead Zone) — not accessible
// const c;      // hoisted to TDZ — not accessible

console.log(a);  // undefined
a = 1;

// Functions are fully hoisted
sayHi(); // "Hi!" — works!
function sayHi() { console.log('Hi!'); }

// Function expressions are NOT fully hoisted
sayBye(); // TypeError: sayBye is not a function
var sayBye = function() { console.log('Bye!'); };
```

---

### Q6: What is the difference between == and ===?

**Answer:**

- `===` (strict equality): No type coercion, compares value AND type
- `==` (loose equality): Performs type coercion before comparing

```javascript
// == coercion rules (avoid using ==)
0 == ''        // true  (both coerce to 0)
0 == '0'       // true  (string '0' coerces to number 0)
'' == '0'      // false (no coercion needed, both strings)
false == '0'   // true  (false → 0, '0' → 0)
null == undefined // true (special rule)
null == 0      // false (null only == undefined)
NaN == NaN     // false (NaN is never equal to anything)

// Always use ===
0 === ''       // false
0 === '0'      // false
null === undefined // false
```

---

### Q7: Explain `call`, `apply`, `bind` with practical examples.

**Answer:**

```javascript
// All three set `this` explicitly

function introduce(greeting, punctuation) {
  return `${greeting}, I'm ${this.name}${punctuation}`;
}

const awais = { name: 'Awais' };

// call - invoke immediately, args individually
introduce.call(awais, 'Hi', '!');    // "Hi, I'm Awais!"

// apply - invoke immediately, args as array
introduce.apply(awais, ['Hi', '!']); // "Hi, I'm Awais!"

// bind - returns NEW function with bound this (doesn't invoke)
const boundIntro = introduce.bind(awais, 'Hi');
boundIntro('!');                     // "Hi, I'm Awais!"

// Practical: borrowing methods
const nums = [5, 2, 8, 1, 9];
Math.max.apply(null, nums);  // 9 (pre-ES6)
Math.max(...nums);           // 9 (modern)

// Practical: converting arguments to array
function legacyFunc() {
  const args = Array.prototype.slice.call(arguments); // [1, 2, 3]
  // Modern: const args = [...arguments]; or use rest params
}
```

---

### Q8: Explain WeakMap, WeakSet, Map, and Set.

**Answer:**

| Feature | Map | WeakMap | Set | WeakSet |
|---|---|---|---|---|
| Key types | Any | Objects only | Values (any) | Objects only |
| Iterable | Yes | No | Yes | No |
| `.size` | Yes | No | Yes | No |
| GC of keys | No | Yes (weak ref) | No | Yes (weak ref) |

```javascript
// Map - key-value pairs with any key type
const map = new Map();
map.set({ id: 1 }, 'user1');   // object key
map.set('name', 'Awais');       // string key
map.set(42, 'answer');          // number key
map.get('name');                // 'Awais'
map.has(42);                    // true
map.size;                       // 3

// Set - unique values only
const set = new Set([1, 2, 2, 3, 3]);
set.size;     // 3 (duplicates removed)
set.add(4);
set.has(2);   // true
[...set];     // [1, 2, 3, 4]

// Practical: remove duplicates
const unique = [...new Set([1, 2, 2, 3])]; // [1, 2, 3]

// WeakMap - keys are weakly referenced (auto GC)
const cache = new WeakMap();
let user = { name: 'Awais' };
cache.set(user, { lastLogin: Date.now() });
user = null; // user AND cache entry can be garbage collected!

// Use cases for WeakMap:
// 1. Caching without memory leaks
// 2. Private data for class instances
// 3. DOM element metadata
// 4. Tracking objects without preventing GC
```

---

### Q9: Explain Proxy and Reflect.

**Answer:**

```javascript
// Proxy - intercept and customize operations on objects
const handler = {
  get(target, prop, receiver) {
    console.log(`Accessing: ${prop}`);
    return Reflect.get(target, prop, receiver);
  },
  set(target, prop, value, receiver) {
    if (prop === 'age' && (value < 0 || value > 150)) {
      throw new RangeError('Invalid age');
    }
    return Reflect.set(target, prop, value, receiver);
  },
  deleteProperty(target, prop) {
    if (prop === 'id') {
      throw new Error('Cannot delete id');
    }
    return Reflect.deleteProperty(target, prop);
  }
};

const user = new Proxy({ id: 1, name: 'Awais', age: 28 }, handler);
user.name;       // logs "Accessing: name", returns 'Awais'
user.age = 200;  // throws RangeError
delete user.id;  // throws Error

// Practical: Reactive state (like Vue 3's reactivity system)
function reactive(obj) {
  return new Proxy(obj, {
    set(target, prop, value) {
      const result = Reflect.set(target, prop, value);
      notifySubscribers(prop, value); // trigger UI update
      return result;
    }
  });
}

// Reflect - provides default behavior for Proxy traps
// Reflect.get, Reflect.set, Reflect.has, Reflect.deleteProperty, etc.
// Always use Reflect in Proxy handlers for correct default behavior
```

---

### Q10: Explain Generators and Iterators.

**Answer:**

```javascript
// Generator - function that can pause and resume
function* fibonacci() {
  let a = 0, b = 1;
  while (true) {
    yield a;         // pause here, return value
    [a, b] = [b, a + b];
  }
}

const fib = fibonacci();
fib.next(); // { value: 0, done: false }
fib.next(); // { value: 1, done: false }
fib.next(); // { value: 1, done: false }
fib.next(); // { value: 2, done: false }

// Practical: Paginated API fetching
async function* fetchPages(url) {
  let page = 1;
  let hasMore = true;

  while (hasMore) {
    const response = await fetch(`${url}?page=${page}`);
    const data = await response.json();
    hasMore = data.hasNextPage;
    page++;
    yield data.items;
  }
}

// Usage
for await (const items of fetchPages('/api/trades')) {
  processTrades(items);
  // automatically fetches next page on each iteration
}

// Iterator protocol - any object with [Symbol.iterator]
const range = {
  from: 1,
  to: 5,
  [Symbol.iterator]() {
    let current = this.from;
    const last = this.to;
    return {
      next() {
        return current <= last
          ? { value: current++, done: false }
          : { done: true };
      }
    };
  }
};

for (const num of range) console.log(num); // 1, 2, 3, 4, 5
[...range]; // [1, 2, 3, 4, 5]
```

---

### Q11: What are memory leaks in JavaScript and how to prevent them?

**Answer:**

```javascript
// 1. Accidental global variables
function leak() {
  name = 'Awais'; // no let/const/var → global!
}
// Fix: use strict mode or always declare variables

// 2. Forgotten timers
const interval = setInterval(() => {
  const data = fetchData(); // keeps reference alive
}, 1000);
// Fix: clearInterval(interval) when done

// 3. Closures holding references
function createHandler() {
  const largeData = new Array(1000000).fill('x');
  return function handler() {
    // largeData is retained even if not used!
    console.log('handling');
  };
}
// Fix: set largeData = null after use, or restructure

// 4. Detached DOM nodes
const elements = [];
function addElement() {
  const el = document.createElement('div');
  document.body.appendChild(el);
  elements.push(el); // reference kept in array
  document.body.removeChild(el); // removed from DOM but still in memory!
}
// Fix: remove from array too, or use WeakRef

// 5. Event listeners not removed
element.addEventListener('click', handler);
// Fix: element.removeEventListener('click', handler)
// Or use AbortController:
const controller = new AbortController();
element.addEventListener('click', handler, { signal: controller.signal });
controller.abort(); // removes all listeners with this signal

// Detection: Chrome DevTools → Memory tab → Heap Snapshot
```

---

## Async JavaScript

### Q12: Promise.all vs Promise.allSettled vs Promise.race vs Promise.any

**Answer:**

```javascript
const p1 = Promise.resolve(1);
const p2 = Promise.reject('error');
const p3 = new Promise(resolve => setTimeout(() => resolve(3), 100));

// Promise.all - fails fast on ANY rejection
await Promise.all([p1, p2, p3]); // REJECTS with 'error'

// Promise.allSettled - waits for ALL, never rejects
await Promise.allSettled([p1, p2, p3]);
// [
//   { status: 'fulfilled', value: 1 },
//   { status: 'rejected', reason: 'error' },
//   { status: 'fulfilled', value: 3 }
// ]

// Promise.race - first to SETTLE (resolve or reject) wins
await Promise.race([p1, p2, p3]); // 1 (p1 settled first)

// Promise.any - first to RESOLVE wins, ignores rejections
await Promise.any([p2, p3]); // 3 (p2 rejected, p3 is first success)
// If ALL reject → AggregateError
```

**When to use:**
| Method | Use Case |
|---|---|
| `all` | Parallel requests where ALL must succeed (load dashboard data) |
| `allSettled` | Independent operations, handle each result (send notifications) |
| `race` | Timeout patterns, first-response caching |
| `any` | Fallback APIs, fastest mirror selection |

```javascript
// Practical: Fetch with timeout
async function fetchWithTimeout(url, ms) {
  return Promise.race([
    fetch(url),
    new Promise((_, reject) =>
      setTimeout(() => reject(new Error('Timeout')), ms)
    )
  ]);
}
```

---

### Q13: Explain async/await error handling patterns.

**Answer:**

```javascript
// Pattern 1: try/catch (most common)
async function fetchUser(id) {
  try {
    const response = await fetch(`/api/users/${id}`);
    if (!response.ok) throw new Error(`HTTP ${response.status}`);
    return await response.json();
  } catch (error) {
    console.error('Failed to fetch user:', error);
    throw error; // re-throw or handle
  }
}

// Pattern 2: Wrapper function (Go-style [error, result])
async function to(promise) {
  try {
    const result = await promise;
    return [null, result];
  } catch (error) {
    return [error, null];
  }
}

const [error, user] = await to(fetchUser(123));
if (error) {
  // handle error
}

// Pattern 3: .catch() on the promise
const user = await fetchUser(123).catch(err => {
  console.error(err);
  return null; // fallback
});

// Pattern 4: Error boundary for multiple operations
async function processOrder(orderId) {
  const [user, order, inventory] = await Promise.all([
    fetchUser(orderId).catch(() => null),     // graceful fallback
    fetchOrder(orderId),                       // must succeed
    checkInventory(orderId).catch(() => ({})), // graceful fallback
  ]);

  if (!order) throw new Error('Order not found');
  // continue with available data...
}

// Anti-pattern: forgetting to await (creates unhandled rejection)
async function bad() {
  doSomethingAsync(); // no await! Error will be unhandled
}
```

---

### Q14: Explain Event Emitter pattern and implement one.

**Answer:**

```javascript
class EventEmitter {
  constructor() {
    this.events = new Map();
  }

  on(event, listener) {
    if (!this.events.has(event)) {
      this.events.set(event, []);
    }
    this.events.get(event).push(listener);
    return this; // for chaining
  }

  off(event, listener) {
    const listeners = this.events.get(event);
    if (listeners) {
      this.events.set(event, listeners.filter(l => l !== listener && l._original !== listener));
    }
    return this;
  }

  once(event, listener) {
    const wrapper = (...args) => {
      listener(...args);
      this.off(event, wrapper);
    };
    wrapper._original = listener;
    return this.on(event, wrapper);
  }

  emit(event, ...args) {
    const listeners = this.events.get(event);
    if (!listeners) return false;
    listeners.forEach(listener => listener(...args));
    return true;
  }
}

// Usage
const emitter = new EventEmitter();
emitter.on('trade', (data) => console.log('Trade:', data));
emitter.once('connect', () => console.log('Connected!'));
emitter.emit('trade', { symbol: 'SOL', price: 95 });
emitter.emit('connect'); // logs once
emitter.emit('connect'); // no output (removed after first)
```

---

## ES6+ Features

### Q15: Explain destructuring, spread/rest, and optional chaining.

**Answer:**

```javascript
// Destructuring
const { name, age, address: { city } = {} } = user; // nested + default
const [first, , third, ...rest] = [1, 2, 3, 4, 5];  // skip + rest
const { id, ...userWithoutId } = user;                // omit a property

// Swap variables
let a = 1, b = 2;
[a, b] = [b, a]; // a=2, b=1

// Spread
const merged = { ...defaults, ...userConfig }; // later wins
const combined = [...arr1, ...arr2];           // concat arrays
const clone = { ...original };                  // shallow clone

// Rest parameters
function sum(...nums) {
  return nums.reduce((a, b) => a + b, 0);
}
sum(1, 2, 3); // 6

// Optional chaining (?.)
const city = user?.address?.city;           // undefined if any is null/undefined
const first = users?.[0]?.name;             // array access
const result = user?.getProfile?.();        // method call
const value = map?.get?.('key');            // method on possibly null object

// Nullish coalescing (??)
const port = config.port ?? 3000;           // only if null/undefined
const name = user.name ?? 'Anonymous';      // '' and 0 are kept (unlike ||)

// Difference: ?? vs ||
0 || 'default';    // 'default' (0 is falsy)
0 ?? 'default';    // 0 (0 is not null/undefined)
'' || 'default';   // 'default'
'' ?? 'default';   // ''
```

---

### Q16: Explain Symbol and its use cases.

**Answer:**

```javascript
// Symbol - unique, immutable identifier
const sym1 = Symbol('description');
const sym2 = Symbol('description');
sym1 === sym2; // false (always unique)

// Use case 1: Private-ish object properties
const _private = Symbol('private');
class User {
  constructor(name) {
    this.name = name;
    this[_private] = 'secret data';
  }
}
const user = new User('Awais');
user.name;      // 'Awais'
user[_private];  // 'secret data' (only if you have the symbol reference)
// Object.keys(user) → ['name'] (symbols not included)

// Use case 2: Well-known symbols (customize language behavior)
class TradeList {
  constructor(trades) { this.trades = trades; }

  // Make iterable
  [Symbol.iterator]() {
    let index = 0;
    const trades = this.trades;
    return {
      next() {
        return index < trades.length
          ? { value: trades[index++], done: false }
          : { done: true };
      }
    };
  }

  // Customize string conversion
  [Symbol.toPrimitive](hint) {
    if (hint === 'number') return this.trades.length;
    if (hint === 'string') return `TradeList(${this.trades.length})`;
    return this.trades.length;
  }
}

// Use case 3: Global symbol registry
const shared = Symbol.for('app.shared'); // creates or retrieves
Symbol.for('app.shared') === shared;      // true (same symbol)
Symbol.keyFor(shared);                    // 'app.shared'
```

---

## TypeScript

### Q17: TypeScript Generics - Explain with practical examples.

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
  timestamp: string;
}

// Usage
type UserResponse = ApiResponse<User>;
type TradeResponse = ApiResponse<Trade[]>;

// Generic with default
interface PaginatedResponse<T, M = Record<string, unknown>> {
  items: T[];
  total: number;
  page: number;
  pageSize: number;
  meta?: M;
}

// Generic class
class Repository<T extends { id: string }> {
  private items: Map<string, T> = new Map();

  save(item: T): void { this.items.set(item.id, item); }
  findById(id: string): T | undefined { return this.items.get(id); }
  findAll(): T[] { return [...this.items.values()]; }
}

const userRepo = new Repository<User>();
userRepo.save({ id: '1', name: 'Awais' }); // ✅
userRepo.save({ name: 'Awais' });            // ❌ missing id
```

---

### Q18: TypeScript Utility Types.

**Answer:**

```typescript
interface User {
  id: string;
  name: string;
  email: string;
  age: number;
  role: 'admin' | 'user';
}

// Partial<T> - all properties optional
type UpdateUserDto = Partial<User>;
// { id?: string; name?: string; email?: string; ... }

// Required<T> - all properties required
type CompleteUser = Required<User>;

// Pick<T, K> - select specific properties
type UserPreview = Pick<User, 'id' | 'name'>;
// { id: string; name: string; }

// Omit<T, K> - exclude specific properties
type CreateUserDto = Omit<User, 'id'>;
// { name: string; email: string; age: number; role: ... }

// Record<K, V> - construct type with keys K and values V
type UserRoles = Record<string, 'admin' | 'user' | 'moderator'>;

// Extract<T, U> - extract types assignable to U
type StringOrNumber = string | number | boolean;
type OnlyStrNum = Extract<StringOrNumber, string | number>; // string | number

// Exclude<T, U> - exclude types assignable to U
type OnlyBool = Exclude<StringOrNumber, string | number>; // boolean

// ReturnType<T> - get function return type
function getUser() { return { id: '1', name: 'Awais' }; }
type UserReturn = ReturnType<typeof getUser>; // { id: string; name: string; }

// Parameters<T> - get function parameter types
type GetUserParams = Parameters<typeof getUser>; // []

// NonNullable<T>
type MaybeString = string | null | undefined;
type DefiniteString = NonNullable<MaybeString>; // string

// Readonly<T>
type ImmutableUser = Readonly<User>;
// All properties are readonly
```

---

### Q19: Conditional Types and `infer` keyword.

**Answer:**

```typescript
// Conditional types - like ternary for types
type IsString<T> = T extends string ? true : false;
type A = IsString<'hello'>; // true
type B = IsString<42>;      // false

// Distributive conditional types (distributes over unions)
type ToArray<T> = T extends any ? T[] : never;
type Result = ToArray<string | number>; // string[] | number[]

// Non-distributive (wrap in tuple)
type ToArrayND<T> = [T] extends [any] ? T[] : never;
type Result2 = ToArrayND<string | number>; // (string | number)[]

// infer - extract types within conditional
type UnpackPromise<T> = T extends Promise<infer U> ? U : T;
type A = UnpackPromise<Promise<string>>; // string
type B = UnpackPromise<number>;          // number

// Practical: Extract array element type
type ElementOf<T> = T extends (infer E)[] ? E : never;
type Item = ElementOf<string[]>; // string

// Practical: Extract function return type (how ReturnType works)
type MyReturnType<T> = T extends (...args: any[]) => infer R ? R : never;

// Practical: Extract props type from React component
type PropsOf<T> = T extends React.ComponentType<infer P> ? P : never;

// Nested infer
type FirstArg<T> = T extends (first: infer F, ...rest: any[]) => any ? F : never;
type F = FirstArg<(name: string, age: number) => void>; // string
```

---

### Q20: What is the `satisfies` operator in TypeScript?

**Answer:**

```typescript
// Problem: type annotation widens the type
type Colors = Record<string, string | string[]>;

// With annotation - loses specificity
const colors1: Colors = {
  red: '#ff0000',
  blue: ['#0000ff', '#0000cc'],
};
colors1.red.toUpperCase(); // ❌ ERROR: Property 'toUpperCase' does not exist on string | string[]

// With satisfies - validates AND keeps narrow type
const colors2 = {
  red: '#ff0000',
  blue: ['#0000ff', '#0000cc'],
} satisfies Colors;

colors2.red.toUpperCase();  // ✅ TypeScript knows it's a string
colors2.blue.map(c => c);   // ✅ TypeScript knows it's string[]

// Another example: config objects
type Config = Record<string, string | number | boolean>;

const config = {
  port: 3000,
  host: 'localhost',
  debug: true,
} satisfies Config;

config.port.toFixed(2);  // ✅ TypeScript knows port is number
// config.port = 'abc';  // ❌ Would error - must be number
```

---

### Q21: Discriminated Unions and Type Guards.

**Answer:**

```typescript
// Discriminated union - union with a common literal property
type Trade =
  | { type: 'market'; symbol: string; amount: number }
  | { type: 'limit'; symbol: string; amount: number; price: number }
  | { type: 'stop'; symbol: string; amount: number; stopPrice: number; limitPrice: number };

function executeTrade(trade: Trade) {
  switch (trade.type) {
    case 'market':
      // TypeScript narrows to market trade
      return placeMarketOrder(trade.symbol, trade.amount);
    case 'limit':
      // TypeScript knows trade.price exists
      return placeLimitOrder(trade.symbol, trade.amount, trade.price);
    case 'stop':
      // TypeScript knows stopPrice and limitPrice exist
      return placeStopOrder(trade.symbol, trade.amount, trade.stopPrice, trade.limitPrice);
  }
}

// Custom type guard
function isLimitTrade(trade: Trade): trade is Extract<Trade, { type: 'limit' }> {
  return trade.type === 'limit';
}

if (isLimitTrade(trade)) {
  console.log(trade.price); // ✅ narrowed
}

// Type guard with assertion
function assertNonNull<T>(value: T | null | undefined, msg?: string): asserts value is T {
  if (value == null) throw new Error(msg ?? 'Value is null');
}

const user = getUser(); // User | null
assertNonNull(user, 'User not found');
user.name; // ✅ TypeScript knows user is User (not null)

// typeof guard
function padLeft(value: string, padding: string | number) {
  if (typeof padding === 'number') {
    return ' '.repeat(padding) + value; // narrowed to number
  }
  return padding + value; // narrowed to string
}

// instanceof guard
if (error instanceof HttpException) {
  error.getStatus(); // narrowed
}

// 'in' guard
if ('price' in trade) {
  trade.price; // narrowed to type that has price
}
```

---

### Q22: Template Literal Types and Mapped Types.

**Answer:**

```typescript
// Template literal types
type HttpMethod = 'GET' | 'POST' | 'PUT' | 'DELETE';
type ApiVersion = 'v1' | 'v2';
type Endpoint = `/${ApiVersion}/${string}`;

type Route = `${HttpMethod} ${Endpoint}`;
// "GET /v1/users" | "POST /v1/users" | "GET /v2/users" | ...

// Event handler types
type EventName = 'click' | 'focus' | 'blur';
type HandlerName = `on${Capitalize<EventName>}`;
// "onClick" | "onFocus" | "onBlur"

// Mapped types - transform each property
type Readonly<T> = { readonly [K in keyof T]: T[K] };
type Optional<T> = { [K in keyof T]?: T[K] };
type Nullable<T> = { [K in keyof T]: T[K] | null };

// Mapped type with key remapping (as clause)
type Getters<T> = {
  [K in keyof T as `get${Capitalize<string & K>}`]: () => T[K];
};

interface User { name: string; age: number; }
type UserGetters = Getters<User>;
// { getName: () => string; getAge: () => number; }

// Remove specific keys
type RemoveReadonly<T> = {
  -readonly [K in keyof T]: T[K]; // remove readonly modifier
};

// Filter properties by type
type StringKeys<T> = {
  [K in keyof T as T[K] extends string ? K : never]: T[K];
};

type UserStrings = StringKeys<User>;
// { name: string; } (age filtered out)
```

---

## Tricky Output Questions

### Q23: What is the output?

```javascript
// Question 1
console.log(typeof null);          // ?
console.log(typeof undefined);     // ?
console.log(typeof NaN);           // ?
console.log(null === undefined);   // ?
console.log(null == undefined);    // ?
console.log(NaN === NaN);          // ?

// Answers:
// "object"    ← historical bug
// "undefined"
// "number"    ← NaN is a number type
// false
// true        ← special rule
// false       ← NaN is never equal to anything

// Question 2
console.log([] == ![]);   // ?
// true!
// ![] is false (empty array is truthy, !truthy = false)
// [] == false → both coerce to 0 → 0 == 0 → true

// Question 3
console.log(+'');     // 0
console.log(+' ');    // 0
console.log(+'0');    // 0
console.log(+'1');    // 1
console.log(+true);   // 1
console.log(+false);  // 0
console.log(+null);   // 0
console.log(+undefined); // NaN
console.log(+{});     // NaN
console.log(+[]);     // 0
console.log(+[1]);    // 1
console.log(+[1,2]);  // NaN

// Question 4
const a = {};
const b = { key: 'b' };
const c = { key: 'c' };

a[b] = 123; // a["[object Object]"] = 123
a[c] = 456; // a["[object Object]"] = 456 (overwrites!)

console.log(a[b]); // 456

// Question 5
var x = 1;
function foo() {
  console.log(x);  // undefined (hoisted var x shadows global)
  var x = 2;
  console.log(x);  // 2
}
foo();
```

---

### Q24: What is the output? (Async)

```javascript
// Question 1
async function foo() {
  console.log('A');
  const result = await Promise.resolve('B');
  console.log(result);
  console.log('C');
}

console.log('D');
foo();
console.log('E');

// Output: D, A, E, B, C
// Explanation:
// 'D' - synchronous
// 'A' - foo runs synchronously until first await
// 'E' - after await, foo is paused, sync continues
// 'B', 'C' - microtask from await resolves

// Question 2
console.log('start');

setTimeout(() => console.log('timeout 1'), 0);

Promise.resolve()
  .then(() => {
    console.log('promise 1');
    setTimeout(() => console.log('timeout 2'), 0);
  })
  .then(() => console.log('promise 2'));

setTimeout(() => console.log('timeout 3'), 0);

console.log('end');

// Output: start, end, promise 1, promise 2, timeout 1, timeout 3, timeout 2
```

---

### Q25: Implement common utility functions.

```javascript
// 1. Debounce
function debounce(fn, delay) {
  let timer;
  return function (...args) {
    clearTimeout(timer);
    timer = setTimeout(() => fn.apply(this, args), delay);
  };
}

// 2. Throttle
function throttle(fn, interval) {
  let lastTime = 0;
  return function (...args) {
    const now = Date.now();
    if (now - lastTime >= interval) {
      lastTime = now;
      fn.apply(this, args);
    }
  };
}

// 3. Deep clone
function deepClone(obj) {
  if (obj === null || typeof obj !== 'object') return obj;
  if (obj instanceof Date) return new Date(obj);
  if (obj instanceof RegExp) return new RegExp(obj);
  if (obj instanceof Map) return new Map([...obj].map(([k, v]) => [deepClone(k), deepClone(v)]));
  if (obj instanceof Set) return new Set([...obj].map(v => deepClone(v)));

  const clone = Array.isArray(obj) ? [] : {};
  for (const key of Reflect.ownKeys(obj)) {
    clone[key] = deepClone(obj[key]);
  }
  return clone;
}
// Modern: structuredClone(obj)

// 4. Flatten array
function flatten(arr, depth = Infinity) {
  return depth > 0
    ? arr.reduce((acc, val) =>
        acc.concat(Array.isArray(val) ? flatten(val, depth - 1) : val), [])
    : arr.slice();
}
// Modern: arr.flat(Infinity)

// 5. Currying
function curry(fn) {
  return function curried(...args) {
    if (args.length >= fn.length) {
      return fn.apply(this, args);
    }
    return (...moreArgs) => curried(...args, ...moreArgs);
  };
}

const add = curry((a, b, c) => a + b + c);
add(1)(2)(3);    // 6
add(1, 2)(3);    // 6
add(1)(2, 3);    // 6
```

---

## References & Deep Dive Resources

### Event Loop & Async
| Topic | Resource |
|---|---|
| Event Loop Visualization | [Loupe - Philip Roberts](http://latentflip.com/loupe/) — Interactive event loop visualizer |
| Event Loop Deep Dive | [Jake Archibald: In The Loop (JSConf)](https://www.youtube.com/watch?v=cCOL7MC4Pl0) — Best video explanation |
| Microtasks vs Macrotasks | [javascript.info - Event Loop](https://javascript.info/event-loop) |
| Promises In-Depth | [javascript.info - Promises](https://javascript.info/promise-basics) |
| async/await | [MDN - async function](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/async_function) |

### Closures, Scope & `this`
| Topic | Resource |
|---|---|
| Closures | [MDN - Closures](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Closures) |
| `this` keyword | [javascript.info - Object methods, this](https://javascript.info/object-methods) |
| Scope Chain | [javascript.info - Variable scope, closure](https://javascript.info/closure) |
| Hoisting Explained | [MDN - Hoisting](https://developer.mozilla.org/en-US/docs/Glossary/Hoisting) |
| call/apply/bind | [javascript.info - Function binding](https://javascript.info/bind) |

### Prototypes & OOP
| Topic | Resource |
|---|---|
| Prototypal Inheritance | [javascript.info - Prototypal inheritance](https://javascript.info/prototype-inheritance) |
| Classes in JS | [MDN - Classes](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Classes) |
| Object.create | [MDN - Object.create()](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/create) |

### ES6+ Features
| Topic | Resource |
|---|---|
| Proxy & Reflect | [javascript.info - Proxy and Reflect](https://javascript.info/proxy) |
| Generators | [javascript.info - Generators](https://javascript.info/generators) |
| Symbols | [javascript.info - Symbol type](https://javascript.info/symbol) |
| WeakMap & WeakSet | [javascript.info - WeakMap and WeakSet](https://javascript.info/weakmap-weakset) |
| Map & Set | [javascript.info - Map and Set](https://javascript.info/map-set) |
| Destructuring | [javascript.info - Destructuring](https://javascript.info/destructuring-assignment) |
| Optional Chaining | [MDN - Optional chaining](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Optional_chaining) |

### TypeScript
| Topic | Resource |
|---|---|
| TS Handbook (Official) | [TypeScript Handbook](https://www.typescriptlang.org/docs/handbook/intro.html) |
| Generics | [TS Handbook - Generics](https://www.typescriptlang.org/docs/handbook/2/generics.html) |
| Utility Types | [TS Handbook - Utility Types](https://www.typescriptlang.org/docs/handbook/utility-types.html) |
| Conditional Types | [TS Handbook - Conditional Types](https://www.typescriptlang.org/docs/handbook/2/conditional-types.html) |
| `satisfies` operator | [TS 4.9 Release Notes](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-4-9.html) |
| Discriminated Unions | [TS Handbook - Narrowing](https://www.typescriptlang.org/docs/handbook/2/narrowing.html#discriminated-unions) |
| Template Literal Types | [TS Handbook - Template Literal Types](https://www.typescriptlang.org/docs/handbook/2/template-literal-types.html) |
| Type Challenges | [type-challenges GitHub](https://github.com/type-challenges/type-challenges) — Practice TS puzzles |
| Matt Pocock TS Tips | [TotalTypeScript.com](https://www.totaltypescript.com/) — Best TS video tutorials |

### Memory & Performance
| Topic | Resource |
|---|---|
| Memory Leaks | [Chrome DevTools - Memory](https://developer.chrome.com/docs/devtools/memory-problems/) |
| Garbage Collection | [javascript.info - Garbage collection](https://javascript.info/garbage-collection) |
| V8 Engine Internals | [V8 Blog](https://v8.dev/blog) |

### Coding Practice
| Topic | Resource |
|---|---|
| Debounce/Throttle | [CSS-Tricks - Debouncing and Throttling](https://css-tricks.com/debouncing-throttling-explained-examples/) |
| Deep Clone | [structuredClone MDN](https://developer.mozilla.org/en-US/docs/Web/API/structuredClone) |
| Currying | [javascript.info - Currying](https://javascript.info/currying-partials) |
| JS Interview Questions | [70 JS Interview Questions](https://github.com/sudheerj/javascript-interview-questions) |

---

> **Back to main**: [INTERVIEW_ROADMAP.md](../INTERVIEW_ROADMAP.md)
> **Next**: [React & Next.js Q&A](./02-react-nextjs.md)
