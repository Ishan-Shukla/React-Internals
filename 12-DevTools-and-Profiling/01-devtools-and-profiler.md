# DevTools Integration and Profiling

## How React DevTools Connects

React DevTools uses a global hook injected by the browser extension:

```javascript
// The extension injects this BEFORE React loads:
window.__REACT_DEVTOOLS_GLOBAL_HOOK__ = {
  renderers: new Map(),
  supportsFiber: true,
  inject: function(renderer) { ... },
  onCommitFiberRoot: function(rendererID, root) { ... },
  onCommitFiberUnmount: function(rendererID, fiber) { ... },
  onPostCommitFiberRoot: function(rendererID, root) { ... },
};
```

React checks for this hook during initialization:

```javascript
// packages/react-reconciler/src/ReactFiberDevToolsHook.js

export function injectInternals(internals) {
  if (typeof __REACT_DEVTOOLS_GLOBAL_HOOK__ === 'undefined') {
    return false;  // DevTools not installed
  }

  const hook = __REACT_DEVTOOLS_GLOBAL_HOOK__;

  try {
    rendererID = hook.inject({
      bundleType: __DEV__ ? 1 : 0,
      version: ReactVersion,
      rendererPackageName: 'react-dom',

      // Functions DevTools can call to inspect fibers:
      findFiberByHostInstance: getClosestInstanceFromNode,
      getCurrentFiber: () => ReactCurrentFiber.current,
      // ...
    });
  } catch (err) {
    // DevTools injection failed — ignore
  }
}
```

## Commit Notifications

After every commit, React notifies DevTools:

```javascript
// Called at the end of commitRoot:
function onCommitRoot(root) {
  if (typeof onCommitFiberRoot === 'function') {
    onCommitFiberRoot(rendererID, root, undefined, root.current.lanes);
  }
}

// DevTools walks the fiber tree to:
// 1. Build the component tree UI
// 2. Show props/state/hooks for each component
// 3. Highlight components that re-rendered
// 4. Show render timing if profiling
```

## The Profiler Component

```jsx
<Profiler id="Navigation" onRender={onRenderCallback}>
  <Navigation />
</Profiler>

function onRenderCallback(
  id,             // "Navigation" — the Profiler id
  phase,          // "mount" | "update" | "nested-update"
  actualDuration, // Time spent rendering this update (ms)
  baseDuration,   // Estimated time for full subtree render without memoization
  startTime,      // When React began rendering this update
  commitTime,     // When React committed this update
) {
  console.log(`${id} ${phase}: ${actualDuration}ms`);
}
```

## Profiler Fiber Internals

```javascript
// When creating a Profiler fiber:
function createFiberFromProfiler(pendingProps) {
  const fiber = createFiber(Profiler, pendingProps, null, mode);
  fiber.type = REACT_PROFILER_TYPE;
  fiber.elementType = REACT_PROFILER_TYPE;

  // Profiler-specific timing fields
  fiber.stateNode = {
    effectDuration: 0,      // Total effect time in subtree
    passiveEffectDuration: 0, // Total passive effect time
  };

  return fiber;
}
```

During rendering, React tracks timing:

```javascript
// In beginWork, when entering a Profiler fiber:
if (enableProfilerTimer) {
  startProfilerTimer(workInProgress);
}

// In completeWork, when leaving a Profiler fiber:
if (enableProfilerTimer) {
  stopProfilerTimerIfRunningAndRecordDelta(workInProgress);
  // Records actualDuration on the fiber
}
```

## What DevTools Shows

```
Component Tree (from fiber tree):
┌──────────────────────────────────────┐
│ ▼ App                                │
│   ▼ Header                           │
│     Logo                             │
│     Navigation                       │
│   ▼ Main                             │
│     ▼ Suspense                       │
│       ▼ AsyncContent ⏳ (suspended)   │
│     Sidebar                          │
│   Footer                             │
└──────────────────────────────────────┘

Props/State Panel (for selected component):
┌──────────────────────────────────────┐
│ AsyncContent                         │
│                                      │
│ Props:                               │
│   query: "search term"               │
│   limit: 10                          │
│                                      │
│ Hooks:                               │
│   1 State: { loading: true }         │
│   2 Effect: ƒ fetchData()            │
│   3 Memo: [item1, item2, item3]      │
│   4 Ref: { current: <div> }          │
│                                      │
│ Rendered by: Main > Suspense         │
│ Source: AsyncContent.jsx:15           │
└──────────────────────────────────────┘
```

## Profiler Flamegraph

```
Profiler Recording — Commit #3 (12.4ms)

                    ┌─ App (0.1ms) ──────────────────────────────┐
                    │                                             │
   ┌─ Header (0.2ms) ──┐    ┌─ Main (0.3ms) ────────┐   Footer
   │                    │    │                        │   (0.1ms)
  Logo   Navigation     │  ┌─List (8.2ms)──┐  Sidebar│
  (0.1ms) (0.1ms)       │  │               │  (0.1ms)│
                         │  Item  Item  Item│         │
                         │  2.1ms 3.0ms 2.4ms        │
                         │  ████  █████ ████          │

Colors:
  █ Cold (fast, < 1ms)
  ██ Warm (moderate, 1-5ms)
  ████ Hot (slow, > 5ms)

"Why did this render?" tooltip:
  List: Props changed (items)
  Item[0]: Props changed (data)
  Item[1]: Props changed (data) + State changed
  Item[2]: Parent rendered
```

## __DEV__ Only Code

React includes extensive development-only checks that are stripped
in production builds:

```javascript
// Component stack traces for warnings
if (__DEV__) {
  const componentStack = getStackByFiberInDevAndProd(fiber);
  console.error(
    'Each child in a list should have a unique "key" prop.%s',
    componentStack
  );
}

// Double-invoking effects to find bugs (StrictMode)
if (__DEV__) {
  if (fiber.mode & StrictMode) {
    // Call effects twice to catch impure effects
    invokeEffectsInDev(fiber);
    invokeEffectsInDev(fiber);  // Second invocation
  }
}

// Prop type validation
if (__DEV__) {
  checkPropTypes(type.propTypes, props, 'prop', getComponentName(type));
}
```

## StrictMode Double-Rendering

```jsx
<StrictMode>
  <App />
</StrictMode>
```

In development, StrictMode:

1. **Double-invokes component functions** — to catch impure render logic
2. **Double-invokes effect setup/cleanup** — to catch missing cleanup
3. **Runs deprecated API warnings** — findDOMNode, legacy context, etc.

```
StrictMode rendering:
  Component()        → discard result
  Component()        → use this result (second call)

StrictMode effects:
  effect setup()     → run cleanup immediately
  effect cleanup()
  effect setup()     → keep this one (second call)

This catches bugs like:
  - Effects that don't clean up subscriptions
  - State updates in render that depend on external state
  - Side effects during render (mutations, API calls)
```
