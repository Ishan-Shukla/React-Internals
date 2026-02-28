# Edge Cases: Render Phase

## 1. setState During Render (Render-Phase Updates)

React allows calling setState during render — but only with specific semantics.
This is NOT the same as calling setState in an event handler.

```jsx
function ScrollTracker({ items }) {
  const [isAtBottom, setIsAtBottom] = useState(false);

  // setState DURING render — React handles this specially
  if (items.length > 100 && !isAtBottom) {
    setIsAtBottom(true);  // ← This triggers a re-render of THIS component
  }

  return <div>{isAtBottom ? 'Scrolled' : 'Top'}</div>;
}
```

### What Happens Internally

```
renderWithHooks(Component) — first call:
  Component(props) runs
    → setState(true) called DURING render
    → didScheduleRenderPhaseUpdate = true
    → Update enqueued on hook's queue

  Component returned JSX, but didScheduleRenderPhaseUpdate is true!

  → React re-runs Component(props) immediately (same render cycle)
  → Uses HooksDispatcherOnRerender (third dispatcher!)
  → Processes the queued update: isAtBottom = true
  → Component returns new JSX

  → If ANOTHER setState fires during this re-run:
    → numberOfReRenders++ (counted)
    → Re-run again

  → If numberOfReRenders > 25:
    → throw Error('Too many re-renders')
    → This prevents infinite loops

Important:
  - The extra re-renders happen IN THE SAME beginWork call
  - No new fiber traversal, no re-scheduling
  - The component just runs again until it stabilizes
  - This is how getDerivedStateFromProps is emulated in hooks
```

```
Normal setState (event handler):      Render-phase setState:
  enqueue → schedule → later render     enqueue → re-run NOW (same beginWork)
  New render cycle                      Same render cycle, just re-execute
  Can be batched with others            Processed immediately, up to 25 times
```

## 2. Throwing During Render

When a component throws, the behavior depends on WHAT was thrown:

```
Component throws during render:
        │
        ├── Threw a Promise (thenable)?
        │   └── SUSPENSE path
        │       → Find nearest Suspense boundary
        │       → Show fallback
        │       → Attach .then(ping) for retry
        │
        ├── Threw an Error?
        │   └── ERROR BOUNDARY path
        │       → Find nearest error boundary (class with getDerivedStateFromError)
        │       → Show fallback UI
        │       → componentDidCatch fires in commit
        │
        └── Threw something else (string, number, etc.)?
            └── ERROR BOUNDARY path (same as Error)
                → Wrapped in Error if not already
```

## 3. Bailout After Rendering (didReceiveUpdate = false)

A component can render and produce identical output. React still tries
to bail out:

```jsx
function MaybeUpdates({ value }) {
  const [count, setCount] = useState(0);
  // Component re-renders because parent passed new props object
  // But the actual VALUE hasn't changed
  return <div>{count}</div>;  // Same output as before
}
```

```
beginWork: oldProps !== newProps (different object)
  → Cannot bail out before render
  → Call Component(props)
  → Hooks process: no state changes → didReceiveUpdate stays false
  → After render: didReceiveUpdate === false
  → bailoutOnAlreadyFinishedWork!
  → Children are CLONED, not re-rendered

This is the "render then bailout" path:
  The component DID render (its function was called),
  but its children DON'T re-render because the output was identical.
```

## 4. Concurrent Render Abandoned Mid-Way

```
Scenario: Rendering a large list in concurrent mode

t=0ms:  startTransition(() => setFilter('search'))
        → Schedule concurrent render at TransitionLane

t=1ms:  workLoopConcurrent begins
        beginWork(App) ✓
        beginWork(FilteredList) ✓
        beginWork(Item1) ✓
        beginWork(Item2) ✓

t=5ms:  shouldYield() → true, yield to browser

t=6ms:  ★ User calls setFilter('search2') inside another startTransition
        → Same TransitionLane (or next transition lane)
        → React decides to RESTART the render with new state
        → Old WIP tree is ABANDONED (not committed, not cleaned up by React
          explicitly — just becomes unreachable for GC)

t=7ms:  prepareFreshStack() → new WIP tree from current
        Start rendering with filter='search2' from scratch
        All work from t=1ms to t=5ms is thrown away

This is safe because:
  - Render phase has no side effects
  - No DOM was touched
  - No effects were run
  - The abandoned WIP tree is just garbage collected
```

## 5. Same Component Rendered in Multiple Positions

```jsx
function App() {
  return (
    <div>
      <Counter />    {/* Fiber A */}
      <Counter />    {/* Fiber B — completely independent! */}
    </div>
  );
}
```

```
Each <Counter /> gets its OWN fiber node with its OWN state:

div
 ├── Counter (fiber A, memoizedState: { count: 3 })
 └── Counter (fiber B, memoizedState: { count: 7 })

These share the same FUNCTION reference (Counter)
but have SEPARATE fiber nodes, separate hooks linked lists,
separate state. Updating one does NOT affect the other.

The reconciler distinguishes them by POSITION (index 0 vs 1)
within the parent's child list.
```

## 6. Recursive Component Rendering

```jsx
function RecursiveTree({ node, depth = 0 }) {
  if (depth > 100) return null;  // Must have a base case!

  return (
    <div style={{ marginLeft: depth * 20 }}>
      {node.name}
      {node.children?.map(child => (
        <RecursiveTree key={child.id} node={child} depth={depth + 1} />
      ))}
    </div>
  );
}
```

```
React handles this fine — each recursion level is a separate fiber:

RecursiveTree (depth=0)
 └── div
      ├── "root"
      ├── RecursiveTree (depth=1, key="a")
      │    └── div
      │         ├── "child-a"
      │         └── RecursiveTree (depth=2, key="a1")
      │              └── div
      │                   └── "leaf"
      └── RecursiveTree (depth=1, key="b")
           └── div
                └── "child-b"

Each RecursiveTree instance has its own fiber and own hooks.
React's iterative traversal (not recursive JS calls) handles this.
Deep trees don't blow the call stack because of the fiber work loop.
```

## 7. Component Returns Different Types Across Renders

```jsx
function Flexible({ mode }) {
  if (mode === 'text') return 'Just a string';
  if (mode === 'number') return 42;
  if (mode === 'null') return null;
  if (mode === 'array') return [<A key="a" />, <B key="b" />];
  if (mode === 'fragment') return <><C /><D /></>;
  if (mode === 'portal') return createPortal(<E />, otherNode);
  return <div>Normal element</div>;
}
```

```
All of these are valid return values. The reconciler handles each:

'string'     → HostText fiber (tag: 6)
42           → HostText fiber (coerced to string "42")
null         → No fiber (component renders nothing)
false        → No fiber (same as null)
undefined    → Error in dev ("Nothing was returned from render")
[A, B]       → Fragment fiber wrapping A and B
<><C/><D/></>→ Fragment fiber wrapping C and D
Portal       → HostPortal fiber
<div>        → HostComponent fiber

When the return type CHANGES across renders:
  Return 'string' then return <div>
  → HostText fiber vs HostComponent fiber
  → Different types → old fiber deleted, new fiber created
  → State of children is lost
```

## 8. useEffect Cleanup Runs After Component Unmounts

```jsx
function Timer() {
  const [count, setCount] = useState(0);

  useEffect(() => {
    const id = setInterval(() => {
      setCount(c => c + 1);  // ← This still fires after unmount!
    }, 1000);

    return () => clearInterval(id);
    // Cleanup runs AFTER unmount, but React handles the
    // setState-after-unmount case gracefully
  }, []);

  return <div>{count}</div>;
}
```

```
What happens when Timer unmounts but cleanup hasn't run yet:

1. Commit phase: Timer fiber is deleted from the tree
2. useLayoutEffect cleanup runs synchronously (if any)
3. Passive effects are scheduled asynchronously

   Between steps 2 and 3, the interval may fire:
     setInterval callback → setCount(c => c + 1)
     → dispatchReducerAction(fiber, queue, action)
     → fiber is detached (return === null)
     → scheduleUpdateOnFiber finds no root
     → Returns without scheduling (silent noop)

4. flushPassiveEffects runs → clearInterval(id)
   → Interval stopped, no more callbacks

React gracefully ignores setState on unmounted components.
In React 18+, no warning is logged (the warning was removed
because it caused more confusion than it prevented bugs).
```

## 9. Key Collision

```jsx
function App() {
  return (
    <div>
      <A key="same" />
      <B key="same" />   {/* ← Same key as A! */}
    </div>
  );
}
```

```
React behavior with duplicate keys:

1. Dev mode: Warning logged
   "Encountered two children with the same key, `same`"

2. Runtime behavior: UNDEFINED (implementation-dependent)
   React may:
   - Match the first element with key="same" to the first fiber
   - Skip or overwrite the second element
   - Produce inconsistent state

3. The reconciler's Map-based lookup (pass 2 of list diffing)
   stores fibers by key. Duplicate keys overwrite:
   map.set("same", fiberA)
   map.set("same", fiberB)  ← fiberA is now unreachable
   → fiberA is never matched → may be incorrectly deleted

This is always a bug. Keys MUST be unique among siblings.
```

## 10. Rendering null/undefined/boolean Children

```jsx
<div>
  {null}              {/* Renders nothing */}
  {undefined}         {/* Renders nothing */}
  {true}              {/* Renders nothing */}
  {false}             {/* Renders nothing */}
  {0}                 {/* ⚠ Renders "0" — common gotcha! */}
  {''}                {/* Renders nothing (empty string) */}
  {NaN}               {/* Renders "NaN" */}
</div>
```

```
The reconciler filters "empty" children:

function reconcileChildFibers(returnFiber, currentFirstChild, newChild) {
  // null, undefined, boolean → treated as empty
  if (typeof newChild === 'undefined' || typeof newChild === 'boolean') {
    newChild = null;
  }

  if (newChild === null) {
    // Delete existing children at this position
    return deleteRemainingChildren(returnFiber, currentFirstChild);
  }

  // Numbers (including 0) and strings are rendered as text:
  if (typeof newChild === 'number' || typeof newChild === 'string') {
    return placeSingleChild(
      reconcileSingleTextNode(returnFiber, currentFirstChild, '' + newChild)
    );
  }
}

Common gotcha:
  {items.length && <List items={items} />}
  When items = [] → items.length = 0 → renders "0" on screen!

  Fix: {items.length > 0 && <List items={items} />}
  Or:  {items.length ? <List items={items} /> : null}
```
