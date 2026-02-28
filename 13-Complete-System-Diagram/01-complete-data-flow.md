# Complete Data Flow: From setState to Pixels

## The Full Journey of a State Update

This traces EVERY step from when you call `setState` to when the browser
paints the new pixels on screen.

```
USER ACTION (e.g., click a button)
│
▼
═══════════════════════════════════════════════════════════════════════
PHASE 0: EVENT DISPATCH
═══════════════════════════════════════════════════════════════════════
│
├── Native click event fires on <div id="root">
│   (React's delegated listener catches it)
│
├── dispatchDiscreteEvent()
│   └── Sets current update priority = DiscreteEventPriority
│
├── Find target fiber from event.target DOM node
│   (via node.__reactFiber$ back-pointer)
│
├── Collect React event listeners walking fiber tree upward
│   (onClickCapture handlers → unshift, onClick handlers → push)
│
├── Create SyntheticEvent wrapper
│
├── Execute listeners in order (capture down, bubble up)
│   └── Your onClick handler runs:  setCount(prev => prev + 1)
│
▼
═══════════════════════════════════════════════════════════════════════
PHASE 1: UPDATE CREATION AND SCHEDULING
═══════════════════════════════════════════════════════════════════════
│
├── dispatchReducerAction(fiber, queue, action)
│
├── requestUpdateLane(fiber)
│   ├── Inside startTransition? → TransitionLane
│   └── Event priority → SyncLane (click)
│
├── Create update object:
│   { lane: SyncLane, action: prev => prev + 1, next: null }
│
├── Enqueue update on hook's queue (circular linked list)
│   queue.pending → update ─┐
│                            └── (circular)
│
├── ★ EAGER STATE CHECK ★
│   ├── Compute: reducer(currentState, action) = newState
│   ├── Object.is(newState, currentState)?
│   │   ├── YES → return (no render needed!)
│   │   └── NO  → continue...
│
├── scheduleUpdateOnFiber(root, fiber, SyncLane)
│   ├── Mark fiber.lanes |= SyncLane
│   ├── Walk UP to root, marking childLanes on each ancestor
│   │   fiber.return.childLanes |= SyncLane
│   │   fiber.return.return.childLanes |= SyncLane
│   │   ...until root
│   └── root.pendingLanes |= SyncLane
│
├── ensureRootIsScheduled(root)
│   ├── getNextLanes(root) → SyncLane
│   ├── SyncLane? → scheduleMicrotask(performSyncWorkOnRoot)
│   │   (For concurrent: scheduleCallback(priority, performConcurrentWorkOnRoot))
│   └── root.callbackNode = scheduled task
│
├── Event handler returns
│   (may have more setState calls → batched into same microtask)
│
▼
═══════════════════════════════════════════════════════════════════════
PHASE 2: RENDER (The "Reconciliation" or "Diffing" phase)
═══════════════════════════════════════════════════════════════════════
│
├── performSyncWorkOnRoot(root)
│   (or performConcurrentWorkOnRoot)
│
├── prepareFreshStack(root, lanes)
│   ├── Create WIP tree root from current tree root
│   │   workInProgress = createWorkInProgress(root.current)
│   └── Reset render-phase globals
│
├── renderRootSync(root, lanes)  [or renderRootConcurrent]
│
├── workLoopSync():              [or workLoopConcurrent with shouldYield]
│   while (workInProgress !== null) {
│     performUnitOfWork(workInProgress);
│   }
│
│   For EACH fiber node (depth-first traversal):
│   ┌──────────────────────────────────────────────────────────────┐
│   │                                                              │
│   │  ★ beginWork(current, wip, renderLanes) ★                    │
│   │  │                                                           │
│   │  ├── BAILOUT CHECK:                                          │
│   │  │   oldProps === newProps && !contextChanged && !updateLanes│
│   │  │   → bailoutOnAlreadyFinishedWork → skip subtree           │
│   │  │                                                           │
│   │  ├── switch (wip.tag):                                       │
│   │  │                                                           │
│   │  │   FunctionComponent:                                      │
│   │  │   ├── Set hooks dispatcher (mount vs update)              │
│   │  │   ├── ★ Call Component(props) ★                           │
│   │  │   │   ├── useState() → process update queue               │
│   │  │   │   │   reducer(0, prev => prev+1) → 1                  │
│   │  │   │   ├── useEffect() → push effect to fiber              │
│   │  │   │   ├── useMemo() → check deps, maybe recompute         │
│   │  │   │   └── Returns new JSX elements                        │
│   │  │   ├── Set ContextOnlyDispatcher (prevent hooks outside)   │
│   │  │   └── reconcileChildren(newElements)                      │
│   │  │       ├── Diff new elements vs current child fibers       │
│   │  │       ├── Reuse matching fibers, create new, mark delete  │
│   │  │       └── Set flags: Placement, Update, ChildDeletion     │
│   │  │                                                           │
│   │  │   HostComponent ('div'):                                  │
│   │  │   └── reconcileChildren(props.children)                   │
│   │  │                                                           │
│   │  │   HostText ("hello"):                                     │
│   │  │   └── return null (leaf node)                             │
│   │  │                                                           │
│   │  └── return wip.child (first child fiber, or null)           │
│   │                                                              │
│   │  If null (leaf node or bailout):                             │
│   │                                                              │
│   │  ★ completeWork(current, wip, renderLanes) ★                 │
│   │  │                                                           │
│   │  ├── HostComponent (mount):                                  │
│   │  │   ├── document.createElement(type)                        │
│   │  │   ├── setInitialProperties(dom, type, props)              │
│   │  │   ├── appendAllChildren(dom, wip) ← build DOM subtree     │
│   │  │   └── wip.stateNode = dom                                 │
│   │  │                                                           │
│   │  ├── HostComponent (update):                                 │
│   │  │   ├── diffProperties(oldProps, newProps)                  │
│   │  │   ├── wip.updateQueue = ['className','new','style',{...}] │
│   │  │   └── wip.flags |= Update                                 │
│   │  │                                                           │
│   │  ├── bubbleProperties(wip)                                   │
│   │  │   wip.subtreeFlags |= child.flags | child.subtreeFlags    │
│   │  │                                                           │
│   │  └── Move to sibling or return to parent                     │
│   │                                                              │
│   └──────────────────────────────────────────────────────────────┘
│
├── Render complete → exitStatus = RootCompleted
│   (or RootInProgress if yielded in concurrent mode)
│
▼
═══════════════════════════════════════════════════════════════════════
PHASE 3: COMMIT (Synchronous, uninterruptible)
═══════════════════════════════════════════════════════════════════════
│
├── commitRoot(root, finishedWork, lanes)
│
├── Schedule passive effects (if any useEffect flags):
│   scheduleCallback(NormalPriority, flushPassiveEffects)
│
├── ── SUB-PHASE 3a: BEFORE MUTATION ──
│   │
│   └── getSnapshotBeforeUpdate() (class components)
│       Read DOM before it changes (scroll position, etc.)
│
├── ── SUB-PHASE 3b: MUTATION ──
│   │
│   ├── Process deletions:
│   │   ├── Remove DOM nodes (removeChild)
│   │   ├── componentWillUnmount (class)
│   │   ├── useLayoutEffect cleanup (function)
│   │   └── Detach refs
│   │
│   ├── Process placements:
│   │   ├── Find host parent DOM node
│   │   ├── Find host sibling DOM node (for insertBefore)
│   │   └── insertBefore / appendChild
│   │
│   ├── Process updates:
│   │   ├── Apply updatePayload to DOM
│   │   │   element.className = 'new-class'
│   │   │   element.style.color = 'red'
│   │   │   etc.
│   │   └── Update text content for HostText
│   │
│   └── useInsertionEffect runs (CSS-in-JS)
│
├── ★★★ TREE SWAP: root.current = finishedWork ★★★
│   (WIP tree is now the current tree)
│
├── ── SUB-PHASE 3c: LAYOUT ──
│   │
│   ├── useLayoutEffect:
│   │   ├── Run cleanup from previous render
│   │   └── Run setup function
│   │
│   ├── componentDidMount / componentDidUpdate
│   │
│   ├── Attach refs (ref.current = domNode)
│   │
│   └── setState callbacks (from class components)
│
├── requestPaint() → signal browser to repaint
│
▼
═══════════════════════════════════════════════════════════════════════
PHASE 4: BROWSER PAINT
═══════════════════════════════════════════════════════════════════════
│
├── Browser performs layout (recalculate positions)
├── Browser performs paint (rasterize pixels)
├── Browser composites layers
└── Pixels appear on screen!
│
▼
═══════════════════════════════════════════════════════════════════════
PHASE 5: PASSIVE EFFECTS (async, after paint)
═══════════════════════════════════════════════════════════════════════
│
├── flushPassiveEffects() (via scheduler callback)
│
├── Phase 1: Run ALL cleanup functions
│   (destroy from previous render, depth-first)
│
├── Phase 2: Run ALL setup functions
│   (create for this render, depth-first)
│
└── ensureRootIsScheduled(root)
    (check for any remaining pending work)
```

## Timing Diagram: Sync Update

```
Time (ms) →
0         5         10        15        20        25
├─────────┼─────────┼─────────┼─────────┼─────────┤

Event     Render Phase          Commit    Browser  Passive
Handler   (beginWork +          Phase     Paint    Effects
          completeWork)
███       ██████████████████    ████      ░░░      ▒▒▒

█ = JavaScript execution on main thread
░ = Browser layout/paint
▒ = Async effects (useEffect)

Detailed:
├─ onClick handler (setState)                    ── 1ms
├─ Microtask: performSyncWorkOnRoot              ── 0ms (scheduling)
├─ beginWork(App)                                ── 0.5ms
├─ beginWork(div)                                ── 0.2ms
├─ beginWork(Counter) → call Counter() function  ── 1ms
├─ beginWork(button) → reconcile text            ── 0.3ms
├─ completeWork(button) → diffProperties         ── 0.5ms
├─ completeWork(Counter)                         ── 0.1ms
├─ completeWork(div)                             ── 0.1ms
├─ completeWork(App)                             ── 0.1ms
├─ commitBeforeMutationEffects                   ── 0.1ms
├─ commitMutationEffects → DOM: textContent = 1  ── 0.5ms
├─ root.current = finishedWork                   ── 0ms
├─ commitLayoutEffects → useLayoutEffect         ── 0.3ms
├─ ──── yield to browser ────                    ── browser paints
├─ flushPassiveEffects → useEffect cleanup       ── 0.5ms
├─ flushPassiveEffects → useEffect setup         ── 0.5ms
└─ done
```

## Timing Diagram: Concurrent Update with Interruption

```
Time (ms) →
0    5    10   15   20   25   30   35   40   45   50
├────┼────┼────┼────┼────┼────┼────┼────┼────┼────┤

Transition Update:
     ████ ░░░ ████ ░░░ ████
     work yield work yield work ──── INTERRUPTED!
                                │
Sync Update (user click):       │
                                ████████ ████ ░░ ▒▒
                                render   commit P  PE

Transition Restart:
                                              ████ ░░░ ████ ████ ░░ ▒▒
                                              work yield work commit P PE

██ = React work    ░░ = Browser paint    ▒▒ = Passive effects

Legend:
  P = Browser paint
  PE = Passive effects (useEffect)

Detailed timeline:
t=0ms:   startTransition(() => setFilter("search"))
         → TransitionLane → scheduleCallback(NormalPriority, render)

t=1ms:   Concurrent render starts
         workLoopConcurrent:
           beginWork(App) ✓
           beginWork(FilteredList) ✓
           beginWork(Item1) ✓

t=5ms:   shouldYield() → YES
         Return continuation to scheduler
         Browser paints (smooth 60fps!)

t=8ms:   Resume transition render
         beginWork(Item2) ✓
         beginWork(Item3) ✓

t=10ms:  shouldYield() → YES, yield again

t=13ms:  Resume...

t=15ms:  ★ USER TYPES A CHARACTER
         → onChange → setState(newInput)
         → SyncLane (higher priority!)
         → ensureRootIsScheduled:
           cancel transition task
           schedule sync task

t=16ms:  SYNC RENDER (workLoopSync — no yielding)
         → Full render with new input value
         → Old transition WIP tree: DISCARDED

t=20ms:  Commit sync update
         → DOM updated with new character
         → Browser paints immediately

t=22ms:  ensureRootIsScheduled
         → Transition still pending!
         → Schedule new transition render

t=23ms:  New transition render starts FROM SCRATCH
         (with both new input AND filter state)

t=35ms:  Transition render completes
         → Commit: filter results appear
         → Browser paints final state
```
