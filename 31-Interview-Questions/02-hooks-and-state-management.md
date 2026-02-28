# Interview Questions: Hooks & State Management

## Q1. How are hooks stored internally in React?

**Answer:**

Hooks are stored as a **linked list** on `fiber.memoizedState`. Each
hook call creates a node in this list:

```
fiber.memoizedState → hook1 → hook2 → hook3 → null
                      (useState) (useEffect) (useMemo)

Each hook node:
{
  memoizedState: <state value or effect object>,
  baseState: <base state for update processing>,
  baseQueue: <pending updates from previous render>,
  queue: <update queue for dispatches>,
  next: <pointer to next hook>
}
```

React uses a **cursor** that advances with each hook call. This is why
hooks must be called in the same order every render — the cursor maps
each call to its corresponding node in the linked list.

React uses **three dispatchers** to enforce rules:
- `HooksDispatcherOnMount` — creates new hook nodes
- `HooksDispatcherOnUpdate` — reads existing nodes
- `HooksDispatcherOnRerender` — for render-phase updates

---

## Q2. Why can't hooks be called conditionally?

**Answer:**

Because hooks rely on **call order** to match hook calls to their stored
state in the linked list.

```
// Mount (condition = true):
Hook 0: useState('A')    → stored as node 0
Hook 1: useState('B')    → stored as node 1  (conditionally called)
Hook 2: useEffect(...)   → stored as node 2

// Update (condition = false):
Hook 0: useState('A')    → reads node 0 ✓
Hook 1: useEffect(...)   → reads node 1 ✗ (expects useState('B')!)
```

React doesn't identify hooks by name. The only identifier is their
**position** in the call sequence. Skipping a hook shifts all subsequent
hooks by one, causing state corruption.

The `eslint-plugin-react-hooks` `rules-of-hooks` rule catches this at
lint time. React also detects order changes in DEV mode.

---

## Q3. How does useState differ from useReducer internally?

**Answer:**

`useState` is implemented **as** `useReducer` with a built-in reducer:

```javascript
// React's internal basicStateReducer:
function basicStateReducer(state, action) {
  return typeof action === 'function' ? action(state) : action;
}

// useState mount:
function mountState(initialState) {
  const hook = mountWorkInProgressHook();
  hook.memoizedState = typeof initialState === 'function'
    ? initialState()    // Lazy initializer
    : initialState;
  const queue = { pending: null, dispatch: null, ... };
  hook.queue = queue;
  const dispatch = dispatchReducerAction.bind(null, fiber, queue);
  return [hook.memoizedState, dispatch];
}
```

**Key differences in practice:**
| Feature | useState | useReducer |
|---------|----------|------------|
| Reducer | Built-in (replace or function) | Custom |
| Eager state | Yes (can skip render) | No |
| Dispatch identity | Stable across renders | Stable across renders |
| Best for | Simple state | Complex state logic |

**Eager state optimization** (useState only):
```
dispatch(sameValue)
  → React computes new state immediately (not during render)
  → If Object.is(newState, currentState): skip scheduling entirely
  → No re-render at all!
```

---

## Q4. Explain the useEffect lifecycle in detail.

**Answer:**

```
Mount:
  1. Component renders → effect object created: { create, destroy, deps }
  2. Commit phase completes, DOM is updated
  3. Browser paints
  4. flushPassiveEffects (async, via MessageChannel)
  5. effect.create() runs → returns cleanup function
  6. effect.destroy = cleanupFn (stored for later)

Update (deps changed):
  1. Component re-renders → new effect object
  2. React compares deps: areHookInputsEqual(newDeps, oldDeps)
  3. Deps differ → effect flagged with HookHasEffect
  4. Commit phase → DOM updated → browser paints
  5. flushPassiveEffects:
     a. Run ALL cleanup functions first (old effect.destroy())
     b. Run ALL setup functions (new effect.create())
  6. Cleanups run before setups across ALL components

Unmount:
  1. Component removed during commit (commitDeletion)
  2. Layout effect cleanups run synchronously
  3. flushPassiveEffects: passive effect cleanups run
```

**Dependency comparison uses `Object.is`:**
```javascript
// Primitives compared by value: Object.is(1, 1) → true
// Objects compared by reference: Object.is({}, {}) → false
// Special: Object.is(NaN, NaN) → true, Object.is(+0, -0) → false
```

---

## Q5. What is the difference between useEffect and useLayoutEffect?

**Answer:**

| | useEffect | useLayoutEffect |
|---|---|---|
| **Timing** | After browser paint (async) | Before browser paint (sync) |
| **Blocks paint?** | No | Yes |
| **Use case** | Data fetching, subscriptions | DOM measurements, mutations |
| **SSR** | Works (skipped on server) | Warning on server (no layout) |

```
Commit timeline:
  DOM mutations applied
    → useLayoutEffect cleanup (sync)
    → useLayoutEffect setup (sync)
    → [if setState in layout effect → synchronous re-render!]
  Tree swap
  Browser paints ← useLayoutEffect blocks this
    → useEffect cleanup (async)
    → useEffect setup (async)
```

**When to use useLayoutEffect:**
- Measuring DOM elements (getBoundingClientRect)
- Scroll position management
- Preventing visual flicker (DOM mutation before paint)
- Focus management

---

## Q6. How does useContext work and why does it bypass React.memo?

**Answer:**

`useContext` reads from a **context stack** maintained during rendering.

```javascript
// Internal: useContext reads from the fiber's context dependency
function readContext(context) {
  const value = context._currentValue;

  // Register this fiber as a consumer of this context
  const contextItem = { context, next: null };
  // Append to fiber.dependencies linked list
  lastContextDependency.next = contextItem;

  return value;
}
```

**Why it bypasses memo:**

When a context value changes, React runs `propagateContextChange` which
**scans the fiber tree** looking for consumers. When it finds one:

```javascript
// Directly marks the consumer fiber for update
fiber.lanes = mergeLanes(fiber.lanes, renderLanes);
// Also marks all parent fibers (up to the provider)
// This bypasses shouldComponentUpdate / React.memo
```

The memo check (`shallowEqual` on props) happens **before** rendering.
But context propagation marks the fiber's lanes **before** the memo check
runs. So the fiber is forced to render regardless of memoization.

**Optimization: split contexts** into smaller pieces so that consumers
only subscribe to the data they actually need.

---

## Q7. What is the purpose of useRef and how is it implemented?

**Answer:**

`useRef` creates a mutable container that persists across renders:

```javascript
// Mount:
function mountRef(initialValue) {
  const hook = mountWorkInProgressHook();
  const ref = { current: initialValue };
  hook.memoizedState = ref;
  return ref;
}

// Update:
function updateRef() {
  const hook = updateWorkInProgressHook();
  return hook.memoizedState;  // Same object! Never changes.
}
```

**Key properties:**
- `ref.current` is mutable — you can assign any value
- Mutating `ref.current` does NOT trigger re-render
- The ref object itself (`{ current: ... }`) is stable (same reference)
- React never reads or tracks `ref.current` during reconciliation

**Ref vs State:**
```
useState: tracked by React, triggers re-render on change
useRef:   invisible to React, never triggers re-render
          Just a plain object stored on the hook's linked list
```

**Common pattern — DOM ref:**
```jsx
<div ref={myRef} />
// React sets myRef.current = domElement during commit (layout phase)
// React sets myRef.current = null during unmount
```

---

## Q8. How does useMemo differ from React.memo?

**Answer:**

| | useMemo | React.memo |
|---|---|---|
| **Level** | Value memoization (inside component) | Component memoization (wraps component) |
| **What it caches** | A computed value | Component render output |
| **Comparison** | `Object.is` on deps array | `shallowEqual` on props (default) |
| **Granularity** | Per-hook | Per-component |

```javascript
// useMemo: caches a value
const sorted = useMemo(() => items.sort(compareFn), [items]);

// React.memo: caches entire component render
const MemoList = React.memo(function List({ items }) {
  return items.map(item => <Item key={item.id} {...item} />);
});
// Re-renders only if props change (shallow comparison)
```

**Internal implementation of React.memo:**
```javascript
function memo(type, compare) {
  return {
    $$typeof: REACT_MEMO_TYPE,
    type,                    // The wrapped component
    compare: compare || null // Custom comparison (or null for shallow)
  };
}
```

When `compare === null`, React may use `SimpleMemoComponent` (tag 15),
a faster code path that skips the full MemoComponent machinery.

---

## Q9. Explain automatic batching in React 18.

**Answer:**

In React 18, **all state updates are batched** regardless of where they
originate. In React 17, only updates inside React event handlers were batched.

```javascript
// React 17:
setTimeout(() => {
  setCount(1);  // → render
  setFlag(true); // → render (TWO renders!)
}, 0);

// React 18:
setTimeout(() => {
  setCount(1);  // batched
  setFlag(true); // batched → ONE render
}, 0);
```

**How it works internally:**

React 18 uses **microtask scheduling** via `ensureRootIsScheduled`:
```
setState(A) → schedule render (but don't start yet)
setState(B) → already scheduled (deduplicate!)
           → End of microtask
           → Now process both updates in one render
```

**Opting out of batching:**
```javascript
import { flushSync } from 'react-dom';

flushSync(() => setCount(1));  // Renders immediately
flushSync(() => setFlag(true)); // Renders immediately (separate render)
```

**What gets batched:** All `setState`/`dispatch` calls between yields —
in event handlers, setTimeout, promises, native event handlers, and even
`fetch().then()`.

---

## Q10. What happens when you call setState with the same value?

**Answer:**

React performs an **eager bailout** — it may skip the re-render entirely.

```javascript
const [count, setCount] = useState(0);
setCount(0);  // Same value → may not re-render
```

**Internal mechanism:**
```javascript
function dispatchSetState(fiber, queue, action) {
  // ...
  const eagerState = typeof action === 'function'
    ? action(currentState)
    : action;

  if (Object.is(eagerState, currentState)) {
    // BAILOUT: same value, don't schedule render
    return;  // ← No render scheduled at all
  }

  // Different value → schedule update
  scheduleUpdateOnFiber(fiber, lane);
}
```

**Caveat:** The bailout isn't always guaranteed:
- If there are already pending updates on the queue, React may not
  be able to determine the final state eagerly
- In that case, React schedules the render anyway but may bail out
  later during `updateReducer` if the final state matches
- The component function may be called, but children won't re-render
  (this is the "render then bailout" path)
