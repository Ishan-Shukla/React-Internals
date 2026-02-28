# How React Uses the Scheduler

## The Bridge: ensureRootIsScheduled

When React receives an update (setState, dispatch, etc.), it must schedule
a render. The function `ensureRootIsScheduled` is the bridge between
React's lane system and the scheduler:

```javascript
// Simplified from ReactFiberRootScheduler.js

function ensureRootIsScheduled(root) {
  // 1. Determine the highest priority lane with pending work
  const nextLanes = getNextLanes(root, NoLanes);

  if (nextLanes === NoLanes) {
    // No work to do → cancel any existing task
    if (root.callbackNode !== null) {
      cancelCallback(root.callbackNode);
      root.callbackNode = null;
    }
    return;
  }

  // 2. Convert lane priority to scheduler priority
  const schedulerPriority = lanesToSchedulerPriority(nextLanes);

  // 3. Check if we already have a scheduled task
  const existingCallbackNode = root.callbackNode;
  if (existingCallbackNode !== null) {
    const existingPriority = root.callbackPriority;
    if (existingPriority === schedulerPriority) {
      // Same priority → reuse existing task, don't schedule a new one
      return;
    }
    // Different priority → cancel old, schedule new
    cancelCallback(existingCallbackNode);
  }

  // 4. Schedule with the scheduler
  let newCallbackNode;

  if (includesSyncLane(nextLanes)) {
    // Sync work → execute immediately via microtask
    scheduleMicrotask(performSyncWorkOnRoot.bind(null, root));
    newCallbackNode = null;
  } else {
    // Concurrent work → schedule through scheduler
    newCallbackNode = scheduleCallback(
      schedulerPriority,
      performConcurrentWorkOnRoot.bind(null, root),
    );
  }

  root.callbackNode = newCallbackNode;
  root.callbackPriority = schedulerPriority;
}
```

## Lane-to-Scheduler Priority Mapping

```javascript
function lanesToSchedulerPriority(lanes) {
  const lane = getHighestPriorityLane(lanes);

  if (lane === SyncLane || lane === SyncBatchedLane) {
    return ImmediatePriority;          // -1ms timeout
  }
  if (lane === InputContinuousLane) {
    return UserBlockingPriority;       // 250ms timeout
  }
  if (lane === DefaultLane) {
    return NormalPriority;             // 5000ms timeout
  }
  if (lane & TransitionLanes) {
    return NormalPriority;             // 5000ms timeout
  }
  if (lane & IdleLane) {
    return IdlePriority;              // ~forever timeout
  }

  return NormalPriority;
}
```

```
Lane                    →  Scheduler Priority  →  Timeout
────────────────────       ──────────────────      ──────
SyncLane                   Immediate              -1ms (NOW)
InputContinuousLane        UserBlocking           250ms
DefaultLane                Normal                 5000ms
TransitionLanes            Normal                 5000ms
RetryLanes                 Normal                 5000ms
IdleLane                   Idle                   ~12 days
```

## Sync vs Concurrent Entry Points

```
                    ensureRootIsScheduled
                            │
              ┌─────────────┴─────────────┐
              ▼                           ▼
        Sync Lanes?                 Concurrent Lanes
              │                           │
              ▼                           ▼
    scheduleMicrotask(            scheduleCallback(
      performSyncWorkOnRoot)        schedulerPriority,
              │                     performConcurrentWorkOnRoot)
              │                           │
              ▼                           ▼
    ┌──────────────────┐         ┌──────────────────┐
    │ workLoopSync()   │         │workLoopConcurrent│
    │                  │         │                  │
    │ while (wip) {    │         │ while (wip &&    │
    │   performUnit()  │         │   !shouldYield())│
    │ }                │         │   performUnit()  │
    │                  │         │ }                │
    │ Cannot pause     │         │ CAN pause        │
    └──────────────────┘         └──────────────────┘
```

## Continuation: Resuming Paused Work

When `workLoopConcurrent` yields before completing, React returns a
**continuation** to the scheduler:

```javascript
function performConcurrentWorkOnRoot(root, didTimeout) {
  // Determine lanes for this render
  const lanes = getNextLanes(root, NoLanes);
  if (lanes === NoLanes) return null;

  // Should we use time slicing or render synchronously?
  const shouldTimeSlice =
    !didTimeout &&
    !includesBlockingLane(lanes) &&
    !includesExpiredLane(root, lanes);

  // ★ RENDER PHASE ★
  const exitStatus = shouldTimeSlice
    ? renderRootConcurrent(root, lanes)   // Can yield
    : renderRootSync(root, lanes);        // Runs to completion

  if (exitStatus === RootInProgress) {
    // Work was INTERRUPTED (shouldYield returned true)
    // Return a continuation — scheduler will call us again
    return performConcurrentWorkOnRoot.bind(null, root);
    //     ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
    //     This continuation keeps the task alive in the scheduler
  }

  if (exitStatus === RootCompleted) {
    // ★ COMMIT PHASE ★
    const finishedWork = root.finishedWork;
    commitRoot(root, finishedWork, lanes);
  }

  // Check if there's more work at different priority
  ensureRootIsScheduled(root);
  return null;  // No continuation — task is done
}
```

## Timeline: Concurrent Render with Yielding

```
Time →

Scheduler Task Queue: [React render (Normal priority)]

t=0ms:   MessageChannel fires → performWorkUntilDeadline
         → performConcurrentWorkOnRoot
         → renderRootConcurrent
         → workLoopConcurrent
           beginWork(App)     ✓  (0.5ms)
           beginWork(div)     ✓  (0.3ms)
           beginWork(Header)  ✓  (0.8ms)
           beginWork(Nav)     ✓  (0.4ms)

t=3ms:     shouldYield()? → No (< 5ms)
           beginWork(Main)    ✓  (1.2ms)
           beginWork(List)    ✓  (0.6ms)

t=5ms:     shouldYield()? → YES (≥ 5ms)
           workLoopConcurrent exits
           renderRootConcurrent returns RootInProgress
           performConcurrentWorkOnRoot returns continuation
           workLoop sees continuation → keeps task in queue

t=5ms:   port.postMessage(null)  → yield to browser

         Browser: handle pending events, layout, paint (~3ms)

t=8ms:   MessageChannel fires → performWorkUntilDeadline (resume)
         → continuation runs
         → renderRootConcurrent (continues from List's child)
         → workLoopConcurrent
           beginWork(Item1)   ✓
           beginWork(Item2)   ✓
           ...
           completeWork(App)  ✓

t=12ms:  Render complete → commitRoot (synchronous, uninterruptible)
         → DOM mutations
         → Layout effects
         → Schedule passive effects (useEffect)

t=13ms:  Browser paints updated UI
```

## Task Cancellation (Priority Interruption)

```
Scenario: User types during a low-priority transition render

t=0ms:  startTransition(() => setFilter(value))
        → Schedule: TransitionLane → NormalPriority

t=2ms:  workLoopConcurrent begins
        beginWork(App)
        beginWork(FilteredList)  ◄── expensive component

t=5ms:  Yield to browser

t=6ms:  User types a character → onChange → setState
        → Schedule: SyncLane → ImmediatePriority
        → ensureRootIsScheduled checks:
          existing task = NormalPriority
          new task = ImmediatePriority (higher!)
          → cancelCallback(existingTask)   ◄── CANCEL transition render!
          → Schedule new sync task

t=7ms:  Sync render runs (workLoopSync)
        → Full render cycle with new input value
        → Commit immediately
        → WIP tree from transition is DISCARDED

t=10ms: After sync render, ensureRootIsScheduled
        → Transition work still pending
        → Schedule new transition render
        → Starts fresh (not resuming old WIP)
```
