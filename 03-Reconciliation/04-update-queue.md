# Update Queue: How State Updates Are Processed

## The Update Object

Every `setState` / `dispatch` call creates an **update object**:

```javascript
// Simplified from ReactFiberClassUpdateQueue.js

const update = {
  lane: lane,              // Priority lane for this update
  tag: UpdateState,        // UpdateState | ReplaceState | ForceUpdate | CaptureUpdate
  payload: null,           // The state value or updater function
  callback: null,          // Optional setState callback
  next: null,              // Circular linked list pointer
};
```

## Circular Linked List Structure

Updates are stored as a **circular linked list** on the fiber's `updateQueue`:

```javascript
const updateQueue = {
  baseState: currentState,   // State before any pending updates
  firstBaseUpdate: null,     // Head of partially-processed updates (from skipped lanes)
  lastBaseUpdate: null,      // Tail of above
  shared: {
    pending: null,           // Circular list of NEW updates (from setState calls)
    lanes: NoLanes,
  },
  callbacks: null,           // setState callbacks to fire after commit
};
```

```
Three rapid setState calls:

setState(1) → update1
setState(2) → update2
setState(3) → update3

shared.pending (circular linked list):

  ┌──────────────────────────────────────┐
  │                                      │
  ▼                                      │
update1 ──next──► update2 ──next──► update3
                                     ▲
                                     │
                         shared.pending (points to LAST)
```

Why circular? So React can find both the first AND last update in O(1):
- `shared.pending` = last update
- `shared.pending.next` = first update

## Processing Updates: processUpdateQueue

```javascript
function processUpdateQueue(workInProgress, renderLanes) {
  const queue = workInProgress.updateQueue;

  // Move pending updates to the base queue
  let pendingQueue = queue.shared.pending;
  if (pendingQueue !== null) {
    queue.shared.pending = null;
    // Break circular list and append to baseUpdate list
    const lastPendingUpdate = pendingQueue;
    const firstPendingUpdate = lastPendingUpdate.next;
    lastPendingUpdate.next = null;
    // Append to end of baseUpdate list
    if (queue.lastBaseUpdate === null) {
      queue.firstBaseUpdate = firstPendingUpdate;
    } else {
      queue.lastBaseUpdate.next = firstPendingUpdate;
    }
    queue.lastBaseUpdate = lastPendingUpdate;
  }

  // Now process the base queue
  let newState = queue.baseState;
  let update = queue.firstBaseUpdate;
  let newBaseState = null;
  let newFirstBaseUpdate = null;
  let newLastBaseUpdate = null;

  while (update !== null) {
    const updateLane = update.lane;

    if (!isSubsetOfLanes(renderLanes, updateLane)) {
      // ═══ SKIP THIS UPDATE ═══
      // Its lane is not included in current render lanes
      // Clone it into the new base queue for later processing
      const clone = cloneUpdate(update);
      if (newLastBaseUpdate === null) {
        newFirstBaseUpdate = newLastBaseUpdate = clone;
        newBaseState = newState;  // Remember state before skip
      } else {
        newLastBaseUpdate.next = clone;
        newLastBaseUpdate = clone;
      }
    } else {
      // ═══ PROCESS THIS UPDATE ═══
      // Also clone any subsequent updates to preserve order
      if (newLastBaseUpdate !== null) {
        const clone = cloneUpdate(update);
        newLastBaseUpdate.next = clone;
        newLastBaseUpdate = clone;
      }

      // Calculate new state
      newState = getStateFromUpdate(update, newState);

      // Collect callbacks
      if (update.callback !== null) {
        workInProgress.flags |= Callback;
      }
    }
    update = update.next;
  }

  workInProgress.memoizedState = newState;
  workInProgress.updateQueue.baseState = newBaseState ?? newState;
  workInProgress.updateQueue.firstBaseUpdate = newFirstBaseUpdate;
  workInProgress.updateQueue.lastBaseUpdate = newLastBaseUpdate;
}
```

## State Computation

```javascript
function getStateFromUpdate(update, prevState) {
  switch (update.tag) {
    case UpdateState: {
      const payload = update.payload;
      if (typeof payload === 'function') {
        // setState(prevState => newState)
        return payload(prevState);
      }
      // setState(newStateObject)
      return Object.assign({}, prevState, payload);
    }
    case ReplaceState:
      return update.payload;
    case ForceUpdate:
      return prevState;  // State unchanged, but force re-render
  }
}
```

## Lane-Based Update Skipping

This is one of the most subtle parts. When an update's lane isn't being
rendered, it's SKIPPED but preserved for later:

```
Scenario: 3 updates with different priorities

Update A: lane = SyncLane      (high priority)    payload: count + 1
Update B: lane = TransitionLane (low priority)    payload: count * 10
Update C: lane = SyncLane      (high priority)    payload: count + 2

Current state: { count: 0 }

═══ Render at SyncLane only ═══

Process A: count = 0 + 1 = 1     ✓ (SyncLane matches)
Process B: SKIP                   ✗ (TransitionLane not rendering)
           newBaseState = 1       (snapshot state before skip)
Process C: count = 1 + 2 = 3     ✓ (SyncLane matches)
           But CLONE C into base  (after a skip, all subsequent
                                   updates go into base queue
                                   to preserve ordering)

Result for this render:
  memoizedState = { count: 3 }
  baseState = { count: 1 }          ◄── state before first skip
  baseQueue = [B(clone), C(clone)]   ◄── will replay later

═══ Later: Render at TransitionLane ═══

Start from baseState = { count: 1 }
Process B: count = 1 * 10 = 10   ✓
Process C: count = 10 + 2 = 12   ✓ (replayed!)

Final state: { count: 12 }

This ensures state consistency:  A → B → C always produces the same
result regardless of how updates are batched across renders.
```

## Visual: Full Update Lifecycle

```
              setState({ count: 1 })
                      │
                      ▼
           ┌─────────────────────┐
           │  Create Update      │
           │  { lane, payload,   │
           │    callback, next } │
           └────────┬────────────┘
                    │
                    ▼
           ┌─────────────────────┐
           │  Enqueue on fiber's │
           │  updateQueue        │
           │  (circular list)    │
           └────────┬────────────┘
                    │
                    ▼
           ┌─────────────────────┐
           │  Schedule render    │
           │  at update's lane   │
           │  (via scheduler)    │
           └────────┬────────────┘
                    │
                    ▼
           ┌─────────────────────┐
           │  During beginWork:  │
           │  processUpdateQueue │
           │  → compute new      │
           │    memoizedState    │
           └────────┬────────────┘
                    │
                    ▼
           ┌─────────────────────┐
           │  Component renders  │
           │  with new state     │
           │  → new elements     │
           │  → reconcile        │
           └─────────────────────┘
```
