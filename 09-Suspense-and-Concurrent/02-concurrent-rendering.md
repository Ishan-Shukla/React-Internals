# Concurrent Rendering

## What Is Concurrent Rendering?

Concurrent rendering means React can **prepare multiple versions of the UI
at the same time** and choose which one to show. It's not parallelism (no
threads) — it's interleaved scheduling on the main thread.

```
Sync Rendering (React 17-):
  Update A: ████████████████████ commit
  Update B:                            ████████████████ commit
  (B waits for A to complete)

Concurrent Rendering (React 18+):
  Update A: ██████ yield ████ yield ██████ commit
  Update B:        ██████                   (higher priority, interleaves)
  (B can interrupt A!)
```

## Enabling Concurrent Mode

```javascript
// React 18: createRoot enables concurrent features
import { createRoot } from 'react-dom/client';
const root = createRoot(document.getElementById('root'));
root.render(<App />);

// Old API (legacy mode, no concurrent features):
ReactDOM.render(<App />, document.getElementById('root'));
```

The difference in the reconciler:

```javascript
// createRoot → sets mode on the root fiber
const root = createFiberRoot(container, ConcurrentRoot, ...);

// The mode propagates to all child fibers
fiber.mode = ConcurrentMode;

// This affects the work loop:
function renderRootConcurrent(root, lanes) {
  // ...
  workLoopConcurrent();   // Uses shouldYield()
}

function renderRootSync(root, lanes) {
  // ...
  workLoopSync();         // No yielding
}
```

## The Two Work Loops

```javascript
// SYNC: Cannot be interrupted
function workLoopSync() {
  while (workInProgress !== null) {
    performUnitOfWork(workInProgress);
  }
}

// CONCURRENT: Yields every ~5ms
function workLoopConcurrent() {
  while (workInProgress !== null && !shouldYield()) {
    performUnitOfWork(workInProgress);
  }
}
```

The ONLY difference is `!shouldYield()`. Everything else (beginWork,
completeWork, the fiber tree) is identical.

## startTransition

Marks updates as non-urgent. The UI stays responsive by:
1. Rendering the transition in the background
2. Keeping the old UI visible during rendering
3. Allowing interruption by urgent updates

```javascript
// packages/react/src/ReactStartTransition.js

export function startTransition(scope) {
  const prevTransition = ReactCurrentBatchConfig.transition;
  ReactCurrentBatchConfig.transition = {};

  try {
    scope();
    // Any setState/dispatch calls inside scope() will:
    // → requestUpdateLane() sees transition is active
    // → returns a TransitionLane instead of SyncLane
  } finally {
    ReactCurrentBatchConfig.transition = prevTransition;
  }
}
```

```javascript
// How requestUpdateLane picks the lane:

function requestUpdateLane(fiber) {
  if (ReactCurrentBatchConfig.transition !== null) {
    // Inside startTransition → assign a TransitionLane
    return claimNextTransitionLane();
  }

  // Outside transition → use event priority
  return getCurrentEventPriority();
  // click → SyncLane, effect → DefaultLane, etc.
}
```

## useTransition

```javascript
function mountTransition() {
  const [isPending, setPending] = mountState(false);

  const start = (callback) => {
    setPending(true);     // Sync update → shows pending state immediately

    startTransition(() => {
      setPending(false);  // Transition update → resolves when transition commits
      callback();         // Your transition updates
    });
  };

  return [isPending, start];
}
```

```
const [isPending, startTransition] = useTransition();

startTransition(() => {
  setSearchResults(expensiveFilter(query));
});

Timeline:
  t=0ms:   startTransition called
            setPending(true)  → SyncLane → immediate render
            → isPending = true (show spinner)

  t=1ms:   setPending(false) + setSearchResults() → TransitionLane
            → background render begins

  t=2ms:   Urgent update? → interrupts transition, handles it first

  t=50ms:  Transition render completes
            → commit: isPending = false, searchResults = new data
            → spinner disappears, results appear
```

## useDeferredValue

Returns a deferred version of a value that lags behind during transitions:

```javascript
function mountDeferredValue(value) {
  const hook = mountWorkInProgressHook();
  hook.memoizedState = value;
  return value;
}

function updateDeferredValue(value) {
  const hook = updateWorkInProgressHook();
  const prevValue = hook.memoizedState;

  if (Object.is(value, prevValue)) {
    return value;  // Same value → return immediately
  }

  // Value changed → check if we can defer
  if (isCurrentTreeHidden() || !includesOnlyNonUrgentLanes(renderLanes)) {
    // Currently rendering at urgent priority → return OLD value
    // Schedule a transition to update to new value
    hook.memoizedState = prevValue;

    // Mark for transition render
    markSkippedUpdateLanes(TransitionLane);

    return prevValue;
  }

  // Rendering at transition priority → use new value
  hook.memoizedState = value;
  return value;
}
```

```
const deferredQuery = useDeferredValue(query);

User types "abc":
  Render 1 (urgent):   query = "abc",  deferredQuery = "ab" (old!)
  Render 2 (transition): query = "abc", deferredQuery = "abc" (updated)

This lets you render the input immediately with the new value,
while expensive components using deferredQuery re-render in a transition.
```

## Interruption: How Higher Priority Wins

```javascript
// In performConcurrentWorkOnRoot:

function performConcurrentWorkOnRoot(root, didTimeout) {
  const lanes = getNextLanes(root, NoLanes);

  // Check if these are the same lanes we're already rendering
  const exitStatus = shouldTimeSlice
    ? renderRootConcurrent(root, lanes)
    : renderRootSync(root, lanes);

  if (exitStatus === RootInProgress) {
    // Rendering was interrupted OR yielded
    // Return continuation
    return performConcurrentWorkOnRoot.bind(null, root);
  }
  // ...
}

// When a higher priority update comes in:
// 1. scheduleUpdateOnFiber(root, fiber, SyncLane)
// 2. ensureRootIsScheduled sees existing task at lower priority
// 3. Cancels existing scheduler task
// 4. Schedules new task at higher priority
// 5. prepareFreshStack() — resets workInProgress to root
// 6. Starts render from scratch with new lanes
// 7. Old WIP tree is abandoned (garbage collected)
```

```
                Transition Render (building WIP tree)
                ████████ INTERRUPTED ████████
t=0:   beginWork(App)
t=1:   beginWork(ExpensiveList)
t=3:   beginWork(Item1)
t=5:   yield → browser handles events
t=6:   ★ User clicks button → SyncLane update!
       → cancelCallback(transitionTask)
       → schedule sync render
t=7:   Sync render starts (FRESH — new WIP tree)
       → workLoopSync: full render of click handler update
       → commit
t=10:  ensureRootIsScheduled
       → transition still pending
       → schedule new transition render
t=11:  New transition render starts FROM SCRATCH
       (old partially-built WIP tree is gone)
```
