# MCQs: Fiber Architecture & Tree Traversal

---

### Q1. What is a Fiber in React?
- A) A lightweight thread for parallel rendering
- B) A JavaScript object representing a unit of work in the component tree
- C) A CSS optimization technique
- D) A type of Web Worker

**Answer: B**
A Fiber is a plain JavaScript object that represents a component, DOM element, or other React node. It holds state, props, effects, and tree pointers (child, sibling, return).

---

### Q2. Which THREE pointers form the fiber tree structure?
- A) `parent`, `firstChild`, `nextSibling`
- B) `child`, `sibling`, `return`
- C) `left`, `right`, `parent`
- D) `prev`, `next`, `root`

**Answer: B**
`child` points to the first child, `sibling` points to the next sibling, and `return` points to the parent fiber. This forms a linked-list tree structure.

---

### Q3. Why does React use a linked-list tree instead of a recursive call stack?
- A) Linked lists use less memory
- B) It enables pausing and resuming rendering (interruptible work)
- C) JavaScript doesn't support recursion
- D) Linked lists are faster for DOM operations

**Answer: B**
The recursive Stack Reconciler couldn't pause mid-traversal because state was on the JS call stack. Fibers store traversal state in objects, enabling pause/resume/restart.

---

### Q4. What is the `alternate` pointer on a Fiber node?
- A) A pointer to the fiber's error boundary
- B) A pointer to the corresponding fiber in the other tree (current ↔ WIP)
- C) A pointer to an alternative component to render
- D) A pointer to the previous version of props

**Answer: B**
`alternate` connects the current tree fiber to its work-in-progress (WIP) counterpart and vice versa. This is the core of React's double buffering technique.

---

### Q5. What is stored in `fiber.memoizedState` for a function component?
- A) The last rendered JSX
- B) The head of the hooks linked list
- C) The component's prop types
- D) The DOM node reference

**Answer: B**
For function components, `memoizedState` points to the first hook node in a linked list. Each hook is `{ memoizedState, queue, next }`. For class components, it holds the class state object.

---

### Q6. What is stored in `fiber.stateNode`?
- A) Always the DOM element
- B) The state object for the component
- C) For host components: DOM node; for class components: class instance; for HostRoot: FiberRoot
- D) The parent fiber's state

**Answer: C**
`stateNode` varies by fiber type: DOM element for host components, class instance for class components, `FiberRoot` for the root fiber.

---

### Q7. What does `fiber.flags` contain?
- A) Feature flags for the React build
- B) Bitmask of side effects (Placement, Update, Deletion, etc.)
- C) CSS class flags
- D) Error severity levels

**Answer: B**
`flags` is a bitmask indicating what side effects this fiber needs during commit: `Placement` (insert), `Update` (modify), `Deletion` (remove), `Ref`, `Snapshot`, etc.

---

### Q8. What is the `Placement` flag used for?
- A) Placing the component in the scheduler queue
- B) Indicating the fiber's DOM node needs to be inserted into the DOM
- C) Placing a context value on the stack
- D) Reserving a memory slot for the fiber

**Answer: B**
`Placement` means this fiber's host node needs to be inserted into the DOM during the commit phase's mutation sub-phase (via `commitPlacement`).

---

### Q9. What does `fiber.lanes` represent?
- A) The physical CPU lanes used for rendering
- B) Bitmask of pending update priorities on this fiber
- C) Network request lanes for data fetching
- D) Animation keyframe lanes

**Answer: B**
`lanes` is a bitmask indicating what priority updates are pending on this specific fiber. Used by `beginWork` to determine if the fiber needs processing.

---

### Q10. What does `fiber.childLanes` represent?
- A) The lanes of the fiber's first child only
- B) Bitmask of pending update priorities in the fiber's subtree
- C) The number of child fibers
- D) The rendering order of children

**Answer: B**
`childLanes` is a bitmask representing pending work in the entire subtree. If `childLanes === NoLanes`, React can skip the subtree entirely (bailout).

---

### Q11. What is `FiberRoot`?
- A) The topmost fiber in the tree
- B) A container object that holds metadata about the React root (not a fiber itself)
- C) The root DOM element
- D) The first component rendered

**Answer: B**
`FiberRoot` is a special object created by `createRoot()`. It holds `current` (the root fiber), `pendingLanes`, `finishedWork`, `callbackNode`, etc. It's the entry point for the work loop.

---

### Q12. What is `HostRoot`?
- A) The DOM container element
- B) The topmost fiber in the tree (tag 3), child of FiberRoot
- C) The root of the server-rendered HTML
- D) The Scheduler's root task

**Answer: B**
`HostRoot` (tag 3) is the root fiber node. `FiberRoot.current` points to it. It's the starting point for fiber tree traversal.

---

### Q13. How does `performUnitOfWork` traverse the fiber tree?
- A) Breadth-first using a queue
- B) Depth-first: go to child (beginWork), when no child go to sibling, when no sibling go up (completeWork)
- C) Random order based on priority
- D) Alphabetical order by component name

**Answer: B**
The traversal is depth-first. For each fiber: `beginWork` processes it and returns the child. When no child exists, `completeWork` runs and the loop moves to sibling, then up via `return`.

---

### Q14. What is the "double buffering" technique in React?
- A) Using two DOM trees and swapping them
- B) Maintaining current and work-in-progress fiber trees, swapping after commit
- C) Buffering two frames of animation
- D) Using two event queues

**Answer: B**
React builds the WIP tree during render, then atomically swaps `fiberRoot.current = finishedWork` during commit. The old current becomes the alternate for the next render.

---

### Q15. When does the tree swap happen during commit?
- A) Before any DOM mutations
- B) After DOM mutations but before layout effects
- C) After all effects have run
- D) Before the render phase

**Answer: B**
The swap (`root.current = finishedWork`) happens after mutation sub-phase but before layout sub-phase. This ensures `componentDidMount`/`useLayoutEffect` see the new tree as current.

---

### Q16. What is the purpose of `createWorkInProgress(current, pendingProps)`?
- A) Creates a brand new fiber from scratch
- B) Reuses the `current.alternate` fiber (if it exists), updating its props
- C) Clones the DOM node for the fiber
- D) Creates a new Scheduler task

**Answer: B**
If an `alternate` already exists from a previous render, React reuses it (resetting fields). Otherwise, it creates a new fiber. This avoids allocating fresh objects every render.

---

### Q17. What fiber tag does a Suspense component have?
- A) `SuspenseComponent (13)`
- B) `LazyComponent (16)`
- C) `FunctionComponent (0)`
- D) `HostComponent (5)`

**Answer: A**
`<Suspense>` fibers have tag `SuspenseComponent` (13). They handle thrown Promises and manage fallback/content switching.

---

### Q18. What fiber tag does a Fragment have?
- A) `Fragment (7)`
- B) `HostComponent (5)`
- C) `FunctionComponent (0)`
- D) Fragments don't have fiber nodes

**Answer: A**
Fragments (`<>...</>` or `<React.Fragment>`) have their own fiber node with tag `Fragment` (7). They exist in the fiber tree but have no corresponding DOM node.

---

### Q19. How many fiber trees exist at most for a single React root?
- A) 1
- B) 2
- C) 3
- D) Unlimited

**Answer: B**
At most 2: the current tree (committed) and the work-in-progress tree (being built). After commit, the WIP becomes current and the old current becomes the alternate.

---

### Q20. What is `fiber.pendingProps`?
- A) Props that are queued but not yet applied
- B) The new props passed in the current render, before processing
- C) Props from the previous render
- D) Props that failed validation

**Answer: B**
`pendingProps` holds the props for the current render. After the fiber is processed, `memoizedProps` is set to `pendingProps`. On the next render, `memoizedProps` becomes the "old" props.

---

### Q21. What is `fiber.memoizedProps`?
- A) Props after memoization optimization
- B) Props from the last completed render
- C) Props that passed shallow comparison
- D) Props wrapped in useMemo

**Answer: B**
`memoizedProps` stores the props that were used in the last committed render. During the next render, they're compared with `pendingProps` to detect changes.

---

### Q22. What is `fiber.updateQueue`?
- A) A list of DOM mutations to apply
- B) A queue of state updates (circular linked list for class/host root components)
- C) A list of child fibers to process
- D) A queue of event handlers

**Answer: B**
For class components and HostRoot, `updateQueue` holds pending state updates in a circular linked list. For function components, updates are stored on individual hook queues.

---

### Q23. What is the `return` pointer on a fiber?
- A) The return value of the component
- B) A pointer to the parent fiber
- C) A pointer to the previous sibling
- D) A pointer to the root fiber

**Answer: B**
`return` points to the parent fiber. It's called "return" because it's the fiber to return to after processing the current fiber (analogous to returning from a function call).

---

### Q24. What is the maximum depth of a fiber tree?
- A) 100 levels
- B) 1000 levels
- C) No inherent limit — limited only by available memory
- D) Limited by the JavaScript call stack size

**Answer: C**
Since React uses an iterative work loop (not recursive function calls), tree depth isn't limited by the call stack. Deep trees may have performance implications but won't cause stack overflow.

---

### Q25. What does `fiber.index` represent?
- A) The fiber's position in a global fiber array
- B) The fiber's position among its siblings (used for reconciliation)
- C) The render count for this fiber
- D) The priority index in the scheduler

**Answer: B**
`index` is the position of this fiber among its parent's children. It's used during list reconciliation to determine if a fiber needs to be moved (`Placement` flag).

---

### Q26. What is stored in `fiber.ref`?
- A) A reference to the parent component
- B) The ref object/callback passed via JSX
- C) A reference to the corresponding DOM node
- D) A reference counter for garbage collection

**Answer: B**
`fiber.ref` stores the ref prop (callback function, `useRef` object, or string ref). During commit, React uses this to attach/detach the ref to the DOM node or class instance.

---

### Q27. What is `fiber.dependencies`?
- A) NPM package dependencies
- B) A linked list of contexts this fiber consumes
- C) Child fiber dependencies
- D) Effect dependency arrays

**Answer: B**
`fiber.dependencies` contains a linked list of context objects this fiber reads via `useContext`. Used by `propagateContextChange` to find context consumers.

---

### Q28. What happens to fiber nodes when a component unmounts?
- A) They are immediately freed from memory
- B) React detaches them (nullifies pointers) to make them eligible for GC
- C) They are moved to a recycling pool
- D) They remain in memory until the page unloads

**Answer: B**
React sets `fiber.return`, `fiber.child`, `fiber.stateNode`, `fiber.memoizedState`, etc. to `null` during deletion, making the fiber and its referenced objects eligible for garbage collection.

---

### Q29. What is the `subtreeFlags` field on a fiber?
- A) Flags for CSS subtree optimization
- B) Bitmask of effect flags that exist anywhere in the fiber's subtree
- C) The number of children in the subtree
- D) Feature flags for the subtree

**Answer: B**
`subtreeFlags` is a bitmask OR of all `flags` in the subtree. During commit, React uses this to quickly skip subtrees with no effects (if `subtreeFlags === NoFlags`).

---

### Q30. How does `bubbleProperties` work in completeWork?
- A) It propagates CSS properties up the DOM tree
- B) It ORs together the `flags` and `subtreeFlags` of all children into the parent's `subtreeFlags`
- C) It copies props from child to parent fiber
- D) It merges context values up the tree

**Answer: B**
`bubbleProperties` walks the child list, ORing each child's `flags | subtreeFlags` into the parent's `subtreeFlags`, and each child's `lanes | childLanes` into the parent's `childLanes`.

---

### Q31. What is the `deletions` array on a fiber?
- A) A history of all deleted children
- B) An array of child fibers that need to be deleted during commit
- C) A list of deleted props
- D) A garbage collection queue

**Answer: B**
During reconciliation, when children are removed, their fibers are added to the parent's `deletions` array. During commit, `commitDeletionEffects` processes this array.

---

### Q32. How does React handle a component that renders the same output?
- A) Always re-renders the entire subtree
- B) If `didReceiveUpdate === false`, it bails out and clones children instead of re-rendering them
- C) Caches the DOM and skips commit entirely
- D) Throws an optimization warning

**Answer: B**
If after calling the component function, no hooks had state changes (`didReceiveUpdate === false`), React calls `bailoutOnAlreadyFinishedWork`, which clones children from the current tree without re-rendering them.

---

### Q33. What is `workInProgress` in the work loop?
- A) A boolean flag indicating if work is in progress
- B) The current fiber being processed
- C) The entire WIP tree
- D) The scheduler's current task

**Answer: B**
`workInProgress` is a pointer to the fiber currently being processed in the work loop. It advances through the tree as `performUnitOfWork` completes each fiber.

---

### Q34. What triggers `prepareFreshStack()`?
- A) Component mount
- B) A new render cycle starting or a render being restarted from scratch
- C) Memory pressure
- D) Browser idle callback

**Answer: B**
`prepareFreshStack` creates a new WIP tree from the current tree when a render begins or when an interrupted render needs to restart (e.g., higher priority update arrived).

---

### Q35. What is the `mode` field on a fiber?
- A) Dark mode / light mode setting
- B) Bitmask of rendering modes (ConcurrentMode, StrictMode, etc.)
- C) The rendering engine mode (Canvas/WebGL/DOM)
- D) Development vs production mode

**Answer: B**
`fiber.mode` is a bitmask indicating which modes are active: `ConcurrentMode`, `StrictMode`, `ProfileMode`, etc. All descendants inherit the parent's mode.

---

### Q36. What is `StrictMode` in terms of fiber mode?
- A) A separate rendering engine
- B) A mode bit that enables double-rendering and double-effect-mounting in development
- C) A production performance optimization
- D) A security hardening feature

**Answer: B**
`StrictMode` sets a mode bit on the fiber. In development, React uses this to double-invoke component functions, effect setups, and cleanups to help detect impure code.

---

### Q37. What is `fiber.elementType` vs `fiber.type`?
- A) They are always the same
- B) `elementType` is the original type from JSX; `type` may be the resolved inner component (e.g., for memo/lazy)
- C) `elementType` is for DOM elements, `type` is for components
- D) `elementType` is a string, `type` is a function

**Answer: B**
For `React.memo(Comp)`, `elementType` is the memo wrapper object while `type` is the inner `Comp` function. For `React.lazy`, `type` is updated to the resolved component after loading.

---

### Q38. How many root fibers can exist on a single page?
- A) Exactly 1
- B) At most 2
- C) As many as there are `createRoot()` calls
- D) Limited by browser memory only

**Answer: C**
Each `createRoot()` call creates a separate `FiberRoot` with its own fiber tree. Multiple React roots can coexist on the same page, each independent.

---

### Q39. What does `fiber.tag` determine?
- A) The HTML tag name
- B) The type of work the fiber represents (FunctionComponent, HostComponent, etc.)
- C) The priority tag for scheduling
- D) A debugging label

**Answer: B**
`fiber.tag` is a numeric constant that determines how `beginWork` and `completeWork` process the fiber. It maps to cases in the switch statement (e.g., `FunctionComponent = 0`, `HostComponent = 5`).

---

### Q40. What is the `OffscreenComponent` fiber used for?
- A) Rendering components outside the viewport
- B) Managing visibility of Suspense content and the Activity API
- C) Server-side rendering of hidden content
- D) Lazy loading images

**Answer: B**
`OffscreenComponent` (now `Activity` in React 19) manages content that can be hidden/shown without unmounting. Suspense uses it internally to toggle between content and fallback.

---

### Q41. What is the effect of setting `fiber.flags |= Update`?
- A) The fiber will be deleted
- B) The fiber's DOM node needs props/attributes updated during commit
- C) The fiber needs a new render
- D) The fiber's state has been updated

**Answer: B**
The `Update` flag tells the commit phase that this fiber's host node needs attribute/property updates applied via `commitUpdate`.

---

### Q42. What is `fiber.memoizedState` for a class component?
- A) The hooks linked list
- B) The class component's `this.state` object
- C) The memoized render output
- D) A cache of computed values

**Answer: B**
For class components, `memoizedState` holds the state object (`this.state`). For function components, it holds the hooks linked list. The meaning varies by fiber tag.

---

### Q43. How does React efficiently check if any work exists in a subtree?
- A) By traversing every fiber in the subtree
- B) By checking `fiber.childLanes !== NoLanes`
- C) By querying the DOM
- D) By polling the Scheduler

**Answer: B**
`childLanes` is a bitmask bubbled up from all descendants. If `childLanes === NoLanes`, no fiber in the subtree has pending updates, and the entire subtree can be skipped.

---

### Q44. What is the `Ref` flag on a fiber?
- A) The fiber has a ref that needs to be attached or detached during commit
- B) The fiber references another fiber
- C) The fiber is referenced by an error boundary
- D) The fiber contains a useRef hook

**Answer: A**
The `Ref` flag is set when a fiber has a `ref` prop. During commit's layout phase, `commitAttachRef` sets the ref to the DOM node or instance.

---

### Q45. What fiber tag does `React.Profiler` receive?
- A) `FunctionComponent (0)`
- B) `Profiler (12)`
- C) `HostComponent (5)`
- D) `Fragment (7)`

**Answer: B**
`<Profiler>` components get tag `Profiler` (12). React measures render timing for the Profiler's subtree and calls the `onRender` callback.

---

### Q46. What is `fiber.actualDuration`?
- A) Time the component has been mounted
- B) Time spent rendering this fiber and its subtree (used by Profiler)
- C) Time until the next scheduled update
- D) Time since the last commit

**Answer: B**
`actualDuration` measures the time spent in `beginWork` + `completeWork` for the fiber and all its descendants. It's only tracked when `ProfileMode` is active.

---

### Q47. In the fiber tree, how are multiple children of a parent connected?
- A) As an array on `parent.children`
- B) As a linked list via `sibling` pointers (first child via `parent.child`)
- C) Via a Map keyed by child index
- D) Via a doubly-linked list

**Answer: B**
`parent.child` points to the first child. Each child's `sibling` pointer links to the next child. The last child's `sibling` is `null`. This is a singly-linked list.

---

### Q48. Can a fiber tree have cycles?
- A) Yes, through `alternate` pointers
- B) No, the tree is strictly acyclic (DAG)
- C) Yes, through `return` pointers forming loops
- D) Only during error recovery

**Answer: B**
The fiber tree is acyclic. While `alternate` creates pairs between current and WIP trees, the tree structure itself (child/sibling/return) has no cycles. `alternate` is a 1-to-1 mapping, not a cycle.

---

### Q49. What is `fiber.expirationTime` in modern React?
- A) The time when the fiber's state expires
- B) Deprecated — replaced by `fiber.lanes` in React 18+
- C) The timeout for async operations
- D) The time until garbage collection

**Answer: B**
The `expirationTime` system was replaced by the Lanes model in React 18. Lanes use bitmasks instead of numeric timestamps for more flexible priority management.

---

### Q50. What is the `ShouldCapture` flag?
- A) Indicates the fiber should capture user input
- B) Indicates the fiber is an error boundary that should capture a thrown error
- C) Indicates the fiber should capture performance metrics
- D) Indicates the fiber should capture screen position

**Answer: B**
During error handling, `ShouldCapture` is set on the nearest error boundary fiber. During the unwind phase, React sees this flag and processes the error recovery on that fiber.
