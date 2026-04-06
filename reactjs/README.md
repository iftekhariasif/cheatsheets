# React.js

React 19 patterns, Server Components, and modern best practices.

## Table of Contents

- [React 19 — What's New](#react-19--whats-new)
- [Server Components vs Client Components](#server-components-vs-client-components)
- [Hooks Deep Dive](#hooks-deep-dive)
- [Suspense and Error Boundaries](#suspense-and-error-boundaries)
- [State Management Patterns](#state-management-patterns)
- [Performance Optimization](#performance-optimization)
- [Forms and Actions](#forms-and-actions)
- [Testing](#testing)

---

## React 19 — What's New

### Actions

Functions that use async transitions. They handle pending states, errors, and optimistic updates automatically.

```jsx
function UpdateName() {
  const [name, setName] = useState("");
  const [error, setError] = useState(null);
  const [isPending, startTransition] = useTransition();

  const handleSubmit = () => {
    startTransition(async () => {
      const error = await updateNameOnServer(name);
      if (error) {
        setError(error);
        return;
      }
      redirect("/profile");
    });
  };

  return (
    <div>
      <input value={name} onChange={(e) => setName(e.target.value)} />
      <button onClick={handleSubmit} disabled={isPending}>
        Update
      </button>
      {error && <p>{error}</p>}
    </div>
  );
}
```

### `useActionState`

Manages form action state — replaces the old `useFormState`.

```jsx
import { useActionState } from "react";

async function addToCart(prevState, formData) {
  const itemId = formData.get("itemId");
  try {
    await api.addToCart(itemId);
    return { success: true, message: "Added to cart" };
  } catch (e) {
    return { success: false, message: e.message };
  }
}

function AddToCartForm({ itemId }) {
  const [state, formAction, isPending] = useActionState(addToCart, null);

  return (
    <form action={formAction}>
      <input type="hidden" name="itemId" value={itemId} />
      <button disabled={isPending}>
        {isPending ? "Adding..." : "Add to Cart"}
      </button>
      {state?.message && <p>{state.message}</p>}
    </form>
  );
}
```

### `useOptimistic`

Show optimistic state while an async action is in progress.

```jsx
import { useOptimistic } from "react";

function Messages({ messages, sendMessage }) {
  const [optimisticMessages, addOptimistic] = useOptimistic(
    messages,
    (currentMessages, newMessage) => [
      ...currentMessages,
      { text: newMessage, sending: true },
    ]
  );

  async function handleSend(formData) {
    const message = formData.get("message");
    addOptimistic(message);
    await sendMessage(message);
  }

  return (
    <>
      {optimisticMessages.map((msg, i) => (
        <div key={i} style={{ opacity: msg.sending ? 0.5 : 1 }}>
          {msg.text}
        </div>
      ))}
      <form action={handleSend}>
        <input name="message" />
        <button>Send</button>
      </form>
    </>
  );
}
```

### `use()` API

Read resources (promises, context) during render. Can be called conditionally — unlike hooks.

```jsx
import { use, Suspense } from "react";

// Reading a promise
function UserProfile({ userPromise }) {
  const user = use(userPromise); // suspends until resolved
  return <h1>{user.name}</h1>;
}

// Usage
<Suspense fallback={<Loading />}>
  <UserProfile userPromise={fetchUser(id)} />
</Suspense>

// Reading context conditionally
function ThemeText({ showTheme }) {
  if (showTheme) {
    const theme = use(ThemeContext); // valid — use() can be conditional
    return <p>Current theme: {theme}</p>;
  }
  return <p>Theme hidden</p>;
}
```

### `ref` as a prop

No more `forwardRef` — refs are passed as regular props.

```jsx
// React 19 — ref is just a prop
function Input({ placeholder, ref }) {
  return <input ref={ref} placeholder={placeholder} />;
}

// Usage
function Form() {
  const inputRef = useRef(null);
  return <Input ref={inputRef} placeholder="Enter name" />;
}
```

### Ref cleanup functions

Like `useEffect` cleanup, but for refs.

```jsx
function VideoPlayer({ src }) {
  return (
    <video
      ref={(el) => {
        if (el) el.play();
        // Cleanup function — runs when ref is detached
        return () => el?.pause();
      }}
      src={src}
    />
  );
}
```

### Document metadata

Render `<title>`, `<meta>`, and `<link>` anywhere in your component tree.

```jsx
function BlogPost({ post }) {
  return (
    <article>
      <title>{post.title}</title>
      <meta name="description" content={post.excerpt} />
      <link rel="canonical" href={`https://example.com/posts/${post.slug}`} />
      <h1>{post.title}</h1>
      <p>{post.content}</p>
    </article>
  );
}
```

### Stylesheet and asset loading

```jsx
// Stylesheets with precedence control
function Component() {
  return (
    <>
      <link rel="stylesheet" href="/base.css" precedence="default" />
      <link rel="stylesheet" href="/theme.css" precedence="high" />
      <div>Content</div>
    </>
  );
}

// Preloading resources
import { preload, prefetchDNS, preconnect } from "react-dom";

function App() {
  preload("/font.woff2", { as: "font" });
  preconnect("https://api.example.com");
  prefetchDNS("https://cdn.example.com");
  return <div>App</div>;
}
```

---

## Server Components vs Client Components

### The mental model

| | Server Components (default) | Client Components (`"use client"`) |
|---|---|---|
| **Runs on** | Server only | Server (SSR) + Client (hydration) |
| **Can use** | `async/await`, DB, filesystem, env vars | Hooks, state, effects, browser APIs |
| **Bundle impact** | Zero — not sent to client | Included in JS bundle |
| **Can import** | Server + Client components | Client components only |
| **Can pass to children** | Any serializable props | Any serializable props |

### Decision tree: which to use?

```
Need useState, useEffect, onClick, onChange, browser APIs?
  → YES → "use client"
  → NO  → Keep as Server Component (default)

Need to fetch data?
  → Server Component: use async/await directly
  → Client Component: use useEffect, SWR, or TanStack Query

Need to access DB, filesystem, env secrets?
  → Server Component only
```

### `"use client"` directive

Marks the **boundary** — everything imported into this file becomes client code.

```jsx
"use client";

import { useState } from "react";

export function Counter() {
  const [count, setCount] = useState(0);
  return <button onClick={() => setCount(count + 1)}>Count: {count}</button>;
}
```

### `"use server"` directive

Marks functions as Server Actions — callable from the client but execute on the server.

```jsx
// actions.js
"use server";

export async function createPost(formData) {
  const title = formData.get("title");
  await db.posts.create({ title });
  revalidatePath("/posts");
}
```

### Composition patterns

Server Components can render Client Components, but not the other way around. Use the **children pattern** to compose them:

```jsx
// ServerWrapper.jsx (Server Component)
import { ClientInteractive } from "./ClientInteractive";

export function ServerWrapper() {
  const data = await db.query("SELECT * FROM items"); // server-only

  return (
    <ClientInteractive>
      {/* These Server Components render on the server, passed as children */}
      <ExpensiveServerComponent data={data} />
    </ClientInteractive>
  );
}

// ClientInteractive.jsx
"use client";

import { useState } from "react";

export function ClientInteractive({ children }) {
  const [isOpen, setIsOpen] = useState(false);
  return (
    <div>
      <button onClick={() => setIsOpen(!isOpen)}>Toggle</button>
      {isOpen && children}
    </div>
  );
}
```

### Serialization boundary

Props passed from Server to Client Components must be serializable:

```
Allowed: strings, numbers, booleans, null, arrays, plain objects,
         Date, Map, Set, TypedArrays, FormData, Server Actions

NOT allowed: functions (except Server Actions), classes, DOM nodes,
             symbols, closures
```

---

## Hooks Deep Dive

### useState

```jsx
// Basic
const [count, setCount] = useState(0);

// Functional updater — use when new state depends on old
setCount((prev) => prev + 1);

// Lazy initialization — expensive computation runs only once
const [data, setData] = useState(() => computeExpensiveInitialValue());

// Objects — always spread to avoid losing fields
const [user, setUser] = useState({ name: "", age: 0 });
setUser((prev) => ({ ...prev, name: "Alice" }));
```

### useEffect

```jsx
// Runs after every render
useEffect(() => {
  console.log("rendered");
});

// Runs once on mount
useEffect(() => {
  const subscription = subscribe();
  return () => subscription.unsubscribe(); // cleanup
}, []);

// Runs when dependencies change
useEffect(() => {
  fetchUser(userId);
}, [userId]);
```

**Common mistakes:**

```jsx
// BAD — object/array in deps (new reference every render = infinite loop)
useEffect(() => { /* ... */ }, [{ id: 1 }]);

// GOOD — use primitive values or useMemo
useEffect(() => { /* ... */ }, [user.id]);

// BAD — missing dependency
const [count, setCount] = useState(0);
useEffect(() => {
  const id = setInterval(() => setCount(count + 1), 1000); // stale!
  return () => clearInterval(id);
}, []);

// GOOD — functional updater
useEffect(() => {
  const id = setInterval(() => setCount((c) => c + 1), 1000);
  return () => clearInterval(id);
}, []);
```

### useRef

```jsx
// DOM reference
function TextInput() {
  const inputRef = useRef(null);
  return (
    <>
      <input ref={inputRef} />
      <button onClick={() => inputRef.current.focus()}>Focus</button>
    </>
  );
}

// Mutable value that doesn't trigger re-renders
function Timer() {
  const intervalRef = useRef(null);
  const countRef = useRef(0);

  useEffect(() => {
    intervalRef.current = setInterval(() => {
      countRef.current += 1; // no re-render
    }, 1000);
    return () => clearInterval(intervalRef.current);
  }, []);
}

// Store previous value
function usePrevious(value) {
  const ref = useRef();
  useEffect(() => {
    ref.current = value;
  });
  return ref.current;
}
```

### useMemo and useCallback

```jsx
// useMemo — memoize a computed value
const sortedList = useMemo(
  () => items.toSorted((a, b) => a.name.localeCompare(b.name)),
  [items]
);

// useCallback — memoize a function reference
const handleClick = useCallback(
  (id) => setSelected(id),
  [setSelected]
);

// When to use:
// - Expensive computations → useMemo
// - Passing callbacks to memoized children → useCallback
// - DON'T use for simple operations — adds overhead
```

### useTransition

Mark state updates as non-urgent — keeps the UI responsive.

```jsx
function Search() {
  const [query, setQuery] = useState("");
  const [results, setResults] = useState([]);
  const [isPending, startTransition] = useTransition();

  function handleChange(e) {
    setQuery(e.target.value); // urgent — update input immediately

    startTransition(() => {
      setResults(filterHugeList(e.target.value)); // non-urgent — can be interrupted
    });
  }

  return (
    <>
      <input value={query} onChange={handleChange} />
      {isPending && <Spinner />}
      <ResultsList results={results} />
    </>
  );
}
```

### useDeferredValue

Defer re-rendering of a value — similar to debouncing but React-aware.

```jsx
function SearchResults({ query }) {
  const deferredQuery = useDeferredValue(query);
  const isStale = query !== deferredQuery;

  return (
    <div style={{ opacity: isStale ? 0.5 : 1 }}>
      <ExpensiveList query={deferredQuery} />
    </div>
  );
}
```

### Custom hooks

Extract reusable stateful logic:

```jsx
// useLocalStorage
function useLocalStorage(key, initialValue) {
  const [value, setValue] = useState(() => {
    try {
      const item = localStorage.getItem(key);
      return item ? JSON.parse(item) : initialValue;
    } catch {
      return initialValue;
    }
  });

  useEffect(() => {
    localStorage.setItem(key, JSON.stringify(value));
  }, [key, value]);

  return [value, setValue];
}

// useDebounce
function useDebounce(value, delay) {
  const [debounced, setDebounced] = useState(value);

  useEffect(() => {
    const timer = setTimeout(() => setDebounced(value), delay);
    return () => clearTimeout(timer);
  }, [value, delay]);

  return debounced;
}

// useFetch
function useFetch(url) {
  const [data, setData] = useState(null);
  const [error, setError] = useState(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    const controller = new AbortController();

    fetch(url, { signal: controller.signal })
      .then((res) => res.json())
      .then(setData)
      .catch((err) => {
        if (err.name !== "AbortError") setError(err);
      })
      .finally(() => setLoading(false));

    return () => controller.abort();
  }, [url]);

  return { data, error, loading };
}
```

---

## Suspense and Error Boundaries

### Suspense for data fetching

```jsx
import { Suspense } from "react";

function App() {
  return (
    <Suspense fallback={<PageSkeleton />}>
      <Dashboard />
    </Suspense>
  );
}

// Async Server Component — suspends automatically
async function Dashboard() {
  const stats = await fetchStats(); // suspends until resolved
  return <StatsGrid stats={stats} />;
}
```

### Nested Suspense boundaries

Each boundary independently shows its own fallback:

```jsx
function Page() {
  return (
    <div className="grid">
      <Suspense fallback={<HeaderSkeleton />}>
        <Header />
      </Suspense>

      <Suspense fallback={<SidebarSkeleton />}>
        <Sidebar />
      </Suspense>

      <Suspense fallback={<ContentSkeleton />}>
        <MainContent />
      </Suspense>
    </div>
  );
}
// Each section loads independently — no waterfall
```

### Error boundaries

Class-based (still the only way in React — no hook equivalent yet):

```jsx
class ErrorBoundary extends React.Component {
  constructor(props) {
    super(props);
    this.state = { hasError: false, error: null };
  }

  static getDerivedStateFromError(error) {
    return { hasError: true, error };
  }

  componentDidCatch(error, errorInfo) {
    console.error("Error boundary caught:", error, errorInfo);
    // Send to error reporting service
  }

  render() {
    if (this.state.hasError) {
      return this.props.fallback || <h1>Something went wrong.</h1>;
    }
    return this.props.children;
  }
}

// Usage
<ErrorBoundary fallback={<ErrorPage />}>
  <Suspense fallback={<Loading />}>
    <Dashboard />
  </Suspense>
</ErrorBoundary>
```

### Combining Suspense + Error Boundary

```jsx
function AsyncSection({ children, fallback, errorFallback }) {
  return (
    <ErrorBoundary fallback={errorFallback}>
      <Suspense fallback={fallback}>
        {children}
      </Suspense>
    </ErrorBoundary>
  );
}

// Usage
<AsyncSection
  fallback={<TableSkeleton />}
  errorFallback={<p>Failed to load users</p>}
>
  <UserTable />
</AsyncSection>
```

---

## State Management Patterns

### Lifting state up

Share state between siblings by moving it to their common parent:

```jsx
function Parent() {
  const [selected, setSelected] = useState(null);

  return (
    <>
      <Sidebar items={items} onSelect={setSelected} />
      <Detail item={selected} />
    </>
  );
}
```

### useReducer — complex state logic

```jsx
const initialState = { items: [], loading: false, error: null };

function cartReducer(state, action) {
  switch (action.type) {
    case "ADD_ITEM":
      return { ...state, items: [...state.items, action.payload] };
    case "REMOVE_ITEM":
      return { ...state, items: state.items.filter((i) => i.id !== action.payload) };
    case "SET_LOADING":
      return { ...state, loading: action.payload };
    case "SET_ERROR":
      return { ...state, error: action.payload };
    default:
      throw new Error(`Unknown action: ${action.type}`);
  }
}

function Cart() {
  const [state, dispatch] = useReducer(cartReducer, initialState);

  const addItem = (item) => dispatch({ type: "ADD_ITEM", payload: item });
  const removeItem = (id) => dispatch({ type: "REMOVE_ITEM", payload: id });

  return (
    <div>
      {state.items.map((item) => (
        <CartItem key={item.id} item={item} onRemove={() => removeItem(item.id)} />
      ))}
    </div>
  );
}
```

### Context API — avoid prop drilling

```jsx
// 1. Create context
const AuthContext = React.createContext(null);

// 2. Provider
function AuthProvider({ children }) {
  const [user, setUser] = useState(null);

  const login = async (credentials) => {
    const user = await api.login(credentials);
    setUser(user);
  };

  const logout = () => setUser(null);

  return (
    <AuthContext.Provider value={{ user, login, logout }}>
      {children}
    </AuthContext.Provider>
  );
}

// 3. Custom hook for consuming
function useAuth() {
  const context = React.useContext(AuthContext);
  if (!context) throw new Error("useAuth must be used within AuthProvider");
  return context;
}

// 4. Usage
function Navbar() {
  const { user, logout } = useAuth();
  return user ? <button onClick={logout}>Logout</button> : <LoginLink />;
}
```

### When to use external state libraries

| Scenario | Solution |
|---|---|
| Component-local state | `useState` / `useReducer` |
| Shared between a few components | Lift state + props |
| App-wide (auth, theme) | Context API |
| Frequent updates across many components | **Zustand** or **Jotai** |
| Complex async state (API caching) | **TanStack Query** or **SWR** |
| Global + DevTools + middleware | **Zustand** (lightweight Redux alternative) |

### Zustand quick example

```jsx
import { create } from "zustand";

const useStore = create((set) => ({
  count: 0,
  increment: () => set((state) => ({ count: state.count + 1 })),
  reset: () => set({ count: 0 }),
}));

function Counter() {
  const { count, increment } = useStore();
  return <button onClick={increment}>{count}</button>;
}
```

---

## Performance Optimization

### React Compiler (React Forget)

Automatically memoizes components and hooks — no more manual `useMemo`/`useCallback`.

```jsx
// With React Compiler, this is automatically optimized:
function ProductList({ products, category }) {
  const filtered = products.filter((p) => p.category === category);
  const sorted = filtered.toSorted((a, b) => a.price - b.price);

  return sorted.map((product) => (
    <ProductCard key={product.id} product={product} />
  ));
}
// Compiler auto-memoizes filtered, sorted, and the callback
```

**Setup:**

```bash
npm install babel-plugin-react-compiler
```

```js
// babel.config.js
module.exports = {
  plugins: ["babel-plugin-react-compiler"],
};
```

### React.memo — manual memoization

Skip re-rendering when props haven't changed:

```jsx
const ExpensiveList = React.memo(function ExpensiveList({ items, onSelect }) {
  return items.map((item) => (
    <div key={item.id} onClick={() => onSelect(item.id)}>
      {item.name}
    </div>
  ));
});

// Only useful when:
// 1. Component renders often with the same props
// 2. Re-rendering is actually expensive
// 3. Parent re-renders frequently but child props stay stable
```

### Code splitting with `lazy()`

```jsx
import { lazy, Suspense } from "react";

// Split by route
const Dashboard = lazy(() => import("./pages/Dashboard"));
const Settings = lazy(() => import("./pages/Settings"));

function App() {
  return (
    <Suspense fallback={<PageLoader />}>
      <Routes>
        <Route path="/dashboard" element={<Dashboard />} />
        <Route path="/settings" element={<Settings />} />
      </Routes>
    </Suspense>
  );
}

// Named export lazy loading
const AdminPanel = lazy(() =>
  import("./AdminPanel").then((mod) => ({ default: mod.AdminPanel }))
);
```

### Virtualization for long lists

Don't render thousands of DOM nodes:

```jsx
import { useVirtualizer } from "@tanstack/react-virtual";

function VirtualList({ items }) {
  const parentRef = useRef(null);

  const virtualizer = useVirtualizer({
    count: items.length,
    getScrollElement: () => parentRef.current,
    estimateSize: () => 50,
  });

  return (
    <div ref={parentRef} style={{ height: "500px", overflow: "auto" }}>
      <div style={{ height: virtualizer.getTotalSize() }}>
        {virtualizer.getVirtualItems().map((virtualItem) => (
          <div
            key={virtualItem.key}
            style={{
              position: "absolute",
              top: virtualItem.start,
              height: virtualItem.size,
              width: "100%",
            }}
          >
            {items[virtualItem.index].name}
          </div>
        ))}
      </div>
    </div>
  );
}
```

### Profiling

```jsx
// React DevTools Profiler — record and inspect renders

// Programmatic Profiler component
import { Profiler } from "react";

function onRender(id, phase, actualDuration) {
  console.log(`${id} (${phase}): ${actualDuration.toFixed(2)}ms`);
}

<Profiler id="Dashboard" onRender={onRender}>
  <Dashboard />
</Profiler>
```

---

## Forms and Actions

### Basic form with `useActionState`

```jsx
"use client";

import { useActionState } from "react";
import { submitForm } from "./actions";

function ContactForm() {
  const [state, formAction, isPending] = useActionState(submitForm, {
    message: "",
    errors: {},
  });

  return (
    <form action={formAction}>
      <div>
        <label htmlFor="email">Email</label>
        <input id="email" name="email" type="email" required />
        {state.errors?.email && <span>{state.errors.email}</span>}
      </div>

      <div>
        <label htmlFor="message">Message</label>
        <textarea id="message" name="message" required />
        {state.errors?.message && <span>{state.errors.message}</span>}
      </div>

      <button type="submit" disabled={isPending}>
        {isPending ? "Sending..." : "Send"}
      </button>

      {state.message && <p>{state.message}</p>}
    </form>
  );
}
```

### Server Action

```jsx
// actions.js
"use server";

import { revalidatePath } from "next/cache";

export async function submitForm(prevState, formData) {
  const email = formData.get("email");
  const message = formData.get("message");

  // Validate
  const errors = {};
  if (!email?.includes("@")) errors.email = "Invalid email";
  if (!message || message.length < 10) errors.message = "Min 10 characters";

  if (Object.keys(errors).length > 0) {
    return { message: "", errors };
  }

  // Save
  await db.contacts.create({ email, message });
  revalidatePath("/contacts");
  return { message: "Sent successfully!", errors: {} };
}
```

### `useFormStatus`

Access form submission status from within a child component:

```jsx
"use client";

import { useFormStatus } from "react-dom";

function SubmitButton() {
  const { pending, data, method, action } = useFormStatus();

  return (
    <button type="submit" disabled={pending}>
      {pending ? "Submitting..." : "Submit"}
    </button>
  );
}

// Must be a child of a <form> — not the component that renders the form
function MyForm() {
  return (
    <form action={submitAction}>
      <input name="name" />
      <SubmitButton /> {/* useFormStatus works here */}
    </form>
  );
}
```

### Progressive enhancement

Forms work without JavaScript — actions run on the server:

```jsx
// This form works even if JS fails to load
function SignupForm() {
  const [state, action] = useActionState(signup, null);

  return (
    <form action={action}>
      <input name="email" type="email" required />
      <input name="password" type="password" required />
      <button type="submit">Sign Up</button>
      {state?.error && <p>{state.error}</p>}
    </form>
  );
}
```

---

## Testing

### Setup

```bash
npm install -D @testing-library/react @testing-library/jest-dom @testing-library/user-event vitest jsdom
```

```js
// vitest.config.js
import { defineConfig } from "vitest/config";

export default defineConfig({
  test: {
    environment: "jsdom",
    setupFiles: ["./test/setup.js"],
  },
});
```

```js
// test/setup.js
import "@testing-library/jest-dom/vitest";
```

### Testing components

```jsx
import { render, screen } from "@testing-library/react";
import userEvent from "@testing-library/user-event";
import { describe, it, expect, vi } from "vitest";
import { Counter } from "./Counter";

describe("Counter", () => {
  it("renders initial count", () => {
    render(<Counter initialCount={5} />);
    expect(screen.getByText("Count: 5")).toBeInTheDocument();
  });

  it("increments on click", async () => {
    const user = userEvent.setup();
    render(<Counter initialCount={0} />);

    await user.click(screen.getByRole("button", { name: /increment/i }));
    expect(screen.getByText("Count: 1")).toBeInTheDocument();
  });

  it("calls onChange when count changes", async () => {
    const user = userEvent.setup();
    const onChange = vi.fn();
    render(<Counter initialCount={0} onChange={onChange} />);

    await user.click(screen.getByRole("button", { name: /increment/i }));
    expect(onChange).toHaveBeenCalledWith(1);
  });
});
```

### Testing async components

```jsx
import { render, screen, waitFor } from "@testing-library/react";

it("loads and displays user data", async () => {
  // Mock the API
  vi.spyOn(global, "fetch").mockResolvedValue({
    ok: true,
    json: () => Promise.resolve({ name: "Alice", email: "alice@example.com" }),
  });

  render(<UserProfile userId="1" />);

  // Wait for async content
  expect(await screen.findByText("Alice")).toBeInTheDocument();
  expect(screen.getByText("alice@example.com")).toBeInTheDocument();

  global.fetch.mockRestore();
});
```

### Testing custom hooks

```jsx
import { renderHook, act } from "@testing-library/react";
import { useCounter } from "./useCounter";

it("increments and decrements", () => {
  const { result } = renderHook(() => useCounter(0));

  expect(result.current.count).toBe(0);

  act(() => result.current.increment());
  expect(result.current.count).toBe(1);

  act(() => result.current.decrement());
  expect(result.current.count).toBe(0);
});

// Test with re-renders
it("resets when initialValue changes", () => {
  const { result, rerender } = renderHook(
    ({ initial }) => useCounter(initial),
    { initialProps: { initial: 0 } }
  );

  act(() => result.current.increment());
  expect(result.current.count).toBe(1);

  rerender({ initial: 10 });
  expect(result.current.count).toBe(10);
});
```

### Query priority (from Testing Library docs)

Use the most accessible query first:

1. `getByRole` — buttons, headings, inputs (best)
2. `getByLabelText` — form fields
3. `getByPlaceholderText` — if no label
4. `getByText` — non-interactive elements
5. `getByDisplayValue` — filled-in form values
6. `getByAltText` — images
7. `getByTitle` — title attribute
8. `getByTestId` — last resort

---

## References

- [React Docs](https://react.dev)
- [React 19 Blog Post](https://react.dev/blog)
- [React RFC: Server Components](https://github.com/reactjs/rfcs/pull/188)
- [Testing Library Docs](https://testing-library.com/docs/react-testing-library/intro)
