# Error Handling: Error Boundaries and Recovery

## Error Boundaries

Error boundaries are React components that catch JavaScript errors in their
child component tree, log them, and display a fallback UI.

```javascript
class ErrorBoundary extends React.Component {
  constructor(props) {
    super(props);
    this.state = { hasError: false };
  }

  // Called during RENDER phase (can recover state)
  static getDerivedStateFromError(error) {
    return { hasError: true };
  }

  // Called during COMMIT phase (for logging)
  componentDidCatch(error, errorInfo) {
    console.error('Caught by boundary:', error);
    console.log('Component stack:', errorInfo.componentStack);
  }

  render() {
    if (this.state.hasError) {
      return this.props.fallback;
    }
    return this.props.children;
  }
}
```

## What Error Boundaries Catch

```
✅ CATCHES:                           ❌ DOES NOT CATCH:
─────────────                         ─────────────────
Errors in render methods              Errors in event handlers
Errors in lifecycle methods           Errors in async code (setTimeout)
Errors in constructors                Errors in SSR
Errors in getDerivedStateFromProps    Errors in the boundary itself
Errors in static methods
```

## Internal: How React Catches Errors

```javascript
// In renderRootSync/renderRootConcurrent:

do {
  try {
    if (isSync) {
      workLoopSync();
    } else {
      workLoopConcurrent();
    }
    break;  // Normal completion
  } catch (thrownValue) {
    handleThrow(root, thrownValue);
  }
} while (true);
```

```javascript
function handleThrow(root, thrownValue) {
  // Reset module-level state
  resetContextDependencies();
  resetHooksOnUnwind(workInProgress);

  if (isThenable(thrownValue)) {
    // It's a Promise → Suspense (covered in Suspense section)
    const wakeable = thrownValue;
    throwException(root, workInProgress, wakeable, renderLanes);
  } else {
    // It's a real error
    throwException(root, workInProgress, thrownValue, renderLanes);
  }

  // Unwind to the nearest boundary
  completeUnitOfWork(workInProgress);
}
```

## throwException: Finding the Error Boundary

```javascript
function throwException(root, sourceFiber, value, rootRenderLanes) {
  // Mark the source fiber as incomplete
  sourceFiber.flags |= Incomplete;

  // Walk UP the fiber tree looking for a boundary
  let workInProgress = sourceFiber.return;

  while (workInProgress !== null) {
    switch (workInProgress.tag) {
      case HostRoot: {
        // Reached the root without finding a boundary
        // → Fatal error, unmount the entire tree
        const errorInfo = createCapturedValueFromError(value);
        const update = createRootErrorUpdate(workInProgress, errorInfo);
        enqueueUpdate(workInProgress, update);
        return;
      }

      case ClassComponent: {
        const instance = workInProgress.stateNode;
        const getDerivedStateFromError = workInProgress.type.getDerivedStateFromError;
        const didCatch = typeof instance.componentDidCatch === 'function';

        if (getDerivedStateFromError || didCatch) {
          // ═══ FOUND AN ERROR BOUNDARY ═══
          const errorInfo = createCapturedValueFromError(value);

          // Create an update that calls getDerivedStateFromError
          const update = createClassErrorUpdate(workInProgress, errorInfo);
          enqueueUpdate(workInProgress, update);

          // Mark for ShouldCapture
          workInProgress.flags |= ShouldCapture;
          workInProgress.lanes |= rootRenderLanes;

          return;
        }
        break;
      }
    }

    workInProgress = workInProgress.return;
  }
}
```

## Unwinding: completeUnitOfWork After Error

```javascript
// After throwException, completeUnitOfWork handles unwinding:

function completeUnitOfWork(unitOfWork) {
  let completedWork = unitOfWork;

  do {
    const current = completedWork.alternate;
    const returnFiber = completedWork.return;

    if ((completedWork.flags & Incomplete) === NoFlags) {
      // ═══ NORMAL PATH ═══
      // completeWork as usual
      completeWork(current, completedWork, renderLanes);
    } else {
      // ═══ ERROR PATH ═══
      // This fiber or its subtree threw an error
      const next = unwindWork(current, completedWork, renderLanes);

      if (next !== null) {
        // Found a boundary that can handle the error
        // Re-render from this boundary
        next.flags &= HostEffectMask;  // Clear error flags
        workInProgress = next;
        return;  // ← Resume rendering from the error boundary
      }

      // No boundary found at this level → continue unwinding up
      if (returnFiber !== null) {
        returnFiber.flags |= Incomplete;
        returnFiber.subtreeFlags = NoFlags;
        returnFiber.deletions = null;
      }
    }

    completedWork = returnFiber;
    workInProgress = completedWork;
  } while (completedWork !== null);
}
```

## Error Flow Visualization

```
Component Tree:
  App
   ├── ErrorBoundary
   │    └── Parent
   │         ├── GoodChild
   │         └── BadChild  ← throws Error!
   └── Sidebar

Error propagation:

  Step 1: BadChild throws during render
          beginWork(BadChild) → throw Error("oops")

  Step 2: handleThrow catches it
          → throwException walks UP from BadChild

  Step 3: Walk up: Parent → no getDerivedStateFromError → skip
          Walk up: ErrorBoundary → HAS getDerivedStateFromError!
          → Found boundary!
          → Create error update on ErrorBoundary
          → Mark ErrorBoundary flags |= ShouldCapture

  Step 4: Unwind via completeUnitOfWork
          BadChild: Incomplete → unwindWork → nothing
          Parent: Incomplete → unwindWork → nothing
          ErrorBoundary: ShouldCapture → unwindWork → returns ErrorBoundary

  Step 5: Re-render from ErrorBoundary
          → getDerivedStateFromError(error) → { hasError: true }
          → ErrorBoundary.render() returns fallback UI
          → Reconcile new children (fallback instead of Parent subtree)

  Step 6: Continue rendering rest of tree
          → Sidebar renders normally

  Step 7: Commit phase
          → DOM updated: ErrorBoundary shows fallback
          → componentDidCatch fires (for logging)

  Result:
    App
     ├── ErrorBoundary → shows fallback UI
     └── Sidebar → renders normally (unaffected!)
```

## Recovery (React 19+)

React 19 improved error recovery by automatically attempting to
re-render the error boundary's children after an error:

```
React 18:
  Error → show fallback → stays on fallback forever

React 19:
  Error → show fallback → on next render/navigation →
  try rendering children again → if success, remove fallback
```

## Error Boundaries in Function Components

As of React 19, error boundaries can be created with function components
using the `use` hook and error-handling primitives. However, the
`componentDidCatch` / `getDerivedStateFromError` class API remains
the primary way to create error boundaries.
