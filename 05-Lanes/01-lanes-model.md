# Lanes: React's Priority Model

## What Are Lanes?

Lanes are React's **bitmask-based priority system**. Each update is assigned
a lane (a single bit), and React uses bitwise operations to efficiently
manage, merge, and compare priorities.

## Why Lanes Replaced Expiration Times

React originally used **expiration times** (a single number) for priority:

```
OLD (Expiration Times):
  Problem: Can't represent multiple priorities simultaneously
  update1: expirationTime = 5000
  update2: expirationTime = 3000
  How to batch them? Which ones to render? Hard to express.

NEW (Lanes):
  update1: lane = 0b0000000000000000000000000010000  (DefaultLane)
  update2: lane = 0b0000000000000000000001000000000  (TransitionLane1)
  Batch:   lanes = 0b0000000000000000000001000010000  (both!)
  Check:   lanes & DefaultLane !== 0  → yes, includes default
```

## Lane Definitions

```javascript
// packages/react-reconciler/src/ReactFiberLane.js

//                                           Bit position
export const NoLane          = 0b0000000000000000000000000000000;
export const SyncLane        = 0b0000000000000000000000000000010;  // 1
export const SyncBatchedLane = 0b0000000000000000000000000000100;  // 2

export const InputContinuousLane =
                               0b0000000000000000000000000001000;  // 3

export const DefaultLane     = 0b0000000000000000000000000010000;  // 4

// Transition lanes (16 lanes for concurrent transitions)
export const TransitionLane1 = 0b0000000000000000000000001000000;  // 6
export const TransitionLane2 = 0b0000000000000000000000010000000;  // 7
export const TransitionLane3 = 0b0000000000000000000000100000000;  // 8
export const TransitionLane4 = 0b0000000000000000000001000000000;  // 9
export const TransitionLane5 = 0b0000000000000000000010000000000;  // 10
export const TransitionLane6 = 0b0000000000000000000100000000000;  // 11
export const TransitionLane7 = 0b0000000000000000001000000000000;  // 12
export const TransitionLane8 = 0b0000000000000000010000000000000;  // 13
export const TransitionLane9 = 0b0000000000000000100000000000000;  // 14
export const TransitionLane10= 0b0000000000000001000000000000000;  // 15
export const TransitionLane11= 0b0000000000000010000000000000000;  // 16
export const TransitionLane12= 0b0000000000000100000000000000000;  // 17
export const TransitionLane13= 0b0000000000001000000000000000000;  // 18
export const TransitionLane14= 0b0000000000010000000000000000000;  // 19
export const TransitionLane15= 0b0000000000100000000000000000000;  // 20
export const TransitionLane16= 0b0000000001000000000000000000000;  // 21

// Retry lanes (for Suspense retries)
export const RetryLane1      = 0b0000000010000000000000000000000;  // 22
export const RetryLane2      = 0b0000000100000000000000000000000;  // 23
export const RetryLane3      = 0b0000001000000000000000000000000;  // 24
export const RetryLane4      = 0b0000010000000000000000000000000;  // 25

export const IdleLane        = 0b0100000000000000000000000000000;  // 29
export const OffscreenLane   = 0b1000000000000000000000000000000;  // 30

// Grouped lane sets
export const TransitionLanes = 0b0000000001111111111111111000000;
export const RetryLanes      = 0b0000011110000000000000000000000;
export const NonIdleLanes    = 0b0000011111111111111111111111111;
```

## Lane Priority Hierarchy

```
Highest Priority (leftmost = lowest bit)
│
│  SyncLane              ██  Discrete events (click, keydown)
│  SyncBatchedLane       ██  Batched sync work
│  InputContinuousLane   ██  Continuous events (drag, scroll, mousemove)
│  DefaultLane           ██  Normal updates (setState from useEffect)
│  TransitionLane1-16    ░░  startTransition() updates
│  RetryLane1-4          ░░  Suspense retries
│  IdleLane              ░░  requestIdleCallback-like work
│  OffscreenLane         ░░  Hidden/prerendered content
│
Lowest Priority
```

## Bitwise Operations on Lanes

```javascript
// Merge lanes (union)
const merged = laneA | laneB;
// 0b0010 | 0b1000 = 0b1010

// Check if a lane is included
const isIncluded = (lanes & lane) !== NoLanes;
// 0b1010 & 0b0010 = 0b0010 (truthy — included!)
// 0b1010 & 0b0100 = 0b0000 (falsy — not included)

// Remove a lane
const remaining = lanes & ~lane;
// 0b1010 & ~0b0010 = 0b1010 & 0b1101 = 0b1000

// Get highest priority (lowest bit)
function getHighestPriorityLane(lanes) {
  return lanes & -lanes;  // Isolate rightmost set bit
}
// 0b1010 & -0b1010 = 0b1010 & 0b0110 = 0b0010

// Check if subset
function isSubsetOfLanes(set, subset) {
  return (set & subset) === subset;
}
```

## How Updates Get Lanes

```javascript
// When you call setState, React assigns a lane based on context:

function requestUpdateLane(fiber) {
  // Inside startTransition?
  if (currentTransition !== null) {
    // Assign a transition lane (round-robin among 16 lanes)
    return claimNextTransitionLane();
  }

  // Get the current event priority
  const eventLane = getCurrentEventPriority();
  //   click/keydown     → SyncLane
  //   drag/scroll       → InputContinuousLane
  //   setTimeout/effect → DefaultLane

  return eventLane;
}
```

## Lanes on Fiber and Root

```javascript
// On each FIBER:
fiber.lanes      // Pending update lanes on THIS fiber
fiber.childLanes // Pending update lanes in THIS fiber's subtree

// On the ROOT:
root.pendingLanes     // All lanes with pending work
root.suspendedLanes   // Lanes suspended by Suspense
root.pingedLanes      // Suspended lanes that resolved
root.expiredLanes     // Lanes that have timed out (must render sync)
root.finishedLanes    // Lanes completed in the last render
```

## getNextLanes: Deciding What to Render

```javascript
function getNextLanes(root, wipLanes) {
  const pendingLanes = root.pendingLanes;
  if (pendingLanes === NoLanes) return NoLanes;

  let nextLanes = NoLanes;

  // Non-idle lanes take priority
  const nonIdlePendingLanes = pendingLanes & NonIdleLanes;

  if (nonIdlePendingLanes !== NoLanes) {
    // Filter out suspended lanes
    const nonIdleUnblockedLanes = nonIdlePendingLanes & ~root.suspendedLanes;

    if (nonIdleUnblockedLanes !== NoLanes) {
      nextLanes = getHighestPriorityLanes(nonIdleUnblockedLanes);
    } else {
      // All non-idle lanes are suspended, check pinged
      const nonIdlePingedLanes = nonIdlePendingLanes & root.pingedLanes;
      if (nonIdlePingedLanes !== NoLanes) {
        nextLanes = getHighestPriorityLanes(nonIdlePingedLanes);
      }
    }
  } else {
    // Only idle lanes remain
    const unblockedLanes = pendingLanes & ~root.suspendedLanes;
    if (unblockedLanes !== NoLanes) {
      nextLanes = getHighestPriorityLanes(unblockedLanes);
    }
  }

  return nextLanes;
}
```

```
Decision flow:

  pendingLanes = SyncLane | TransitionLane1 | IdleLane
                         │
                         ▼
              Non-idle pending? → Yes
                         │
                         ▼
              Filter suspended → SyncLane | TransitionLane1
                         │
                         ▼
              Highest priority → SyncLane
                         │
                         ▼
              getHighestPriorityLanes(SyncLane)
                         │
                         ▼
              renderLanes = SyncLane
              (TransitionLane1 and IdleLane wait)
```

## Multiple Transition Lanes: Why 16?

Each `startTransition` gets its own lane (round-robin):

```
startTransition(() => setA(1))  → TransitionLane1
startTransition(() => setB(2))  → TransitionLane2
startTransition(() => setC(3))  → TransitionLane3
...
startTransition(() => setQ(17)) → TransitionLane1 (wraps around)

This allows React to:
1. Render transitions independently
2. Commit one transition without waiting for others
3. Cancel a specific transition without affecting others
```

## Lane Expiration: Starvation Prevention

Low-priority lanes can be starved by continuous high-priority updates.
React prevents this by **expiring** lanes:

```javascript
function markStarvedLanesAsExpired(root, currentTime) {
  const pendingLanes = root.pendingLanes;
  const expirationTimes = root.expirationTimes;  // Array, one per lane

  let lanes = pendingLanes;
  while (lanes > 0) {
    const index = pickArbitraryLaneIndex(lanes);
    const lane = 1 << index;

    const expirationTime = expirationTimes[index];

    if (expirationTime === NoTimestamp) {
      // Lane just became pending → set expiration
      expirationTimes[index] = computeExpirationTime(lane, currentTime);
    } else if (expirationTime <= currentTime) {
      // Lane has EXPIRED → mark it
      root.expiredLanes |= lane;
      // Expired lanes are rendered synchronously (can't be interrupted)
    }

    lanes &= ~lane;  // Move to next lane
  }
}
```
