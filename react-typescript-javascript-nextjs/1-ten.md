---

## Q1. What is the difference between `==` and `===` in JavaScript? Explain type coercion with an example.

**Answer:**

`===` is a strict equality check — no type coercion occurs, meaning the JavaScript engine does not convert data types before comparison.

`==` is a loose equality check — the JS engine converts data types before comparing.

**Examples:**

```javascript
"5" == 5; // true  — JS converts "5" to number 5
"5" === 5; // false — different types, no conversion
```

**Bonus:** Special case worth knowing:

```javascript
null == undefined; // true
null === undefined; // false
```

---

## Q2. What is a closure in JavaScript? Explain with a real-world use case.

**Answer:**

A closure is a function that remembers the scope from when it was created, even after the outer scope has finished executing.

**Real-world example — Debouncing:**

Debouncing means fixing a time period X, after which a request is created only when the user stops typing for X milliseconds.

The outer function executes and exits, but the **timer persists in the closure scope** so the next request can be correctly scheduled after the user stops typing.

```javascript
function debounce(fn, delay) {
  let timer; // this timer persists via closure
  return function (...args) {
    clearTimeout(timer);
    timer = setTimeout(() => fn(...args), delay);
  };
}
```

---

## Q3. Explain the JavaScript Event Loop. What is the difference between Microtask and Macrotask queues?

**Answer:**

JavaScript is single-threaded, but browsers need to handle async tasks like user clicks, timers, and network requests. The **Event Loop** is the coordinator that enables async behavior without multithreading.

**Microtask Queue** (higher priority):

- Promises (`.then`, `.catch`)
- MutationObserver

**Macrotask Queue** (lower priority):

- `setTimeout`, `setInterval`
- Browser APIs

**Rule:** The microtask queue is **fully consumed** before the next macrotask is picked up. After each macrotask, the event loop checks for microtasks again.

**Code example:**

```javascript
console.log("1");
setTimeout(() => console.log("2"), 0);
Promise.resolve().then(() => console.log("3"));
console.log("4");
```

**Output:**

```
1
4
3
2
```

**Reasoning:** `1` and `4` are synchronous. `3` (Promise) is a microtask — runs before `2` (setTimeout macrotask).

---

## Q4. What is the difference between `interface` and `type` in TypeScript? When would you use each?

**Answer:**

| Feature             | `interface`    | `type`         |
| ------------------- | -------------- | -------------- |
| Declaration merging | ✅ Allowed     | ❌ Not allowed |
| Union types         | ❌             | ✅             |
| Intersection        | via `extends`  | via `&`        |
| Error messages      | Easier to read | More explicit  |

**Rule of thumb:** Use `interface` first. Switch to `type` when you need unions or something that can't be expressed with interface.

```typescript
// Only possible with type
type Status = "active" | "inactive" | "pending";

// Interface merging
interface User {
  name: string;
}
interface User {
  age: number;
} // valid — merges both
```

**Explain this:**

```typescript
type ReadonlyUser = Readonly<Partial<User>>;
```

- `Partial<User>` — makes all User fields optional
- `Readonly<...>` — makes all fields immutable

**Use case:** API response drafts, form state objects where fields shouldn't be mutated.

---

## Q5. What is the difference between `useMemo` and `useCallback`?

**Answer:**

- **`useMemo`** — memoizes the **return value** of an expensive computation
- **`useCallback`** — memoizes a **function reference**

```typescript
// useMemo — prevents expensive recalculation
const sorted = useMemo(() => expensiveSort(list), [list]);

// useCallback — stable function reference
const handleClick = useCallback(() => doSomething(id), [id]);
```

Use `useCallback` when:

- Passing callbacks to child components wrapped in `React.memo`
- Using a function as a dependency of `useEffect`

**Equivalence:** `useCallback(fn, deps)` is equivalent to `useMemo(() => fn, deps)`

**Bug in this code:**

```typescript
const MyComponent = () => {
  const fetchData = async () => {
    await api.getData();
  };

  useEffect(() => {
    fetchData();
  }, [fetchData]); // ⚠️ infinite loop!
};
```

`fetchData` is recreated on every render → `useEffect` dependency changes → infinite loop.

**Fix:**

```typescript
// Option 1
const fetchData = useCallback(async () => {
  await api.getData();
}, []);

// Option 2 (cleanest)
useEffect(() => {
  const fetchData = async () => await api.getData();
  fetchData();
}, []);
```

---

## Q6. How would you solve prop drilling? Compare Context API vs Zustand vs Redux.

**Answer:**

**Prop drilling** is passing props through every parent level to reach a deeply nested child component.

**Context API:**

- Best for static or slow-changing global data (theme, locale, auth user)
- Wrap components in a Provider and consume anywhere
- ⚠️ Problem: re-renders every consumer when context value changes
- ✅ Fix: split context (e.g. `UserContext` and `UIContext` separately) to reduce consumers per context

**Zustand:**

- Lightweight external state
- Components subscribe only to what they use — no unnecessary re-renders
- No Provider wrapper needed
- Update state from anywhere

**Redux Toolkit:**

- Built for large, complex state management
- More boilerplate than Zustand
- **Best-in-class DevTools** — time-travel debugging, state snapshots — easier to debug complex flows

**When to use what:**

- Small app / simple global state → Context API
- Medium app / performance sensitive → Zustand
- Large enterprise app / complex state / team needs debuggability → Redux Toolkit

---

## Q7. What are React Server Components (RSC)? How are they different from Client Components?

**Answer:**

**Before RSC:** Every React component ran in the browser. The server sent HTML for initial paint, then React hydrated everything — all component logic, data fetching, and rendering happened client-side.

**RSC flips this:** Some components never ship to the browser at all. They run only on the server, render to HTML, and send the result down — zero JS bundle impact.

**What RSC enables:**

- DB calls, secret keys, heavy libraries stay on the server
- Nothing sensitive reaches the browser
- Smaller client bundles

**What RSC cannot do:**

- No `useState`, `useEffect`, or any hooks
- No event handlers (`onClick`, etc.)
- Stateless and non-interactive by design

**RSC and Client Components can coexist** — pass RSC output as `children` to a client component. RSC also enables **streaming** via `Suspense`.

**Bug in this code:**

```typescript
"use server" // ❌ wrong directive

const UserProfile = () => {
  const [count, setCount] = useState(0); // ❌ hooks not allowed in RSC
  return <button onClick={() => setCount(count + 1)}>{count}</button> // ❌ event handler
}
```

**Fix:** Change `"use server"` to `"use client"`.

---

## Q8. Explain TypeScript Generics. What is the difference between these two functions?

**Answer:**

Generics let you write one function that preserves the exact type of whatever you pass in. `T` is a type variable — a placeholder filled in when the function is called. TypeScript infers it automatically.

```typescript
function identity1<T>(arg: T): T {
  return arg;
}
function identity2<T extends object>(arg: T): T {
  return arg;
}
```

- `identity1` — accepts **any** type (primitives, objects, arrays, etc.)
- `identity2` — accepts only **objects** (`{id: 1}`, `[1,2,3]`, etc.) — primitives like `string`, `number` are excluded

`extends object` means T must satisfy the constraint of being an object type.

**Implement `pick`:**

```typescript
function pick<T, K extends keyof T>(obj: T, keys: K[]): Pick<T, K> {
  return keys.reduce(
    (acc, key) => {
      acc[key] = obj[key];
      return acc;
    },
    {} as Pick<T, K>,
  );
}

// Usage
const user = { id: 1, name: "Pranshu", email: "x@x.com", age: 22 };
pick(user, ["id", "name"]); // { id: 1, name: "Pranshu" }
```

---

## Q9. How would you diagnose and fix React performance issues? Explain reconciliation and virtual DOM diffing.

**Answer:**

**Diagnosis steps:**

1. Open React DevTools Profiler — find slow interactions
2. Check for unnecessary re-renders when nothing changed
3. Look for cascading re-renders (waterfall effect)
4. Add console logs to find components rendering multiple times

**Fixes:**

- `useMemo` / `useCallback` to cache expensive computations and stable references
- **Lazy loading** — code split so only needed bundles are fetched
- **Virtualization** for large lists (react-window, react-virtual)
- **Tree shaking** — import only what you need from libraries
- **Parallel requests** — fetch user profile and posts simultaneously instead of sequentially

**Reconciliation & Virtual DOM:**

Every time state changes, React needs the minimum set of DOM changes to reflect the new UI. Real DOM operations are expensive, so React maintains a **Virtual DOM** — a lightweight JS object tree.

When state changes → React builds a new VDOM tree → diffs against previous → applies only the differences to real DOM.

**Diffing algorithm:**

- If root element type changes → tear down entire old tree, build fresh
- **Keys** identify elements across renders so React knows which elements to update vs recreate

**Concurrent Mode (Fiber):**
Old reconciliation was synchronous and blocking — froze the UI for large trees.

Fiber scheduler now:

- Does a unit of work → checks if time deadline is exceeded → pauses if needed → resumes on next frame
- **Prioritizes urgent updates** (user input) over less urgent ones (data fetching)
- Enables Suspense, streaming, and concurrent features

---

## Q10. Design a real-time collaborative code editor on the frontend.

**Answer:**

### Challenges

1. **Simultaneous edits** — two users type at the same time, both must see each other's changes
2. **Conflict resolution** — User A deletes line 3 while User B edits line 3
3. **Cursor sync** — every user's cursor/selection must show in real time
4. **Performance** — large files, syntax highlighting, 50+ collaborators, low latency
5. **Offline resilience** — brief disconnections must not lose work

### Tech Stack

| Tool             | Purpose                                                |
| ---------------- | ------------------------------------------------------ |
| **CodeMirror 6** | Editor — lightweight, modular, transaction-based       |
| **Yjs**          | CRDT conflict resolution — peer-to-peer, works offline |
| **y-websocket**  | WebSocket provider for Yjs                             |
| **Zustand**      | Local UI state (sidebar, settings)                     |
| **React**        | Component layer                                        |
| **TypeScript**   | Type safety across ops                                 |

### Conflict Resolution — OT vs CRDT

**Operational Transformation (OT):**

```
Initial: "hello"
User A: insert("!", 5)    → "hello!"
User B: insert(" world", 5) → "hello world"

Server transforms A's op against B's:
B arrived first → state is "hello world"
A's insert was at index 5, B inserted 6 chars at 5
Transform A: insert("!", 5 + 6) = insert("!", 11)
Result: "hello world!" ✅
```

Problem: Need transform function for every operation pair. Gets complex fast.

**Yjs CRDT (better approach):**

Every character gets a unique ID (clientID + clock) and a reference to what comes before/after it.

```
"hello": h(c1,0) → e(c1,1) → l(c1,2) → l(c1,3) → o(c1,4)

User A inserts "!" after o(c1,4):  !(c1,5) origin=o(c1,4)
User B inserts " world" after o(c1,4): (c2,0..5) origin=o(c1,4)

CRDT resolves via clientID tiebreak — same result on every client:
Result: "hello world!" ✅ — no server arbitration needed
```

### State Management Architecture

```
Yjs (shared, persisted)     → document content, comments, file structure
Yjs Awareness (ephemeral)   → cursor positions, presence, who is typing
Zustand (local UI)          → active tab, sidebar, theme, font size
React state (component)     → dropdowns, tooltips, search terms
```

### Performance Optimizations

- **Lazy load** language support — keep bundle small
- **Virtualize** large files — don't render off-screen lines
- **Throttle** cursor position WebSocket messages — not every keystroke
- **Debounce** server persistence — don't save every keystroke to DB

---

## 📊 Final Score: 94/100 🏆

| Q   | Topic                        | Score  |
| --- | ---------------------------- | ------ |
| 1   | == vs ===                    | 9/10   |
| 2   | Closures                     | 10/10  |
| 3   | Event Loop                   | 10/10  |
| 4   | Interface vs Type            | 7.5/10 |
| 5   | useMemo/useCallback          | 8.5/10 |
| 6   | State Management             | 9.5/10 |
| 7   | React Server Components      | 9.5/10 |
| 8   | Generics                     | 10/10  |
| 9   | Performance + Reconciliation | 10/10  |
| 10  | System Design                | 10/10  |
