# useState and useReducer Internals

## useState Is Just useReducer

Internally, `useState` is implemented as `useReducer` with a built-in reducer:

```javascript
function mountState(initialState) {
  if (typeof initialState === 'function') {
    initialState = initialState();  // Lazy initialization
  }
  // useState delegates to useReducer with a basic reducer
  return mountReducer(basicStateReducer, initialState);
}

function basicStateReducer(state, action) {
  return typeof action === 'function' ? action(state) : action;
}

// setState(5)              → basicStateReducer(prev, 5)       → 5
// setState(prev => prev+1) → basicStateReducer(prev, fn)      → fn(prev)
```

## mountReducer (First Render)

```javascript
function mountReducer(reducer, initialArg, init) {
  const hook = mountWorkInProgressHook();

  // Compute initial state
  const initialState = init !== undefined ? init(initialArg) : initialArg;

  hook.memoizedState = initialState;
  hook.baseState = initialState;

  // Create the update queue
  const queue = {
    pending: null,         // Circular linked list of dispatched updates
    lanes: NoLanes,
    dispatch: null,        // The dispatch/setState function
    lastRenderedReducer: reducer,
    lastRenderedState: initialState,
  };
  hook.queue = queue;

  // Create the dispatch function (stable across renders)
  const dispatch = (queue.dispatch = dispatchReducerAction.bind(
    null,
    currentlyRenderingFiber,
    queue,
  ));

  return [hook.memoizedState, dispatch];
}
```

## The dispatch Function (setState / dispatch)

This runs **outside** of rendering — when user clicks a button, etc.

```javascript
function dispatchReducerAction(fiber, queue, action) {
  // 1. Determine priority lane
  const lane = requestUpdateLane(fiber);

  // 2. Create update object
  const update = {
    lane: lane,
    revertLane: NoLane,
    action: action,        // The value or function passed to setState
    hasEagerState: false,
    eagerState: null,
    next: null,
  };

  if (isRenderPhaseUpdate(fiber)) {
    // setState during render → queue differently
    enqueueRenderPhaseUpdate(queue, update);
  } else {
    // Normal path: enqueue and schedule
    const root = enqueueConcurrentHookUpdate(fiber, queue, update, lane);

    if (root !== null) {
      // ═══ EAGER STATE OPTIMIZATION ═══
      // If the component isn't currently rendering, try to compute
      // new state eagerly. If it's the same as current → bail out!
      const currentState = queue.lastRenderedState;
      const eagerState = queue.lastRenderedReducer(currentState, action);
      update.hasEagerState = true;
      update.eagerState = eagerState;

      if (Object.is(eagerState, currentState)) {
        // State didn't change → don't schedule a render!
        return;
      }

      // State changed → schedule render
      scheduleUpdateOnFiber(root, fiber, lane);
    }
  }
}
```

### Eager State Optimization

```
User clicks a button that calls setCount(5):

1. Current state: 5
2. Action: 5
3. Eager computation: basicStateReducer(5, 5) = 5
4. Object.is(5, 5) → true
5. BAIL OUT — no render scheduled!

This is why:
  const [count, setCount] = useState(5);
  setCount(5);  // Calling with same value → no re-render
```

## updateReducer (Re-renders)

```javascript
function updateReducer(reducer, initialArg, init) {
  const hook = updateWorkInProgressHook();
  const queue = hook.queue;

  queue.lastRenderedReducer = reducer;

  const current = currentHook;
  let baseQueue = current.baseQueue;

  // Move pending updates to base queue (same pattern as class updateQueue)
  const pendingQueue = queue.pending;
  if (pendingQueue !== null) {
    if (baseQueue !== null) {
      // Merge pending into base
      const baseFirst = baseQueue.next;
      const pendingFirst = pendingQueue.next;
      baseQueue.next = pendingFirst;
      pendingQueue.next = baseFirst;
    }
    current.baseQueue = baseQueue = pendingQueue;
    queue.pending = null;
  }

  if (baseQueue !== null) {
    const first = baseQueue.next;
    let newState = current.baseState;
    let newBaseState = null;
    let newBaseQueueFirst = null;
    let newBaseQueueLast = null;
    let update = first;

    do {
      const updateLane = update.lane;

      if (!isSubsetOfLanes(renderLanes, updateLane)) {
        // ═══ SKIP (wrong lane) ═══
        // Clone into new base queue
        const clone = { ...update, next: null };
        if (newBaseQueueLast === null) {
          newBaseQueueFirst = newBaseQueueLast = clone;
          newBaseState = newState;
        } else {
          newBaseQueueLast = newBaseQueueLast.next = clone;
        }
      } else {
        // ═══ PROCESS ═══
        if (newBaseQueueLast !== null) {
          // After a skip, clone all subsequent updates
          const clone = { ...update, lane: NoLane, next: null };
          newBaseQueueLast = newBaseQueueLast.next = clone;
        }

        // Compute new state
        const action = update.action;
        if (update.hasEagerState) {
          newState = update.eagerState;  // Use pre-computed state
        } else {
          newState = reducer(newState, action);
        }
      }
      update = update.next;
    } while (update !== null && update !== first);

    // Check if state actually changed
    if (!Object.is(newState, hook.memoizedState)) {
      markWorkInProgressReceivedUpdate();  // Set didReceiveUpdate = true
    }

    hook.memoizedState = newState;
    hook.baseState = newBaseState ?? newState;
    hook.baseQueue = newBaseQueueLast;
    queue.lastRenderedState = newState;
  }

  return [hook.memoizedState, queue.dispatch];
}
```

## Batching: Multiple setState Calls

```javascript
function handleClick() {
  setCount(c => c + 1);   // Creates update1, enqueues
  setName('Bob');          // Creates update2, enqueues
  setAge(25);              // Creates update3, enqueues
  // React 18: ALL batched into a single render
  // Only ONE scheduleUpdateOnFiber happens (or is deduplicated)
}
```

```
handleClick() runs synchronously:

setCount(c => c + 1)
  → create update { action: c => c+1, lane: SyncLane }
  → enqueue on count hook's queue
  → scheduleUpdateOnFiber(root, fiber, SyncLane)
    → ensureRootIsScheduled(root)
      → scheduleMicrotask(performSyncWorkOnRoot)  ◄── scheduled

setName('Bob')
  → create update { action: 'Bob', lane: SyncLane }
  → enqueue on name hook's queue
  → scheduleUpdateOnFiber(root, fiber, SyncLane)
    → ensureRootIsScheduled(root)
      → existing callback same priority → REUSE (noop)

setAge(25)
  → create update { action: 25, lane: SyncLane }
  → enqueue on age hook's queue
  → scheduleUpdateOnFiber(root, fiber, SyncLane)
    → ensureRootIsScheduled(root)
      → existing callback same priority → REUSE (noop)

handleClick() returns → call stack empty → microtask fires
  → performSyncWorkOnRoot
  → ONE render that processes ALL three updates
  → ONE commit
```

## Visual: useState Lifecycle

```
                    ┌──────────────────────┐
                    │    First Render       │
                    │                      │
                    │  useState(0)         │
                    │    │                 │
                    │    ▼                 │
                    │  mountState(0)       │
                    │    │                 │
                    │    ▼                 │
                    │  Create hook:        │
                    │  { memoizedState: 0, │
                    │    queue: {...},     │
                    │    next: null }      │
                    │    │                 │
                    │    ▼                 │
                    │  return [0, dispatch]│
                    └──────────┬───────────┘
                               │
                    User clicks button
                               │
                    ┌──────────▼───────────┐
                    │  dispatch(1)         │
                    │    │                 │
                    │    ▼                 │
                    │  Create update:      │
                    │  { action: 1,        │
                    │    lane: SyncLane }  │
                    │    │                 │
                    │    ▼                 │
                    │  Enqueue update      │
                    │  Schedule render     │
                    └──────────┬───────────┘
                               │
                    ┌──────────▼───────────┐
                    │    Re-Render          │
                    │                      │
                    │  useState(0)         │
                    │  (initialState       │
                    │   ignored!)          │
                    │    │                 │
                    │    ▼                 │
                    │  updateReducer()     │
                    │    │                 │
                    │    ▼                 │
                    │  Process update:     │
                    │  reducer(0, 1) = 1   │
                    │    │                 │
                    │    ▼                 │
                    │  return [1, dispatch]│
                    │  (same dispatch fn)  │
                    └──────────────────────┘
```
