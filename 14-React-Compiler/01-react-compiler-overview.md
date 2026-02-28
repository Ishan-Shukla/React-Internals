# React Compiler (React Forget)

## What Is the React Compiler?

The React Compiler is a **build-time tool** that automatically adds
memoization to your React components. It eliminates the need for manual
`useMemo`, `useCallback`, and `React.memo` by analyzing your code at
compile time and inserting fine-grained caching.

```
BEFORE (manual memoization):                AFTER (compiler does it for you):

function ProductList({ items, tax }) {      function ProductList({ items, tax }) {
  const totals = useMemo(                     // Compiler auto-memoizes!
    () => items.map(i => i.price + tax),      const totals = items.map(
    [items, tax]                                i => i.price + tax
  );                                          );
  const handleClick = useCallback(            const handleClick = (id) => {
    (id) => { selectItem(id); },                selectItem(id);
    [selectItem]                              };
  );                                          return (
  return (                                      <ul>
    <ul>                                          {totals.map((t, i) => (
      {totals.map((t, i) => (                       <Item
        <Item                                         total={t}
          key={i}                                     onClick={handleClick}
          total={t}                                 />
          onClick={handleClick}                   ))}
        />                                      </ul>
      ))}                                     );
    </ul>                                   }
  );
}
```

## Why a Compiler?

Manual memoization has fundamental problems:

```
Problem 1: MISSED MEMOIZATION
  function App({ data }) {
    const filtered = data.filter(x => x.active);  // ← new array every render
    return <ExpensiveList items={filtered} />;     // ← always re-renders!
    // Developer forgot useMemo → performance bug
  }

Problem 2: OVER-MEMOIZATION
  function App({ data }) {
    const value = useMemo(() => data.length, [data]); // ← useMemo is overhead
    // data.length is trivially cheap, memoization costs MORE than recomputing
  }

Problem 3: STALE CLOSURES
  function App({ id }) {
    const handler = useCallback(() => {
      fetch(`/api/${id}`);
    }, []);  // ← forgot 'id' in deps → stale closure bug!
  }

Problem 4: DEPENDENCY ARRAYS ARE ERROR-PRONE
  useMemo(() => compute(a, b, c), [a, b]);  // ← forgot 'c' → stale result
```

The compiler solves ALL of these by analyzing data flow at build time.

## How the Compiler Works

### Phase 1: Parse and Build HIR (High-level IR)

The compiler converts your component into an intermediate representation:

```
function Counter({ initial }) {       HIR:
  const [count, setCount] =           ┌─────────────────────────┐
    useState(initial);                 │ Block 0:                │
  const doubled = count * 2;           │   $1 = LoadProp initial │
  const handleClick = () => {          │   $2 = CallHook useState│
    setCount(c => c + 1);             │   $3 = Destructure $2   │
  };                                   │   $4 = $3[0] (count)   │
  return (                             │   $5 = $3[1] (setCount)│
    <div onClick={handleClick}>        │   $6 = $4 * 2 (doubled)│
      {doubled}                        │   $7 = ArrowFunction    │
    </div>                             │   $8 = JSX div          │
  );                                   │   Return $8             │
}                                      └─────────────────────────┘
```

### Phase 2: Analyze Dependencies

The compiler builds a **dependency graph** — which values depend on which:

```
Dependency Graph:

  initial ──────► useState ──────► [count, setCount]
                                       │         │
                                       ▼         │
                                    doubled      │
                                       │         │
                                       ▼         ▼
                                    JSX div ◄── handleClick

  Reactive dependencies:
    doubled depends on: count
    handleClick depends on: setCount (stable)
    JSX depends on: doubled, handleClick
```

### Phase 3: Insert Cache Slots

The compiler inserts caching code using a hidden `useMemoCache` hook:

```javascript
// COMPILER OUTPUT (simplified):

function Counter({ initial }) {
  const $ = useMemoCache(4);  // ← Compiler-generated cache with 4 slots

  const [count, setCount] = useState(initial);

  // Cache slot 0: doubled (depends on count)
  let doubled;
  if ($[0] !== count) {
    doubled = count * 2;
    $[0] = count;
    $[1] = doubled;
  } else {
    doubled = $[1];
  }

  // Cache slot 2: handleClick (depends on setCount — stable, cached once)
  let handleClick;
  if ($[2] === Symbol.for("react.memo_cache_sentinel")) {
    handleClick = () => {
      setCount(c => c + 1);
    };
    $[2] = handleClick;
  } else {
    handleClick = $[2];
  }

  // Cache slot 3: JSX (depends on doubled, handleClick)
  let t0;
  if ($[1] !== doubled || $[2] !== handleClick) {
    t0 = <div onClick={handleClick}>{doubled}</div>;
    $[3] = t0;
  } else {
    t0 = $[3];
  }

  return t0;
}
```

### useMemoCache: The Runtime Primitive

```javascript
// This is the single runtime hook the compiler uses

function useMemoCache(size) {
  const hook = updateWorkInProgressHook();

  let cache = hook.memoizedState;
  if (cache === null) {
    // First render: create cache array
    cache = new Array(size);
    cache.fill(Symbol.for("react.memo_cache_sentinel"));
    hook.memoizedState = cache;
  }

  return cache;
}

// The cache is just an array on the hooks linked list
// Each slot stores a cached value
// Sentinel value means "not yet computed"
```

## Granularity: Component-Level vs Value-Level

```
React.memo (component-level):
  ┌──────────────────────────┐
  │ Component                │
  │                          │ ← ALL or NOTHING
  │  value1 ← recomputed    │    If ANY prop changes,
  │  value2 ← recomputed    │    EVERYTHING recomputes
  │  value3 ← recomputed    │
  │  jsx    ← recreated     │
  └──────────────────────────┘

React Compiler (value-level):
  ┌──────────────────────────┐
  │ Component                │
  │                          │ ← FINE-GRAINED
  │  value1 ← recomputed ✓  │    Only the specific values
  │  value2 ← CACHED     ░  │    that actually changed
  │  value3 ← CACHED     ░  │    get recomputed
  │  jsx    ← CACHED     ░  │
  └──────────────────────────┘
```

## The Rules of React (Compiler Requirements)

The compiler assumes your code follows the **Rules of React**. If it
detects violations, it skips memoizing that code:

```
RULE 1: Components and hooks must be PURE during render
  ✅ const doubled = count * 2;
  ❌ const doubled = (sideEffect(), count * 2);

RULE 2: No mutation of props, state, or hook return values
  ✅ const newItems = [...items, newItem];
  ❌ items.push(newItem);

RULE 3: No reading from refs during render
  ✅ useEffect(() => { ref.current.focus(); });
  ❌ const width = ref.current.offsetWidth; // in render!

RULE 4: No calling setState from render (except with prev => next pattern)
  ✅ setCount(prev => prev + 1);  // Okay in some patterns
  ❌ setCount(count + 1);          // During render → loop

When the compiler detects a violation:
  → It does NOT crash
  → It simply SKIPS memoization for that section
  → Your code still works, just without optimization
```

## Compiler vs Manual: What Changes

```
                        Manual                  Compiler
                        ────────                ────────
useMemo                 You write it            Auto-inserted as cache checks
useCallback             You write it            Auto-inserted as cache checks
React.memo              You wrap components     Component output is cached
Dependency arrays       You maintain them       Compiler computes them
Stale closures          Your bug to find        Impossible (compiler tracks deps)
Over-memoization        Common mistake          Compiler knows the cost
Missed memoization      Common mistake          Compiler catches everything
```

## Using the Compiler

```bash
# Install
npm install -D babel-plugin-react-compiler

# babel.config.js
module.exports = {
  plugins: [
    ['babel-plugin-react-compiler', {
      // Optional: restrict to specific directories
      sources: (filename) => {
        return filename.includes('src/');
      },
    }],
  ],
};

# Next.js (built-in support)
// next.config.js
module.exports = {
  experimental: {
    reactCompiler: true,
  },
};
```

## Opting Out

```javascript
// Opt out a specific component:
function LegacyComponent() {
  "use no memo";  // ← Directive tells compiler to skip this component
  // ... code that violates Rules of React
}
```

## Architecture Diagram

```
┌──────────────────────────────────────────────────────────┐
│                    BUILD TIME                              │
│                                                           │
│  Your Code ──► Babel/SWC ──► React Compiler Plugin ──►   │
│                               │                           │
│                    ┌──────────┴──────────┐                │
│                    │  1. Parse to HIR     │                │
│                    │  2. Analyze deps     │                │
│                    │  3. Determine cache  │                │
│                    │     boundaries       │                │
│                    │  4. Insert           │                │
│                    │     useMemoCache     │                │
│                    │     + cache checks   │                │
│                    └──────────┬──────────┘                │
│                               │                           │
│                    Optimized Code                         │
└───────────────────────────────┼───────────────────────────┘
                                │
┌───────────────────────────────┼───────────────────────────┐
│                    RUNTIME                                 │
│                                                           │
│  useMemoCache(N) → array of N cache slots on fiber        │
│  Each slot: sentinel → first compute → cached thereafter  │
│  Invalidation: dependency value !== cached → recompute    │
└───────────────────────────────────────────────────────────┘
```
