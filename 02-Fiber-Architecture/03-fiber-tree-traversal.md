# Fiber Tree Traversal

## The Linked List Structure

A fiber tree is NOT a traditional tree with a `children[]` array. Instead,
it uses three pointers per node: `child`, `sibling`, and `return`:

```
                    App
                     │
                   child
                     │
                     ▼
    ┌─── return ─── div ─── sibling ──► null
    │                │
    │              child
    │                │
    │                ▼
    │    ┌─ return ─ H1 ── sibling ──► P ── sibling ──► Counter ──► null
    │    │                             │                 │
    │    │                           child             child
App ◄────┘                             │                 │
    ▲                                  ▼                 ▼
    │                              "Hello"            Button ──► null
    │                                                   │
    └─── (all nodes eventually return to App) ──────────┘
```

## Why Linked List Instead of Array?

1. **O(1) insertion/deletion** — No array shifting
2. **Iterative traversal** — No recursion needed (key for pausability)
3. **Natural work ordering** — child → sibling → return gives depth-first order
4. **Memory efficient** — Three pointers vs. variable-length arrays

## The Traversal Algorithm: performUnitOfWork

React traverses the fiber tree using a depth-first algorithm that alternates
between "going down" (beginWork) and "going up" (completeWork):

```javascript
// Simplified from ReactFiberWorkLoop.js

function workLoopSync() {
  while (workInProgress !== null) {
    performUnitOfWork(workInProgress);
  }
}

function performUnitOfWork(unitOfWork) {
  const current = unitOfWork.alternate;

  // STEP 1: Process this fiber and get its first child
  const next = beginWork(current, unitOfWork, renderLanes);

  // Props are now committed for this fiber
  unitOfWork.memoizedProps = unitOfWork.pendingProps;

  if (next !== null) {
    // Has a child → move down
    workInProgress = next;
  } else {
    // No child → complete this fiber and move to sibling/parent
    completeUnitOfWork(unitOfWork);
  }
}

function completeUnitOfWork(unitOfWork) {
  let completedWork = unitOfWork;

  do {
    const current = completedWork.alternate;
    const returnFiber = completedWork.return;

    // STEP 2: Complete work for this fiber
    completeWork(current, completedWork, renderLanes);

    // Bubble flags up to parent
    if (returnFiber !== null) {
      returnFiber.subtreeFlags |= completedWork.subtreeFlags;
      returnFiber.subtreeFlags |= completedWork.flags;
    }

    // Move to sibling if it exists
    const siblingFiber = completedWork.sibling;
    if (siblingFiber !== null) {
      workInProgress = siblingFiber;
      return;  // Exit — performUnitOfWork will call beginWork on sibling
    }

    // No sibling → go up to parent
    completedWork = returnFiber;
    workInProgress = completedWork;
  } while (completedWork !== null);
}
```

## Step-by-Step Traversal Example

Given this component tree:

```jsx
<App>
  <div>
    <H1>Title</H1>
    <List>
      <Item />
      <Item />
    </List>
  </div>
</App>
```

The traversal order is:

```
Step  Action           Fiber      Direction    workInProgress
─────────────────────────────────────────────────────────────
 1    beginWork        App        ↓ down       div
 2    beginWork        div        ↓ down       H1
 3    beginWork        H1         ↓ down       "Title"
 4    beginWork        "Title"    (leaf)       —
 5    completeWork     "Title"    ↑ up         H1
 6    completeWork     H1         → sibling    List
 7    beginWork        List       ↓ down       Item[0]
 8    beginWork        Item[0]    (leaf)       —
 9    completeWork     Item[0]    → sibling    Item[1]
10    beginWork        Item[1]    (leaf)       —
11    completeWork     Item[1]    ↑ up         List
12    completeWork     List       ↑ up         div
13    completeWork     div        ↑ up         App
14    completeWork     App        DONE         null

Visual path:
                        App
                      ↓1  ↑14
                       div
                     ↓2   ↑13
               ┌──────────────────┐
             ↓3 H1  →6→    List ↑12
             ↓4  ↑5       ↓7   ↑11
           "Title"    Item[0] → Item[1]
                      ↓8  ↑9   ↓10 ↑11
```

## Concurrent Mode: Yielding During Traversal

In concurrent mode, React checks `shouldYield()` between units of work:

```javascript
function workLoopConcurrent() {
  while (workInProgress !== null && !shouldYield()) {
    //                              ^^^^^^^^^^^^^^
    //                              This is the key difference!
    performUnitOfWork(workInProgress);
  }
}
```

```
workInProgress     shouldYield?     Action
──────────────     ────────────     ────────────────
App                No               beginWork(App)
div                No               beginWork(div)
H1                 No               beginWork(H1)
"Title"            YES!             ── PAUSE ──
                                    yield to browser
                                    browser handles events, paints
                                    ── RESUME ──
"Title"            No               beginWork("Title")
...                ...              ...
```

Because the position is tracked by the `workInProgress` pointer (not the call
stack), React can simply stop the loop and resume later from exactly where it
left off.

## Flags Bubbling

During `completeUnitOfWork`, React bubbles effect flags UP the tree:

```
        App (subtreeFlags = Placement | Update)
         │
        div (subtreeFlags = Placement | Update)
         │
    ┌────┴─────┐
    H1         List (subtreeFlags = Placement)
  (flags:       │
   Update)  ┌───┴───┐
            Item    Item
          (flags:  (no flags)
          Placement)
```

This is crucial for the commit phase: React can skip entire subtrees
where `subtreeFlags === NoFlags`, avoiding unnecessary DOM walks.
