# MCQs: Reconciliation & Diffing Algorithm

---

### Q1. What is the time complexity of React's diffing algorithm?
- A) O(n³)
- B) O(n²)
- C) O(n log n)
- D) O(n)

**Answer: D**
React uses heuristics to achieve O(n) diffing instead of the O(n³) optimal tree diff. The two heuristics: different types mean different trees, and keys identify stable elements.

---

### Q2. What happens when an element's type changes at the same position?
- A) React updates the existing fiber's props
- B) React destroys the entire old subtree and creates a new one
- C) React merges the old and new subtrees
- D) React throws an error

**Answer: B**
When the type changes (e.g., `<div>` → `<span>`, or `<ComponentA>` → `<ComponentB>`), React unmounts the entire old subtree (destroying state) and mounts a new one from scratch.

---

### Q3. What is the purpose of keys in list reconciliation?
- A) To encrypt element data
- B) To uniquely identify elements among siblings for efficient reordering
- C) To set CSS class names
- D) To define rendering priority

**Answer: B**
Keys allow React to match old and new elements in a list, identifying which were added, removed, or moved without unnecessarily destroying and recreating fibers.

---

### Q4. What is the first pass of React's list reconciliation algorithm?
- A) Sort elements by key
- B) Walk old and new lists in parallel, matching by key, stopping at first mismatch
- C) Build a Map of all elements
- D) Delete all old elements

**Answer: B**
Pass 1 compares elements at the same index. If keys match, the fiber is reused. The first mismatch breaks the loop, and remaining elements go to Pass 2.

---

### Q5. What is the second pass of list reconciliation?
- A) Re-sort the elements
- B) Put remaining old fibers in a Map by key, then look up matches for remaining new elements
- C) Delete all remaining elements
- D) Re-run the first pass in reverse

**Answer: B**
Pass 2 builds a `Map<key, fiber>` from remaining old fibers. For each remaining new element, React looks up by key. Found = reuse and move; not found = create new. Leftover in map = delete.

---

### Q6. What is `lastPlacedIndex` used for in list reconciliation?
- A) The last index in the DOM
- B) Tracking the rightmost old-list index that was "in place" to determine which fibers need moving
- C) The last index in the new list
- D) The index of the last placed DOM node

**Answer: B**
`lastPlacedIndex` tracks the maximum old index of fibers that don't need moving. If a reused fiber's old index is less than `lastPlacedIndex`, it needs a `Placement` flag (move).

---

### Q7. Why are index-based keys an anti-pattern for dynamic lists?
- A) Indexes are not unique
- B) When items are reordered/removed, the wrong state gets associated with the wrong items
- C) Indexes cause performance issues
- D) React ignores numeric keys

**Answer: B**
Index keys cause fibers to match by position, not identity. Removing item at index 1 causes item at index 2 to inherit index 1's state/fiber — corrupting uncontrolled state.

---

### Q8. When is using index as key acceptable?
- A) Always
- B) Never
- C) When the list is static and never reordered
- D) Only for class components

**Answer: C**
Index keys are safe when the list content and order never change (e.g., a static menu). For dynamic lists with add/remove/reorder, use stable unique IDs.

---

### Q9. What does `reconcileSingleElement` do?
- A) Reconciles a list of elements
- B) Reconciles when the new children is a single React element
- C) Reconciles text nodes
- D) Reconciles fragments

**Answer: B**
When `reconcileChildFibers` receives a single element (not array), it uses `reconcileSingleElement` which compares the key and type with existing children to find a match.

---

### Q10. In `reconcileSingleElement`, if a matching key is found but type differs, what happens?
- A) The fiber is updated with the new type
- B) The matching fiber and ALL remaining siblings are deleted, new fiber created
- C) Only the matching fiber is deleted
- D) React throws an error

**Answer: B**
When the key matches but type differs, React deletes that fiber AND all its siblings (since we're going from multiple to single, and the only key match failed). Then creates a new fiber.

---

### Q11. What does `beginWork` return?
- A) The updated DOM node
- B) The child fiber to process next (or null if no children)
- C) A boolean indicating success
- D) The commit instructions

**Answer: B**
`beginWork` processes the current fiber and returns `workInProgress.child` — the first child fiber to process next. Returning `null` means no children, triggering `completeWork`.

---

### Q12. What is the bailout optimization in `beginWork`?
- A) Skipping the render phase entirely
- B) When `oldProps === newProps` and no pending lanes, skip the component and clone children
- C) Caching DOM mutations
- D) Batching multiple components

**Answer: B**
If props haven't changed (same reference) and there are no pending updates on this fiber (`!includesSomeLane(renderLanes, fiber.lanes)`), React can skip calling the component and reuse existing children.

---

### Q13. What condition must be true for `bailoutOnAlreadyFinishedWork`?
- A) The component must be memoized
- B) `oldProps === newProps` (reference equality) and no pending lanes on the fiber
- C) The component must return `null`
- D) The component must be a class with `shouldComponentUpdate` returning false

**Answer: B**
The bailout checks: `oldProps === newProps` (reference equality, not deep comparison), `!hasScheduledUpdateOrContext`, and the legacy context hasn't changed.

---

### Q14. What does `completeWork` do for a `HostComponent`?
- A) Calls the component function
- B) Creates the DOM element (on mount) or diffs props (on update), appends children
- C) Runs useEffect
- D) Dispatches events

**Answer: B**
On mount, `completeWork` calls `createElement`, appends already-created child DOM nodes (`appendAllChildren`), and sets initial props. On update, it calls `diffProperties` to compute a change list.

---

### Q15. What is the `updatePayload` in `completeWork`?
- A) The new props object
- B) An array of `[propKey, propValue]` pairs representing changes to apply during commit
- C) The DOM mutation instructions
- D) The event handler updates

**Answer: B**
`diffProperties` returns an array like `['className', 'new-class', 'style', {color: 'red'}]` — alternating key-value pairs of changed props. Stored on `fiber.updateQueue` for commit.

---

### Q16. What does `appendAllChildren` do?
- A) Appends all React elements to an array
- B) Walks the fiber's children/siblings to find host nodes and appends them to the parent DOM node
- C) Appends children to the event queue
- D) Appends children to the update queue

**Answer: B**
During `completeWork`, `appendAllChildren` traverses child fibers (skipping component fibers) to find `HostComponent`/`HostText` fibers and appends their DOM nodes to the parent DOM element being constructed.

---

### Q17. What is `didReceiveUpdate`?
- A) A flag indicating the component received an update from the parent
- B) A module-level flag that tracks whether any hook processed a state change during the current render
- C) A flag on the fiber indicating it has pending updates
- D) A flag indicating the DOM was updated

**Answer: B**
`didReceiveUpdate` is a module-level boolean. During `renderWithHooks`, if any hook (useState, useReducer) processes an update that changes state, it's set to `true`. If `false` after render, React can bail out.

---

### Q18. What is the "render then bailout" path?
- A) The component is never rendered
- B) The component renders (function is called) but produces identical output, so children are cloned
- C) The render is cancelled mid-way
- D) The component renders but commit is skipped

**Answer: B**
When `oldProps !== newProps` (different object references), React must call the component. But if no hooks had state changes (`didReceiveUpdate === false`), React bails out for children.

---

### Q19. What does `reconcileChildrenArray` do when the new array is empty?
- A) Nothing
- B) Deletes all existing child fibers
- C) Renders a placeholder
- D) Throws an error

**Answer: B**
An empty new children array means all old children should be removed. `deleteRemainingChildren` walks the old sibling list and marks each fiber for deletion.

---

### Q20. How does React handle going from one child to many?
- A) Wraps them in a Fragment automatically
- B) Uses `reconcileChildrenArray` for the new list, the single old child may match one of the new elements
- C) Throws an error about inconsistent children
- D) Ignores the extra children

**Answer: B**
React detects the new children is an array and uses `reconcileChildrenArray`. The old single fiber may match one of the new elements by key/type, while others are created fresh.

---

### Q21. In the diffing algorithm, what does "same type" mean?
- A) Same HTML tag or same component function/class reference
- B) Same visual appearance
- C) Same props
- D) Same number of children

**Answer: A**
"Same type" means `oldFiber.type === newElement.type` — same string for host elements (`'div' === 'div'`) or same function/class reference for components.

---

### Q22. What happens when a component is rendered at a different position without a key?
- A) State is preserved
- B) State is destroyed because position changed
- C) State is transferred to the new position
- D) React logs a warning

**Answer: B**
Without keys, React identifies components by position. A component at position 0 and one at position 1 are considered different instances. Moving a component destroys its state.

---

### Q23. How does `processUpdateQueue` handle multiple queued updates?
- A) Only processes the last update
- B) Processes all updates sequentially, each building on the previous state
- C) Processes them in parallel
- D) Randomly selects one update

**Answer: B**
Updates form a circular linked list. `processUpdateQueue` walks the list, applying each update's reducer to the accumulated state. The final state becomes the new `memoizedState`.

---

### Q24. What happens when an update's lane doesn't match the current render lanes?
- A) The update is discarded
- B) The update is skipped and kept in the queue for a future render
- C) React throws an error
- D) The update is applied anyway

**Answer: B**
If the update's lane isn't included in `renderLanes`, it's skipped during this render but remains in the queue. `baseState` is preserved at the point before the first skipped update.

---

### Q25. What is the `baseState` in an update queue?
- A) The initial state from useState
- B) The state up to (but not including) the first skipped update
- C) The base class component's state
- D) The default context value

**Answer: B**
`baseState` represents the state computed before any skipped updates. When those updates' lanes match in a future render, they're replayed from `baseState` to ensure consistency.

---

### Q26. What does `cloneChildFibers` do?
- A) Deep clones the entire subtree
- B) Creates WIP fibers for each child by reusing alternates, without re-rendering them
- C) Copies DOM nodes
- D) Duplicates the component function

**Answer: B**
During bailout, `cloneChildFibers` creates WIP fibers from the current tree's children (reusing alternates) without calling any component functions — the children are carried over as-is.

---

### Q27. What is the significance of `shouldComponentUpdate` returning `false`?
- A) The component is unmounted
- B) React skips calling `render()` and reuses existing children
- C) The component renders but commit is skipped
- D) Props are not updated

**Answer: B**
When `shouldComponentUpdate` returns `false`, React bails out — it doesn't call `render()` and clones existing children. State updates still apply, but the render output is reused.

---

### Q28. How does React handle the `defaultProps` static property?
- A) They're resolved at compile time
- B) React merges `defaultProps` with the passed props before passing to the component
- C) They're applied inside the component via destructuring defaults
- D) React ignores `defaultProps` for function components

**Answer: B**
For class components and legacy function components, React resolves `defaultProps` before calling the component: `resolvedProps = { ...defaultProps, ...props }`.

---

### Q29. What is `reconcileSingleTextNode` used for?
- A) Reconciling a single React element
- B) Reconciling when the new child is a string or number
- C) Reconciling inline styles
- D) Reconciling text input values

**Answer: B**
When `reconcileChildFibers` receives a string or number, it uses `reconcileSingleTextNode` to create or update a `HostText` fiber.

---

### Q30. What is the `Deletion` flag on a fiber?
- A) The fiber has been deleted from the source code
- B) The fiber and its subtree should be removed from the DOM during commit
- C) The fiber's state should be cleared
- D) The fiber should be removed from the scheduler

**Answer: B**
`Deletion` tells the commit phase to remove this fiber's DOM node, run cleanup effects, detach refs, and recursively process the subtree for cleanup.

---

### Q31. How does `getHostSibling` work during `commitPlacement`?
- A) Returns the next DOM sibling directly
- B) Walks the fiber tree to find the next host node that's already placed in the DOM
- C) Queries the DOM using `nextSibling`
- D) Uses a pre-computed sibling map

**Answer: B**
When inserting a DOM node, React needs the insertion point (`insertBefore`). `getHostSibling` walks right through the fiber tree, skipping non-host and newly-placed fibers, to find an existing DOM sibling.

---

### Q32. What makes `getHostSibling` potentially O(n)?
- A) It searches the entire DOM
- B) It may walk through many component fibers (which have no DOM nodes) and drill into their children
- C) It sorts the sibling list
- D) It queries the server

**Answer: B**
For deeply nested component trees, `getHostSibling` must skip over many non-host fibers. In the worst case (many wrapper components), it traverses a significant portion of the tree.

---

### Q33. What does `commitUpdate` receive?
- A) The full new props object
- B) The `updatePayload` (array of changed prop key-value pairs) from `diffProperties`
- C) A DOM mutation instruction set
- D) The fiber node

**Answer: B**
`commitUpdate(instance, updatePayload, type, oldProps, newProps)` receives the pre-computed `updatePayload` array and applies the specific changed properties to the DOM node.

---

### Q34. How does React handle `style` prop changes?
- A) Replaces the entire style object
- B) Diffs old and new style objects, applying only changed properties
- C) Uses CSS classes instead
- D) Ignores style changes

**Answer: B**
React diffs the old and new style objects. Removed properties are set to `''` (empty string). Changed properties are set to their new values. Only modified properties are touched.

---

### Q35. What is `reconcileChildFibers` vs `mountChildFibers`?
- A) They are identical
- B) `reconcileChildFibers` tracks side effects (deletions); `mountChildFibers` doesn't (initial mount)
- C) `reconcileChildFibers` is for class components; `mountChildFibers` is for function components
- D) `mountChildFibers` is faster because it skips diffing

**Answer: B**
Both call the same logic, but `reconcileChildFibers` (used on updates) sets `shouldTrackSideEffects = true`, which tracks deletions and placements. `mountChildFibers` (initial mount) sets it to `false`.

---

### Q36. What optimization does `shouldTrackSideEffects = false` provide?
- A) Skips the diffing algorithm
- B) During initial mount, doesn't mark individual fibers with Placement since the entire tree is new
- C) Disables error tracking
- D) Skips event handler attachment

**Answer: B**
On initial mount, all nodes need to be placed. Instead of marking each fiber individually, React places the entire tree at once during commit, avoiding redundant per-fiber Placement flags.

---

### Q37. What happens when you render `{condition && <Component />}` and condition changes?
- A) Component is hidden with CSS
- B) `true→false`: Component's fiber is deleted (unmount). `false→true`: New fiber created (mount)
- C) Component is detached but state is preserved
- D) Nothing changes

**Answer: B**
Conditional rendering causes full mount/unmount cycles. `false` produces no fiber, while `<Component />` produces one. The transition between them creates/destroys the fiber.

---

### Q38. How does React compare `oldProps` and `newProps` for bailout?
- A) Deep equality comparison
- B) Reference equality (`===`)
- C) Shallow equality (`shallowEqual`)
- D) JSON.stringify comparison

**Answer: B**
The initial bailout check uses `===` (reference equality). If the parent re-rendered, it creates new props objects, so `===` fails even if values are the same. That's why `React.memo` exists.

---

### Q39. What does `React.memo` use for comparison by default?
- A) Reference equality (`===`)
- B) `shallowEqual` (shallow comparison of each prop)
- C) Deep equality
- D) `JSON.stringify` comparison

**Answer: B**
`React.memo` defaults to `shallowEqual`, which compares each prop with `Object.is`. This catches the case where a new props object has the same values as the old one.

---

### Q40. What is the role of `fiber.alternate.memoizedState` during update?
- A) It stores the previous render's error
- B) It provides the current (committed) state to compare against pending updates
- C) It stores the alternate rendering strategy
- D) It holds the context value

**Answer: B**
`current.memoizedState` (via alternate) holds the committed state. During update, `workInProgress` starts with this and processes pending updates to compute new state.

---

### Q41. What is `createFiberFromElement`?
- A) Creates a DOM element from a fiber
- B) Creates a fiber node from a React element during reconciliation
- C) Creates a React element from a fiber
- D) Creates a fragment from an element

**Answer: B**
When reconciliation determines a new child needs a fiber, `createFiberFromElement` creates one, setting the tag based on the element's type (string → HostComponent, function → IndeterminateComponent, etc.).

---

### Q42. What is the purpose of `placeSingleChild`?
- A) Places a child in the DOM
- B) Marks a newly created single child fiber with the `Placement` flag if side effects are being tracked
- C) Places a child at a specific position
- D) Selects which child to render

**Answer: B**
`placeSingleChild` adds the `Placement` flag to a fiber that was created (not reused) during `reconcileSingleElement`, indicating it needs DOM insertion during commit.

---

### Q43. In what order does React process effects during the commit phase?
- A) Random order
- B) In fiber tree order (depth-first), but layout effects are child-first (bottom-up)
- C) Alphabetical by component name
- D) By priority level

**Answer: B**
Effects are processed in depth-first order following the fiber tree. Layout effects fire child-first (because `completeWork` processes children before parents). Passive effects follow the same order.

---

### Q44. What does `markUpdate` do on a fiber?
- A) Marks it for deletion
- B) Sets `fiber.flags |= Update`, indicating props/state changed and DOM needs updating
- C) Adds it to the scheduler queue
- D) Marks it as a context consumer

**Answer: B**
During `completeWork`, if `diffProperties` returns a non-null `updatePayload`, `markUpdate` adds the `Update` flag so `commitUpdate` processes it during commit.

---

### Q45. How does React handle moving a DOM node to a new position?
- A) `removeChild` then `appendChild`
- B) Sets `Placement` flag, then `commitPlacement` uses `insertBefore` with the correct sibling
- C) Swaps CSS `order` property
- D) Recreates the entire parent

**Answer: B**
Moved fibers get the `Placement` flag. During commit, `commitPlacement` calls `insertBefore(movedNode, referenceNode)` where `referenceNode` is found via `getHostSibling`.

---

### Q46. What is the maximum number of DOM operations React uses to reconcile `[A,B,C,D] → [D,A,B,C]`?
- A) 4 operations
- B) 3 operations
- C) 1 operation
- D) 0 operations

**Answer: C**
React's algorithm moves A, B, C (3 have lower old index than D's `lastPlacedIndex`). But actually, the optimal would be to move just D to front. React's algorithm isn't optimal for all permutations — it may perform 3 moves here.

---

### Q47. Why doesn't React use the optimal list diff algorithm?
- A) It's too complex to implement
- B) The optimal algorithm is O(n²) or higher; React trades optimality for O(n) speed
- C) React's algorithm IS optimal
- D) Browser DOM APIs don't support optimal operations

**Answer: B**
The optimal list diff (minimum edit distance) is O(n²). React's `lastPlacedIndex` heuristic is O(n) but may perform more moves than necessary. For typical UI lists, the tradeoff is worthwhile.

---

### Q48. What happens if a child has a key but its sibling doesn't?
- A) React treats the un-keyed child as having `key=null`
- B) React throws a warning about mixing keyed and un-keyed children
- C) React ignores the keyed child's key
- D) Both A and B

**Answer: D**
Un-keyed children get `key = null`. React logs a DEV warning about mixing keyed and un-keyed children, as it prevents effective reconciliation.

---

### Q49. What is `reconcileChildrenIterator` used for?
- A) Reconciling array children
- B) Reconciling iterable children (generators, Maps, Sets) that implement the Iterator protocol
- C) Reconciling text children
- D) Iterating over fiber trees

**Answer: B**
Besides arrays, React supports any iterable as children. `reconcileChildrenIterator` handles the general `Symbol.iterator` protocol.

---

### Q50. How does React reconcile when a component returns a different type across renders?
- A) Smoothly transitions between types
- B) Destroys the old fiber entirely and creates a new one for the new type
- C) Morphs the existing fiber to the new type
- D) Keeps both and shows the new one on top

**Answer: B**
Returning `<div>` then `<span>` on the next render means type changed. React deletes the old `HostComponent('div')` fiber and creates a new `HostComponent('span')` fiber. All child state is lost.
