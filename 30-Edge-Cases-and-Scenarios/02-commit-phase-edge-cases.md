# Edge Cases: Commit Phase

## 1. Effect Execution Order Across Component Tree

```jsx
function Parent() {
  useEffect(() => { console.log('Parent effect'); return () => console.log('Parent cleanup'); });
  useLayoutEffect(() => { console.log('Parent layout effect'); return () => console.log('Parent layout cleanup'); });
  return <Child />;
}

function Child() {
  useEffect(() => { console.log('Child effect'); return () => console.log('Child cleanup'); });
  useLayoutEffect(() => { console.log('Child layout effect'); return () => console.log('Child layout cleanup'); });
  return <div />;
}
```

```
On MOUNT — execution order:

  1. Child layout effect        ← layout effects fire bottom-up (child first!)
  2. Parent layout effect       ← then parent
  3. (browser paints)
  4. Child effect               ← passive effects fire bottom-up
  5. Parent effect

On UNMOUNT — cleanup order:

  1. Parent layout cleanup      ← layout cleanups fire top-down (parent first!)
  2. Child layout cleanup
  3. Parent cleanup             ← passive cleanups fire... during next flush
  4. Child cleanup

WHY bottom-up for mount?
  → completeWork builds the tree bottom-up
  → Effects are collected in that order
  → Layout effects run in commit order (child → parent)

This means a parent's layout effect can read DOM measurements that
include changes made by children's layout effects.
```

## 2. Ref Timing: When Refs Are Set and Cleared

```jsx
function App({ show }) {
  const divRef = useRef(null);

  useLayoutEffect(() => {
    // ✓ divRef.current is available here
    console.log(divRef.current);  // <div> element or null
  });

  useEffect(() => {
    // ✓ divRef.current is also available here
    console.log(divRef.current);
  });

  return show ? <div ref={divRef}>Hello</div> : null;
}
```

```
Ref assignment happens during the LAYOUT sub-phase of commit:

  Before Mutation phase:
    → refs are NOT yet updated
    → divRef.current still points to old element (or null)

  Mutation phase:
    → DOM is mutated (element inserted/removed)
    → But refs NOT yet updated

  Layout phase (commitAttachRef / commitDetachRef):
    → Old refs are detached (set to null)
    → New refs are attached (set to DOM element)
    → useLayoutEffect runs — refs ARE available ✓
    → class componentDidMount/Update runs — refs ARE available ✓

  Passive effects (useEffect):
    → Runs after paint — refs ARE available ✓

Ref clearing order on unmount:
  1. commitDeletionEffectsOnFiber walks the deleted subtree
  2. safelyDetachRef(fiber) sets ref to null
  3. This happens BEFORE cleanup effects run
  4. So cleanup functions can still read the ref? NO —
     ref is already null by the time cleanup runs!

  Common gotcha:
    useEffect(() => {
      const node = ref.current;  // Capture ref in setup
      return () => {
        // ref.current is null here!
        // But `node` still has the reference
        node.removeEventListener(...);  // ✓ works
      };
    }, []);
```

## 3. Multiple DOM Mutations in Single Commit

```jsx
function App({ items }) {
  return (
    <ul>
      {items.map(item => (
        <li key={item.id}>{item.text}</li>
      ))}
    </ul>
  );
}

// Render 1: items = [A, B, C]
// Render 2: items = [C, A, D]  ← reorder + add + remove
```

```
The commit phase applies ALL mutations in one synchronous pass:

  Mutation sub-phase processes the effect list:

  1. commitPlacement(D)    → insertBefore/appendChild for new <li>D</li>
  2. commitPlacement(C)    → move <li>C</li> to new position
  3. commitPlacement(A)    → move <li>A</li> to new position
  4. commitDeletion(B)     → remove <li>B</li>

  All these DOM operations happen synchronously.
  The browser does NOT repaint between them.
  User sees the final state atomically.

  getHostSibling complexity:
    When React needs to insertBefore, it must find the next
    sibling that is actually in the DOM. This requires walking
    the fiber tree because:
    - Some fibers are component fibers (no DOM node)
    - Some fibers are newly inserted (not in DOM yet)
    - Must skip over non-host and non-placed fibers

    getHostSibling walks right siblings and drills into children
    to find the first HostComponent/HostText that's already placed.
    This can be O(n) in worst case for deeply nested components.
```

## 4. flushSync Inside Effects

```jsx
function Problematic() {
  useEffect(() => {
    // flushSync inside an effect — forces synchronous re-render
    flushSync(() => {
      setSomeState(newValue);
    });
    // By this point, the DOM has been updated synchronously
  });

  useLayoutEffect(() => {
    // flushSync inside a layout effect — even more dangerous
    flushSync(() => {
      setSomeState(newValue);
    });
    // This can cause nested commits!
  });
}
```

```
flushSync forces synchronous rendering:

  Normal flow:
    setState → schedule → workLoop (async) → commit

  flushSync flow:
    flushSync(() => setState(x))
      → schedule with SyncLane
      → immediately call performSyncWorkOnRoot
      → render phase runs NOW
      → commit phase runs NOW
      → DOM is updated before flushSync returns

  Inside useLayoutEffect:
    commitRoot running
      → useLayoutEffect fires
        → flushSync(() => setState(x))
          → WARNING: React is already committing!
          → React defers the sync update until after current commit
          → But processes it before returning to the browser
          → Effectively: nested render + commit cycle

  Inside useEffect:
    → This is safer — passive effects run after commit
    → flushSync works normally
    → But still causes synchronous re-render mid-effect-batch
    → Other pending passive effects may not have run yet

  React 18+ logs a warning:
    "flushSync was called from inside a lifecycle method"
```

## 5. Commit Phase Interrupted by Error

```jsx
function App() {
  return (
    <ErrorBoundary>
      <ComponentA />  {/* Has useLayoutEffect that throws */}
      <ComponentB />  {/* Also has effects */}
    </ErrorBoundary>
  );
}
```

```
What happens when an error occurs DURING commit:

  Commit is normally synchronous and un-interruptible.
  But errors can still occur in:
    - useLayoutEffect
    - componentDidMount / componentDidUpdate
    - ref callbacks

  Error during commit:
    1. commitMutationEffects runs → DOM is already mutated
    2. commitLayoutEffects starts
       → ComponentA's useLayoutEffect throws!
    3. captureCommitPhaseError(fiber, error)
       → Error is captured
       → Schedules error recovery on the nearest error boundary
    4. Remaining layout effects for OTHER components still run
       (React tries to be resilient — doesn't abort all effects)
    5. After commit completes, error boundary processes the error
       → New render cycle with the error boundary showing fallback

  Key point: DOM mutations CANNOT be rolled back.
    If the error happens in layout effects, the DOM already reflects
    the new state. The error boundary's fallback will be a new render
    that replaces the broken UI — not a rollback.

  This is different from render-phase errors:
    Render phase → no DOM touched → can retry or show boundary
    Commit phase → DOM already changed → can only show boundary on top
```

## 6. Passive Effect Cleanup Timing

```jsx
function App() {
  const [count, setCount] = useState(0);

  useEffect(() => {
    console.log(`setup ${count}`);
    return () => console.log(`cleanup ${count}`);
  }, [count]);

  return <button onClick={() => setCount(c => c + 1)}>{count}</button>;
}
```

```
Click sequence: count 0 → 1 → 2

  Render 0 (mount):
    commit → schedule passive effects
    flushPassiveEffects:
      setup 0                     ← no cleanup on mount

  Click → count = 1:
    render → commit → schedule passive effects
    flushPassiveEffects:
      cleanup 0                   ← cleanup runs FIRST (old effect)
      setup 1                     ← then setup with new value

  Click → count = 2:
    render → commit → schedule passive effects
    flushPassiveEffects:
      cleanup 1                   ← cleanup of previous
      setup 2

  IMPORTANT: Between commit and flushPassiveEffects,
  the browser MAY paint. This means:

    t=0: DOM updated (shows "1")
    t=1: Browser paints "1" to screen
    t=2: cleanup 0 runs (old interval cleared, etc.)
    t=3: setup 1 runs (new interval started, etc.)

  There's a window where the DOM shows new content
  but the old effect hasn't cleaned up yet.
  This is usually fine but matters for subscriptions.

  React 18 guarantees:
    - All CLEANUP functions for the update run before any SETUP functions
    - Order: all cleanups → all setups (not interleaved)
    - Cleanups run in the same order as the components in the tree
```

## 7. commitDeletion: Recursive Cleanup

```jsx
function Parent() {
  return (
    <div ref={parentRef}>
      <Child />
    </div>
  );
}

function Child() {
  useEffect(() => {
    return () => console.log('child effect cleanup');
  });
  useLayoutEffect(() => {
    return () => console.log('child layout cleanup');
  });
  return <span ref={childRef}>text</span>;
}

// When Parent unmounts:
```

```
commitDeletion walks the ENTIRE subtree being removed:

  commitDeletionEffectsOnFiber(Parent):
    │
    ├── Process Parent fiber:
    │   → safelyDetachRef(parentRef)        ← detach parent ref
    │
    ├── Recurse into children:
    │   │
    │   ├── Process div (HostComponent):
    │   │   → Will be removed from DOM
    │   │
    │   ├── Process Child (FunctionComponent):
    │   │   → safelyCallDestroy: layout effect cleanup
    │   │     → "child layout cleanup"       ← synchronous!
    │   │   → Schedule passive effect cleanup
    │   │     → "child effect cleanup"        ← deferred
    │   │
    │   └── Process span (HostComponent):
    │       → safelyDetachRef(childRef)
    │
    └── Remove top-level DOM node:
        → parentRef.current.remove()
        → This removes div and all DOM children in one operation

  Order:
    1. Layout effect cleanups (synchronous, during commit)
    2. Ref detachments (synchronous, during commit)
    3. DOM removal (synchronous, single removeChild call)
    4. Passive effect cleanups (deferred, in flushPassiveEffects)

  The subtree walk is RECURSIVE through the fiber tree
  but only removes the TOP-LEVEL DOM node (the rest comes with it).
  However, it must visit every fiber to run cleanups and detach refs.
```

## 8. Tree Swap: The Atomic Switch

```
After all mutations and layout effects:

  BEFORE swap:
    fiberRoot.current → currentTree (what's on screen)
    workInProgress tree has all updates applied

  THE SWAP (single assignment):
    fiberRoot.current = finishedWork;  // ← THIS IS THE SWAP

  AFTER swap:
    fiberRoot.current → finishedWork (new current)
    old current tree becomes the "alternate" for next render

  Why this placement matters:
    - Swap happens AFTER mutation, BEFORE layout effects
    - So layout effects (componentDidMount/Update) see the NEW tree as current
    - If componentDidMount calls setState, React sees the correct current tree
    - Error boundaries in layout phase use the new tree for recovery

  The swap is a SINGLE POINTER ASSIGNMENT — atomic, no partial state.
  This is the moment the "double buffer" flips.
```

## 9. Effect Dependencies: Reference vs Value Equality

```jsx
function App() {
  const [count, setCount] = useState(0);
  const config = { theme: 'dark' };  // ← New object every render!

  useEffect(() => {
    console.log('effect runs');
  }, [config]);  // ← Runs EVERY render because config is a new object

  useEffect(() => {
    console.log('count effect');
  }, [count]);  // ← Only runs when count changes (number comparison)
}
```

```
React's dependency comparison (areHookInputsEqual):

  function areHookInputsEqual(nextDeps, prevDeps) {
    for (let i = 0; i < prevDeps.length; i++) {
      if (Object.is(nextDeps[i], prevDeps[i])) {
        continue;
      }
      return false;
    }
    return true;
  }

  Object.is comparison:
    Object.is(1, 1)           → true  (same number)
    Object.is('a', 'a')       → true  (same string)
    Object.is(obj, obj)       → true  (same reference)
    Object.is({}, {})         → false (different objects!)
    Object.is(NaN, NaN)       → true  (unlike ===)
    Object.is(+0, -0)         → false (unlike ===)

  Common dependency traps:
    [{ a: 1 }]        → New object each render, always re-runs
    [someArray]        → Same array reference? Depends on creation
    [[1,2,3]]          → New array literal, always re-runs
    [fn]               → New function each render (unless useCallback)
    [count]            → Primitive, compared by value ✓
    []                 → Empty deps, only runs on mount ✓
    undefined          → No deps, runs every render
```

## 10. Offscreen / Activity Commit Behavior (React 19)

```
When a component is hidden via the Activity API (formerly Offscreen):

  Visible → Hidden:
    1. React does NOT unmount the component
    2. React does NOT remove DOM nodes
    3. React DOES run passive effect cleanups (visibility effects)
    4. DOM is hidden via CSS (display: none) or detached
    5. State is PRESERVED in the fiber tree

  Hidden → Visible:
    1. React does NOT remount (no mount effects)
    2. React re-runs passive effect setups (visibility effects)
    3. DOM is shown again
    4. State is exactly as it was when hidden

  This is different from conditional rendering:
    {show && <Component />}
      → Unmounts: all effects cleaned up, state destroyed, DOM removed
      → Remounts: fresh state, mount effects, new DOM

    <Activity mode={show ? 'visible' : 'hidden'}>
      <Component />
    </Activity>
      → Hides: visibility cleanups only, state preserved, DOM hidden
      → Shows: visibility setups only, state restored, DOM shown

  Use case: tab-based UIs where hidden tabs should keep state
```
