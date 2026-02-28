# beginWork: Entering a Fiber

## What beginWork Does

`beginWork` is called for every fiber during the render phase. Its job is to:

1. Determine if this fiber needs to be updated
2. If yes, process the update (call the component, compute new children)
3. Return the first child fiber (or null if leaf node)

```
beginWork(current, workInProgress, renderLanes)
   │
   ├── current = the fiber from the current tree (null on mount)
   ├── workInProgress = the fiber being processed in the WIP tree
   └── renderLanes = which priority lanes we're rendering
```

## The Bailout Optimization

Before doing any work, beginWork checks if this fiber can **bail out**:

```javascript
// Simplified from ReactFiberBeginWork.js

function beginWork(current, workInProgress, renderLanes) {
  if (current !== null) {
    // This is an UPDATE (not first mount)
    const oldProps = current.memoizedProps;
    const newProps = workInProgress.pendingProps;

    if (
      oldProps === newProps &&                   // Same props object
      !hasContextChanged() &&                    // No context change
      !includesSomeLane(renderLanes, updateLanes) // No pending update
    ) {
      // ═══════════════════════════════════════
      // BAILOUT! Skip this fiber entirely.
      // ═══════════════════════════════════════
      return bailoutOnAlreadyFinishedWork(
        current, workInProgress, renderLanes
      );
    }
  }

  // ... proceed with actual work
}
```

The bailout can also skip the **entire subtree**:

```javascript
function bailoutOnAlreadyFinishedWork(current, workInProgress, renderLanes) {
  // Check if any CHILDREN have pending work
  if (!includesSomeLane(renderLanes, workInProgress.childLanes)) {
    // No child has pending work either → skip ENTIRE subtree
    return null;  // ◄── returning null means "no child to process"
  }

  // Children have work → clone children and continue
  cloneChildFibers(current, workInProgress);
  return workInProgress.child;
}
```

```
Bailout visualization:

        App (no update)
         │
    ┌────┴────┐
    │         │
  Header    Main (has update)
  (bail!)     │
    │      ┌──┴──┐
  SKIP    List  Sidebar
  entire  (has   (bail!)
  subtree  update)  │
              │    SKIP
           ┌──┴──┐
          Item  Item
          (upd) (bail!)
```

## Switch by Fiber Tag

After the bailout check, beginWork switches on the fiber's tag:

```javascript
function beginWork(current, workInProgress, renderLanes) {
  // ... bailout check above ...

  switch (workInProgress.tag) {
    case FunctionComponent:
      return updateFunctionComponent(
        current, workInProgress, workInProgress.type, renderLanes
      );

    case ClassComponent:
      return updateClassComponent(
        current, workInProgress, workInProgress.type, renderLanes
      );

    case HostRoot:
      return updateHostRoot(current, workInProgress, renderLanes);

    case HostComponent:  // 'div', 'span', etc.
      return updateHostComponent(current, workInProgress, renderLanes);

    case HostText:  // "Hello world"
      return null;  // Text nodes have no children

    case SuspenseComponent:
      return updateSuspenseComponent(current, workInProgress, renderLanes);

    case MemoComponent:
      return updateMemoComponent(current, workInProgress, renderLanes);

    case LazyComponent:
      return mountLazyComponent(current, workInProgress, renderLanes);

    case ContextProvider:
      return updateContextProvider(current, workInProgress, renderLanes);

    case ContextConsumer:
      return updateContextConsumer(current, workInProgress, renderLanes);

    case ForwardRef:
      return updateForwardRef(current, workInProgress, renderLanes);

    // ... more cases
  }
}
```

## Function Component Processing

```javascript
function updateFunctionComponent(current, workInProgress, Component, renderLanes) {
  // Set up the hooks dispatcher
  // (mount vs update dispatcher — different implementations)
  prepareToReadContext(workInProgress);

  // ★ CALL THE COMPONENT FUNCTION ★
  // This is where YOUR component code runs
  // All hook calls (useState, useEffect, etc.) happen here
  let nextChildren = renderWithHooks(
    current,
    workInProgress,
    Component,          // Your function: function App() { ... }
    workInProgress.pendingProps,
    {},                 // context (legacy)
    renderLanes,
  );

  // Check if we can bail out after rendering
  if (current !== null && !didReceiveUpdate) {
    // Component rendered but nothing changed
    bailoutHooks(current, workInProgress, renderLanes);
    return bailoutOnAlreadyFinishedWork(current, workInProgress, renderLanes);
  }

  // Reconcile children: diff new elements vs existing child fibers
  reconcileChildren(current, workInProgress, nextChildren, renderLanes);

  return workInProgress.child;  // Return first child to continue traversal
}
```

## Class Component Processing

```javascript
function updateClassComponent(current, workInProgress, Component, renderLanes) {
  const instance = workInProgress.stateNode;

  if (instance === null) {
    // ═══ MOUNTING ═══
    // 1. Create class instance
    const instance = new Component(workInProgress.pendingProps);
    workInProgress.stateNode = instance;
    instance._reactInternals = workInProgress;  // Link back to fiber

    // 2. Process initial state
    instance.state = workInProgress.memoizedState;

    // 3. Call getDerivedStateFromProps
    applyDerivedStateFromProps(workInProgress, Component);

    // 4. Call componentDidMount (scheduled for commit phase)
    workInProgress.flags |= Update;
  } else {
    // ═══ UPDATING ═══
    // 1. Process update queue (setState calls)
    processUpdateQueue(workInProgress, renderLanes);

    // 2. getDerivedStateFromProps
    applyDerivedStateFromProps(workInProgress, Component);

    // 3. shouldComponentUpdate check
    const shouldUpdate = checkShouldComponentUpdate(
      workInProgress, oldProps, newProps, oldState, newState
    );

    if (!shouldUpdate) {
      return bailoutOnAlreadyFinishedWork(current, workInProgress, renderLanes);
    }
  }

  // ★ CALL RENDER METHOD ★
  const nextChildren = instance.render();

  reconcileChildren(current, workInProgress, nextChildren, renderLanes);
  return workInProgress.child;
}
```

## Host Component Processing (DOM Elements)

```javascript
function updateHostComponent(current, workInProgress, renderLanes) {
  // Host components ('div', 'span') just reconcile their children
  // No state, no lifecycle — just props → children

  const nextProps = workInProgress.pendingProps;
  let nextChildren = nextProps.children;

  // Optimization: if the only child is a string/number, treat as
  // direct text content (no HostText fiber needed)
  const isDirectTextChild =
    typeof nextChildren === 'string' || typeof nextChildren === 'number';

  if (isDirectTextChild) {
    // Don't create a separate HostText fiber
    // Will be set directly as textContent in completeWork
    nextChildren = null;
  }

  reconcileChildren(current, workInProgress, nextChildren, renderLanes);
  return workInProgress.child;
}
```

## beginWork Flow Diagram

```
                    beginWork(current, wip, lanes)
                              │
                    ┌─────────┴─────────┐
                    ▼                   ▼
              current !== null?    current === null
              (is update?)         (is mount)
                    │                   │
                    ▼                   │
              Can bail out?             │
              (same props,              │
               no update,               │
               no context change)       │
                 │       │              │
                YES      NO             │
                 │       │              │
                 ▼       └──────┬───────┘
            bailout             ▼
            (skip or     switch (wip.tag)
             clone        ┌──────┬──────┬────────┐
             children)    ▼      ▼      ▼        ▼
                       FuncComp Class  Host    Suspense...
                          │      │      │
                          ▼      ▼      ▼
                       render  render  reconcile
                       Hooks() ()      children
                          │      │      │
                          └──────┴──────┘
                                 │
                                 ▼
                     reconcileChildren(nextChildren)
                                 │
                                 ▼
                      return wip.child (or null)
```
