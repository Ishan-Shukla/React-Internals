# Memory Management and GC Considerations

## Fiber Tree Memory

React maintains two fiber trees (current + WIP) which means double the
fiber node count at peak. After commit, the old current tree becomes the
alternate for next render:

```
Memory during render:

  Current tree: N fibers × ~1KB each     = N KB
  WIP tree:     N fibers × ~1KB each     = N KB  (being built)
  Elements:     M elements × ~0.1KB each = M×0.1 KB  (temporary)
                                          ─────────
  Peak during render:                     ~2N + 0.1M KB

After commit:
  Current tree (was WIP): N KB           = N KB
  Alternate fibers: N KB                 = N KB  (reused next render)
  Elements: garbage collected            = 0 KB
                                          ─────
  Steady state:                           ~2N KB

The alternate tree is NOT garbage collected — it's kept alive
by the `alternate` pointers for reuse in the next render.
```

## Fiber Cleanup on Unmount

When a component unmounts, React must carefully clean up to avoid memory leaks:

```javascript
function detachFiberMutation(fiber) {
  // Cut the fiber's connection to the tree
  // This allows garbage collection of the subtree

  // Clear the alternate
  const alternate = fiber.alternate;
  if (alternate !== null) {
    alternate.return = null;
  }
  fiber.return = null;

  // Clear child/sibling pointers
  fiber.child = null;
  fiber.sibling = null;

  // Clear instance references
  fiber.stateNode = null;

  // Clear update queue
  fiber.updateQueue = null;
  fiber.memoizedState = null;
  fiber.dependencies = null;

  // Clear effect references
  fiber.deletions = null;
}
```

```
Before unmount:
  Parent ──child──► Deleted ──child──► GrandChild
    ▲                  │                   │
    └── return ────────┘                   │
    ▲                                      │
    └── return ────────────────────────────┘

  stateNode ──► <div> (DOM node)
  memoizedState ──► hooks linked list ──► closures ──► ???

After cleanup:
  Parent ──child──► (null)
  Deleted: return=null, child=null, stateNode=null, memoizedState=null
  GrandChild: return=null, child=null, stateNode=null

  Now GC can collect: Deleted fiber, GrandChild fiber,
  DOM nodes, hook objects, closures, effect objects
```

## Common Memory Leak Patterns

### Leak 1: Uncleared Subscriptions

```javascript
// ❌ LEAK: subscription not cleaned up
function Component() {
  useEffect(() => {
    const sub = eventBus.subscribe('update', handleUpdate);
    // Missing: return () => sub.unsubscribe();
  }, []);
  // On unmount: subscription stays active
  // handleUpdate closure retains reference to fiber's scope
  // GC cannot collect the component's state/props
}

// ✅ FIX: always return cleanup
function Component() {
  useEffect(() => {
    const sub = eventBus.subscribe('update', handleUpdate);
    return () => sub.unsubscribe();  // ← cleanup!
  }, []);
}
```

### Leak 2: Timers Not Cleared

```javascript
// ❌ LEAK
function Poller() {
  useEffect(() => {
    const id = setInterval(() => {
      fetchData();  // Closure retains component scope
    }, 5000);
    // Missing cleanup!
  }, []);
}

// ✅ FIX
function Poller() {
  useEffect(() => {
    const id = setInterval(fetchData, 5000);
    return () => clearInterval(id);
  }, []);
}
```

### Leak 3: Closures Retaining Large Objects

```javascript
// ❌ POTENTIAL LEAK: closure retains entire bigData array
function Component({ bigData }) {
  const handleClick = useCallback(() => {
    console.log(bigData.length);  // Closure captures bigData reference
  }, [bigData]);
  // Even after re-render with new bigData, the OLD callback
  // (if stored somewhere) still references old bigData

  return <Child onClick={handleClick} />;
}

// Each render creates a new closure that captures the current bigData.
// If a previous version of handleClick is stored externally
// (e.g., in an event listener that wasn't cleaned up),
// the old bigData cannot be GC'd.
```

### Leak 4: Refs Holding DOM Nodes

```javascript
// ❌ LEAK: ref to unmounted DOM node
const nodeRef = useRef(null);

useEffect(() => {
  nodeRef.current = document.createElement('div');
  document.body.appendChild(nodeRef.current);

  // Missing cleanup — the div stays in the DOM forever!
  // And nodeRef.current prevents GC of the ref object
}, []);

// ✅ FIX
useEffect(() => {
  const node = document.createElement('div');
  document.body.appendChild(node);
  nodeRef.current = node;

  return () => {
    document.body.removeChild(node);
    nodeRef.current = null;
  };
}, []);
```

## Effect Cleanup Timing and Memory

```
Component lifecycle and memory:

Mount:
  Fiber created ──► hooks allocated ──► effects queued
  [memory: fiber + hooks + closures]

Update:
  New closures created for hooks ──► old closures eligible for GC
  (but only AFTER cleanup runs and new effects fire)

  Between renders, BOTH old and new closures exist briefly:
    Old effect cleanup closure: retains old scope
    New effect setup closure: retains new scope
  This is the peak memory for effects.

Unmount:
  1. useLayoutEffect cleanup runs (sync)
  2. useEffect cleanup runs (async)
  3. Fiber references cleared (detachFiberMutation)
  4. Everything eligible for GC

  If cleanup doesn't run (bug), closures from effects
  retain references to: fiber scope → props → state → DOM refs
```

## Hook Memory Layout

```
fiber.memoizedState ──► Hook1 ──next──► Hook2 ──next──► Hook3 ──► null
                        │                │                │
                    memoizedState    memoizedState    memoizedState
                        │                │                │
                        ▼                ▼                ▼
                    state value      effect object    [memoValue, deps]
                                         │
                                    ┌────┴────┐
                                  create    destroy
                                (closure)  (closure)
                                    │          │
                                captures   captures
                                component  component
                                 scope      scope

Each hook's closures (effect create/destroy, setState callbacks)
capture variables from the component's render scope.

If the component renders 100 times, there are 100 versions of
each closure created — but only the latest should be reachable.
Previous versions are GC'd when the hook linked list is replaced.
```

## React DevTools Memory Overhead

```
With DevTools attached:

  React sends fiber tree data to DevTools via the global hook:
  - Component names, props, state, hooks for every fiber
  - Re-render timing data when profiling
  - Component stack traces

  This means DevTools keeps its OWN copy of component data.
  Memory usage can be significantly higher with DevTools open.

  In production builds:
  - Component names may be minified
  - __DEV__ only data is stripped
  - Less memory overhead from React internals
```

## Performance: Why Certain Patterns Are Slow at the Fiber Level

### Pattern 1: Object/Array Props Without Memoization

```javascript
// ❌ SLOW: new object every render
function Parent() {
  return <Child style={{ color: 'red' }} />;
  //             ^^^^^^^^^^^^^^^^^^^^
  //             New object every render!
  //             pendingProps !== memoizedProps
  //             → beginWork cannot bail out
  //             → Child always re-renders
}

// At the fiber level:
// beginWork: oldProps === newProps? NO (different object refs)
// → Cannot bail out → must call Child() → must reconcile children
```

### Pattern 2: Render Functions as Props

```javascript
// ❌ SLOW: new function every render
function Parent() {
  return <List renderItem={(item) => <Item data={item} />} />;
  //           ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  //           New function every render!
  //           List ALWAYS re-renders
}
```

### Pattern 3: Context Value as New Object

```javascript
// ❌ SLOW: triggers propagateContextChange every render
function Provider({ children }) {
  const [count, setCount] = useState(0);
  return (
    <MyContext.Provider value={{ count, setCount }}>
    {/* New object → Object.is fails → ALL consumers re-render */}
      {children}
    </MyContext.Provider>
  );
}
```

### Pattern 4: Large Lists Without Keys

```javascript
// ❌ SLOW: index-based reconciliation
{items.map((item, i) => <Item key={i} data={item} />)}

// At the fiber level:
// When items change order → keys are indices → wrong fibers matched
// → unnecessary prop updates + state corruption
// reconcileChildrenArray does extra work diffing wrong pairs
```

### Pattern 5: Deep Component Trees with Frequent Root Updates

```javascript
// ❌ SLOW: state at root → entire tree re-evaluated
function App() {
  const [mousePos, setMousePos] = useState({ x: 0, y: 0 });
  // mousemove updates → beginWork called on EVERY fiber
  // Even with bailouts, React still traverses until it finds
  // the fibers that actually need updating (via childLanes)
}

// ✅ BETTER: colocate state near where it's used
function MouseTracker() {
  const [mousePos, setMousePos] = useState({ x: 0, y: 0 });
  // Only MouseTracker's subtree is evaluated
}
```

## Measuring Memory

```javascript
// Performance API
performance.measureUserAgentSpecificMemory().then(result => {
  console.log('Total JS heap:', result.bytes);
  result.breakdown.forEach(entry => {
    console.log(entry.types, entry.bytes);
  });
});

// Chrome DevTools → Memory tab
// 1. Take heap snapshot
// 2. Search for "FiberNode" to see all fibers
// 3. Check retained size to find leaks
// 4. Compare snapshots before/after navigation

// React Profiler
// DevTools → Profiler → Record
// Shows render counts, timing per component
// Identifies unnecessary re-renders
```
