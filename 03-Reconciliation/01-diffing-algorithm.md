# The Diffing Algorithm

## The Problem

Given two trees, the optimal algorithm to find the minimum number of
transformations has O(n^3) complexity. For a tree of 1000 nodes, that's
1 billion comparisons — far too slow.

## React's O(n) Heuristics

React uses two heuristics that reduce the problem to O(n):

### Heuristic 1: Different Types → Different Trees

If the root element type changes, React tears down the entire old subtree
and builds a new one from scratch. No attempt to diff children.

```
OLD:                    NEW:                    RESULT:
<div>                   <section>               Destroy entire old
  <Header />             <Header />             subtree, build new
  <Main />               <Main />               from scratch
  <Footer />             <Footer />
</div>                  </section>

Even though Header, Main, Footer are the same,
React won't try to reuse them — the parent type changed.
```

### Heuristic 2: Keys Identify Stable Elements

Elements with the same `key` in the same list position are matched:

```
OLD:                         NEW:
<ul>                         <ul>
  <li key="a">Apple</li>      <li key="c">Cherry</li>  ← new
  <li key="b">Banana</li>     <li key="a">Apple</li>   ← moved
  <li key="c">Cherry</li>     <li key="b">Banana</li>  ← moved
</ul>                        </ul>

With keys: React knows a, b, c are the same items → MOVES them (3 ops)
Without keys: React updates all three in place → 3 mutations + potential bugs
```

## The Reconciliation Decision Tree

```
             Compare old fiber with new element
                          │
             ┌────────────┴────────────┐
             ▼                         ▼
        Same type?                Different type
             │                         │
        ┌────┴────┐                    ▼
        ▼         ▼              DELETE old fiber
    Same key?  No key            CREATE new fiber
        │      (both)            (no reuse at all)
        ▼         ▼
    ┌───┴───┐     │
    ▼       ▼     ▼
  Match   Key     Reuse fiber
  found   changed Update props
    │       │     Recurse into
    ▼       ▼     children
  Reuse   DELETE old
  fiber   CREATE new
  at new
  position
```

## Single Element Reconciliation

When a fiber has a single child:

```javascript
// Simplified from ReactChildFiber.js

function reconcileSingleElement(returnFiber, currentFirstChild, element) {
  const key = element.key;
  let child = currentFirstChild;

  // Walk existing children looking for a match
  while (child !== null) {
    if (child.key === key) {
      // Key matches
      if (child.elementType === element.type) {
        // Type matches too → REUSE this fiber!
        deleteRemainingChildren(returnFiber, child.sibling);
        const existing = useFiber(child, element.props);
        existing.return = returnFiber;
        return existing;
      }
      // Key matches but type differs → delete all old, create new
      deleteRemainingChildren(returnFiber, child);
      break;
    } else {
      // Key doesn't match → delete this child, check next sibling
      deleteChild(returnFiber, child);
    }
    child = child.sibling;
  }

  // No match found → create a brand new fiber
  const created = createFiberFromElement(element);
  created.return = returnFiber;
  return created;
}
```

## Multi-Element Reconciliation (List Diffing)

This is the most complex part. React reconciles children arrays in two passes:

### Pass 1: Walk Both Lists in Order

```javascript
// Simplified — the actual code is in reconcileChildrenArray()

// Compare old children (linked list) with new children (array)
let oldFiber = currentFirstChild;  // linked list
let newIdx = 0;                    // array index
let lastPlacedIndex = 0;           // tracks DOM ordering

for (; oldFiber !== null && newIdx < newChildren.length; newIdx++) {
  const newChild = newChildren[newIdx];

  if (oldFiber.key !== newChild.key) {
    break;  // Keys diverge → stop first pass, enter pass 2
  }

  // Keys match → check type
  if (oldFiber.type === newChild.type) {
    // Reuse fiber, update props
    newFiber = useFiber(oldFiber, newChild.props);
  } else {
    // Different type → create new
    newFiber = createFiberFromElement(newChild);
    deleteChild(returnFiber, oldFiber);
  }

  oldFiber = oldFiber.sibling;  // advance old list
}
```

### Pass 2: Handle Remaining (Moves, Inserts, Deletes)

```javascript
if (newIdx === newChildren.length) {
  // New list is exhausted → delete all remaining old fibers
  deleteRemainingChildren(returnFiber, oldFiber);
  return;
}

if (oldFiber === null) {
  // Old list is exhausted → insert all remaining new children
  for (; newIdx < newChildren.length; newIdx++) {
    const newFiber = createFiberFromElement(newChildren[newIdx]);
    newFiber.flags |= Placement;
    // ... link into tree
  }
  return;
}

// Both lists have remaining items → build a Map for O(1) lookup
const existingChildren = new Map();
let existingChild = oldFiber;
while (existingChild !== null) {
  const key = existingChild.key !== null ? existingChild.key : existingChild.index;
  existingChildren.set(key, existingChild);
  existingChild = existingChild.sibling;
}

// Walk remaining new children, find matches in map
for (; newIdx < newChildren.length; newIdx++) {
  const newChild = newChildren[newIdx];
  const key = newChild.key !== null ? newChild.key : newIdx;
  const matchedFiber = existingChildren.get(key);

  if (matchedFiber) {
    newFiber = useFiber(matchedFiber, newChild.props);
    existingChildren.delete(key);

    // Determine if the node needs to be MOVED
    if (matchedFiber.index < lastPlacedIndex) {
      newFiber.flags |= Placement;  // Needs to be moved
    } else {
      lastPlacedIndex = matchedFiber.index;
    }
  } else {
    newFiber = createFiberFromElement(newChild);
    newFiber.flags |= Placement;  // Brand new insertion
  }
}

// Delete any old fibers that weren't matched
existingChildren.forEach(child => deleteChild(returnFiber, child));
```

## Visual: List Reconciliation

```
OLD: [A, B, C, D]          NEW: [B, D, A, C]

Pass 1: Compare in order
  old[0]=A vs new[0]=B  →  keys differ, STOP pass 1

Pass 2: Build map from remaining old fibers
  Map: { A→fiber0, B→fiber1, C→fiber2, D→fiber3 }

  Process new[0]=B: found in map at index 1, lastPlaced=1
  Process new[1]=D: found in map at index 3, lastPlaced=3
  Process new[2]=A: found in map at index 0, 0 < 3 → MOVE
  Process new[3]=C: found in map at index 2, 2 < 3 → MOVE

Result:
  B → stays (reuse, no move)
  D → stays (reuse, no move)
  A → Placement flag (move after D)
  C → Placement flag (move after A)

DOM operations: move A, move C (2 operations instead of rebuilding all 4)
```

## Why Keys Matter: The Index Key Anti-pattern

```
// BAD: Using index as key
{items.map((item, i) => <Item key={i} data={item} />)}

When items = [A, B, C] → keys = [0, 1, 2]
Remove B:
  items = [A, C] → keys = [0, 1]

  React sees:
    key=0: was A, still A → reuse (correct)
    key=1: was B, now C   → UPDATE B's fiber with C's props (WRONG!)
    key=2: was C, gone     → DELETE (WRONG!)

  React updates B→C and deletes C, instead of just deleting B.
  If Items have local state, B's state now lives on C's component!

// GOOD: Using stable unique key
{items.map(item => <Item key={item.id} data={item} />)}

  key="a": was A, still A → reuse
  key="b": was B, gone    → DELETE (correct!)
  key="c": was C, still C → reuse
```
