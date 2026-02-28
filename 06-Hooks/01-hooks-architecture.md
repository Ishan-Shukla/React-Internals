# Hooks Architecture

## The Core Idea

Hooks are not magic. They're implemented as a **linked list of hook objects**
stored on the fiber's `memoizedState` field. Each hook call (useState,
useEffect, etc.) corresponds to one node in this list.

```
function MyComponent() {
  const [count, setCount] = useState(0);       // Hook 1
  const [name, setName] = useState('Alice');   // Hook 2
  useEffect(() => { ... }, [count]);           // Hook 3
  const doubled = useMemo(() => count * 2, [count]); // Hook 4
  return <div>{count} {name} {doubled}</div>;
}
```

```
fiber.memoizedState (head of linked list):

  Hook1 (useState)    Hook2 (useState)    Hook3 (useEffect)   Hook4 (useMemo)
  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐    ┌──────────────┐
  │memoizedState:│    │memoizedState:│    │memoizedState:│    │memoizedState:│
  │  0           │───►│  "Alice"     │───►│  {create,    │───►│  2           │
  │              │next│              │next│   destroy,   │next│              │
  │queue: {...}  │    │queue: {...}  │    │   deps}      │    │queue: null   │
  │baseState: 0  │    │baseState:    │    │              │    │              │
  │              │    │  "Alice"     │    │              │    │              │
  └──────────────┘    └──────────────┘    └──────────────┘    └──────────────┘
```

## Why Hooks Must Be Called in the Same Order

React doesn't identify hooks by name. It identifies them by **position**
in the linked list. The Nth hook call always accesses the Nth node:

```
Render 1:                           Render 2 (same order):
  useState(0)    → Hook[0] ✓         useState(0)    → Hook[0] ✓
  useState('')   → Hook[1] ✓         useState('')   → Hook[1] ✓
  useEffect(fn)  → Hook[2] ✓         useEffect(fn)  → Hook[2] ✓

Render 2 (WRONG — conditional hook):
  // if (condition) useState(0)  ← SKIPPED
  useState('')   → Hook[0] ✗ ← Gets useState(0)'s state! WRONG!
  useEffect(fn)  → Hook[1] ✗ ← Gets useState('')'s data! WRONG!
```

## The Dispatcher Pattern

React swaps the hook implementations depending on the phase:

```javascript
// packages/react/src/ReactHooks.js

export function useState(initialState) {
  const dispatcher = resolveDispatcher();
  return dispatcher.useState(initialState);
}

export function useEffect(create, deps) {
  const dispatcher = resolveDispatcher();
  return dispatcher.useEffect(create, deps);
}

function resolveDispatcher() {
  // This is set by the reconciler before calling your component
  return ReactCurrentDispatcher.current;
}
```

The reconciler sets different dispatchers:

```javascript
// packages/react-reconciler/src/ReactFiberHooks.js

// Before calling component function:
if (current === null || current.memoizedState === null) {
  // MOUNT — first render
  ReactCurrentDispatcher.current = HooksDispatcherOnMount;
} else {
  // UPDATE — re-render
  ReactCurrentDispatcher.current = HooksDispatcherOnUpdate;
}

// After component returns:
ReactCurrentDispatcher.current = ContextOnlyDispatcher;
//                               ^^^^^^^^^^^^^^^^^^
//                               Throws error if hooks called outside component
```

## Mount vs Update Dispatchers

```javascript
const HooksDispatcherOnMount = {
  useState: mountState,
  useEffect: mountEffect,
  useReducer: mountReducer,
  useMemo: mountMemo,
  useCallback: mountCallback,
  useRef: mountRef,
  useLayoutEffect: mountLayoutEffect,
  useContext: readContext,
  // ...
};

const HooksDispatcherOnUpdate = {
  useState: updateState,
  useEffect: updateEffect,
  useReducer: updateReducer,
  useMemo: updateMemo,
  useCallback: updateCallback,
  useRef: updateRef,
  useLayoutEffect: updateLayoutEffect,
  useContext: readContext,
  // ...
};

const ContextOnlyDispatcher = {
  useState: throwInvalidHookError,  // "Hooks can only be called inside..."
  useEffect: throwInvalidHookError,
  // all hooks → throw
};
```

## The Hook Object

```javascript
const hook = {
  memoizedState: null,   // The hook's current value
                         //   useState: the state value
                         //   useEffect: the effect object
                         //   useMemo: [memoizedValue, deps]
                         //   useRef: { current: value }
                         //   useCallback: [callback, deps]

  baseState: null,       // Base state before pending updates
  baseQueue: null,       // First update that was skipped (lane-based)
  queue: null,           // Queue of pending updates (dispatch calls)
  next: null,            // Next hook in the linked list
};
```

## renderWithHooks: The Entry Point

```javascript
// Simplified from ReactFiberHooks.js

function renderWithHooks(current, workInProgress, Component, props, renderLanes) {
  // Store current fiber for hooks to access
  currentlyRenderingFiber = workInProgress;

  // Reset hook state on the WIP fiber
  workInProgress.memoizedState = null;  // Will be rebuilt by hook calls
  workInProgress.updateQueue = null;

  // Set the right dispatcher
  ReactCurrentDispatcher.current =
    current === null || current.memoizedState === null
      ? HooksDispatcherOnMount
      : HooksDispatcherOnUpdate;

  // ★ CALL YOUR COMPONENT FUNCTION ★
  // All useState, useEffect, etc. calls happen here
  let children = Component(props);

  // Handle render-phase updates (setState during render)
  // React allows this but re-runs the component
  if (didScheduleRenderPhaseUpdate) {
    let numberOfReRenders = 0;
    do {
      didScheduleRenderPhaseUpdate = false;
      numberOfReRenders++;
      if (numberOfReRenders > 25) {
        throw new Error('Too many re-renders');
      }

      // Reset and re-render
      workInProgress.memoizedState = null;
      workInProgress.updateQueue = null;
      ReactCurrentDispatcher.current = HooksDispatcherOnRerender;

      children = Component(props);
    } while (didScheduleRenderPhaseUpdate);
  }

  // Prevent hook calls outside of render
  ReactCurrentDispatcher.current = ContextOnlyDispatcher;

  currentlyRenderingFiber = null;
  currentHook = null;        // Reset for next component
  workInProgressHook = null;

  return children;
}
```

## Hook Traversal During Update

```javascript
// During mount: create new hooks
function mountWorkInProgressHook() {
  const hook = {
    memoizedState: null,
    baseState: null,
    baseQueue: null,
    queue: null,
    next: null,
  };

  if (workInProgressHook === null) {
    // First hook in the component
    currentlyRenderingFiber.memoizedState = hook;
  } else {
    // Append to the list
    workInProgressHook.next = hook;
  }
  workInProgressHook = hook;
  return hook;
}

// During update: walk the existing hook list
function updateWorkInProgressHook() {
  // Get the next hook from the CURRENT fiber (previous render)
  let nextCurrentHook;
  if (currentHook === null) {
    const current = currentlyRenderingFiber.alternate;
    nextCurrentHook = current.memoizedState;  // First hook
  } else {
    nextCurrentHook = currentHook.next;  // Next hook in list
  }

  currentHook = nextCurrentHook;

  // Clone it for the WIP fiber
  const newHook = {
    memoizedState: currentHook.memoizedState,
    baseState: currentHook.baseState,
    baseQueue: currentHook.baseQueue,
    queue: currentHook.queue,
    next: null,
  };

  if (workInProgressHook === null) {
    currentlyRenderingFiber.memoizedState = newHook;
  } else {
    workInProgressHook.next = newHook;
  }
  workInProgressHook = newHook;
  return newHook;
}
```

## Complete Hook Lifecycle

```
                ┌─────────────────────────────────────┐
                │       Component Renders             │
                │                                     │
                │  1. Set dispatcher (mount/update)   │
                │  2. Call Component(props)           │
                │     │                               │
                │     ├─ useState() → mountState      │
                │     │   or updateState              │
                │     │   → returns [state, dispatch] │
                │     │                               │
                │     ├─ useEffect() → mountEffect    │
                │     │   or updateEffect             │
                │     │   → pushes effect to fiber    │
                │     │                               │
                │     ├─ useMemo() → mountMemo        │
                │     │   or updateMemo               │
                │     │   → returns memoized value    │
                │     │                               │
                │     └─ returns JSX elements         │
                │                                     │
                │  3. Reconcile children              │
                │  4. Set ContextOnlyDispatcher       │
                └─────────────────────────────────────┘
```
