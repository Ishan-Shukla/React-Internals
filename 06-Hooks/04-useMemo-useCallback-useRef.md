# useMemo, useCallback, and useRef Internals

## useMemo

### Mount

```javascript
function mountMemo(nextCreate, deps) {
  const hook = mountWorkInProgressHook();
  const nextDeps = deps === undefined ? null : deps;

  // ★ Call the factory function NOW ★
  const nextValue = nextCreate();

  // Store [value, deps] tuple
  hook.memoizedState = [nextValue, nextDeps];
  return nextValue;
}
```

### Update

```javascript
function updateMemo(nextCreate, deps) {
  const hook = updateWorkInProgressHook();
  const nextDeps = deps === undefined ? null : deps;
  const prevState = hook.memoizedState;  // [prevValue, prevDeps]

  if (nextDeps !== null) {
    const prevDeps = prevState[1];
    if (areHookInputsEqual(nextDeps, prevDeps)) {
      // ═══ DEPS UNCHANGED → return cached value ═══
      return prevState[0];
    }
  }

  // ═══ DEPS CHANGED → recompute ═══
  const nextValue = nextCreate();
  hook.memoizedState = [nextValue, nextDeps];
  return nextValue;
}
```

```
hook.memoizedState for useMemo:

  [computedValue, [dep1, dep2, dep3]]
   ▲               ▲
   │               └── deps snapshot for comparison
   └── the cached value

useMemo(() => expensiveSort(items), [items])

Render 1: items = [3,1,2]  → compute → [1,2,3], deps = [[3,1,2]]
Render 2: items = [3,1,2]  → same ref → return cached [1,2,3]
Render 3: items = [5,4]    → diff ref → recompute → [4,5], deps = [[5,4]]
```

## useCallback

useCallback is literally useMemo for functions:

### Mount

```javascript
function mountCallback(callback, deps) {
  const hook = mountWorkInProgressHook();
  const nextDeps = deps === undefined ? null : deps;

  // Store [callback, deps] — note: does NOT call the callback!
  hook.memoizedState = [callback, nextDeps];
  return callback;
}
```

### Update

```javascript
function updateCallback(callback, deps) {
  const hook = updateWorkInProgressHook();
  const nextDeps = deps === undefined ? null : deps;
  const prevState = hook.memoizedState;

  if (nextDeps !== null) {
    const prevDeps = prevState[1];
    if (areHookInputsEqual(nextDeps, prevDeps)) {
      // Return PREVIOUS callback (same reference)
      return prevState[0];
    }
  }

  // Deps changed → store and return NEW callback
  hook.memoizedState = [callback, nextDeps];
  return callback;
}
```

### The Equivalence

```javascript
// These are functionally identical:
useCallback(fn, deps)
useMemo(() => fn, deps)

// The only difference: useMemo CALLS the factory, useCallback stores it directly
// useMemo:     memoizedState = [factory(), deps]
// useCallback: memoizedState = [callback, deps]
```

## useRef

useRef is the simplest hook — it creates a mutable container that persists
across renders with NO dependency tracking and NO re-renders on change.

### Mount

```javascript
function mountRef(initialValue) {
  const hook = mountWorkInProgressHook();
  const ref = { current: initialValue };

  if (__DEV__) {
    Object.seal(ref);  // Prevent adding new properties
  }

  hook.memoizedState = ref;
  return ref;
}
```

### Update

```javascript
function updateRef(initialValue) {
  const hook = updateWorkInProgressHook();
  return hook.memoizedState;  // Return the SAME object (same reference!)
}
```

That's it. No dep comparison. No recomputation. Just return the same object.

```
hook.memoizedState for useRef:

  { current: value }
  ▲
  └── This is the SAME object across all renders
      Mutating .current does NOT trigger re-renders
      React never reads or writes .current itself

Render 1: const ref = useRef(0)    → ref = { current: 0 }
Render 2: const ref = useRef(0)    → ref = { current: 0 } (same object!)
          ref.current = 5           → { current: 5 } (mutation, no re-render)
Render 3: const ref = useRef(0)    → ref = { current: 5 } (still same object)
```

## Comparison Table

```
                 useMemo              useCallback          useRef
                 ───────              ───────────          ──────
Stores:          computed value       callback function    { current: val }
Recomputes:      when deps change     when deps change     NEVER
Returns:         cached value         cached function      same object always
Triggers render: no                   no                   no
Memoized as:     [value, deps]        [fn, deps]           { current }
Mount behavior:  calls factory fn     stores fn            creates {current}
Update behavior: compares deps        compares deps        returns same obj
```

## Why useRef Doesn't Cause Re-renders

```
useState:
  dispatch(newValue)
    → creates update object
    → enqueues on fiber
    → scheduleUpdateOnFiber()  ← THIS triggers re-render
    → re-render processes update queue
    → returns new state

useRef:
  ref.current = newValue
    → plain JavaScript property assignment
    → React doesn't know about it
    → no update object created
    → no scheduleUpdateOnFiber
    → NO re-render
```

## useContext: Reading Context

```javascript
function readContext(context) {
  // Get the current value from the nearest Provider
  const value = context._currentValue;

  // Track this fiber as a context consumer
  // (so React can find it when context changes)
  const contextItem = {
    context: context,
    memoizedValue: value,
    next: null,
  };

  // Add to the fiber's dependencies list
  if (lastContextDependency === null) {
    currentlyRenderingFiber.dependencies = {
      lanes: NoLanes,
      firstContext: contextItem,
    };
  } else {
    lastContextDependency.next = contextItem;
  }
  lastContextDependency = contextItem;

  return value;
}
```

Note: `useContext` doesn't create a hook node in the linked list!
It reads directly from the context and registers a dependency.
This is why `useContext` CAN be called conditionally (though React
still discourages it for consistency).

## useId: Generating Stable IDs

```javascript
function mountId() {
  const hook = mountWorkInProgressHook();

  const root = getWorkInProgressRoot();
  const identifierPrefix = root.identifierPrefix;  // from createRoot options

  // Generate ID from the component's position in the tree
  const treeId = getTreeId();  // Based on fiber index path

  const id = ':' + identifierPrefix + 'r' + treeId + ':';
  hook.memoizedState = id;
  return id;
}

function updateId() {
  const hook = updateWorkInProgressHook();
  return hook.memoizedState;  // Same ID across renders
}
```

The ID is derived from the fiber's position in the tree, which is
deterministic. This means the same ID is generated during SSR and
hydration, ensuring server and client IDs match.
