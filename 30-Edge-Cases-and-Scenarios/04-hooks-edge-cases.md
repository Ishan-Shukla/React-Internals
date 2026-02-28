# Edge Cases: Hooks

## 1. Conditional Hook Call (Rules of Hooks Violation)

```jsx
function Bad({ loggedIn }) {
  if (loggedIn) {
    const [user, setUser] = useState(null);  // ❌ Conditional hook!
  }
  const [theme, setTheme] = useState('dark');
}
```

```
Why this breaks:

  Hooks are stored as a linked list on fiber.memoizedState:
    hook1 → hook2 → hook3 → null

  On mount (loggedIn = true):
    Hook 0: useState(null)   → user state
    Hook 1: useState('dark') → theme state

  On update (loggedIn = false):
    Hook 0: useState('dark') → React THINKS this is "user" state!
    → theme gets the value that was stored for user
    → Complete state corruption

  React's internal cursor:
    let currentHook = fiber.memoizedState;  // Start of linked list

    // Each hook call advances the cursor:
    function nextHook() {
      currentHook = currentHook.next;
    }

    // If you skip a hook call, the cursor is off by one
    // All subsequent hooks read the WRONG state

  React detects this in DEV mode:
    "React has detected a change in the order of Hooks"
    "called by ComponentName. This will lead to bugs."

  The eslint-plugin-react-hooks catches this at lint time.
```

## 2. useState Lazy Initializer Called Twice in StrictMode

```jsx
function App() {
  const [data, setData] = useState(() => {
    console.log('initializing');  // Called TWICE in dev + StrictMode
    return expensiveComputation();
  });
}
```

```
StrictMode behavior:

  Development + StrictMode:
    1. First render call: initializer runs → "initializing"
    2. Second render call: initializer runs → "initializing"
    → But React uses the result from the first call
    → Second call is for detecting side effects in initializer

  Production:
    1. Initializer runs once → result stored in hook.memoizedState
    2. Never called again (not even on re-renders)

  The lazy initializer SHOULD be pure:
    ✓ useState(() => computeValue(props))           // Pure
    ✓ useState(() => JSON.parse(localStorage.get()))  // Pure-ish (reads external)
    ✗ useState(() => { sideEffect(); return val; })  // Impure — StrictMode exposes this

  On re-render, useState ignores the initializer entirely:
    function updateState(initialState) {
      // initialState parameter is COMPLETELY IGNORED
      // State comes from hook.memoizedState
      return updateReducer(basicStateReducer, initialState);
    }
```

## 3. useReducer vs useState: When Actions Queue

```jsx
function App() {
  const [count, dispatch] = useReducer((state, action) => {
    switch (action) {
      case 'increment': return state + 1;
      case 'reset': return 0;
    }
  }, 0);

  const handleClick = () => {
    dispatch('increment');  // Queued
    dispatch('increment');  // Queued
    dispatch('reset');      // Queued
    dispatch('increment');  // Queued
    // Final result: 1 (not 3!)
  };
}
```

```
Update queue processing:

  All dispatches in the same event handler are batched.
  They form a linked list of updates:

  queue: increment → increment → reset → increment

  Processing (in processUpdateQueue):
    state = 0
    apply increment: state = 0 + 1 = 1
    apply increment: state = 1 + 1 = 2
    apply reset:     state = 0
    apply increment: state = 0 + 1 = 1

  Final state: 1

  This is the same as Redux reducer behavior.

  With useState (which uses basicStateReducer internally):
    setState(1)           → update: { action: 1 }
    setState(prev => prev + 1) → update: { action: fn }
    setState(0)           → update: { action: 0 }

    Processing:
      state = initialState
      apply action=1:    state = 1 (value replaces)
      apply action=fn:   state = fn(1) = 2 (function called)
      apply action=0:    state = 0 (value replaces)

    Final state: 0
```

## 4. useEffect vs useLayoutEffect: Timing Differences

```jsx
function MeasureAndAnimate() {
  const ref = useRef();
  const [height, setHeight] = useState(0);

  // ❌ useEffect: causes visible flash
  useEffect(() => {
    setHeight(ref.current.getBoundingClientRect().height);
    // DOM updated → browser paints (wrong height) → effect runs → setState
    // → re-render → paint again (correct height) = FLASH!
  }, []);

  // ✓ useLayoutEffect: no flash
  useLayoutEffect(() => {
    setHeight(ref.current.getBoundingClientRect().height);
    // DOM updated → layout effect runs → setState → synchronous re-render
    // → browser paints (correct height from start) = NO FLASH
  }, []);

  return <div ref={ref} style={{ minHeight: height }}>Content</div>;
}
```

```
Timing difference:

  useLayoutEffect:
    Commit phase:
      1. DOM mutations applied
      2. useLayoutEffect cleanup runs  ← synchronous
      3. useLayoutEffect setup runs    ← synchronous
      4. If setState called → synchronous re-render!
      5. Browser paints

  useEffect:
    Commit phase:
      1. DOM mutations applied
      2. Browser paints              ← user sees intermediate state
      3. (some time later)
      4. useEffect cleanup runs      ← asynchronous
      5. useEffect setup runs        ← asynchronous
      6. If setState called → new render scheduled

  useLayoutEffect blocks paint.
  useEffect does not block paint.

  When to use which:
    useLayoutEffect:
      - DOM measurements (getBoundingClientRect)
      - DOM mutations that must happen before paint
      - Scroll position management
      - Tooltip positioning
      - Focus management

    useEffect:
      - Data fetching
      - Event subscriptions
      - Logging/analytics
      - Anything that doesn't affect visual output
```

## 5. Custom Hook Shares Hook State Per Instance

```jsx
function useCounter(initial = 0) {
  const [count, setCount] = useState(initial);
  const increment = useCallback(() => setCount(c => c + 1), []);
  return { count, increment };
}

function App() {
  const counter1 = useCounter(0);   // Independent state!
  const counter2 = useCounter(10);  // Independent state!

  return (
    <div>
      {counter1.count} {/* 0 */}
      {counter2.count} {/* 10 */}
    </div>
  );
}
```

```
Custom hooks are NOT shared singletons.

  Each call to useCounter creates NEW hooks on the fiber's list:

  fiber.memoizedState:
    hook0 (useState for counter1.count) → value: 0
      ↓
    hook1 (useCallback for counter1.increment) → memoized fn
      ↓
    hook2 (useState for counter2.count) → value: 10
      ↓
    hook3 (useCallback for counter2.increment) → memoized fn
      ↓
    null

  Custom hooks are just a CONVENTION for grouping hook calls.
  They have no special internal representation.
  React doesn't know "useCounter" exists — it only sees
  the individual useState/useCallback calls.

  If two COMPONENTS both call useCounter:
    ComponentA fiber: hook0 (count=5), hook1 (increment)
    ComponentB fiber: hook0 (count=3), hook1 (increment)
    → Completely independent! Different fibers, different hook lists.
```

## 6. useMemo / useCallback Aren't Guarantees

```jsx
function App() {
  const expensiveValue = useMemo(() => {
    return heavyComputation();  // React may recompute this!
  }, [dep]);
}
```

```
React's documentation says:
  "You may rely on useMemo as a performance optimization,
   not as a semantic guarantee."

When React MIGHT discard memoized values:

  1. Component unmounts and remounts:
     → New fiber, new hook list, memoized values gone
     → useMemo recomputes

  2. React future feature — "forgetting" memoized values:
     → React may discard memoized values to free memory
     → For offscreen components or under memory pressure
     → Your code must work correctly even if useMemo re-runs

  3. React Compiler auto-memoization:
     → The compiler uses useMemoCache (different from useMemo)
     → Has its own cache invalidation strategy

  What IS guaranteed:
    - useMemo won't recompute if deps haven't changed (Object.is)
    - useCallback returns the same function if deps haven't changed
    - Both use the same areHookInputsEqual comparison

  Internal implementation:
    function updateMemo(nextCreate, deps) {
      const hook = updateWorkInProgressHook();
      const prevDeps = hook.memoizedState[1];

      if (areHookInputsEqual(deps, prevDeps)) {
        return hook.memoizedState[0];  // Return cached value
      }

      const nextValue = nextCreate();
      hook.memoizedState = [nextValue, deps];
      return nextValue;
    }
```

## 7. useRef Mutation During Render

```jsx
function Bad() {
  const ref = useRef(0);
  ref.current += 1;  // ❌ Mutation during render!

  return <div>{ref.current}</div>;
}

function Good() {
  const ref = useRef(0);

  useEffect(() => {
    ref.current += 1;  // ✓ Mutation in effect
  });

  return <div>Rendered</div>;
}
```

```
Why ref mutation during render is dangerous:

  Refs are NOT part of React's state system.
  Mutating a ref during render:
    1. Makes render impure (different result on re-render)
    2. In concurrent mode, render may be called multiple times
       → ref.current gets incremented multiple times
    3. In StrictMode, double render → ref.current += 2 instead of 1

  Refs are "escape hatches" from React's model.
  They should only be mutated in:
    - Event handlers
    - useEffect / useLayoutEffect
    - Never during render

  Internal difference:
    useState: fiber.memoizedState → tracked, triggers re-render
    useRef: fiber.memoizedState → { current: value }
      → NOT tracked, never triggers re-render
      → Just a plain mutable object stored on the hook
      → React never reads or processes it during reconciliation
```

## 8. useId Deterministic Generation

```jsx
function Form() {
  const id = useId();  // ":r1:" (deterministic!)
  return (
    <div>
      <label htmlFor={id}>Name</label>
      <input id={id} />
    </div>
  );
}
```

```
useId generates IDs based on the FIBER TREE POSITION:

  Algorithm (simplified):
    1. Each fiber tracks its position in the tree
    2. Position = path from root to this fiber
    3. Path encoded as a compact string

    function useId() {
      const hook = mountWorkInProgressHook();
      const root = getWorkInProgressRoot();

      // Build ID from tree position
      const id = ':' + identifierPrefix + treeId + ':';
      hook.memoizedState = id;
      return id;
    }

  Tree-based encoding:
    Root
     ├── App         → position: ""
     │   ├── Form    → position: "0" (first child)
     │   │   └── useId → id: ":r0:"
     │   └── Form    → position: "1" (second child)
     │       └── useId → id: ":r1:"
     └── Footer      → position: "H" (sibling)
         └── useId   → id: ":rH:"

  Why tree-based (not counter-based)?
    - Counters differ between server and client
      (server renders sequentially, client renders in tree order)
    - Tree position is the SAME on server and client
    - Enables hydration — server ID matches client ID

  The encoding uses base-32 to keep IDs short:
    0-9, a-v → 32 characters for each tree level
```

## 9. Hooks and Error Boundaries

```jsx
function BuggyHook() {
  const [data, setData] = useState(null);

  useEffect(() => {
    throw new Error('Effect error!');
  }, []);

  return <div>{data}</div>;
}
```

```
Errors in hooks are caught differently:

  Error in render (component body):
    → Caught during render phase
    → React unwinds to nearest error boundary
    → Shows error boundary fallback
    → Standard error boundary behavior

  Error in useEffect/useLayoutEffect:
    → Caught during commit phase (effect execution)
    → captureCommitPhaseError(fiber, error)
    → Schedules update on nearest error boundary
    → Error boundary re-renders with fallback
    → BUT: DOM mutations from this commit are already applied

  Error in useEffect cleanup:
    → Caught during next commit's passive effect flush
    → safelyCallDestroy wraps cleanup in try/catch
    → Error is captured and propagated to boundary

  Error in event handler:
    → NOT caught by React error boundaries!
    → Propagates as normal JavaScript error
    → Must use try/catch in the handler itself

  Error in useState initializer:
    → Caught during render (initializer runs during render)
    → Same as "error in render"

  Error in useReducer reducer:
    → Caught during render (reducer runs during render for state calc)
    → Same as "error in render"
```

## 10. useContext and Re-renders

```jsx
const ThemeContext = createContext('light');

const MemoChild = React.memo(function MemoChild() {
  console.log('MemoChild renders');
  return <div>I'm memoized</div>;
});

function Consumer() {
  const theme = useContext(ThemeContext);  // ← subscribes to context
  console.log('Consumer renders');
  return (
    <div>
      {theme}
      <MemoChild />
    </div>
  );
}
```

```
Context change behavior:

  When ThemeContext value changes:
    1. propagateContextChange scans fiber tree
    2. Finds Consumer (has context dependency in its fiber)
    3. Schedules update on Consumer's fiber
    4. Consumer re-renders, reads new theme value

  Does MemoChild re-render?
    → NO! React.memo prevents it (no prop changes)
    → Consumer re-renders but MemoChild bails out

  Can you skip Consumer's re-render with memo?
    const MemoConsumer = React.memo(Consumer);
    → NO! Context updates BYPASS memo.
    → propagateContextChange marks the fiber directly
    → memo's shouldComponentUpdate/comparison is ignored
    → This is by design — context must propagate

  Optimization: split contexts
    Instead of:
      <BigContext.Provider value={{ theme, user, locale }}>
    Use:
      <ThemeContext.Provider value={theme}>
        <UserContext.Provider value={user}>
          <LocaleContext.Provider value={locale}>

    → Components only subscribe to what they need
    → Changing theme doesn't re-render components that only use user

  Internal mechanism:
    fiber.dependencies = {
      lanes: NoLanes,
      firstContext: {
        context: ThemeContext,
        next: null  // linked list of consumed contexts
      }
    };

    propagateContextChange walks the tree, checks each fiber's
    dependencies list, and marks matching fibers for update.
```
