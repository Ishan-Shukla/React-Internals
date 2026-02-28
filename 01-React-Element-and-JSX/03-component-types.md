# Component Types and Special Elements

## How React Identifies Component Types

When the reconciler encounters an element, it checks the `type` field and
the `$$typeof` tag to determine how to process it:

```javascript
// Simplified from ReactFiber.js — createFiberFromTypeAndProps()

function createFiberFromElement(element) {
  const type = element.type;

  if (typeof type === 'string') {
    // 'div', 'span', 'input' → HostComponent
    return createFiberFromHostComponent(element);
  }

  if (typeof type === 'function') {
    if (type.prototype && type.prototype.isReactComponent) {
      // Class component — has isReactComponent on prototype
      return createFiberFromClassComponent(element);
    }
    // Function component
    return createFiberFromFunctionComponent(element);
  }

  if (typeof type === 'object' && type !== null) {
    switch (type.$$typeof) {
      case REACT_MEMO_TYPE:
        return createFiberFromMemo(element);
      case REACT_LAZY_TYPE:
        return createFiberFromLazy(element);
      case REACT_FORWARD_REF_TYPE:
        return createFiberFromForwardRef(element);
      case REACT_PROVIDER_TYPE:
        return createFiberFromProvider(element);
      case REACT_CONTEXT_TYPE:
        return createFiberFromConsumer(element);
    }
  }

  // Fragment, Suspense, Profiler are Symbols
  switch (type) {
    case REACT_FRAGMENT_TYPE:
      return createFiberFromFragment(element);
    case REACT_SUSPENSE_TYPE:
      return createFiberFromSuspense(element);
    case REACT_PROFILER_TYPE:
      return createFiberFromProfiler(element);
  }
}
```

## Fiber Tag Types

Each fiber has a `tag` field — a numeric constant identifying its kind:

```javascript
// packages/react-reconciler/src/ReactWorkTags.js

export const FunctionComponent       = 0;
export const ClassComponent          = 1;
export const IndeterminateComponent  = 2;  // Before first render
export const HostRoot                = 3;  // Root of fiber tree
export const HostPortal              = 4;  // ReactDOM.createPortal
export const HostComponent           = 5;  // 'div', 'span', etc.
export const HostText                = 6;  // Plain text nodes
export const Fragment                = 7;
export const Mode                    = 8;  // <StrictMode>, <ConcurrentMode>
export const ContextConsumer         = 9;
export const ContextProvider         = 10;
export const ForwardRef              = 11;
export const Profiler                = 12;
export const SuspenseComponent       = 13;
export const MemoComponent           = 14;
export const SimpleMemoComponent     = 15; // memo() on a plain function
export const LazyComponent           = 16;
export const OffscreenComponent      = 22;
```

## Component Type Decision Tree

```
                        element.type
                            │
            ┌───────────────┼──────────────────┐
            ▼               ▼                  ▼
        string          function            object/symbol
            │               │                  │
            ▼           ┌───┴───┐         ┌────┴─────────┐
       HostComponent    │       │         │   $$typeof   │
       (tag: 5)     has .proto  no        │              │
                   .isReact     proto     ▼              ▼
                   Component?       REACT_MEMO_TYPE  Symbol check
                       │            REACT_LAZY_TYPE  ┌────────┐
                   ┌───┴───┐        REACT_FWD_REF    │Fragment│
                   ▼       ▼        REACT_PROVIDER   │Suspense│
              ClassComp  FuncComp   REACT_CONTEXT    │Profiler│
              (tag: 1)   (tag: 0)                    └────────┘
```

## React.memo Internals

```javascript
// packages/react/src/ReactMemo.js

export function memo(type, compare) {
  const elementType = {
    $$typeof: REACT_MEMO_TYPE,
    type: type,                    // The wrapped component
    compare: compare || null,      // Custom comparison function
  };
  return elementType;
}
```

When the reconciler hits a memo fiber:

```
                  memo(MyComponent)
                        │
                        ▼
              ┌─────────────────┐
              │ Has props        │
              │ changed?         │
              │                  │
              │ Uses compare fn  │
              │ if provided,     │
              │ else shallowEqual│
              └────────┬────────┘
                  ┌────┴────┐
                  ▼         ▼
               Changed    Same
                  │         │
                  ▼         ▼
            Re-render    Bail out
            child        (reuse old fiber)
```

## React.lazy Internals

```javascript
// packages/react/src/ReactLazy.js

export function lazy(ctor) {
  const payload = {
    _status: Uninitialized,  // -1
    _result: ctor,           // The () => import() function
  };

  const lazyType = {
    $$typeof: REACT_LAZY_TYPE,
    _payload: payload,
    _init: lazyInitializer,
  };

  return lazyType;
}

function lazyInitializer(payload) {
  if (payload._status === Uninitialized) {
    const ctor = payload._result;
    const thenable = ctor();       // Call import()

    payload._status = Pending;
    payload._result = thenable;

    thenable.then(
      moduleObject => {
        payload._status = Resolved;
        payload._result = moduleObject.default;
      },
      error => {
        payload._status = Rejected;
        payload._result = error;
      }
    );
  }

  if (payload._status === Resolved) {
    return payload._result;        // Return the component
  }
  throw payload._result;           // Throw the promise (Suspense catches it)
}
```

```
            lazy(() => import('./Heavy'))
                        │
              ┌─────────┴──────────┐
              ▼                    ▼
         First render          After load
              │                    │
              ▼                    ▼
         Call import()         Status = Resolved
         Status = Pending      Return component
         Throw promise         Render normally
              │
              ▼
         Suspense catches
         Shows fallback
              │
              ▼
         Promise resolves
         Re-render subtree
```

## forwardRef Internals

```javascript
// packages/react/src/ReactForwardRef.js

export function forwardRef(render) {
  const elementType = {
    $$typeof: REACT_FORWARD_REF_TYPE,
    render: render,  // (props, ref) => ReactElement
  };
  return elementType;
}
```

The reconciler calls `type.render(props, ref)` instead of `type(props)`,
passing the ref as the second argument.

## Context Internals

```javascript
// packages/react/src/ReactContext.js

export function createContext(defaultValue) {
  const context = {
    $$typeof: REACT_CONTEXT_TYPE,
    _currentValue: defaultValue,   // Used by renderers
    _currentValue2: defaultValue,  // Used by secondary renderer
    Provider: null,
    Consumer: null,
  };

  // The Provider element type
  context.Provider = {
    $$typeof: REACT_PROVIDER_TYPE,
    _context: context,
  };

  // The Consumer is the context itself (reads _currentValue)
  context.Consumer = context;

  return context;
}
```

Context value propagation:

```
         <ThemeContext.Provider value="dark">
                      │
    ┌─────────────────┼──────────────────┐
    ▼                 ▼                  ▼
  <Header>          <Main>            <Footer>
    │                 │                  │
    ▼                 ▼                  ▼
  useContext       No context         useContext
  (ThemeContext)   usage              (ThemeContext)
    │                                    │
    ▼                                    ▼
  "dark"                               "dark"

When Provider value changes:
1. React walks the subtree
2. Finds all fibers that consume this context
3. Marks them for re-render (even if parent bails out via memo/shouldComponentUpdate)
4. Context consumers ALWAYS re-render when context value changes
```
