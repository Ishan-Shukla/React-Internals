# Scheduler: Cooperative Multitasking

## Why React Needs Its Own Scheduler

The browser's main thread handles everything: JavaScript, layout, paint, user
input. If React hogs the thread, the UI freezes. React's scheduler implements
**cooperative multitasking** — React voluntarily yields control back to the
browser between chunks of work.

```
WITHOUT Scheduler (blocks browser):
├─ React work (50ms)  ███████████████████████████████████████████████████│
├─ User click         ──────────── WAITING ──────────── finally handled  │
├─ Animation frame    ──────────── WAITING ──────────── finally runs     │
├─ Browser paint      ──────────── WAITING ──────────── finally paints   │

WITH Scheduler (yields every ~5ms):
├─ React ██│Paint│██│Click│██│Paint│██│██│Paint│██│Commit│Paint│
├─ User click      ↑ handled immediately (< 5ms delay)
├─ Animation     ↑ runs on time
├─ Paint       ↑ smooth 60fps
```

## Architecture

The scheduler is a **standalone package** (`packages/scheduler/`) with zero
React-specific knowledge. It manages a generic priority task queue:

```
┌──────────────────────────────────────────────────────┐
│                    SCHEDULER                         │
│                                                      │
│  ┌───────────────┐    ┌───────────────┐              │
│  │  Task Queue   │    │  Timer Queue  │              │
│  │  (min-heap)   │    │  (min-heap)   │              │
│  │               │    │               │              │
│  │  Ready tasks  │    │  Delayed tasks│              │
│  │  sorted by    │    │  sorted by    │              │
│  │  expiration   │    │  start time   │              │
│  └──────┬────────┘    └──────┬────────┘              │
│         │                    │                       │
│         │    timer fires     │                       │
│         │◄───────────────────┘                       │
│         │                                            │
│         ▼                                            │
│  ┌───────────────────────┐                           │
│  │  workLoop()           │                           │
│  │  Pick highest-priority│                           │
│  │  task, execute it     │                           │
│  │  Check shouldYield()  │──► Yes? → MessageChannel  │
│  │  after each task      │         post message to   │
│  └───────────────────────┘         schedule next tick│
└──────────────────────────────────────────────────────┘
```

## Priority Levels

```javascript
// packages/scheduler/src/SchedulerPriorities.js

export const ImmediatePriority    = 1;   // Timeout: -1ms (already expired!)
export const UserBlockingPriority = 2;   // Timeout: 250ms
export const NormalPriority       = 3;   // Timeout: 5000ms (5 seconds)
export const LowPriority          = 4;   // Timeout: 10000ms (10 seconds)
export const IdlePriority         = 5;   // Timeout: maxSigned31BitInt (~12 days)
```

Each priority has a **timeout**. The expiration time determines task ordering:

```
expirationTime = currentTime + timeout

Example at currentTime = 1000:
  ImmediatePriority:    1000 + (-1)   = 999   (already expired!)
  UserBlockingPriority: 1000 + 250    = 1250
  NormalPriority:       1000 + 5000   = 6000
  LowPriority:         1000 + 10000  = 11000
  IdlePriority:        1000 + MAX_INT = practically never expires

Tasks with LOWER expiration times run FIRST (min-heap).
```

## The Task Object

```javascript
const task = {
  id: taskIdCounter++,            // Unique ID for stable sorting
  callback: callback,             // The work function to execute
  priorityLevel: priority,        // One of the 5 priority levels
  startTime: startTime,           // When the task becomes ready
  expirationTime: expirationTime, // Deadline for the task
  sortIndex: -1,                  // Used by min-heap (= expirationTime or startTime)
};
```

## Scheduling a Task

```javascript
// packages/scheduler/src/Scheduler.js

function unstable_scheduleCallback(priorityLevel, callback, options) {
  const currentTime = getCurrentTime();  // performance.now()

  // Determine start time
  let startTime = currentTime;
  if (options?.delay > 0) {
    startTime = currentTime + options.delay;
  }

  // Calculate expiration from priority
  const timeout = priorityTimeouts[priorityLevel];
  const expirationTime = startTime + timeout;

  const newTask = {
    id: taskIdCounter++,
    callback,
    priorityLevel,
    startTime,
    expirationTime,
    sortIndex: -1,
  };

  if (startTime > currentTime) {
    // ═══ DELAYED TASK ═══
    // Not ready yet → put in timer queue
    newTask.sortIndex = startTime;
    push(timerQueue, newTask);

    // If this is the earliest delayed task, set a timeout
    if (peek(taskQueue) === null && newTask === peek(timerQueue)) {
      requestHostTimeout(handleTimeout, startTime - currentTime);
    }
  } else {
    // ═══ READY TASK ═══
    // Ready now → put in task queue
    newTask.sortIndex = expirationTime;
    push(taskQueue, newTask);

    // Request a callback to process the queue
    if (!isHostCallbackScheduled && !isPerformingWork) {
      isHostCallbackScheduled = true;
      requestHostCallback(flushWork);
    }
  }

  return newTask;
}
```

## The Work Loop

```javascript
function workLoop(initialTime) {
  let currentTime = initialTime;

  // Move ready timers to task queue
  advanceTimers(currentTime);

  currentTask = peek(taskQueue);

  while (currentTask !== null) {
    if (currentTask.expirationTime > currentTime && shouldYieldToHost()) {
      // Task hasn't expired AND we're out of time → yield
      break;
    }

    const callback = currentTask.callback;
    if (typeof callback === 'function') {
      currentTask.callback = null;
      currentPriorityLevel = currentTask.priorityLevel;

      const didUserCallbackTimeout =
        currentTask.expirationTime <= currentTime;

      // ★ EXECUTE THE TASK ★
      const continuationCallback = callback(didUserCallbackTimeout);

      if (typeof continuationCallback === 'function') {
        // Task returned a continuation → it's not done yet
        // Keep it in the queue with the same priority
        currentTask.callback = continuationCallback;
      } else {
        // Task is done → remove from queue
        if (currentTask === peek(taskQueue)) {
          pop(taskQueue);
        }
      }

      advanceTimers(getCurrentTime());
    } else {
      // Callback was cancelled → remove from queue
      pop(taskQueue);
    }

    currentTask = peek(taskQueue);
  }

  // Return whether there's more work
  return currentTask !== null;
}
```

## shouldYieldToHost: The Yield Decision

```javascript
function shouldYieldToHost() {
  const timeElapsed = getCurrentTime() - startTime;

  if (timeElapsed < frameInterval) {
    // frameInterval is ~5ms by default
    // We still have time in this frame
    return false;
  }

  // We've used our time slice → yield
  return true;
}
```

## MessageChannel: The Scheduling Mechanism

```javascript
// React uses MessageChannel to schedule work, NOT setTimeout

const channel = new MessageChannel();
const port = channel.port2;

channel.port1.onmessage = performWorkUntilDeadline;

function requestHostCallback(callback) {
  scheduledHostCallback = callback;
  port.postMessage(null);  // Schedule a macrotask
}

function performWorkUntilDeadline() {
  if (scheduledHostCallback !== null) {
    const currentTime = getCurrentTime();
    startTime = currentTime;

    const hasMoreWork = scheduledHostCallback(currentTime);

    if (hasMoreWork) {
      // More work to do → schedule another tick
      port.postMessage(null);
    } else {
      isHostCallbackScheduled = false;
      scheduledHostCallback = null;
    }
  }
}
```

**Why MessageChannel instead of setTimeout?**

```
setTimeout(fn, 0):
  ├─ Minimum 4ms delay (browser enforced after 5 nested calls)
  ├─ Clamped to 1ms in workers
  └─ Too slow for 5ms time slices

requestAnimationFrame:
  ├─ Fires before paint (~16ms intervals)
  ├─ Doesn't fire in background tabs
  └─ Wrong timing for React's needs

MessageChannel:
  ├─ Near-zero delay macrotask
  ├─ Fires AFTER microtasks (Promise.then)
  ├─ Fires BEFORE next paint
  └─ Perfect for yielding and resuming
```

## Min-Heap Implementation

```javascript
// packages/scheduler/src/SchedulerMinHeap.js
// Textbook min-heap using an array

export function push(heap, node) {
  const index = heap.length;
  heap.push(node);
  siftUp(heap, node, index);
}

export function peek(heap) {
  return heap.length === 0 ? null : heap[0];
}

export function pop(heap) {
  if (heap.length === 0) return null;
  const first = heap[0];
  const last = heap.pop();
  if (last !== first) {
    heap[0] = last;
    siftDown(heap, last, 0);
  }
  return first;
}

// Compare by sortIndex, then by id (insertion order) for stability
function compare(a, b) {
  const diff = a.sortIndex - b.sortIndex;
  return diff !== 0 ? diff : a.id - b.id;
}
```
