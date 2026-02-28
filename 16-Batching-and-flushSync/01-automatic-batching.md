# Automatic Batching and flushSync

## What Is Batching?

Batching means **grouping multiple state updates into a single re-render**.
Without batching, each `setState` would trigger a separate render cycle:

```
WITHOUT Batching (3 renders):          WITH Batching (1 render):
  setCount(1)  → render → commit        setCount(1)  → enqueue
  setName('A') → render → commit        setName('A') → enqueue
  setAge(25)   → render → commit        setAge(25)   → enqueue
                                         ─── all enqueued ───
  3 render cycles!                       → 1 render → 1 commit
  3 DOM updates!                         1 DOM update!
```

## React 17 vs React 18 Batching

### React 17: Only Batched Inside React Event Handlers

```javascript
// React 17 behavior:

function handleClick() {
  setCount(1);    // ┐
  setName('Bob'); // ├── BATCHED (inside React event handler)
  setAge(25);     // ┘  → 1 render

  setTimeout(() => {
    setCount(2);  // → render 1  ← NOT batched!
    setName('Al');// → render 2  ← NOT batched!
  }, 0);

  fetch('/api').then(() => {
    setCount(3);  // → render 1  ← NOT batched!
    setFlag(true);// → render 2  ← NOT batched!
  });
}
```

### React 18: Batched EVERYWHERE (Automatic Batching)

```javascript
// React 18 behavior (with createRoot):

function handleClick() {
  setCount(1);    // ┐
  setName('Bob'); // ├── BATCHED ✓
  setAge(25);     // ┘

  setTimeout(() => {
    setCount(2);  // ┐
    setName('Al');// ├── BATCHED ✓ (new in React 18!)
  }, 0);          // ┘

  fetch('/api').then(() => {
    setCount(3);  // ┐
    setFlag(true);// ├── BATCHED ✓ (new in React 18!)
  });             // ┘
}
```

## How Automatic Batching Works Internally

The key is `ensureRootIsScheduled` — it deduplicates render scheduling:

```javascript
function ensureRootIsScheduled(root) {
  const nextLanes = getNextLanes(root, NoLanes);
  const schedulerPriority = lanesToSchedulerPriority(nextLanes);

  const existingCallbackNode = root.callbackNode;

  if (existingCallbackNode !== null) {
    const existingPriority = root.callbackPriority;
    if (existingPriority === schedulerPriority) {
      // ═══ SAME PRIORITY → REUSE EXISTING TASK ═══
      // This is the batching mechanism!
      // Multiple setState at the same priority → single scheduled render
      return;
    }
    // Different priority → cancel and reschedule
    cancelCallback(existingCallbackNode);
  }

  // Schedule new render task
  const newCallbackNode = scheduleCallback(schedulerPriority, performWork);
  root.callbackNode = newCallbackNode;
  root.callbackPriority = schedulerPriority;
}
```

```
Batching mechanism — step by step:

setCount(1):
  → Create update, enqueue on count hook
  → scheduleUpdateOnFiber → ensureRootIsScheduled
  → No existing task → schedule microtask ★1
  → root.callbackNode = task1

setName('Bob'):
  → Create update, enqueue on name hook
  → scheduleUpdateOnFiber → ensureRootIsScheduled
  → Existing task at SAME priority → RETURN (noop!) ★2
  → root.callbackNode = task1 (unchanged)

setAge(25):
  → Create update, enqueue on age hook
  → scheduleUpdateOnFiber → ensureRootIsScheduled
  → Existing task at SAME priority → RETURN (noop!) ★3
  → root.callbackNode = task1 (unchanged)

★ handleClick() returns → call stack empty

Microtask fires → task1 runs:
  → performSyncWorkOnRoot
  → ONE render processes ALL THREE updates
  → count hook queue: [update(1)]
  → name hook queue: [update('Bob')]
  → age hook queue: [update(25)]
  → ONE commit → ONE DOM update
```

## Why React 18 Batches Everywhere

In React 17, synchronous setState calls outside React events executed
synchronously because React used a `isBatchingUpdates` flag:

```javascript
// React 17 (legacy):
let isBatchingUpdates = false;

function batchedUpdates(fn) {
  const prev = isBatchingUpdates;
  isBatchingUpdates = true;     // ← Set flag
  try {
    fn();                       // ← Your event handler
  } finally {
    isBatchingUpdates = prev;   // ← Unset flag
    if (!isBatchingUpdates) {
      flushSyncCallbackQueue(); // ← Flush immediately
    }
  }
}

// React wrapped event handlers in batchedUpdates()
// But setTimeout/Promise callbacks were NOT wrapped
// So setState in those → isBatchingUpdates is false → flush immediately
```

React 18 replaced this with a **scheduling-based** approach:

```javascript
// React 18:
// There's no isBatchingUpdates flag
// Instead, ALL updates are enqueued and scheduled via microtask/scheduler
// The render happens when the call stack is empty (microtask) or
// when the scheduler picks it up

// setState → enqueue → scheduleUpdateOnFiber → ensureRootIsScheduled
//   → scheduleMicrotask(performSyncWorkOnRoot)
//
// Microtask only fires AFTER current synchronous code completes
// So multiple setStates → multiple enqueues → but only ONE microtask render
```

## flushSync: The Batching Escape Hatch

`flushSync` forces React to flush updates synchronously, breaking out of batching:

```javascript
import { flushSync } from 'react-dom';

function handleClick() {
  flushSync(() => {
    setCount(1);  // → render + commit IMMEDIATELY
  });
  // DOM is already updated here!
  console.log(document.getElementById('count').textContent); // "1"

  flushSync(() => {
    setName('Bob');  // → render + commit IMMEDIATELY
  });
  // DOM updated again
}
```

### flushSync Internals

```javascript
// Simplified from ReactFiberWorkLoop.js

export function flushSync(fn) {
  const prevIsFlushingSyncWork = isFlushingSyncWork;
  isFlushingSyncWork = true;

  try {
    if (fn) {
      return fn();  // Execute your setState calls
    }
  } finally {
    isFlushingSyncWork = prevIsFlushingSyncWork;
    // ★ Force immediate flush ★
    // Don't wait for microtask — process NOW
    flushSyncWorkOnAllRoots();
  }
}

function flushSyncWorkOnAllRoots() {
  // Process ALL pending sync-priority work immediately
  for (const root of allRoots) {
    const lanes = getNextLanes(root, NoLanes);
    if (includesSyncLane(lanes)) {
      performSyncWorkOnRoot(root, lanes);
    }
  }
}
```

```
Normal batching:                        flushSync:

setCount(1)  → enqueue                 flushSync(() => {
setName('A') → enqueue                   setCount(1) → enqueue
─── microtask ───                       })
render both                             ─── flushSyncWorkOnAllRoots() ───
commit                                  render count
                                        commit count
                                        ← DOM updated HERE
                                        flushSync(() => {
                                          setName('A') → enqueue
                                        })
                                        ─── flushSyncWorkOnAllRoots() ───
                                        render name
                                        commit name
```

## When to Use flushSync

```javascript
// Use case 1: Read DOM immediately after state update
function handleAdd() {
  flushSync(() => {
    setItems([...items, newItem]);
  });
  // DOM has the new item → can scroll to it
  listRef.current.lastChild.scrollIntoView();
}

// Use case 2: Third-party library integration
function handleChange(value) {
  flushSync(() => {
    setState(value);
  });
  // DOM updated → library can read accurate DOM measurements
  thirdPartyLibrary.recalculate();
}
```

## Batching Across Different Contexts

```
Context                     React 17        React 18
──────────────────────      ─────────       ─────────
React event handler         Batched ✓       Batched ✓
setTimeout/setInterval      NOT batched     Batched ✓
Promise.then                NOT batched     Batched ✓
Native event listener       NOT batched     Batched ✓
requestAnimationFrame       NOT batched     Batched ✓
queueMicrotask              NOT batched     Batched ✓
MutationObserver            NOT batched     Batched ✓
flushSync                   N/A             Forces immediate flush
```

## Visual: Batching Timeline

```
Without batching (React 17, in setTimeout):

│ setTimeout callback │ render │ commit │ render │ commit │ render │ commit │
│ setState(a)         │   a    │  DOM   │   b    │  DOM   │   c    │  DOM   │
│ setState(b)         │        │ update │        │ update │        │ update │
│ setState(c)         │        │        │        │        │        │        │

With batching (React 18):

│ setTimeout callback │ microtask │ render │ commit │
│ setState(a)         │           │ a+b+c  │  DOM   │
│ setState(b)         │           │ (one   │ update │
│ setState(c)         │ fires ──► │ render)│ (once) │
│                     │           │        │        │

Time saved: ~2 render cycles + 2 DOM updates
```
