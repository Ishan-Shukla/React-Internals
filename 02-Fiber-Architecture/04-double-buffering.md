# Double Buffering: Current and Work-In-Progress Trees

## The Concept

React maintains **two copies** of the fiber tree at any time, inspired by
double buffering in graphics programming:

1. **Current tree** — reflects what's currently on screen
2. **Work-in-progress (WIP) tree** — being built/updated during render

```
FiberRoot
├── current ──────► Current Tree (on screen)
│                     App
│                      │
│                     div
│                    /   \
│                  H1     P
│
└── (during render)
    workInProgress ──► WIP Tree (being built)
                        App*
                         │
                        div*
                       /   \
                     H1     P*  ◄── updated
```

## The alternate Pointer

Every fiber has an `alternate` field pointing to its counterpart in the
other tree:

```
  CURRENT TREE                              WIP TREE

  App (current)  ◄──── alternate ────►  App (wip)
    │                                     │
  div (current)  ◄──── alternate ────►  div (wip)
    │                                     │
   H1 (current)  ◄──── alternate ────►  H1 (wip)
```

```javascript
currentFiber.alternate === wipFiber;   // true
wipFiber.alternate === currentFiber;   // true
```

## How Double Buffering Works

### Phase 1: Render (Building WIP Tree)

```
 CURRENT (on screen)              WIP (being built)
┌──────────────────┐            ┌──────────────────┐
│ App              │ alternate  │ App              │
│  count: 0       ◄──────────►│  count: 1  ◄── updated!
│                  │            │                  │
│ div              │ alternate  │ div              │
│                 ◄──────────►│                  │
│                  │            │                  │
│ H1               │ alternate  │ H1               │
│  "Count: 0"    ◄──────────►│  "Count: 1" ◄── updated!
└──────────────────┘            └──────────────────┘
                                         ▲
                                    being built
                                  (not visible yet)
```

### Phase 2: Commit (Swap Trees)

```
After commit, just swap the pointer:

  FiberRoot.current = WIP tree;  // One pointer swap!

 OLD CURRENT (now alternate)     NEW CURRENT (was WIP)
┌──────────────────┐            ┌──────────────────┐
│ App              │            │ App              │
│  count: 0       │◄──────────►│  count: 1        │ ◄── NOW ON SCREEN
│                  │            │                  │
│ div              │            │ div              │
│                  │◄──────────►│                  │
│                  │            │                  │
│ H1               │            │ H1               │
│  "Count: 0"     │◄──────────►│  "Count: 1"      │ ◄── NOW ON SCREEN
└──────────────────┘            └──────────────────┘
    will become WIP               is now current
    on next render
```

## Fiber Reuse: createWorkInProgress

When React starts a new render, it doesn't create fibers from scratch.
It reuses fibers from the previous alternate tree:

```javascript
// Simplified from ReactFiber.js

function createWorkInProgress(current, pendingProps) {
  let workInProgress = current.alternate;

  if (workInProgress === null) {
    // First render: create a brand new fiber
    workInProgress = createFiber(
      current.tag,
      pendingProps,
      current.key,
      current.mode,
    );
    workInProgress.type = current.type;
    workInProgress.stateNode = current.stateNode;

    // Link them together
    workInProgress.alternate = current;
    current.alternate = workInProgress;
  } else {
    // Subsequent render: REUSE the existing alternate fiber
    workInProgress.pendingProps = pendingProps;
    workInProgress.type = current.type;

    // Reset effects
    workInProgress.flags = NoFlags;
    workInProgress.subtreeFlags = NoFlags;
    workInProgress.deletions = null;
  }

  // Copy fields from current
  workInProgress.childLanes = current.childLanes;
  workInProgress.lanes = current.lanes;
  workInProgress.child = current.child;
  workInProgress.memoizedProps = current.memoizedProps;
  workInProgress.memoizedState = current.memoizedState;
  workInProgress.updateQueue = current.updateQueue;

  return workInProgress;
}
```

## Why Double Buffering?

### 1. Atomic Screen Updates

The WIP tree is invisible to the user. Only when ALL changes are computed
does React swap trees and apply DOM mutations. The user never sees a
half-rendered state.

### 2. Abort Without Damage

If React needs to discard work (e.g., a higher-priority update arrives),
it simply throws away the WIP tree. The current tree is untouched:

```
Higher priority update arrives mid-render:

Current:  App (count: 0)     ← still on screen, untouched
WIP:      App (count: 1)     ← DISCARDED (thrown away)
New WIP:  App (count: 2)     ← start fresh with latest state
```

### 3. Memory Efficiency

Instead of allocating new fibers every render, React alternates between
two sets of fiber objects. The "old" tree becomes the "scratch pad" for
the next render:

```
Render 1:  Tree A = current,  Tree B = WIP       (build B)
Commit 1:  Tree B = current,  Tree A = alternate
Render 2:  Tree B = current,  Tree A = WIP       (reuse A)
Commit 2:  Tree A = current,  Tree B = alternate
Render 3:  Tree A = current,  Tree B = WIP       (reuse B)
...and so on, ping-ponging between the two trees
```

## FiberRoot vs HostRoot

```
                    ┌─────────────────┐
                    │   FiberRoot     │ ◄── THE root object
                    │                 │     Created by createRoot()
                    │ .current ────────────► HostRoot Fiber (tag: 3)
                    │ .containerInfo ──────► <div id="root"> (DOM)
                    │ .pendingLanes   │
                    │ .finishedWork   │     (WIP root after render)
                    │ .callbackNode   │     (scheduler task)
                    └─────────────────┘
                                              │
                                            child
                                              │
                                              ▼
                                         App Fiber

FiberRoot: A plain object (NOT a fiber). One per createRoot() call.
HostRoot:  A fiber node (tag: 3). The topmost fiber in the tree.
           FiberRoot.current points to the current HostRoot fiber.
```
