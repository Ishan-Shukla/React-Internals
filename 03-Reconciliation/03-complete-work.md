# completeWork: Bubbling Up

## What completeWork Does

`completeWork` is called for each fiber as the traversal moves **upward**.
While `beginWork` processes fibers top-down (parent → child), `completeWork`
processes them bottom-up (child → parent). Its responsibilities:

1. **Create DOM instances** for host components (on mount)
2. **Diff props** for existing host components (on update)
3. **Bubble flags** — aggregate effect flags from children to parent
4. **Build the effect list** — track which fibers need DOM work

## When completeWork Runs

```
Traversal order (beginWork ↓ then completeWork ↑):

          App
        ↓1  ↑8
         div
       ↓2   ↑7
    ┌────────────┐
   H1 →5→      P
  ↓3  ↑4      ↓5  ↑6
 "Hi"        "World"

beginWork order:   App → div → H1 → "Hi" → P → "World"
completeWork order: "Hi" → H1 → "World" → P → div → App
```

## The Switch Statement

```javascript
// Simplified from ReactFiberCompleteWork.js

function completeWork(current, workInProgress, renderLanes) {
  const newProps = workInProgress.pendingProps;

  switch (workInProgress.tag) {
    case FunctionComponent:
    case ClassComponent:
    case Fragment:
    case ContextProvider:
    case ContextConsumer:
    case MemoComponent:
      // These components don't directly produce host instances
      // Just bubble up flags
      bubbleProperties(workInProgress);
      return null;

    case HostComponent:
      return completeHostComponent(current, workInProgress, newProps);

    case HostText:
      return completeHostText(current, workInProgress, newProps);

    case SuspenseComponent:
      return completeSuspenseComponent(current, workInProgress);

    case HostRoot:
      return completeHostRoot(current, workInProgress);
  }
}
```

## Host Component: Mount vs Update

### Mount (current === null): Create DOM Node

```javascript
function completeHostComponent(current, workInProgress, newProps) {
  const type = workInProgress.type;  // 'div', 'span', etc.

  if (current !== null && workInProgress.stateNode !== null) {
    // ═══ UPDATE PATH ═══
    updateHostComponent(current, workInProgress, type, newProps);
  } else {
    // ═══ MOUNT PATH ═══

    // 1. Create the actual DOM element
    const instance = document.createElement(type);
    //              ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
    //              This is where the real DOM node is born!

    // 2. Set initial properties (className, style, event listeners, etc.)
    setInitialProperties(instance, type, newProps);

    // 3. Append all child DOM nodes that were created by child fibers
    appendAllChildren(instance, workInProgress);

    // 4. Store DOM reference on the fiber
    workInProgress.stateNode = instance;

    // 5. Handle refs, autofocus, etc.
    if (shouldAutoFocusHostComponent(type, newProps)) {
      workInProgress.flags |= Update;
    }
  }

  bubbleProperties(workInProgress);
  return null;
}
```

### appendAllChildren: Building the DOM Tree Bottom-Up

```javascript
// This function walks WIP children and appends their DOM nodes
function appendAllChildren(parent, workInProgress) {
  let node = workInProgress.child;

  while (node !== null) {
    if (node.tag === HostComponent || node.tag === HostText) {
      // Direct DOM child → append to parent DOM node
      parent.appendChild(node.stateNode);
    } else if (node.child !== null) {
      // Not a DOM node (e.g., FunctionComponent) → look deeper
      // Components don't produce DOM, so we dig into their children
      node.child.return = node;
      node = node.child;
      continue;
    }

    if (node === workInProgress) return;

    // Walk siblings and back up
    while (node.sibling === null) {
      if (node.return === null || node.return === workInProgress) return;
      node = node.return;
    }
    node.sibling.return = node.return;
    node = node.sibling;
  }
}
```

```
Example: Building DOM tree bottom-up

Fiber Tree:              DOM Construction:
  App (FuncComp)
   │                     Step 1: completeWork(H1)
  div (Host)               → createElement('h1')
   │                       → appendChild(textNode "Title")
  ┌┴┐                     → h1.stateNode = <h1>Title</h1>
  H1  P (Host)
  │   │                  Step 2: completeWork(P)
"Title" "Body"             → createElement('p')
                           → appendChild(textNode "Body")
                           → p.stateNode = <p>Body</p>

                         Step 3: completeWork(div)
                           → createElement('div')
                           → appendAllChildren(div, ...)
                             → walks children: H1 is Host → append
                             → walks siblings: P is Host → append
                           → div.stateNode = <div><h1>Title</h1><p>Body</p></div>

                         Step 4: completeWork(App)
                           → FunctionComponent → no DOM work, just bubble flags

The DOM tree is fully constructed before being attached to the document!
```

### Update Path: Prop Diffing

```javascript
function updateHostComponent(current, workInProgress, type, newProps) {
  const oldProps = current.memoizedProps;

  if (oldProps === newProps) {
    // Referentially equal → nothing changed
    return;
  }

  // Compute the update payload (list of changed properties)
  const updatePayload = diffProperties(
    workInProgress.stateNode,
    type,
    oldProps,
    newProps,
  );

  // Store as [key1, value1, key2, value2, ...]
  workInProgress.updateQueue = updatePayload;

  if (updatePayload) {
    // Mark this fiber for DOM update during commit
    workInProgress.flags |= Update;
  }
}
```

### diffProperties: What Changed?

```javascript
function diffProperties(domElement, tag, lastProps, nextProps) {
  let updatePayload = null;

  // Phase 1: Find removed props
  for (const propKey in lastProps) {
    if (nextProps.hasOwnProperty(propKey) || !lastProps.hasOwnProperty(propKey)) {
      continue;
    }
    // Prop was removed
    (updatePayload = updatePayload || []).push(propKey, null);
  }

  // Phase 2: Find added/changed props
  for (const propKey in nextProps) {
    const nextProp = nextProps[propKey];
    const lastProp = lastProps[propKey];

    if (nextProp === lastProp || (nextProp == null && lastProp == null)) {
      continue;
    }

    if (propKey === 'style') {
      // Diff style objects individually
      const styleDiff = diffStyleProperties(lastProp, nextProp);
      if (styleDiff) {
        (updatePayload = updatePayload || []).push(propKey, styleDiff);
      }
    } else {
      (updatePayload = updatePayload || []).push(propKey, nextProp);
    }
  }

  return updatePayload;
  // Returns: ['className', 'new-class', 'style', {color: 'red'}]
  //    or null if nothing changed
}
```

## bubbleProperties: Aggregating Flags

```javascript
function bubbleProperties(completedWork) {
  let subtreeFlags = NoFlags;
  let child = completedWork.child;

  while (child !== null) {
    subtreeFlags |= child.subtreeFlags;  // grandchild+ flags
    subtreeFlags |= child.flags;          // direct child flags
    child = child.sibling;
  }

  completedWork.subtreeFlags |= subtreeFlags;
}
```

```
After bubbling:

  App  (flags: NoFlags, subtreeFlags: Placement | Update)
   │
  div  (flags: NoFlags, subtreeFlags: Placement | Update)
   │
  ┌┴─────────┐
  H1         List
  (flags:    (flags: NoFlags, subtreeFlags: Placement)
   Update)     │
             ┌─┴──┐
            Item  Item
          (flags: (flags:
         Placement) NoFlags)

Commit phase uses subtreeFlags to skip branches with no work.
```
