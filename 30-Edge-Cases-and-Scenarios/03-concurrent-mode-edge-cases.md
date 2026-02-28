# Edge Cases: Concurrent Mode & Transitions

## 1. Stale Closure Problem

```jsx
function Counter() {
  const [count, setCount] = useState(0);

  useEffect(() => {
    const id = setInterval(() => {
      setCount(count + 1);  // ← BUG: `count` is captured at 0 forever
    }, 1000);
    return () => clearInterval(id);
  }, []);  // ← Empty deps: closure captures count=0

  return <div>{count}</div>;  // Always shows 1
}
```

```
Why this happens:

  Mount: count = 0, effect captures count = 0
    setInterval callback: () => setCount(0 + 1) — always sets to 1!

  The effect closure captured count=0 and never updates.
  Each interval tick calls setCount(0 + 1) = setCount(1).

  Fix 1: Functional updater
    setCount(c => c + 1);  // Uses latest state, not closure

  Fix 2: Include count in deps
    useEffect(() => {
      const id = setInterval(() => setCount(count + 1), 1000);
      return () => clearInterval(id);
    }, [count]);  // Re-creates interval each time count changes
    // This works but creates/destroys intervals frequently

  Fix 3: useRef to hold mutable value
    const countRef = useRef(count);
    countRef.current = count;  // Update ref each render
    useEffect(() => {
      const id = setInterval(() => {
        setCount(countRef.current + 1);  // Reads latest via ref
      }, 1000);
      return () => clearInterval(id);
    }, []);

How concurrent mode makes this WORSE:
  In concurrent mode, a component can render multiple times
  before committing. Each render creates a new closure.
  If you store a closure from a render that gets abandoned,
  it captures stale values from a render that never committed.
  This is why React emphasizes pure render functions.
```

## 2. Tearing in Concurrent Rendering

```jsx
// External mutable store
let externalState = { count: 0 };

function ComponentA() {
  // Reads external state during render
  return <div>A: {externalState.count}</div>;
}

function ComponentB() {
  // Also reads external state during render
  return <div>B: {externalState.count}</div>;
}
```

```
Tearing scenario in concurrent mode:

  t=0ms: React starts rendering (concurrent)
         ComponentA renders: reads externalState.count = 0
         Output: "A: 0"

  t=3ms: shouldYield() → true, React yields to browser

  t=4ms: External event fires:
         externalState.count = 1  ← state mutated!

  t=5ms: React resumes rendering
         ComponentB renders: reads externalState.count = 1
         Output: "B: 1"

  t=6ms: React commits:
         Screen shows: "A: 0" and "B: 1"  ← TORN!
         Same state, same render, but different values!

  This CANNOT happen with React state (useState/useReducer)
  because React controls when state updates are applied.
  It only happens with external mutable stores.

  Fix: useSyncExternalStore
    const count = useSyncExternalStore(
      store.subscribe,
      store.getSnapshot
    );

    React takes a snapshot before rendering and checks it
    after rendering. If the snapshot changed mid-render,
    React restarts the render synchronously to prevent tearing.
```

## 3. Transition Interrupted by Urgent Update

```jsx
function SearchApp() {
  const [query, setQuery] = useState('');
  const [results, setResults] = useState([]);
  const [isPending, startTransition] = useTransition();

  const handleChange = (e) => {
    setQuery(e.target.value);           // ← Urgent: InputLane
    startTransition(() => {
      setResults(search(e.target.value)); // ← Transition: TransitionLane
    });
  };
}
```

```
User types "abc" quickly:

  t=0ms: Type "a"
    → setQuery("a") at InputLane (urgent)
    → setResults(search("a")) at TransitionLane1
    → React renders urgent update first
    → Input shows "a" immediately

  t=1ms: Transition render starts for "a" results
    → beginWork(App) with TransitionLane1
    → beginWork(ResultsList) starts...

  t=10ms: Type "b" (user is fast)
    → setQuery("ab") at InputLane (urgent)
    → setResults(search("ab")) at TransitionLane2

    → INTERRUPT! Urgent update preempts transition
    → Transition render for "a" is ABANDONED
    → All that work in progress tree is thrown away
    → prepareFreshStack()

  t=11ms: Urgent update renders
    → Input shows "ab"
    → isPending = true (transition still pending)

  t=12ms: Type "c"
    → setQuery("abc") at InputLane
    → setResults(search("abc")) at TransitionLane3

    → Previous transition (for "ab") also abandoned
    → Only the latest transition will complete

  t=13ms: Urgent renders "abc"
  t=20ms: No more typing — transition for "abc" completes
    → Results for "abc" shown
    → isPending = false

  The "a" and "ab" transition renders were completely wasted.
  This is by design — React prioritizes responsiveness over efficiency.
  The abandoned renders had no side effects, so no cleanup needed.
```

## 4. useDeferredValue Creates "Ghost" Renders

```jsx
function App() {
  const [text, setText] = useState('');
  const deferredText = useDeferredValue(text);

  return (
    <>
      <input value={text} onChange={e => setText(e.target.value)} />
      <ExpensiveList text={deferredText} />
    </>
  );
}
```

```
User types "hi":

  State changes: text="" → text="h" → text="hi"

  Render 1 (urgent): text="h", deferredText="" (still old!)
    → Input shows "h"
    → ExpensiveList renders with "" (previous value)
    → Committed to screen

  Render 2 (deferred): text="h", deferredText="h"
    → This is a "background" render at lower priority
    → Input still shows "h"
    → ExpensiveList renders with "h"
    → BUT — user types "i" before this commits...

  Render 3 (urgent): text="hi", deferredText="h" (still deferred!)
    → INTERRUPTS Render 2
    → Input shows "hi"
    → ExpensiveList renders with "h" (deferred hasn't caught up)
    → Committed

  Render 4 (deferred): text="hi", deferredText="hi"
    → Finally matches
    → ExpensiveList renders with "hi"
    → Committed — screen fully consistent

  Key insight: deferredText LAGS behind text.
  There can be renders where they disagree.
  React tells you: Object.is(text, deferredText) → false when lagging.
  isPending from useTransition, or comparing values, lets you show spinners.
```

## 5. Concurrent Features Disabled for Sync Lanes

```
Not all updates are concurrent! React uses sync rendering for:

  1. SyncLane updates:
     → flushSync(() => setState(x))
     → Legacy root (ReactDOM.render, not createRoot)
     → All of these use performSyncWorkOnRoot
     → workLoopSync: NO yielding, NO interruption

  2. Default updates in createRoot:
     → onClick handler → setState(x)
     → Gets DefaultLane (not SyncLane)
     → Uses performConcurrentWorkOnRoot
     → workLoopConcurrent: CAN yield, CAN be interrupted

  3. Transition updates:
     → startTransition(() => setState(x))
     → Gets TransitionLane
     → Concurrent rendering
     → Lowest priority, most interruptible

  This means "concurrent mode" isn't all-or-nothing.
  Different updates in the SAME app use different rendering modes
  based on their lane priority.

  Edge case: If a sync update and transition update are pending,
  React processes the sync update first (synchronously),
  then starts the transition (concurrently).
  The transition can reuse work from the sync render
  if the fibers haven't changed.
```

## 6. Double Rendering in StrictMode

```jsx
// With <StrictMode>:
function App() {
  console.log('render');  // Logs TWICE in development!
  const [count, setCount] = useState(() => {
    console.log('init');  // Logs TWICE on mount!
    return 0;
  });

  useEffect(() => {
    console.log('effect setup');
    return () => console.log('effect cleanup');
  }, []);
  // Mount: setup → cleanup → setup (effect runs twice!)

  return <div>{count}</div>;
}
```

```
StrictMode double-invocation (DEV only):

  On mount:
    1. Component function called (first time)
    2. Component function called (second time) ← StrictMode
       → console.log is suppressed on second call
    3. Only ONE fiber/result is used

    4. Effects mount: setup runs
    5. Effects unmount: cleanup runs     ← StrictMode simulates unmount
    6. Effects remount: setup runs again ← StrictMode simulates remount

  Purpose:
    - Double render: catches impure render functions
      (side effects during render that produce different results)
    - Double effects: catches missing cleanup
      (if your effect doesn't clean up properly, the remount breaks)

  In PRODUCTION:
    - Components render ONCE
    - Effects mount ONCE
    - No double invocation
    - StrictMode has zero runtime cost

  What StrictMode does NOT double:
    - Event handlers (onClick, onChange, etc.)
    - Commit-phase lifecycle (componentDidMount is doubled for class)
    - Passive effects ARE doubled (setup → cleanup → setup)
    - Layout effects ARE doubled
```

## 7. Time-Slicing Granularity

```
React yields every ~5ms (configurable by scheduler):

  workLoopConcurrent() {
    while (workInProgress !== null && !shouldYield()) {
      performUnitOfWork(workInProgress);
    }
  }

  shouldYield checks:
    currentTime >= deadline  (deadline = startTime + 5ms)

  But React can only yield BETWEEN fiber units of work.
  It cannot yield in the MIDDLE of a component's render function.

  Edge case: Component render takes 50ms
    → React calls Component(props)
    → Component does heavy computation for 50ms
    → shouldYield is NOT checked during this time
    → Browser is blocked for 50ms
    → After Component returns, shouldYield → true
    → React yields, but the damage (jank) is done

  This is why you should:
    1. Keep individual component renders fast (<16ms)
    2. Use memo/useMemo for expensive computations
    3. Move heavy work to Web Workers
    4. Split large lists with virtualization

  React's time-slicing helps with MANY small components,
  not ONE expensive component.
```

## 8. Race Condition with Rapid State Updates

```jsx
function DataFetcher({ id }) {
  const [data, setData] = useState(null);

  useEffect(() => {
    let cancelled = false;

    fetch(`/api/data/${id}`)
      .then(res => res.json())
      .then(result => {
        if (!cancelled) {
          setData(result);
        }
      });

    return () => { cancelled = true; };
  }, [id]);

  return <div>{data?.name}</div>;
}
```

```
Race condition scenario:

  t=0ms:  id=1 → fetch("/api/data/1") starts
  t=5ms:  id=2 → cleanup: cancelled=true for id=1
                → fetch("/api/data/2") starts
  t=10ms: id=3 → cleanup: cancelled=true for id=2
                → fetch("/api/data/3") starts
  t=50ms: fetch for id=2 resolves! (server was fast)
          → cancelled=true → setData NOT called ✓
  t=100ms: fetch for id=1 resolves (slow server)
           → cancelled=true → setData NOT called ✓
  t=200ms: fetch for id=3 resolves
           → cancelled=false → setData(result3) ✓

  Without the cancellation flag:
    → fetch for id=1 resolves last (slow)
    → setData(result1) overwrites result3
    → UI shows data for id=1 while displaying id=3!

  React 19's use() hook + Suspense handles this automatically:
    function DataFetcher({ id }) {
      const data = use(fetchData(id));  // Suspends
      return <div>{data.name}</div>;
    }
    // React manages the race condition via Suspense boundaries
```

## 9. Lane Starvation Prevention

```
Problem: If urgent updates keep arriving, transitions never complete.

  t=0ms:   Transition starts rendering
  t=5ms:   Urgent click → interrupt, restart
  t=10ms:  Transition resumes
  t=15ms:  Urgent click → interrupt again
  t=20ms:  Transition resumes
  ... (transition never finishes!)

React's starvation prevention:

  Each pending lane has an expiration time:
    markStarvedLanesAsExpired(root, currentTime)

  When a lane's expiration time passes:
    → The lane is marked as EXPIRED
    → includesExpiredLane(nextLanes) returns true
    → React uses workLoopSync (not concurrent!)
    → The expired transition renders synchronously
    → Cannot be interrupted

  Expiration times by priority:
    SyncLane:        -1 (expires immediately)
    InputLane:       250ms
    DefaultLane:     5000ms (5 seconds)
    TransitionLane:  5000ms (5 seconds)
    IdleLane:        never (can be starved forever)

  So a transition stuck for >5 seconds will force
  synchronous rendering, even at the cost of jank.
  This prevents indefinite starvation.
```

## 10. Entangled Lanes: Updates That Must Render Together

```jsx
function App() {
  const [a, setA] = useState(0);
  const [b, setB] = useState(0);

  const handleClick = () => {
    setA(1);                           // DefaultLane
    startTransition(() => setB(1));    // TransitionLane
  };
}
```

```
These updates are on DIFFERENT lanes:
  setA(1) → DefaultLane (bit 5)
  setB(1) → TransitionLane1 (bit 7)

React processes them in separate renders:
  Render 1: Process DefaultLane → a=1, b=0
  Render 2: Process TransitionLane → a=1, b=1

But some updates MUST render together (entangled):

  Case 1: Multiple setState in same startTransition
    startTransition(() => {
      setA(1);   // TransitionLane1
      setB(1);   // TransitionLane1 (same lane!)
    });
    → Both get the SAME transition lane
    → Always processed in the same render

  Case 2: Entangled lanes via entangleLanes()
    When React detects that certain lanes produce
    intermediate states that shouldn't be visible,
    it entangles them:
    root.entangledLanes |= lane;
    root.entanglements[laneIndex] |= entangledLanes;

    → getNextLanes() returns entangled lanes together
    → They render in the same pass

  Case 3: Suspense entanglement
    When a Suspense boundary shows a fallback due to a
    transition, the lanes that caused it are entangled
    with the retry lanes. This ensures the retry commits
    atomically with the content that triggered suspension.
```
