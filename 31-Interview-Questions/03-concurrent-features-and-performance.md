# Interview Questions: Concurrent Features & Performance

## Q1. What is concurrent rendering and how does it differ from synchronous rendering?

**Answer:**

**Synchronous rendering** (workLoopSync): Once React starts rendering,
it processes the entire tree without stopping. The main thread is
blocked until commit.

**Concurrent rendering** (workLoopConcurrent): React can **pause**
rendering between fiber units of work, yield to the browser for
higher-priority tasks (user input, paint), and **resume** or
**restart** later.

```
Synchronous:
  [===RENDER (can't stop)===][COMMIT] → paint

Concurrent:
  [==RENDER==][yield][==RENDER==][yield][==RENDER==][COMMIT] → paint
                ↑                   ↑
           browser work         browser work
```

**Key properties of concurrent rendering:**
- Render phase has NO side effects → safe to abandon/restart
- React can interrupt a render for a higher-priority update
- The WIP tree is discarded if interrupted (garbage collected)
- Only the commit phase touches the DOM (synchronous, uninterruptible)

**When concurrent rendering is used:**
- `createRoot` (React 18+) enables concurrent features
- `startTransition` updates → concurrent rendering
- `useDeferredValue` deferred renders → concurrent
- Default updates (click handlers) → still synchronous via SyncLane

---

## Q2. Explain startTransition and useTransition.

**Answer:**

`startTransition` marks state updates as **non-urgent**, allowing React
to interrupt them for higher-priority work:

```jsx
const [isPending, startTransition] = useTransition();

startTransition(() => {
  setSearchResults(heavyFilter(query)); // TransitionLane (low priority)
});
setInputValue(query); // DefaultLane (high priority — renders first)
```

**Internal mechanism:**
```javascript
function startTransition(callback) {
  const prevTransition = ReactCurrentBatchConfig.transition;
  ReactCurrentBatchConfig.transition = {};  // Mark: inside transition

  callback();  // setState calls inside get TransitionLane

  ReactCurrentBatchConfig.transition = prevTransition;
}
```

**Behavior:**
- Updates inside `startTransition` get `TransitionLane` (lower priority)
- These renders CAN be interrupted by urgent updates (click, keypress)
- React keeps showing the **old UI** while the transition renders
- `isPending = true` while the transition is in progress
- If a new transition starts before the old one finishes, the old one
  is abandoned

**When NOT to use:**
- Urgent UI feedback (text input value, toggles) — should be immediate
- Side effects — transitions only affect rendering, not fetches

---

## Q3. What is useDeferredValue and how does it work?

**Answer:**

`useDeferredValue` returns a **lagging copy** of a value that updates
with lower priority:

```jsx
const deferredQuery = useDeferredValue(query);
// query updates immediately (urgent)
// deferredQuery updates later (deferred)
```

**Internal implementation (simplified):**
```javascript
function updateDeferredValue(value) {
  const hook = updateWorkInProgressHook();
  const prevValue = hook.memoizedState;

  if (Object.is(value, prevValue)) {
    return value;  // No change
  }

  // Schedule a deferred render with the new value
  // Current render uses the OLD value
  const deferredLane = requestDeferredLane();
  // The fiber gets updated in the deferred render with the new value
  return prevValue;  // Return old value for now
}
```

**Relationship to startTransition:**
- `useDeferredValue` is like a `startTransition` that React manages
- `startTransition`: you control which `setState` is deferred
- `useDeferredValue`: you defer a value coming from a parent/prop
- Both result in lower-priority renders that can be interrupted

---

## Q4. What is "tearing" and how does React prevent it?

**Answer:**

**Tearing** occurs when different components in the same render show
different values from a shared mutable source — the UI is "torn":

```
Component A renders: reads externalStore = 0  (before yield)
--- React yields to browser ---
externalStore = 1  (mutated externally!)
Component B renders: reads externalStore = 1  (after yield)

Screen shows: A=0, B=1 → TORN! Same render, different values.
```

**Prevention with useSyncExternalStore:**
```javascript
const value = useSyncExternalStore(
  store.subscribe,  // Subscribe to changes
  store.getSnapshot  // Read current value
);
```

React takes a **snapshot** before rendering. After rendering, it checks
if the snapshot changed. If it did, React **restarts the render
synchronously** (non-interruptible), guaranteeing all components see
the same value.

**For React state (useState/useReducer):**
- Tearing CANNOT happen because React controls state reads
- State is read from the fiber, which doesn't change mid-render

---

## Q5. How does React's Suspense work internally?

**Answer:**

Suspense catches **thrown promises** during rendering:

```
Component renders → throws a Promise
  → throwException() catches it
  → Walks UP the fiber tree to find nearest <Suspense>
  → Marks Suspense boundary to show fallback
  → Attaches promise.then(ping) for retry
  → Promise resolves → ping() → schedules re-render
  → Component renders again (hopefully with data)
```

**Suspense boundary states:**
```
                   ┌─────────────┐
  mount ────────── │  Show content│
                   └──────┬──────┘
                     child suspends
                   ┌──────▼──────┐
                   │Show fallback │
                   └──────┬──────┘
                    promise resolves
                   ┌──────▼──────┐
                   │ Show content │ (retry render)
                   └─────────────┘
```

**Key internal details:**
- Suspense is a real fiber (tag = `SuspenseComponent`)
- It wraps content in an `OffscreenComponent` (can hide/show)
- The thrown promise is a "wakeable" — React attaches a ping listener
- Multiple children can suspend — all their promises are tracked
- Nested Suspense boundaries: innermost one catches first

---

## Q6. What is Selective Hydration?

**Answer:**

React 18 can **prioritize hydrating** the parts of the page the user
is interacting with, even if other parts haven't hydrated yet.

```
Server sends:
  <nav>...</nav>                    ← hydrates immediately
  <!--$--><sidebar>...</sidebar>    ← Suspense boundary A
  <!--$--><content>...</content>    ← Suspense boundary B

User clicks on content area (not yet hydrated):
  1. React captures the click event
  2. Identifies the Suspense boundary containing the target
  3. BOOSTS that boundary's hydration priority
  4. Pauses sidebar hydration
  5. Hydrates content FIRST
  6. Replays the captured click event
  7. Resumes sidebar hydration
```

**Internal mechanism:**
- `queueExplicitHydrationTarget` upgrades lane priority
- Events on un-hydrated areas are stored in a replay queue
- After hydration, events are replayed in order
- Uses `InputContinuousLane` for boosted boundaries

---

## Q7. How does React handle error boundaries?

**Answer:**

Error boundaries are **class components** that implement
`getDerivedStateFromError` and/or `componentDidCatch`:

```jsx
class ErrorBoundary extends React.Component {
  state = { hasError: false };

  static getDerivedStateFromError(error) {
    return { hasError: true };  // Update state to show fallback
  }

  componentDidCatch(error, errorInfo) {
    logError(error, errorInfo.componentStack);  // Side effect: logging
  }

  render() {
    if (this.state.hasError) return <Fallback />;
    return this.props.children;
  }
}
```

**Internal flow when a component throws:**
```
Component throws Error during render
  → throwException(root, error, sourceFiber, rootRenderLanes)
    → Walk UP fiber tree looking for error boundary
    → Found: getDerivedStateFromError? → Create update with error
    → Schedule re-render of boundary with error state
  → unwindWork(): clean up incomplete work from boundary to root
  → Re-render: boundary shows fallback, componentDidCatch fires in commit
```

**What error boundaries DON'T catch:**
- Errors in event handlers (use try/catch)
- Async errors (setTimeout, promises — unless in effects)
- Server-side rendering errors (different mechanism)
- Errors in the error boundary itself

**Why no function component error boundary?**
- `getDerivedStateFromError` is a static method (no hooks equivalent)
- React team may add `useErrorBoundary` or similar in future
- Current workaround: use a class component wrapper

---

## Q8. What is the React Compiler and what does it do?

**Answer:**

The React Compiler (formerly React Forget) **automatically memoizes**
components and values at build time, eliminating the need for manual
`useMemo`, `useCallback`, and `React.memo`:

```jsx
// Input:
function TodoList({ todos, filter }) {
  const filtered = todos.filter(t => t.status === filter);
  return filtered.map(t => <Todo key={t.id} todo={t} />);
}

// Compiler output (simplified):
function TodoList({ todos, filter }) {
  const $ = useMemoCache(4);  // Internal cache slots
  let filtered;
  if ($[0] !== todos || $[1] !== filter) {
    filtered = todos.filter(t => t.status === filter);
    $[0] = todos;
    $[1] = filter;
    $[2] = filtered;
  } else {
    filtered = $[2];
  }
  // Similar memoization for the JSX output...
}
```

**How it works:**
1. Parses components into **HIR** (High-level Intermediate Representation)
2. Analyzes data flow and dependencies
3. Identifies reactive values (values that change between renders)
4. Inserts `useMemoCache` calls with dependency tracking
5. Each "slot" in the cache is checked against its dependencies

**Requirements:**
- Components must follow the **Rules of React** (pure renders, etc.)
- Currently a Babel plugin
- Validates rules before optimizing; skips invalid components

---

## Q9. How does React prevent XSS attacks?

**Answer:**

React employs multiple layers of defense:

**1. Text content escaping:**
```javascript
// React escapes all string content before insertion:
<div>{userInput}</div>
// Internally: textNode.textContent = userInput;
// textContent auto-escapes HTML entities
// "<script>" becomes the literal text, not an HTML tag
```

**2. Attribute escaping:**
```javascript
<div title={userInput} />
// React uses setAttribute which auto-escapes
// No attribute injection possible
```

**3. $$typeof Symbol:**
```javascript
// React elements include: $$typeof: Symbol.for('react.element')
// Symbols can't exist in JSON (not serializable)
// Prevents injection of fake React elements via API responses
```

**4. dangerouslySetInnerHTML:**
```javascript
// The only way to insert raw HTML:
<div dangerouslySetInnerHTML={{ __html: html }} />
// The ugly name is intentional — forces explicit acknowledgment
// React does NOT sanitize this — developer responsibility
```

**5. URL sanitization (React 16.9+):**
```javascript
<a href={userInput}>Link</a>
// React warns/blocks javascript: URLs
// Prevents: <a href="javascript:alert('xss')">
```

---

## Q10. Explain React's memory management and common leak patterns.

**Answer:**

**Fiber cleanup:**
After commit, React detaches deleted fibers:
```javascript
// For each deleted fiber:
fiber.return = null;      // Detach from parent
fiber.child = null;       // Release children
fiber.stateNode = null;   // Release DOM node
fiber.memoizedState = null; // Release hooks/state
fiber.dependencies = null;  // Release context refs
```

**Common leak patterns:**

1. **Uncleared subscriptions:**
```javascript
useEffect(() => {
  window.addEventListener('resize', handler);
  // Missing: return () => window.removeEventListener('resize', handler);
}, []);
```

2. **Closure retention:**
```javascript
useEffect(() => {
  const bigData = fetchLargeDataset();  // 100MB
  return () => {
    // bigData still in closure scope — retained until cleanup runs
  };
}, []);
```

3. **Stale timer/interval:**
```javascript
useEffect(() => {
  setInterval(callback, 1000);
  // Missing: return () => clearInterval(id);
}, []);
// Interval fires forever, callback retains fiber references
```

4. **Ref holding DOM nodes:**
```javascript
const nodes = useRef([]);
items.forEach(item => {
  // ❌ Array grows but never shrinks
  nodes.current.push(document.createElement('div'));
});
```

**Memory best practices:**
- Always return cleanup from effects
- Use `AbortController` for fetch cleanup
- Avoid storing large objects in state/refs
- Clear refs when components handling changes
