# Context Propagation Internals

## The Problem Context Solves

Passing data through many layers of components without explicit props
at every level ("prop drilling"):

```
WITHOUT Context:                        WITH Context:
  App (theme="dark")                      App
   └── Layout (theme="dark")               └── ThemeProvider value="dark"
        └── Page (theme="dark")                  └── Layout (no theme prop!)
             └── Card (theme="dark")                  └── Page (no theme prop!)
                  └── Button (theme="dark")                └── Card (no theme prop!)
                                                                └── Button
                                                                    useContext → "dark"
```

## Context Data Structure

```javascript
// createContext returns:
const ThemeContext = {
  $$typeof: Symbol(react.context),
  _currentValue: defaultValue,     // Current value (primary renderer)
  _currentValue2: defaultValue,    // Current value (secondary renderer)
  Provider: {
    $$typeof: Symbol(react.provider),
    _context: ThemeContext,         // Back-reference
  },
  Consumer: ThemeContext,           // Consumer IS the context itself
};
```

## Provider: How Values Are Pushed Down

When React processes a `<Context.Provider>` fiber in `beginWork`:

```javascript
// Simplified from ReactFiberBeginWork.js

function updateContextProvider(current, workInProgress, renderLanes) {
  const context = workInProgress.type._context;
  const newValue = workInProgress.pendingProps.value;
  const oldValue = workInProgress.memoizedProps?.value;

  // ★ Push the new value onto the context stack ★
  pushProvider(workInProgress, context, newValue);

  if (oldValue !== undefined) {
    // Check if value changed
    if (!Object.is(oldValue, newValue)) {
      // ═══ CONTEXT VALUE CHANGED ═══
      // Must find all consumers in the subtree and mark them for re-render
      propagateContextChange(workInProgress, context, renderLanes);
    }
  }

  // Continue reconciling children
  reconcileChildren(current, workInProgress, newProps.children, renderLanes);
  return workInProgress.child;
}
```

## The Context Value Stack

React maintains a **stack** of context values as it traverses the tree.
This handles nested providers correctly:

```javascript
// Internal stack (simplified)
const valueStack = [];
let index = -1;

function pushProvider(providerFiber, context, nextValue) {
  index++;
  valueStack[index] = context._currentValue;
  context._currentValue = nextValue;  // Set new value globally
}

function popProvider(context) {
  context._currentValue = valueStack[index];  // Restore previous value
  valueStack[index] = null;
  index--;
}
```

```
Tree traversal with nested providers:

<ThemeContext.Provider value="dark">         stack: ["dark"]
  <Layout>                                  _currentValue = "dark"
    <ThemeContext.Provider value="light">    stack: ["dark", "light"]
      <Panel>                               _currentValue = "light"
        useContext(ThemeContext) → "light"   ← reads _currentValue
      </Panel>
    </ThemeContext.Provider>                 pop → stack: ["dark"]
    <Sidebar>                               _currentValue = "dark"
      useContext(ThemeContext) → "dark"      ← reads _currentValue
    </Sidebar>
  </Layout>
</ThemeContext.Provider>                    pop → stack: []
```

## propagateContextChange: The Deep Scan

When a Provider's value changes, React must find ALL consumers in the subtree.
This is a **deep walk** that cannot be short-circuited by `memo` or `shouldComponentUpdate`:

```javascript
// Simplified from ReactFiberNewContext.js

function propagateContextChange(workInProgress, context, renderLanes) {
  let fiber = workInProgress.child;

  while (fiber !== null) {
    let nextFiber = null;

    // Check if this fiber reads from this context
    const list = fiber.dependencies;
    if (list !== null) {
      nextFiber = fiber.child;

      let dependency = list.firstContext;
      while (dependency !== null) {
        if (dependency.context === context) {
          // ═══ FOUND A CONSUMER ═══

          // For class components: enqueue a forced update
          if (fiber.tag === ClassComponent) {
            const update = createUpdate(renderLanes);
            update.tag = ForceUpdate;
            enqueueUpdate(fiber, update);
          }

          // Mark the fiber and all its ancestors with the render lane
          fiber.lanes |= renderLanes;
          if (fiber.alternate !== null) {
            fiber.alternate.lanes |= renderLanes;
          }

          // Walk UP, marking childLanes on all ancestors
          scheduleContextWorkOnParentPath(fiber.return, renderLanes);

          // Mark the dependency list
          list.lanes |= renderLanes;

          break;  // Found the matching context, stop checking deps
        }
        dependency = dependency.next;
      }
    } else {
      // No dependencies — check children
      if (fiber.tag === ContextProvider && fiber.type === workInProgress.type) {
        // This is a NESTED provider for the same context
        // Its subtree gets a different value — DON'T scan into it
        nextFiber = null;
      } else {
        nextFiber = fiber.child;
      }
    }

    // Depth-first traversal
    if (nextFiber !== null) {
      nextFiber.return = fiber;
      fiber = nextFiber;
    } else {
      // Go to sibling or back up
      fiber = fiber;
      while (fiber !== null) {
        if (fiber === workInProgress) return;
        if (fiber.sibling !== null) {
          fiber.sibling.return = fiber.return;
          fiber = fiber.sibling;
          break;
        }
        fiber = fiber.return;
      }
    }
  }
}
```

## Why Context Bypasses memo and shouldComponentUpdate

```
<ThemeContext.Provider value={newTheme}>
  <App>                                    ← re-renders (parent of provider)
    <MemoizedHeader />                     ← memo BAILS OUT (props same)
      <Logo />                             ← SKIPPED (parent bailed out)
      <NavItem />                          ← SKIPPED... unless it reads context!

  BUT: propagateContextChange runs BEFORE the normal reconciliation!
  It walks the ENTIRE subtree looking for consumers, regardless of memo.

  If <NavItem> uses useContext(ThemeContext):
    → propagateContextChange finds it
    → Marks NavItem.lanes |= renderLane
    → Marks all ancestors' childLanes
    → Even though MemoizedHeader bails out,
      the childLanes on Header are set
    → bailoutOnAlreadyFinishedWork sees childLanes
    → Clones children instead of skipping subtree
    → NavItem re-renders with new context value!

  ┌──────────────────────────────────────────────────────────┐
  │  MemoizedHeader (memo)                                   │
  │  ├── Props same? YES → bailout?                          │
  │  │   └── BUT childLanes has work → clone children        │
  │  │                                                       │
  │  ├── Logo (no context) → actually skipped ✓              │
  │  └── NavItem (reads ThemeContext)                         │
  │      → lanes marked by propagateContextChange            │
  │      → RE-RENDERS despite memo parent! ✓                 │
  └──────────────────────────────────────────────────────────┘
```

## How useContext Registers a Consumer

```javascript
// From ReactFiberHooks.js

function readContext(context) {
  const value = context._currentValue;

  // Create a context dependency entry
  const contextItem = {
    context: context,
    memoizedValue: value,
    next: null,
  };

  // Append to the fiber's dependency list
  if (lastContextDependency === null) {
    currentlyRenderingFiber.dependencies = {
      lanes: NoLanes,
      firstContext: contextItem,
    };
    lastContextDependency = contextItem;
  } else {
    lastContextDependency = lastContextDependency.next = contextItem;
  }

  return value;
}
```

```
fiber.dependencies for a component that reads 2 contexts:

  {
    lanes: NoLanes,
    firstContext: ─────► { context: ThemeCtx, value: "dark", next: ──► }
                                                                       │
                         { context: AuthCtx, value: {user}, next: null }◄┘
  }

propagateContextChange walks every fiber's dependency list
looking for a matching context reference.
```

## Performance Implications

```
Context change cost = O(N) where N = total fibers under the Provider

  <AppContext.Provider>        ← value changes
    └── 5000 fiber nodes       ← ALL 5000 are visited by propagateContextChange
        (only 3 read context)  ← but React doesn't know until it checks each one

This is why:
1. Don't put your ENTIRE app under one mega-context
2. Split contexts by update frequency
3. Context is not a replacement for proper state management in large apps

BAD:
  <AppContext.Provider value={{ theme, user, cart, notifications, ... }}>
    <EntireApp />  ← EVERYTHING re-scans on ANY value change
  </AppContext.Provider>

GOOD:
  <ThemeContext.Provider value={theme}>
    <UserContext.Provider value={user}>
      <CartContext.Provider value={cart}>
        <App />  ← Only relevant subtrees are scanned
      </CartContext.Provider>
    </UserContext.Provider>
  </ThemeContext.Provider>
```

## Context + useMemo Pattern (Before Compiler)

```javascript
// Common optimization: memoize the context value object
function ThemeProvider({ children }) {
  const [theme, setTheme] = useState('dark');

  // Without useMemo: new object every render → all consumers re-render!
  // const value = { theme, setTheme };

  // With useMemo: same object if theme hasn't changed
  const value = useMemo(() => ({ theme, setTheme }), [theme]);

  return (
    <ThemeContext.Provider value={value}>
      {children}
    </ThemeContext.Provider>
  );
}

// Why this matters:
// Object.is({ theme: 'dark' }, { theme: 'dark' }) → false (new object!)
// Without useMemo, EVERY re-render of ThemeProvider triggers
// propagateContextChange, even if theme didn't change.
```
