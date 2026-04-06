# JavaScript

Modern JavaScript essentials for working with React and Next.js.

## Table of Contents

- [ES2024+ Features](#es2024-features)
- [Modules: ESM vs CJS](#modules-esm-vs-cjs)
- [Async Patterns](#async-patterns)
- [Destructuring and Spread](#destructuring-and-spread)
- [Array and Object Methods](#array-and-object-methods)
- [Template Literals and Tagged Templates](#template-literals-and-tagged-templates)
- [Optional Chaining and Nullish Coalescing](#optional-chaining-and-nullish-coalescing)
- [Closures and Scope](#closures-and-scope)
- [Tooling: Node.js 22+](#tooling-nodejs-22)

---

## ES2024+ Features

### `Object.groupBy()` and `Map.groupBy()`

Groups array elements by a callback return value.

```js
const people = [
  { name: "Alice", age: 30 },
  { name: "Bob", age: 25 },
  { name: "Charlie", age: 30 },
];

// Returns a null-prototype object
const byAge = Object.groupBy(people, (person) => person.age);
// { 25: [{ name: "Bob", age: 25 }], 30: [{ name: "Alice", age: 30 }, { name: "Charlie", age: 30 }] }

// Returns a Map — useful when keys aren't strings
const byAgeMap = Map.groupBy(people, (person) => person.age);
// Map { 25 => [...], 30 => [...] }
```

**Before (manual grouping):**

```js
const byAge = people.reduce((acc, person) => {
  (acc[person.age] ||= []).push(person);
  return acc;
}, {});
```

### `Promise.withResolvers()`

Creates a promise along with its `resolve` and `reject` functions — no more wrapping in `new Promise()`.

```js
// Before
let resolve, reject;
const promise = new Promise((res, rej) => {
  resolve = res;
  reject = rej;
});

// After
const { promise, resolve, reject } = Promise.withResolvers();

// Practical use: event-driven resolution
function waitForEvent(emitter, event) {
  const { promise, resolve } = Promise.withResolvers();
  emitter.once(event, resolve);
  return promise;
}
```

### `ArrayBuffer.prototype.transfer()`

Transfers ownership of an `ArrayBuffer` to a new one, detaching the original (zero-copy when possible).

```js
const buffer = new ArrayBuffer(8);
const newBuffer = buffer.transfer(); // original is now detached
// buffer.byteLength === 0
// newBuffer.byteLength === 8
```

### Well-formed Unicode strings

```js
// Check if a string is well-formed UTF-16
"hello".isWellFormed();       // true
"\uD800".isWellFormed();      // false (lone surrogate)

// Fix it
"\uD800".toWellFormed();      // "\uFFFD" (replacement character)
```

### `Atomics.waitAsync()`

Non-blocking version of `Atomics.wait()` — works on the main thread.

```js
const sab = new SharedArrayBuffer(4);
const int32 = new Int32Array(sab);

const result = Atomics.waitAsync(int32, 0, 0);
// result.value is a Promise that resolves when index 0 changes
result.value.then((val) => console.log(val)); // "ok" or "timed-out"
```

---

## Modules: ESM vs CJS

### Syntax comparison

```js
// ============ CommonJS (CJS) ============
const fs = require("fs");
const { readFile } = require("fs");
module.exports = { myFunc };
module.exports = myFunc;

// ============ ES Modules (ESM) ============
import fs from "fs";
import { readFile } from "fs";
export { myFunc };
export default myFunc;
```

### Enabling ESM in Node.js

In `package.json`:

```json
{
  "type": "module"
}
```

| File extension | `"type": "module"` | `"type": "commonjs"` (default) |
|---|---|---|
| `.js` | ESM | CJS |
| `.mjs` | ESM | ESM |
| `.cjs` | CJS | CJS |

### Interop gotchas

```js
// Importing CJS from ESM — works
import cjsModule from "./lib.cjs"; // gets module.exports as default

// Importing ESM from CJS — requires dynamic import
const esmModule = await import("./lib.mjs");

// Named exports from CJS may not destructure cleanly
// This may fail depending on the module:
import { specificFunc } from "./lib.cjs"; // unreliable
// Prefer:
import lib from "./lib.cjs";
const { specificFunc } = lib;
```

### Conditional exports (package.json)

Lets a package expose different entry points for ESM and CJS consumers:

```json
{
  "name": "my-lib",
  "exports": {
    ".": {
      "import": "./dist/index.mjs",
      "require": "./dist/index.cjs",
      "types": "./dist/index.d.ts"
    },
    "./utils": {
      "import": "./dist/utils.mjs",
      "require": "./dist/utils.cjs"
    }
  }
}
```

### Key differences

| Feature | CJS | ESM |
|---|---|---|
| Loading | Synchronous | Asynchronous |
| `this` at top level | `module.exports` | `undefined` |
| `__filename` / `__dirname` | Available | Use `import.meta.url` + `fileURLToPath` |
| Dynamic import | `require()` anywhere | `import()` returns a Promise |
| Tree-shaking | Not possible | Supported by bundlers |

```js
// ESM replacement for __dirname
import { fileURLToPath } from "url";
import { dirname } from "path";

const __filename = fileURLToPath(import.meta.url);
const __dirname = dirname(__filename);
```

---

## Async Patterns

### async/await basics

```js
async function fetchUser(id) {
  try {
    const response = await fetch(`/api/users/${id}`);
    if (!response.ok) throw new Error(`HTTP ${response.status}`);
    return await response.json();
  } catch (error) {
    console.error("Failed to fetch user:", error);
    throw error; // re-throw to let the caller handle it
  }
}
```

### Promise combinators

```js
const urls = ["/api/users", "/api/posts", "/api/comments"];
const fetches = urls.map((url) => fetch(url).then((r) => r.json()));

// Promise.all — fails fast if ANY rejects
const [users, posts, comments] = await Promise.all(fetches);

// Promise.allSettled — waits for ALL, never rejects
const results = await Promise.allSettled(fetches);
results.forEach((result) => {
  if (result.status === "fulfilled") console.log(result.value);
  if (result.status === "rejected") console.error(result.reason);
});

// Promise.race — resolves/rejects with the FIRST settled
const fastest = await Promise.race(fetches);

// Promise.any — resolves with the FIRST fulfilled (ignores rejections)
const firstSuccess = await Promise.any(fetches);
// Throws AggregateError only if ALL reject
```

### AbortController

Cancel fetch requests, timeouts, or any async operation.

```js
const controller = new AbortController();
const { signal } = controller;

// Cancel after 5 seconds
setTimeout(() => controller.abort(), 5000);

try {
  const response = await fetch("/api/slow-endpoint", { signal });
  const data = await response.json();
} catch (error) {
  if (error.name === "AbortError") {
    console.log("Request was cancelled");
  } else {
    throw error;
  }
}
```

**Reusable timeout helper:**

```js
async function fetchWithTimeout(url, options = {}, timeoutMs = 5000) {
  const controller = new AbortController();
  const timeout = setTimeout(() => controller.abort(), timeoutMs);

  try {
    return await fetch(url, { ...options, signal: controller.signal });
  } finally {
    clearTimeout(timeout);
  }
}
```

### Sequential vs parallel execution

```js
// SEQUENTIAL — each waits for the previous (slow)
const user = await fetchUser(1);
const posts = await fetchPosts(user.id);
const comments = await fetchComments(posts[0].id);

// PARALLEL — independent operations run concurrently (fast)
const [user, products, notifications] = await Promise.all([
  fetchUser(1),
  fetchProducts(),
  fetchNotifications(),
]);

// PARALLEL with individual error handling
const [userResult, productsResult] = await Promise.allSettled([
  fetchUser(1),
  fetchProducts(),
]);
```

### Async iteration

```js
// for-await-of — consuming async iterables
async function* fetchPages(baseUrl) {
  let page = 1;
  while (true) {
    const res = await fetch(`${baseUrl}?page=${page}`);
    const data = await res.json();
    if (data.items.length === 0) break;
    yield data.items;
    page++;
  }
}

for await (const items of fetchPages("/api/products")) {
  console.log(`Got ${items.length} items`);
}
```

---

## Destructuring and Spread

### Object destructuring

```js
const user = { name: "Alice", age: 30, role: "admin", city: "NYC" };

// Basic
const { name, age } = user;

// Rename
const { name: userName, role: userRole } = user;

// Defaults
const { theme = "dark", lang = "en" } = user;

// Rest — collect remaining properties
const { name, ...rest } = user; // rest = { age: 30, role: "admin", city: "NYC" }

// Nested
const response = { data: { user: { id: 1, email: "a@b.com" } } };
const { data: { user: { id, email } } } = response;
```

### Array destructuring

```js
const [first, second, ...remaining] = [1, 2, 3, 4, 5];
// first = 1, second = 2, remaining = [3, 4, 5]

// Skip elements
const [, , third] = [1, 2, 3]; // third = 3

// Swap variables
let a = 1, b = 2;
[a, b] = [b, a]; // a = 2, b = 1

// Default values
const [x = 0, y = 0] = [10]; // x = 10, y = 0
```

### Function parameter destructuring

```js
// Object params with defaults
function createUser({ name, role = "viewer", active = true } = {}) {
  return { name, role, active };
}
createUser({ name: "Alice" }); // { name: "Alice", role: "viewer", active: true }

// Array params
function getFirstAndLast([first, ...rest]) {
  return { first, last: rest.at(-1) };
}
```

### Spread operator

```js
// Shallow copy objects (does NOT deep clone)
const original = { a: 1, b: { c: 2 } };
const copy = { ...original };
copy.b.c = 99;
original.b.c; // 99 — nested objects are shared!

// Merge objects (last one wins)
const defaults = { theme: "light", lang: "en", debug: false };
const userPrefs = { theme: "dark", debug: true };
const config = { ...defaults, ...userPrefs };
// { theme: "dark", lang: "en", debug: true }

// Spread arrays
const arr1 = [1, 2, 3];
const arr2 = [0, ...arr1, 4]; // [0, 1, 2, 3, 4]

// Convert iterables
const chars = [..."hello"]; // ["h", "e", "l", "l", "o"]
const unique = [...new Set([1, 2, 2, 3])]; // [1, 2, 3]
```

### Shallow copy pitfall — use `structuredClone` for deep copies

```js
const original = { a: 1, b: { c: 2 }, d: [3, 4] };

// Shallow copy — nested references are shared
const shallow = { ...original };

// Deep copy — fully independent
const deep = structuredClone(original);
deep.b.c = 99;
original.b.c; // still 2
```

---

## Array and Object Methods

### Array methods cheat sheet

```js
const nums = [1, 2, 3, 4, 5];

// map — transform each element
nums.map((n) => n * 2);               // [2, 4, 6, 8, 10]

// filter — keep elements that pass a test
nums.filter((n) => n > 3);            // [4, 5]

// find / findIndex — first match
nums.find((n) => n > 3);              // 4
nums.findIndex((n) => n > 3);         // 3

// findLast / findLastIndex — last match
nums.findLast((n) => n < 4);          // 3
nums.findLastIndex((n) => n < 4);     // 2

// some / every — boolean tests
nums.some((n) => n > 4);              // true
nums.every((n) => n > 0);             // true

// reduce — accumulate a single value
nums.reduce((sum, n) => sum + n, 0);  // 15

// flatMap — map then flatten one level
[[1, 2], [3, 4]].flatMap((x) => x);              // [1, 2, 3, 4]
["hello world", "foo bar"].flatMap((s) => s.split(" ")); // ["hello", "world", "foo", "bar"]

// flat — flatten nested arrays
[1, [2, [3, [4]]]].flat();            // [1, 2, [3, [4]]]
[1, [2, [3, [4]]]].flat(Infinity);    // [1, 2, 3, 4]

// at — access by index (supports negative)
nums.at(0);                            // 1
nums.at(-1);                           // 5

// toSorted, toReversed, with — non-mutating versions
nums.toSorted((a, b) => b - a);       // [5, 4, 3, 2, 1] (original unchanged)
nums.toReversed();                     // [5, 4, 3, 2, 1]
nums.with(2, 99);                      // [1, 2, 99, 4, 5] (replace index 2)
```

### Object methods cheat sheet

```js
const user = { name: "Alice", age: 30, role: "admin" };

// Keys, values, entries
Object.keys(user);    // ["name", "age", "role"]
Object.values(user);  // ["Alice", 30, "admin"]
Object.entries(user);  // [["name", "Alice"], ["age", 30], ["role", "admin"]]

// Object.fromEntries — inverse of Object.entries
const entries = [["a", 1], ["b", 2]];
Object.fromEntries(entries); // { a: 1, b: 2 }

// Transform object values
const doubled = Object.fromEntries(
  Object.entries({ a: 1, b: 2, c: 3 }).map(([key, val]) => [key, val * 2])
);
// { a: 2, b: 4, c: 6 }

// Filter object keys
const filtered = Object.fromEntries(
  Object.entries(user).filter(([key]) => key !== "age")
);
// { name: "Alice", role: "admin" }

// Object.hasOwn — preferred over .hasOwnProperty()
Object.hasOwn(user, "name"); // true
Object.hasOwn(user, "foo");  // false

// structuredClone — deep copy
const deep = structuredClone(user);
```

### Map and Set

```js
// Map — ordered key-value pairs, any type as key
const map = new Map();
map.set("key", "value");
map.set(42, "number key");
map.set(true, "boolean key");
map.get("key");     // "value"
map.has(42);        // true
map.size;           // 3
map.delete(42);

// Iterate
for (const [key, value] of map) { /* ... */ }

// Set — unique values
const set = new Set([1, 2, 2, 3, 3]);
set.size;           // 3
set.has(2);         // true
set.add(4);
set.delete(1);

// Deduplicate an array
const unique = [...new Set([1, 1, 2, 3, 3])]; // [1, 2, 3]
```

---

## Template Literals and Tagged Templates

### Basic template literals

```js
const name = "Alice";
const age = 30;

// String interpolation
const greeting = `Hello, ${name}! You are ${age} years old.`;

// Multi-line strings
const html = `
  <div class="card">
    <h2>${name}</h2>
    <p>Age: ${age}</p>
  </div>
`;

// Expressions inside ${}
const message = `Status: ${age >= 18 ? "adult" : "minor"}`;
const calc = `Total: ${(100 * 1.15).toFixed(2)}`;
```

### Tagged templates

A function that processes template literal parts. Used by libraries like `styled-components`, `lit-html`, `graphql-tag`.

```js
function highlight(strings, ...values) {
  return strings.reduce((result, str, i) => {
    return result + str + (values[i] !== undefined ? `**${values[i]}**` : "");
  }, "");
}

const name = "Alice";
const role = "admin";
highlight`User ${name} has role ${role}`;
// "User **Alice** has role **admin**"
```

**SQL safety example:**

```js
function sql(strings, ...values) {
  const text = strings.reduce((q, str, i) => q + str + (i < values.length ? `$${i + 1}` : ""), "");
  return { text, values };
}

const id = 5;
const query = sql`SELECT * FROM users WHERE id = ${id}`;
// { text: "SELECT * FROM users WHERE id = $1", values: [5] }
```

**styled-components pattern:**

```js
// This is what styled-components does under the hood
const Button = styled.button`
  background: ${(props) => props.primary ? "blue" : "gray"};
  color: white;
  padding: 8px 16px;
`;
```

### `String.raw`

Returns the raw string without processing escape sequences.

```js
String.raw`Hello\nWorld`;   // "Hello\\nWorld" (literal backslash-n)
String.raw`C:\Users\name`;  // "C:\\Users\\name" (no escaping needed)
```

---

## Optional Chaining and Nullish Coalescing

### Optional chaining (`?.`)

Safely access deeply nested properties. Returns `undefined` if any part is `null` or `undefined`.

```js
const user = {
  name: "Alice",
  address: {
    street: "123 Main St",
  },
};

// Property access
user.address?.street;       // "123 Main St"
user.address?.zipCode;      // undefined
user.contact?.email;        // undefined (no error)

// Method calls
const arr = null;
arr?.map((x) => x);         // undefined
"hello"?.toUpperCase();     // "HELLO"

// Array indexing
const users = [{ name: "Alice" }];
users?.[0]?.name;           // "Alice"
users?.[5]?.name;           // undefined

// Function calls
const callback = null;
callback?.();               // undefined (no error)

// Chaining multiple levels
const company = {};
company?.departments?.[0]?.manager?.name; // undefined
```

### Nullish coalescing (`??`)

Returns the right-hand side only when the left is `null` or `undefined` — NOT for `0`, `""`, or `false`.

```js
// ?? vs ||
const count = 0;
count || 10;    // 10  — wrong! 0 is falsy
count ?? 10;    // 0   — correct! 0 is not null/undefined

const text = "";
text || "default";    // "default" — wrong if empty string is valid
text ?? "default";    // ""        — correct

const flag = false;
flag || true;    // true  — wrong if false is intentional
flag ?? true;    // false — correct
```

### When to use which

| Value | `\|\|` returns fallback? | `??` returns fallback? |
|---|---|---|
| `null` | Yes | Yes |
| `undefined` | Yes | Yes |
| `0` | Yes | **No** |
| `""` | Yes | **No** |
| `false` | Yes | **No** |
| `NaN` | Yes | **No** |

**Rule of thumb:** Use `??` when `0`, `""`, or `false` are valid values. Use `||` when you want to fall back on any falsy value.

### Nullish coalescing assignment (`??=`)

```js
const config = { timeout: 0, retries: null };

config.timeout ??= 5000;  // stays 0 (0 is not null/undefined)
config.retries ??= 3;     // becomes 3
config.debug ??= false;   // becomes false (was undefined)
```

### Combining `?.` and `??`

```js
const user = { settings: { theme: "" } };

// Get nested value with fallback
const theme = user.settings?.theme ?? "dark";  // "" (empty string is not nullish)
const lang = user.settings?.lang ?? "en";      // "en" (undefined triggers fallback)
```

---

## Closures and Scope

### Lexical scope

JavaScript uses lexical (static) scoping — a function can access variables from the scope where it was **defined**, not where it's called.

```js
const multiplier = 3;

function multiply(x) {
  return x * multiplier; // accesses multiplier from outer scope
}

multiply(5); // 15
```

### Closures

A closure is a function that retains access to its outer scope's variables even after the outer function has returned.

```js
function createCounter(initial = 0) {
  let count = initial; // enclosed variable

  return {
    increment: () => ++count,
    decrement: () => --count,
    getCount: () => count,
  };
}

const counter = createCounter(10);
counter.increment(); // 11
counter.increment(); // 12
counter.getCount();  // 12
// count is private — no direct access from outside
```

### Classic gotcha: closures in loops

```js
// BUG: all callbacks log 3
for (var i = 0; i < 3; i++) {
  setTimeout(() => console.log(i), 100);
}
// logs: 3, 3, 3 — var is function-scoped, all closures share the same i

// FIX 1: use let (block-scoped — each iteration gets its own i)
for (let i = 0; i < 3; i++) {
  setTimeout(() => console.log(i), 100);
}
// logs: 0, 1, 2

// FIX 2: IIFE (old-school)
for (var i = 0; i < 3; i++) {
  ((j) => {
    setTimeout(() => console.log(j), 100);
  })(i);
}
// logs: 0, 1, 2
```

### Stale closures in React

This is the #1 closure pitfall in modern development. Hooks capture values at render time.

```jsx
function Timer() {
  const [count, setCount] = useState(0);

  useEffect(() => {
    const id = setInterval(() => {
      // BUG: count is always 0 — stale closure!
      console.log(count);
      setCount(count + 1); // always sets to 1
    }, 1000);
    return () => clearInterval(id);
  }, []); // empty deps = closure captures initial count

  // FIX 1: use functional updater
  useEffect(() => {
    const id = setInterval(() => {
      setCount((prev) => prev + 1); // always has latest value
    }, 1000);
    return () => clearInterval(id);
  }, []);

  // FIX 2: include count in deps (but re-creates interval each render)
  useEffect(() => {
    const id = setInterval(() => {
      setCount(count + 1);
    }, 1000);
    return () => clearInterval(id);
  }, [count]);

  // FIX 3: useRef for mutable latest value
  const countRef = useRef(count);
  countRef.current = count;

  useEffect(() => {
    const id = setInterval(() => {
      console.log(countRef.current); // always current
    }, 1000);
    return () => clearInterval(id);
  }, []);

  return <div>{count}</div>;
}
```

### Practical closure patterns

```js
// Private state via closure
function createAuthManager(secret) {
  const token = encrypt(secret);
  return {
    getToken: () => token,
    isValid: () => verify(token),
  };
}

// Memoization
function memoize(fn) {
  const cache = new Map();
  return (...args) => {
    const key = JSON.stringify(args);
    if (cache.has(key)) return cache.get(key);
    const result = fn(...args);
    cache.set(key, result);
    return result;
  };
}
const expensiveCalc = memoize((n) => /* heavy computation */ n * n);

// Partial application
function partial(fn, ...presetArgs) {
  return (...laterArgs) => fn(...presetArgs, ...laterArgs);
}
const add = (a, b) => a + b;
const add10 = partial(add, 10);
add10(5); // 15
```

---

## Tooling: Node.js 22+

### Built-in test runner

No need for Jest or Vitest for simple tests.

```js
// math.test.js
import { describe, it } from "node:test";
import assert from "node:assert/strict";

describe("math", () => {
  it("adds numbers", () => {
    assert.strictEqual(1 + 2, 3);
  });

  it("handles async", async () => {
    const result = await Promise.resolve(42);
    assert.strictEqual(result, 42);
  });

  it("throws on invalid input", () => {
    assert.throws(() => JSON.parse("invalid"), SyntaxError);
  });
});
```

```bash
# Run tests
node --test

# Run specific file
node --test math.test.js

# With glob pattern
node --test **/*.test.js

# Coverage
node --test --experimental-test-coverage
```

### Watch mode

Auto-restart on file changes — no `nodemon` needed.

```bash
# Watch and re-run
node --watch server.js

# Watch mode for tests
node --test --watch
```

### Native `.env` support

Built-in since Node.js 20.6+ — no `dotenv` package required.

```bash
# .env
DB_HOST=localhost
DB_PORT=5432
API_KEY=secret123
```

```bash
# Load .env automatically
node --env-file=.env server.js

# Multiple env files (later files override earlier)
node --env-file=.env --env-file=.env.local server.js
```

```js
// Access in code
process.env.DB_HOST; // "localhost"
```

### Permission model

Restrict what Node.js can access — useful for running untrusted code.

```bash
# Allow read access to specific directory only
node --experimental-permission --allow-fs-read=/app/data server.js

# Allow write to specific path
node --experimental-permission --allow-fs-write=/tmp server.js

# Allow child processes
node --experimental-permission --allow-child-process server.js

# Allow network (specific host)
node --experimental-permission --allow-worker server.js

# Combine permissions
node --experimental-permission \
  --allow-fs-read=/app \
  --allow-fs-write=/tmp \
  server.js
```

```js
// Check permissions at runtime
process.permission.has("fs.read");           // true/false
process.permission.has("fs.write", "/tmp");  // true/false
```

### Other useful flags

```bash
# TypeScript support (Node.js 22.6+ with --experimental-strip-types)
node --experimental-strip-types app.ts

# Import JSON
node --experimental-json-modules app.js

# Single executable application
node --experimental-sea-config sea-config.json
```

---

## References

- [MDN JavaScript Reference](https://developer.mozilla.org/en-US/docs/Web/JavaScript)
- [Node.js Docs](https://nodejs.org/docs/latest/api/)
- [TC39 Proposals](https://github.com/tc39/proposals)
- [Node.js 22 Release Notes](https://nodejs.org/en/blog/announcements/v22-release-announce)
