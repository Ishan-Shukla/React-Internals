# Architecture Evolution: Stack to Fiber

## The Old Stack Reconciler (React < 16)

Before React 16, reconciliation was **synchronous and recursive**. React would
walk the entire component tree in a single, uninterruptible call stack:

```
                    ┌───────────────────────────────────┐
                    │     STACK RECONCILER (old)        │
                    │                                   │
 setState() ──────► │  mountComponent(App)              │
                    │    ├─ mountComponent(Header)      │
                    │    │   └─ mountComponent(Logo)    │
                    │    ├─ mountComponent(Main)        │ ◄── Cannot pause!
                    │    │   ├─ mountComponent(List)    │     Blocks main thread
                    │    │   │   ├─ mountComponent(Item)│    until complete
                    │    │   │   ├─ mountComponent(Item)│
                    │    │   │   └─ mountComponent(Item)│
                    │    │   └─ mountComponent(Sidebar) │
                    │    └─ mountComponent(Footer)      │
                    │                                   │
                    │  ───► Commit all changes at once  │
                    └───────────────────────────────────┘
```

### Problems with Stack

1. **Blocked main thread**: A large tree update (e.g., 1000 list items) could
   freeze the UI for 100ms+. No user input, no animations, nothing.

2. **No prioritization**: A text input keystroke and a background data fetch
   had equal priority. Everything was FIFO.

3. **No interruption**: Once React started reconciling, it ran to completion.
   You couldn't cancel stale work or restart with fresher data.

4. **Synchronous only**: Every update was synchronous. No way to defer or
   batch work across frames.

```
Timeline: Stack Reconciler

Browser Frame (~16ms)
├────────────────────────────────────────────────────────┤
│  React reconcile (30ms)  ██████████████████████████████│█████████████████
│  User click event        ─────────────────────────────────── BLOCKED ───►
│  Animation frame         ─────────────────────────────────── BLOCKED ───►
│  Browser paint           ─────────────────────────────────── BLOCKED ───►
├────────────────────────────────────────────────────────┤
                           Frame dropped!                 Finally paints
```

## The Fiber Reconciler (React 16+)

The Fiber rewrite transformed reconciliation from a recursive call-stack walk
into an **iterative linked-list traversal** that can be paused and resumed.

```
                    ┌───────────────────────────────────────┐
                    │      FIBER RECONCILER (new)           │
                    │                                       │
 setState() ──────► │  workLoopConcurrent() {               │
                    │    while (workInProgress && !yield) { │
                    │      workInProgress =                 │
                    │        performUnitOfWork(wip);        │ ◄── Can pause
                    │    }                                  │     after each
                    │  }                                    │     unit of work!
                    │                                       │
                    │  Unit 1: beginWork(App)          ✓    │
                    │  Unit 2: beginWork(Header)       ✓    │
                    │  ── yield to browser ──               │ ◄── Browser paints
                    │  Unit 3: beginWork(Main)         ✓    │
                    │  Unit 4: beginWork(List)         ✓    │
                    │  ── yield to browser ──               │ ◄── Handle click
                    │  Unit 5: beginWork(Item)         ✓    │
                    │  ...                                  │
                    │                                       │
                    │  All units done → Commit              │
                    └───────────────────────────────────────┘
```

### Key Insight: From Call Stack to Linked List

The call stack is owned by the JavaScript engine — you can't pause it. But a
linked list is a data structure in **user space** that React fully controls:

```
CALL STACK (can't pause)          LINKED LIST (can pause anywhere)
─────────────────────             ──────────────────────────────
reconcile(App)                    App ──► Header ──► Logo
  reconcile(Header)                        │
    reconcile(Logo)                        ▼
  reconcile(Main)                 Main ──► List ──► Item ──► Item
    reconcile(List)                        │
      reconcile(Item)                      ▼            ▲
      reconcile(Item)                    Sidebar        │
  reconcile(Footer)                        │         PAUSE HERE,
                                           ▼         resume later!
                                         Footer
```

## What Fiber Enabled

| Capability | Stack | Fiber |
|-----------|-------|-------|
| Pause rendering | No | Yes |
| Abort stale work | No | Yes |
| Priority-based scheduling | No | Yes |
| Concurrent rendering | No | Yes |
| Suspense | No | Yes |
| Transitions | No | Yes |
| Time slicing | No | Yes |
| Incremental hydration | No | Yes |

## Timeline Comparison

```
Stack Reconciler:
Frame 1          Frame 2          Frame 3
├────────────────┼────────────────┼────────────────┤
│████████████████████████████████ │                │  ◄── 30ms reconcile
│                 BLOCKED         │  Paint         │      blocks everything
│                                 │                │

Fiber Reconciler:
Frame 1          Frame 2          Frame 3
├────────────────┼────────────────┼────────────────┤
│██████ Paint ░░░│██████ Paint ░░░│████ Commit     │
│ work  ▲   idle │ work  ▲   idle │work       Paint│
│       │        │       │        │                │
│    yield       │    yield       │                │
│                                                  │
└──── Same 30ms of work, spread across frames ─────┘

██ = React work    ░░ = idle time    ▲ = yield point
```

## The Fiber Contract

The fundamental contract of the Fiber architecture:

1. **Work is split into units** — each fiber node = one unit of work
2. **Each unit is fast** — typically < 1ms per fiber
3. **After each unit, React checks if it should yield** — via `shouldYield()`
4. **If time is up, React yields** — browser can paint, handle events
5. **React resumes where it left off** — the fiber linked list remembers position
6. **Higher-priority work can interrupt** — React discards stale WIP tree and restarts
