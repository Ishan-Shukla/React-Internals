# Complete Architecture Map

## The Entire React System — One Diagram

```
╔══════════════════════════════════════════════════════════════════════════╗
║                         YOUR APPLICATION CODE                            ║
║                                                                          ║
║  function App() {                                                        ║
║    const [state, setState] = useState(init);                             ║
║    useEffect(() => { ... }, [dep]);                                      ║
║    return <div onClick={handler}>{state}</div>;                          ║
║  }                                                                       ║
║                                                                          ║
║  const root = createRoot(container);                                     ║
║  root.render(<App />);                                                   ║
╚═══════════════════════════════════╤══════════════════════════════════════╝
                                    │
                    ┌───────────────┴───────────────┐
                    ▼                               ▼
╔═══════════════════════════════╗ ╔═══════════════════════════════════════╗
║     REACT PACKAGE             ║ ║     REACT-DOM PACKAGE                 ║
║     (react)                   ║ ║     (react-dom)                       ║
║                               ║ ║                                       ║
║  ┌─────────────────────────┐  ║ ║  ┌─────────────────────────────────┐  ║
║  │ createElement / jsx()   │  ║ ║  │ createRoot()                    │  ║
║  │ → ReactElement objects  │  ║ ║  │ → FiberRoot + HostRoot fiber    │  ║
║  └─────────────────────────┘  ║ ║  │ → Attach event listeners        │  ║
║                               ║ ║  └─────────────────────────────────┘  ║
║  ┌─────────────────────────┐  ║ ║                                       ║
║  │ Hooks Dispatcher        │  ║ ║  ┌─────────────────────────────────┐  ║
║  │ (routes to reconciler)  │  ║ ║  │ Host Config                     │  ║
║  │ useState → dispatcher   │  ║ ║  │ createElement()                 │  ║
║  │ useEffect → dispatcher  │  ║ ║  │ appendChild()                   │  ║
║  └─────────────────────────┘  ║ ║  │ removeChild()                   │  ║
║                               ║ ║  │ setTextContent()                │  ║
║  ┌─────────────────────────┐  ║ ║  │ commitUpdate()                  │  ║
║  │ Special Types           │  ║ ║  └─────────────────────────────────┘  ║
║  │ memo, lazy, forwardRef  │  ║ ║                                       ║
║  │ createContext           │  ║ ║  ┌─────────────────────────────────┐  ║
║  │ Suspense, Fragment      │  ║ ║  │ Event System                    │  ║
║  └─────────────────────────┘  ║ ║  │ Delegation on root container    │  ║
║                               ║ ║  │ SyntheticEvent creation         │  ║
╚═══════════════════════════════╝ ║  │ Priority mapping                │  ║
                                  ║  │ Listener collection             │  ║
                                  ║  └─────────────────────────────────┘  ║
                                  ╚════════════════╤══════════════════════╝
                                                   │
                              ┌────────────────────┘
                              ▼
╔════════════════════════════════════════════════════════════════════════════╗
║                     REACT-RECONCILER PACKAGE                               ║
║                     (The Core Algorithm)                                   ║
║                                                                            ║
║  ┌─────────────────────────────────────────────────────────────────────┐   ║
║  │                        FIBER TREE                                   │   ║
║  │                                                                     │   ║
║  │   CURRENT TREE ◄──alternate──► WORK-IN-PROGRESS TREE                │   ║
║  │   (on screen)                   (being built)                       │   ║
║  │                                                                     │   ║
║  │   FiberRoot.current ──► HostRoot                                    │   ║
║  │                           │                                         │   ║
║  │                          App (tag:0, FunctionComponent)             │   ║
║  │                           │  memoizedState → hooks linked list      │   ║
║  │                           │  updateQueue → effects list             │   ║
║  │                          div (tag:5, HostComponent)                 │   ║
║  │                           │  stateNode → <div> DOM node             │   ║
║  │                           │  flags → Update | Ref                   │   ║
║  │                         text (tag:6, HostText)                      │   ║
║  │                           │  stateNode → Text DOM node              │   ║
║  └─────────────────────────────────────────────────────────────────────┘   ║
║                                                                            ║
║  ┌───────────────────────┐  ┌───────────────────────┐  ┌────────────────┐  ║
║  │    WORK LOOP          │  │    HOOKS SYSTEM       │  │   LANES        │  ║
║  │                       │  │                       │  │                │  ║
║  │ workLoopSync()        │  │ HooksDispatcherOnMount│  │ SyncLane       │  ║
║  │ workLoopConcurrent()  │  │ HooksDispatcherOnUpd  │  │ InputContLane  │  ║
║  │                       │  │                       │  │ DefaultLane    │  ║
║  │ ┌──────────────────┐  │  │ mountState()          │  │ TransitionLane │  ║
║  │ │ beginWork()      │  │  │ updateState()         │  │ RetryLane      │  ║
║  │ │ Process fiber    │  │  │ mountEffect()         │  │ IdleLane       │  ║
║  │ │ Call component   │  │  │ updateEffect()        │  │                │  ║
║  │ │ Reconcile kids   │  │  │ mountMemo()           │  │ getNextLanes() │  ║
║  │ └────────┬─────────┘  │  │ updateMemo()          │  │ markRootUpd()  │  ║
║  │          │            │  │                       │  │ mergeLanes()   │  ║
║  │ ┌────────▼────────┐   │  │ hook = {              │  └────────────────┘  ║
║  │ │ completeWork()  │   │  │   memoizedState,      │                      ║
║  │ │ Create DOM      │   │  │   baseState,          │  ┌────────────────┐  ║
║  │ │ Diff props      │   │  │   queue: { pending }, │  │ UPDATE QUEUE   │  ║
║  │ │ Bubble flags    │   │  │   next              } │  │                │  ║
║  │ └─────────────────┘   │  └───────────────────────┘  │ Circular list  │  ║
║  └───────────────────────┘                             │ of updates     │  ║
║                                                        │ per fiber      │  ║
║  ┌───────────────────────┐  ┌───────────────────────┐  │                │  ║
║  │   COMMIT PHASE        │  │   CHILD RECONCILER    │  │ processUpdate  │  ║
║  │                       │  │                       │  │  Queue()       │  ║
║  │ 1. Before Mutation    │  │ reconcileSingleElem() │  └────────────────┘  ║
║  │    getSnapshotBefore  │  │ reconcileChildArray() │                      ║
║  │                       │  │                       │  ┌────────────────┐  ║
║  │ 2. Mutation           │  │ Two-pass diffing:     │  │ SUSPENSE       │  ║
║  │    DOM insert/update  │  │ Pass 1: linear scan   │  │                │  ║
║  │    DOM delete         │  │ Pass 2: Map lookup    │  │ throwException │  ║
║  │                       │  │                       │  │ ping/retry     │  ║
║  │ 3. root.current swap  │  │ Key matching          │  │ fallback mgmt  │  ║
║  │                       │  │ Type matching         │  │ Offscreen      │  ║
║  │ 4. Layout             │  │ Placement flags       │  └────────────────┘  ║
║  │    useLayoutEffect    │  └───────────────────────┘                      ║
║  │    componentDidMount  │                                                 ║
║  │    Ref attachment     │  ┌───────────────────────┐                      ║
║  │                       │  │  ERROR HANDLING       │                      ║
║  │ 5. Passive (async)    │  │                       │                      ║
║  │    useEffect          │  │  throwException()     │                      ║
║  │                       │  │  Find error boundary  │                      ║
║  └───────────────────────┘  │  Unwind fiber tree    │                      ║
║                             │  Re-render boundary   │                      ║
║                             └───────────────────────┘                      ║
╚═══════════════════════════════════╤════════════════════════════════════════╝
                                    │
                                    ▼
╔═══════════════════════════════════════════════════════════════════════════╗
║                         SCHEDULER PACKAGE                                 ║
║                         (scheduler)                                       ║
║                                                                           ║
║  ┌─────────────────────────┐  ┌─────────────────────────┐                 ║
║  │   TASK QUEUE            │  │   TIMER QUEUE           │                 ║
║  │   (min-heap)            │  │   (min-heap)            │                 ║
║  │                         │  │                         │                 ║
║  │  sorted by expiration   │  │  sorted by start time   │                 ║
║  │                         │  │  (delayed tasks)        │                 ║
║  │  ┌───┐ ┌───┐ ┌───┐      │  │                         │                 ║
║  │  │ 1 │ │ 2 │ │ 3 │ ...  │  │  timer fires → move to  │                 ║
║  │  └───┘ └───┘ └───┘      │  │  task queue             │                 ║
║  └────────────┬────────────┘  └─────────────────────────┘                 ║
║               │                                                           ║
║  ┌────────────▼────────────┐  ┌──────────────────────────┐                ║
║  │   WORK LOOP             │  │  YIELD MECHANISM         │                ║
║  │                         │  │                          │                ║
║  │  while (task &&         │  │  shouldYieldToHost()     │                ║
║  │    !shouldYield()) {    │  │  │                       │                ║
║  │    task.callback();     │  │  ├─ elapsed < 5ms?       │                ║
║  │  }                      │  │  │  → false (keep going) │                ║
║  │                         │  │  └─ elapsed >= 5ms?      │                ║
║  │  continuation?          │  │     → true (yield!)      │                ║
║  │  → keep task alive      │  │                          │                ║
║  │  done?                  │  │  MessageChannel          │                ║
║  │  → pop from queue       │  │  → near-zero delay       │                ║
║  └─────────────────────────┘  │  → schedule next tick    │                ║
║                               └──────────────────────────┘                ║
║  Priority Levels:                                                         ║
║  ┌─────────────────────────────────────────────────────┐                  ║
║  │ Immediate    -1ms    (expired! run NOW)             │                  ║
║  │ UserBlocking 250ms   (click, input)                 │                  ║
║  │ Normal       5000ms  (default, transitions)         │                  ║
║  │ Low          10000ms (background work)              │                  ║
║  │ Idle         ~∞      (only when nothing else)       │                  ║
║  └─────────────────────────────────────────────────────┘                  ║
╚═══════════════════════════════════╤═══════════════════════════════════════╝
                                    │
                                    ▼
╔═══════════════════════════════════════════════════════════════════════════╗
║                         BROWSER                                           ║
║                                                                           ║
║  ┌──────────┐  ┌───────────┐  ┌────────────┐  ┌───────────┐               ║
║  │  Event   │  │  Layout   │  │   Paint    │  │ Composite │               ║
║  │  Loop    │  │  (reflow) │  │ (rasterize)│  │  (GPU)    │               ║
║  │          │  │           │  │            │  │           │               ║
║  │ Tasks    │  │ Calculate │  │ Convert    │  │ Layer     │               ║
║  │ Microtask│  │ positions │  │ to pixels  │  │ composit. │               ║
║  │ rAF      │  │ & sizes   │  │            │  │           │               ║
║  │ Paint    │  │           │  │            │  │           │               ║
║  └──────────┘  └───────────┘  └────────────┘  └───────────┘               ║
║                                                                           ║
║                    ┌──────────────────┐                                   ║
║                    │   DOM            │                                   ║
║                    │   The actual     │                                   ║
║                    │   document that  │                                   ║
║                    │   the user sees  │                                   ║
║                    └──────────────────┘                                   ║
╚═══════════════════════════════════════════════════════════════════════════╝
```

## Package Dependency Graph

```
                    ┌──────────┐
                    │  react   │
                    │ (public  │
                    │  API)    │
                    └────┬─────┘
                         │ provides element types,
                         │ hook dispatch routing,
                         │ shared types
                         │
              ┌──────────┴──────────────┐
              │                         │
              ▼                         ▼
    ┌──────────────────┐      ┌────────────────────┐
    │ react-reconciler │      │    react-dom       │
    │ (core algorithm) │◄─────│ (DOM host config)  │
    │                  │      │ (event system)     │
    │ • Fiber tree     │      │ • createElement    │
    │ • Work loop      │      │ • appendChild      │
    │ • Hooks impl     │      │ • diffProperties   │
    │ • Reconciliation │      │ • Event delegation │
    │ • Commit phases  │      └────────────────────┘
    │ • Lanes          │
    └────────┬─────────┘
             │ uses for time-slicing
             │ and task scheduling
             ▼
    ┌──────────────────┐      ┌────────────────────┐
    │   scheduler      │      │    shared          │
    │ (task queue,     │      │ (constants, types  │
    │  shouldYield,    │      │  feature flags,    │
    │  priorities)     │      │  utilities)        │
    └──────────────────┘      └────────────────────┘

Key: ──► means "depends on" / "uses"
```

## Connection Summary

| When this happens... | ...these internals are involved |
|---|---|
| JSX compiles | `jsx()` → `ReactElement` objects |
| `createRoot()` | `FiberRoot` + `HostRoot` fiber + event listeners on container |
| `root.render(<App />)` | Schedule sync render → `performSyncWorkOnRoot` |
| Component renders | `beginWork` → `renderWithHooks` → hooks dispatcher → `reconcileChildren` |
| `setState()` | Update object → enqueue → `scheduleUpdateOnFiber` → `ensureRootIsScheduled` |
| Props change | Parent re-renders → `beginWork` on child → bailout check → reconcile |
| List diffing | `reconcileChildrenArray` → two-pass (linear + Map) |
| DOM creation | `completeWork` → `document.createElement` → `appendAllChildren` |
| DOM update | `completeWork` → `diffProperties` → `updatePayload` |
| Commit | `commitMutationEffects` → DOM writes → tree swap → `commitLayoutEffects` |
| `useEffect` | Pushed as effect in render → runs async after paint in `flushPassiveEffects` |
| `useLayoutEffect` | Pushed as effect in render → runs sync in `commitLayoutEffects` |
| Click event | Root listener → find fiber → collect listeners → create SyntheticEvent → execute |
| `startTransition` | Sets transition flag → `requestUpdateLane` returns TransitionLane |
| Suspense | Component throws Promise → `throwException` → show fallback → ping on resolve |
| Error | Component throws Error → `throwException` → find boundary → unwind → re-render |
| Context change | Provider value changes → walk subtree → mark consumers → force re-render |
| `shouldYield()` | Scheduler checks elapsed time → if > 5ms → pause work loop → `MessageChannel` → resume |
