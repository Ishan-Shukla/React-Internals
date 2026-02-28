# Interview Questions: React Fundamentals & Architecture

## Q1. What is the Virtual DOM and how does React use it?

**Answer:**

The "Virtual DOM" is a programming concept where a lightweight JavaScript
representation of the UI is kept in memory. In React, this manifests as
**React Elements** (plain objects returned by `createElement`/JSX) and
**Fibers** (internal work nodes).

```
JSX → React Element (plain object) → Fiber (internal work node) → DOM node

React Element:
{
  $$typeof: Symbol(react.element),
  type: 'div',
  key: null,
  props: { children: 'Hello' }
}
```

React doesn't literally diff two virtual DOM trees. It:
1. Builds a **fiber tree** representing the desired UI
2. Compares each fiber with its previous version (**reconciliation**)
3. Computes the minimal set of DOM mutations
4. Applies mutations in a single synchronous **commit phase**

The key insight is that React Elements are cheap to create (plain objects),
while DOM operations are expensive. React batches and minimizes DOM work.

---

## Q2. What is the Fiber architecture and why was it introduced?

**Answer:**

Fiber replaced the original **Stack Reconciler** (React ≤15) with an
**incremental, interruptible** rendering model.

**Problem with Stack Reconciler:**
- Recursive tree traversal using the JavaScript call stack
- Once started, rendering couldn't be paused
- Long renders blocked the main thread → janky UI

**Fiber Solution:**
- Each component becomes a **Fiber node** (a JavaScript object)
- Fibers form a linked list tree (child/sibling/return pointers)
- React processes one fiber at a time in a **work loop**
- Can **yield** to the browser between fibers (concurrent mode)
- Can **interrupt** and restart rendering for higher-priority updates

```
// Fiber work loop (concurrent):
while (workInProgress !== null && !shouldYield()) {
  performUnitOfWork(workInProgress);
}
// If shouldYield() → true, React pauses and gives control to the browser
```

A Fiber node contains: `type`, `key`, `stateNode`, `memoizedState`,
`memoizedProps`, `lanes`, `flags`, `child`, `sibling`, `return`,
`alternate`, and more.

---

## Q3. Explain the double buffering technique in React.

**Answer:**

React maintains **two fiber trees** at all times:

1. **Current tree** — represents what's currently on screen
2. **Work-in-progress (WIP) tree** — being built during render

```
fiberRoot.current → currentTree (on screen)
                     ↕ alternate pointers
                    wipTree (being built)
```

During render, React clones fibers from current to WIP (reusing via
`alternate` pointers) and applies updates. When the render is complete,
React **swaps** by a single pointer assignment:

```javascript
fiberRoot.current = finishedWork;  // The atomic swap
```

The old current tree becomes the alternate for the next render (reused
as the WIP tree). This avoids allocating new fiber nodes from scratch.

This mirrors the **double buffering** technique in graphics — draw to
an offscreen buffer, then swap it in atomically.

---

## Q4. Describe the complete lifecycle of a setState call.

**Answer:**

```
1. EVENT PHASE:
   onClick handler runs → setState(newValue) called

2. SCHEDULE PHASE:
   → Create update object: { lane, action, next }
   → Enqueue on hook's update queue (circular linked list)
   → Determine lane (SyncLane for click, TransitionLane for startTransition)
   → Call ensureRootIsScheduled(root)
   → Scheduler schedules a task via MessageChannel

3. RENDER PHASE (asynchronous, may yield):
   → performConcurrentWorkOnRoot / performSyncWorkOnRoot
   → workLoop: for each fiber, call beginWork
     → beginWork processes component (calls function, runs hooks)
     → Reconciles children (diffing algorithm)
   → completeWork: bottom-up, creates DOM nodes, diffs props

4. COMMIT PHASE (synchronous, cannot yield):
   → Before Mutation: read DOM (getSnapshotBeforeUpdate)
   → Mutation: apply DOM changes (insertBefore, removeChild, etc.)
   → Tree Swap: fiberRoot.current = finishedWork
   → Layout: useLayoutEffect, componentDidMount/Update, refs attached

5. PASSIVE EFFECTS (asynchronous, after paint):
   → Browser paints the screen
   → flushPassiveEffects: useEffect cleanups then setups
```

---

## Q5. What is the reconciliation algorithm? What are its heuristics?

**Answer:**

Reconciliation is React's process of determining which fibers changed
and what DOM updates are needed. It uses two key **heuristics** to
achieve O(n) time complexity (vs O(n³) for optimal tree diff):

1. **Different types → different trees**: If a `<div>` changes to a
   `<span>`, React destroys the entire subtree and rebuilds it.
   No attempt to reuse children.

2. **Keys identify stable elements**: Among siblings, elements with
   the same `key` are assumed to be the same logical element even if
   their position changes.

**Single element reconciliation:**
```
Compare key → same key? → Compare type → same type? → UPDATE (reuse fiber)
                                        → diff type? → DELETE old, CREATE new
             → diff key? → DELETE old, CREATE new
```

**List reconciliation (two-pass):**
```
Pass 1: Walk old and new lists in order, match by key
        Break when first mismatch found
Pass 2: Put remaining old fibers in a Map by key
        Walk remaining new children, look up matches in Map
        Matched → reuse and move; Unmatched → create new
        Leftover in Map → delete
```

---

## Q6. What are React Lanes and how do they replace the old priority system?

**Answer:**

Lanes are a **bitmask-based priority model** (31 bits) that replaced
the numeric `expirationTime` system. Each bit represents a "lane"
(a category of work):

```
SyncLane:           0b0000000000000000000000000000010  (bit 1)
InputContinuousLane:0b0000000000000000000000000001000  (bit 3)
DefaultLane:        0b0000000000000000000000000100000  (bit 5)
TransitionLane1:    0b0000000000000000000000001000000  (bit 6)
...
IdleLane:           0b0100000000000000000000000000000  (bit 29)
```

**Advantages over expirationTime:**
- **Batch related updates**: Multiple transitions can share lanes
  (bitwise OR: `lanes |= TransitionLane2`)
- **Check membership**: `(lanes & SyncLane) !== 0` — O(1)
- **Select work**: `getNextLanes()` picks the highest-priority pending lanes
- **Entangle lanes**: Force related lanes to render together

**Key operations:**
```javascript
mergeLanes(a, b)       → a | b         // Combine
intersectLanes(a, b)   → a & b         // Check overlap
removeLanes(set, lane) → set & ~lane   // Remove
isSubsetOfLanes(set, subset) → (set & subset) === subset
```

---

## Q7. How does the Scheduler work internally?

**Answer:**

React's Scheduler (`packages/scheduler`) manages task execution with:

1. **Min-heap priority queue** — tasks sorted by `expirationTime`
2. **5 priority levels**: Immediate (−1ms), User-blocking (250ms),
   Normal (5000ms), Low (10000ms), Idle (never)
3. **Time-slicing**: ~5ms time slices via `shouldYieldToHost()`
4. **MessageChannel**: Uses `postMessage` for scheduling (not `setTimeout`
   which has a minimum 4ms delay)

```
Scheduler work loop:
  while (peek(taskQueue) !== null) {
    const task = peek(taskQueue);
    if (task.expirationTime > currentTime && shouldYield()) {
      break;  // Yield to browser
    }
    const callback = task.callback;
    const continuationCallback = callback(didTimeout);
    if (typeof continuationCallback === 'function') {
      task.callback = continuationCallback;  // Resume later
    } else {
      pop(taskQueue);  // Task done
    }
  }
```

React maps Lanes to Scheduler priorities:
- SyncLane → ImmediatePriority (blocks everything)
- InputContinuousLane → UserBlockingPriority
- DefaultLane → NormalPriority
- TransitionLanes → NormalPriority
- IdleLane → IdlePriority

---

## Q8. What is the difference between beginWork and completeWork?

**Answer:**

These are the two phases of processing each fiber:

| | beginWork | completeWork |
|---|---|---|
| **Direction** | Top-down (parent → child) | Bottom-up (child → parent) |
| **Purpose** | Process component, reconcile children | Create DOM, diff props |
| **Key work** | Call component function, run hooks, diff children list | Call `createElement`, `diffProperties`, append children |
| **Output** | Returns `child` fiber (or null) | Returns `sibling` fiber (or parent) |
| **Bailout** | Can skip subtree if no updates | Bubbles effect flags up to parent |

```
Traversal order for: App → div → [A, B]

  beginWork(App)        ← process App
    beginWork(div)      ← process div
      beginWork(A)      ← process A (no children)
      completeWork(A)   ← create/update A's DOM
      beginWork(B)      ← process B (sibling)
      completeWork(B)   ← create/update B's DOM
    completeWork(div)   ← append A, B to div
  completeWork(App)     ← bubble flags
```

---

## Q9. How does React handle events differently from native DOM events?

**Answer:**

React uses **event delegation** — a single listener at the root for each
event type, not individual listeners on each element.

```
Native DOM:  Each <button onClick> gets its own addEventListener
React:       root.addEventListener('click', listener)  ← ONE listener
```

When an event fires:
1. Native event reaches root listener
2. React finds the **fiber** for `event.target` (via internal instance map)
3. Walks **up the fiber tree** collecting event handlers
4. Creates a **SyntheticEvent** wrapping the native event
5. Calls handlers in order (capture phase down, bubble phase up)

**Key differences from native:**
- Events bubble through the **fiber tree**, not DOM tree
  (relevant for Portals — events bubble to fiber parent, not DOM parent)
- Event priorities map to React lanes (click → SyncLane, mousemove → InputContinuousLane)
- `wheel`, `touchstart`, `touchmove` registered as **passive** by default
- `onScroll` doesn't bubble in React (matches native behavior)

---

## Q10. What is the purpose of $$typeof in React elements?

**Answer:**

`$$typeof: Symbol(react.element)` is a **security measure** against XSS
attacks via JSON injection.

**The attack:**
```javascript
// If a server returns user-controlled JSON:
{ "type": "div", "props": { "dangerouslySetInnerHTML": { "__html": "<script>..." } } }

// Without $$typeof, React would render this as a valid element!
```

**The defense:**
```javascript
// React elements include:
$$typeof: Symbol.for('react.element')

// Symbols CANNOT be serialized to JSON
JSON.stringify(Symbol.for('react.element'))  → undefined

// So any JSON from an API cannot contain valid React elements
// React checks $$typeof before rendering — missing/wrong → rejected
```

In environments without Symbol support, React falls back to
`0xeac7` (a magic number that looks like "React").
