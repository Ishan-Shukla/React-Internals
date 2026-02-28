# IndeterminateComponent Resolution

## The Problem: Function or Class?

When React first encounters a component, it doesn't know if it's a
**function component** or a **class component** just by looking at the
`type` field — both are JavaScript functions:

```javascript
// Function component
function Greeting() {
  return <h1>Hello</h1>;
}
typeof Greeting === 'function'  // true

// Class component
class Welcome extends React.Component {
  render() {
    return <h1>Hello</h1>;
  }
}
typeof Welcome === 'function'   // true (classes are functions in JS!)
```

## How React Distinguishes Them

React uses a prototype flag set by `React.Component`:

```javascript
// packages/react/src/ReactBaseClasses.js

function Component(props, context, updater) {
  this.props = props;
  this.context = context;
  this.refs = {};
  this.updater = updater;
}

// ★ THE KEY MARKER ★
Component.prototype.isReactComponent = {};

// React checks:
function shouldConstruct(Component) {
  const prototype = Component.prototype;
  return !!(prototype && prototype.isReactComponent);
  // true for class components
  // false for function components
}
```

## IndeterminateComponent: The First-Render Tag

When a fiber is first created for a function-typed element, React assigns
it the tag `IndeterminateComponent` (2) because it hasn't confirmed what
kind of component it is yet:

```javascript
// In createFiberFromTypeAndProps:

function createFiberFromTypeAndProps(type, key, pendingProps, mode, lanes) {
  let fiberTag = IndeterminateComponent;  // ← Default: we don't know yet

  if (typeof type === 'function') {
    if (shouldConstruct(type)) {
      // Has isReactComponent on prototype → class component
      fiberTag = ClassComponent;
    }
    // Otherwise stays IndeterminateComponent
    // Will be resolved on first render
  } else if (typeof type === 'string') {
    fiberTag = HostComponent;
  }
  // ... other type checks

  const fiber = createFiber(fiberTag, pendingProps, key, mode);
  fiber.type = type;
  fiber.lanes = lanes;
  return fiber;
}
```

Wait — why not just check `shouldConstruct` and set `FunctionComponent`?
Because there's a third case: **components that return class instances**
(a legacy pattern).

## Resolution in beginWork

The first time `beginWork` processes an `IndeterminateComponent`, it calls
the function and inspects the return value:

```javascript
// Simplified from ReactFiberBeginWork.js

function mountIndeterminateComponent(
  current, workInProgress, Component, renderLanes
) {
  const props = workInProgress.pendingProps;

  // Prepare hooks dispatcher (assume function component for now)
  prepareToReadContext(workInProgress, renderLanes);

  // ★ CALL THE FUNCTION ★
  let value = renderWithHooks(
    null,
    workInProgress,
    Component,
    props,
    {},
    renderLanes,
  );

  // Now inspect what was returned:

  if (
    typeof value === 'object' &&
    value !== null &&
    typeof value.render === 'function' &&
    value.$$typeof === undefined
  ) {
    // ═══ RETURNED A CLASS INSTANCE ═══
    // This is a legacy pattern where a function returns { render() {} }
    // Extremely rare in modern React

    workInProgress.tag = ClassComponent;

    // Adopt the returned object as the class instance
    const instance = value;
    workInProgress.stateNode = instance;
    instance._reactInternals = workInProgress;

    // Initialize class lifecycle
    mountClassInstance(workInProgress, Component, props, renderLanes);

    // Render using the instance
    return finishClassComponent(
      null, workInProgress, Component, true, false, renderLanes
    );
  } else {
    // ═══ RETURNED REACT ELEMENTS (normal function component) ═══

    // Resolve the tag permanently
    workInProgress.tag = FunctionComponent;

    // Reconcile the returned elements as children
    reconcileChildren(null, workInProgress, value, renderLanes);

    return workInProgress.child;
  }
}
```

## The Resolution Flow

```
First render of a function-typed component:

  createFiberFromTypeAndProps(MyComponent)
    │
    ├── typeof MyComponent === 'function'
    ├── shouldConstruct(MyComponent)?
    │   ├── YES → tag = ClassComponent (resolved immediately)
    │   └── NO  → tag = IndeterminateComponent (resolve later)
    │
    ▼
  beginWork encounters IndeterminateComponent
    │
    ├── Call MyComponent(props)
    │
    ├── What did it return?
    │   │
    │   ├── JSX elements / null / string / number
    │   │   → tag = FunctionComponent ★ (permanently resolved)
    │   │   → reconcileChildren(elements)
    │   │
    │   └── Object with .render() method
    │       → tag = ClassComponent ★ (permanently resolved)
    │       → mountClassInstance()
    │       → finishClassComponent()
    │
    ▼
  Subsequent renders: tag is already FunctionComponent or ClassComponent
    → beginWork uses the correct updateFunctionComponent or
      updateClassComponent path directly
    → IndeterminateComponent never appears again for this fiber
```

## Why This Matters for Contributors

### 1. The IndeterminateComponent Case in beginWork

If you're reading `ReactFiberBeginWork.js`, you'll see
`IndeterminateComponent` handled in the switch statement.
Understanding it prevents confusion:

```javascript
function beginWork(current, workInProgress, renderLanes) {
  switch (workInProgress.tag) {
    case IndeterminateComponent:
      // ← Only hit on FIRST RENDER of a function component
      return mountIndeterminateComponent(...);

    case FunctionComponent:
      // ← Hit on ALL SUBSEQUENT renders
      return updateFunctionComponent(...);

    case ClassComponent:
      return updateClassComponent(...);

    // ...
  }
}
```

### 2. Hot Module Replacement (HMR) Edge Case

During development with HMR, when a component is hot-reloaded, React
may need to re-resolve it. The component function reference changes,
and React creates a new fiber. If the new function happens to be a
different type (someone refactored function → class), the
IndeterminateComponent path handles this gracefully.

### 3. Lazy Components

`React.lazy` components go through a similar resolution process:

```javascript
case LazyComponent: {
  const elementType = workInProgress.elementType;
  // Resolve the lazy component
  const Component = resolveLazy(elementType);
  // The resolved component might be a function or class
  // React creates the appropriate fiber on first resolution
  workInProgress.type = Component;

  if (shouldConstruct(Component)) {
    workInProgress.tag = ClassComponent;
    return updateClassComponent(current, workInProgress, Component, ...);
  } else {
    workInProgress.tag = FunctionComponent;
    return updateFunctionComponent(current, workInProgress, Component, ...);
  }
}
```

## SimpleMemoComponent Optimization

When `React.memo` wraps a plain function component (no custom comparison),
React uses `SimpleMemoComponent` (tag 15) which is faster than the
full `MemoComponent` (tag 14) path:

```javascript
// In beginWork, after resolving IndeterminateComponent:

if (workInProgress.type !== workInProgress.elementType) {
  // elementType might be memo(Component) while type is Component
  // Check if this is a simple memo wrapper
  const outerType = workInProgress.elementType;

  if (outerType.$$typeof === REACT_MEMO_TYPE) {
    const compare = outerType.compare;
    if (compare === null) {
      // No custom compare function → SimpleMemoComponent
      // Uses default shallow comparison — faster path
      workInProgress.tag = SimpleMemoComponent;
    }
  }
}
```

```
Tag resolution outcomes:

  typeof type         prototype.isReactComponent    Tag
  ───────────         ──────────────────────────    ───
  'string'            N/A                           HostComponent (5)
  'function'          true                          ClassComponent (1)
  'function'          false/undefined               IndeterminateComponent (2)
                                                    → FunctionComponent (0)
  object (memo)       N/A                           MemoComponent (14)
                                                    → SimpleMemoComponent (15)
  object (lazy)       N/A                           LazyComponent (16)
                                                    → FunctionComponent or ClassComponent
  object (forwardRef) N/A                           ForwardRef (11)
  Symbol (fragment)   N/A                           Fragment (7)
  Symbol (suspense)   N/A                           SuspenseComponent (13)
  Symbol (profiler)   N/A                           Profiler (12)
```

## PureComponent vs Component

```javascript
// PureComponent adds automatic shouldComponentUpdate:

class PureComponent extends Component {}
PureComponent.prototype.isPureReactComponent = true;
//                       ^^^^^^^^^^^^^^^^^^^^^
//                       Another prototype flag!

// In beginWork for ClassComponent:
function checkShouldComponentUpdate(fiber, oldProps, newProps, oldState, newState) {
  const instance = fiber.stateNode;

  if (instance.isPureReactComponent) {
    // ★ Automatic shallow comparison ★
    return (
      !shallowEqual(oldProps, newProps) ||
      !shallowEqual(oldState, newState)
    );
  }

  if (typeof instance.shouldComponentUpdate === 'function') {
    return instance.shouldComponentUpdate(newProps, newState);
  }

  return true;  // Default: always update
}
```
