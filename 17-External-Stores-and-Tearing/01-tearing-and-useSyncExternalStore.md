# Tearing and useSyncExternalStore

## What Is Tearing?

Tearing occurs when **different parts of the UI show different values from
the same data source** during a single render. This is a visual inconsistency
that was impossible in synchronous rendering but becomes possible in
concurrent rendering.

```
SYNCHRONOUS rendering (no tearing possible):
  External store: value = "A"

  Render starts:
    Component1 reads store → "A"
    Component2 reads store → "A"
    Component3 reads store → "A"
  Commit: all show "A" ✓

CONCURRENT rendering (tearing possible):
  External store: value = "A"

  Render starts:
    Component1 reads store → "A"
    Component2 reads store → "A"
    ── React YIELDS to browser ──
    ── External code changes store to "B" ──
    ── React RESUMES ──
    Component3 reads store → "B"    ← DIFFERENT VALUE!
  Commit:
    Component1: "A"
    Component2: "A"   ← INCONSISTENT!
    Component3: "B"

  This is TEARING: the UI is "torn" between two versions of truth.
```

## Why Only External Stores Have This Problem

React's own state (`useState`, `useReducer`) is stored on **fibers** which
React fully controls. During a concurrent render, React reads from the
**work-in-progress fiber** which is frozen for that render:

```
React state (safe):
  useState value is on the fiber → React controls reads
  Even if dispatch is called during render, updates are queued
  The WIP tree always sees a consistent snapshot

External stores (unsafe):
  Redux store, Zustand, MobX, global variables, DOM state
  These live OUTSIDE React's fiber tree
  Any code can mutate them at any time
  React has no control over when they change
```

## The Problem in Detail

```javascript
// External store (e.g., Redux-like)
let currentValue = 'A';
const listeners = new Set();

const store = {
  getValue: () => currentValue,
  subscribe: (listener) => {
    listeners.add(listener);
    return () => listeners.delete(listener);
  },
};

// Some external code (websocket, timer, etc.)
setTimeout(() => {
  currentValue = 'B';         // ← Mutation happens between yields!
  listeners.forEach(l => l());
}, 5);

// Components reading the store
function Display1() {
  const value = store.getValue();  // Might read "A"
  return <div>{value}</div>;
}

function Display2() {
  const value = store.getValue();  // Might read "B" if React yielded!
  return <div>{value}</div>;
}
```

## useSyncExternalStore: The Solution

```javascript
import { useSyncExternalStore } from 'react';

function Display() {
  const value = useSyncExternalStore(
    store.subscribe,    // How to subscribe to changes
    store.getValue,     // How to get current value (client)
    store.getValue,     // How to get current value (server — optional)
  );
  return <div>{value}</div>;
}
```

## useSyncExternalStore Internals

### Mount

```javascript
function mountSyncExternalStore(subscribe, getSnapshot, getServerSnapshot) {
  const hook = mountWorkInProgressHook();
  const fiber = currentlyRenderingFiber;

  // Get the initial snapshot
  const nextSnapshot = getSnapshot();
  hook.memoizedState = nextSnapshot;

  // Store metadata for update checking
  const inst = {
    value: nextSnapshot,
    getSnapshot: getSnapshot,
  };
  hook.queue = inst;

  // Subscribe via useEffect (to trigger re-renders on change)
  mountEffect(subscribeToStore.bind(null, fiber, inst, subscribe), [subscribe]);

  // ★ KEY: Also push a LAYOUT effect to check for changes ★
  // This catches synchronous mutations between render and commit
  pushStoreConsistencyCheck(fiber, getSnapshot, nextSnapshot);

  return nextSnapshot;
}
```

### The Consistency Check (Anti-Tearing Mechanism)

```javascript
function pushStoreConsistencyCheck(fiber, getSnapshot, renderedSnapshot) {
  // Add a layout-phase check
  fiber.flags |= StoreConsistency;

  const check = {
    getSnapshot: getSnapshot,
    value: renderedSnapshot,
  };

  // This runs during commitLayoutEffects:
  // It calls getSnapshot() AGAIN and compares to what was rendered
  // If different → the store changed during render → FORCE SYNC RE-RENDER
}

// During commit (layout phase):
function commitStoreConsistencyCheck(fiber) {
  const checks = fiber.updateQueue.stores;

  for (let i = 0; i < checks.length; i++) {
    const check = checks[i];
    const currentValue = check.getSnapshot();

    if (!Object.is(currentValue, check.value)) {
      // ═══ TEARING DETECTED! ═══
      // Store value changed since we rendered
      // Force a SYNCHRONOUS re-render to fix it
      forceStoreRerender(fiber);
    }
  }
}
```

### Update

```javascript
function updateSyncExternalStore(subscribe, getSnapshot) {
  const hook = updateWorkInProgressHook();
  const prevSnapshot = hook.memoizedState;

  // Get current snapshot
  const nextSnapshot = getSnapshot();

  if (!Object.is(prevSnapshot, nextSnapshot)) {
    // Store value changed → update
    hook.memoizedState = nextSnapshot;
    markWorkInProgressReceivedUpdate();
  }

  // Re-push consistency check
  pushStoreConsistencyCheck(
    currentlyRenderingFiber, getSnapshot, nextSnapshot
  );

  // Re-subscribe if subscribe function changed
  updateEffect(
    subscribeToStore.bind(null, currentlyRenderingFiber, hook.queue, subscribe),
    [subscribe]
  );

  return nextSnapshot;
}
```

### Subscribe to Store

```javascript
function subscribeToStore(fiber, inst, subscribe) {
  const handleStoreChange = () => {
    // Check if snapshot actually changed
    if (checkIfSnapshotChanged(inst)) {
      // ★ Force a SYNC lane update ★
      // This ensures the re-render can't be interrupted
      // (preventing tearing on the re-render too)
      forceStoreRerender(fiber);
    }
  };

  // Subscribe and return unsubscribe for cleanup
  return subscribe(handleStoreChange);
}

function checkIfSnapshotChanged(inst) {
  try {
    const nextValue = inst.getSnapshot();
    const prevValue = inst.value;
    return !Object.is(prevValue, nextValue);
  } catch {
    return true;  // Error reading snapshot → assume changed
  }
}

function forceStoreRerender(fiber) {
  // Schedule at SyncLane — cannot be interrupted!
  scheduleUpdateOnFiber(fiber, SyncLane);
}
```

## How It Prevents Tearing

```
CONCURRENT render with useSyncExternalStore:

Step 1: Render starts
  Component1: useSyncExternalStore → snapshot = "A"
  Component2: useSyncExternalStore → snapshot = "A"

Step 2: React yields
  External code changes store: "A" → "B"
  Store notifies subscribers → handleStoreChange fires
  → forceStoreRerender at SyncLane

Step 3: React resumes... but wait!
  SyncLane update was scheduled (higher priority)
  → React ABORTS current concurrent render
  → Starts NEW sync render (uninterruptible)

Step 4: Sync render
  Component1: useSyncExternalStore → snapshot = "B"
  Component2: useSyncExternalStore → snapshot = "B"
  Component3: useSyncExternalStore → snapshot = "B"
  → All consistent! No tearing! ✓

Step 5: Commit
  → All components show "B" ✓
```

```
Even if the store changes DURING the sync render:

Step 1: Sync render (uninterruptible)
  Component1 reads "B"
  ── store changes to "C" ──
  Component2 reads "C" ← POTENTIAL TEAR!

Step 2: Layout phase — consistency check runs
  check.value ("B" from Component1 render) !== getSnapshot() ("C")
  → TEARING DETECTED
  → Force another sync re-render

Step 3: Second sync render
  Component1 reads "C"
  Component2 reads "C"
  → Consistent! ✓

This loop converges because sync renders are uninterruptible
and store changes are discrete events.
```

## Using with Popular Libraries

```javascript
// Redux
import { useSyncExternalStore } from 'react';
import { useSelector } from 'react-redux';
// react-redux v8+ uses useSyncExternalStore internally

// Zustand
import { create } from 'zustand';
const useStore = create((set) => ({ count: 0 }));
// Zustand uses useSyncExternalStore internally

// Custom store
function useMyStore(selector) {
  return useSyncExternalStore(
    myStore.subscribe,
    () => selector(myStore.getState()),
    () => selector(myStore.getInitialState()), // For SSR
  );
}
```

## getServerSnapshot: SSR Support

```javascript
const value = useSyncExternalStore(
  subscribe,
  getSnapshot,          // Client: read from actual store
  getServerSnapshot,    // Server: return initial/default value
);

// During SSR (react-dom/server):
// - There's no store to subscribe to
// - getServerSnapshot provides the initial value
// - Must return the SAME value on server and initial client render
//   (otherwise hydration mismatch)
```
