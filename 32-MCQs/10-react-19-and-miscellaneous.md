# MCQs: React 19, Security, Performance & Miscellaneous

---

### Q1. What is the `use()` hook in React 19?
- A) A universal hook for using any React feature
- B) A hook that can read Promises (suspending if unresolved) and Contexts, callable conditionally
- C) A replacement for all other hooks
- D) A hook for using external libraries

**Answer: B**
`use(promise)` suspends if pending, returns value if resolved. `use(context)` reads context. Unlike other hooks, it can be called inside conditions and loops.

---

### Q2. What is `useActionState` in React 19?
- A) A hook for Redux actions
- B) A hook that wraps an async action function, returning `[state, formAction, isPending]` for progressive form handling
- C) A hook for managing action history
- D) A hook for server-side actions only

**Answer: B**
`const [state, action, isPending] = useActionState(fn, initialState)` wraps an action (sync or async). `action` can be passed to `<form action={action}>`.

---

### Q3. What is a Server Action in React 19?
- A) An API route
- B) A function marked with `'use server'` that executes on the server but can be called from client components
- C) A Redux action that runs on the server
- D) A GraphQL mutation

**Answer: B**
```javascript
'use server';
async function saveData(formData) {
  await db.save(formData.get('name'));
}
```
Client components call this function; React serializes the call to the server.

---

### Q4. How does React 19 handle `ref` differently from React 18?
- A) Refs are removed
- B) `ref` is passed as a regular prop — `forwardRef` is no longer needed for function components
- C) Refs are now async
- D) Refs only work on class components

**Answer: B**
In React 19, function components receive `ref` as a regular prop in `props.ref`. `forwardRef` still works but is no longer required.

---

### Q5. What new capability do ref callbacks have in React 19?
- A) They can be async
- B) They can return a cleanup function (called when ref is detached)
- C) They receive the fiber instead of the DOM node
- D) They are called during render

**Answer: B**
```jsx
<div ref={(node) => {
  setup(node);
  return () => cleanup(node);  // New: cleanup on detach
}} />
```

---

### Q6. What does `<form action={fn}>` do in React 19?
- A) Sets the HTML action attribute to a URL
- B) Triggers `fn` on form submission, with React managing pending state and progressive enhancement
- C) Redirects to `fn`
- D) Nothing special

**Answer: B**
React intercepts form submission, calls `fn(formData)`, manages pending state via `useFormStatus`, and provides progressive enhancement (works without JS if the action is a URL).

---

### Q7. What does `useFormStatus` read?
- A) Form validation errors
- B) The pending/action/data/method state of the nearest parent `<form>` with an action
- C) The number of form fields
- D) The form's submission history

**Answer: B**
`const { pending, data, method, action } = useFormStatus()`. `pending` is `true` while the form's action is executing. Must be used inside a component that's a child of a `<form>`.

---

### Q8. What does React 19's `preload()` function do?
- A) Preloads component code
- B) Adds `<link rel="preload">` to the document head for early resource loading
- C) Pre-renders a route
- D) Preloads state from cache

**Answer: B**
`preload('/font.woff2', { as: 'font', type: 'font/woff2', crossOrigin: 'anonymous' })` tells the browser to start downloading the resource before it's actually needed.

---

### Q9. What is `prefetchDNS()` in React 19?
- A) Prefetches DNS records for a domain to reduce connection latency
- B) Fetches data before rendering
- C) Caches DNS responses
- D) Resolves domain names server-side

**Answer: A**
`prefetchDNS('https://api.example.com')` adds `<link rel="dns-prefetch">` to resolve DNS early, reducing latency for subsequent requests to that domain.

---

### Q10. What is the `precedence` prop on `<link>` stylesheets in React 19?
- A) CSS specificity value
- B) Controls the insertion order of stylesheets — React groups and orders them by precedence
- C) Loading priority
- D) z-index value

**Answer: B**
`<link rel="stylesheet" href="..." precedence="high" />` tells React where to insert this stylesheet relative to others. React maintains correct cascade order regardless of component render timing.

---

### Q11. How does React prevent XSS through `$$typeof`?
- A) By encrypting element data
- B) Symbols can't be serialized to JSON, so forged React elements from API responses are rejected
- C) By validating HTML tags
- D) By using Content Security Policy

**Answer: B**
`$$typeof: Symbol.for('react.element')` can't appear in JSON payloads. If an attacker injects a JSON object mimicking a React element, it lacks the Symbol and React rejects it.

---

### Q12. What does React do with `javascript:` URLs?
- A) Renders them normally
- B) Blocks/warns about them to prevent XSS (`<a href="javascript:alert('xss')">`)
- C) Converts them to `https:`
- D) Ignores the `href` attribute

**Answer: B**
React 16.9+ warns about `javascript:` URLs in href, action, and src attributes. Some versions actively block them. This prevents click-based XSS attacks.

---

### Q13. What does `dangerouslySetInnerHTML` bypass?
- A) The reconciler
- B) React's text content escaping — raw HTML is inserted directly via `innerHTML`
- C) The commit phase
- D) State management

**Answer: B**
Normal `{text}` content is escaped via `textContent`. `dangerouslySetInnerHTML={{ __html: rawHtml }}` uses `innerHTML`, which parses and executes HTML/scripts. Developer is responsible for sanitization.

---

### Q14. What is the memory impact of a fiber vs a React Element?
- A) Fibers use less memory
- B) Fibers are heavier — they store state, effects, lanes, tree pointers; Elements are lightweight plain objects
- C) They use the same memory
- D) Elements are heavier

**Answer: B**
React Elements have ~5 fields (`$$typeof, type, key, ref, props`). Fibers have 25+ fields including state, effects, lanes, tree pointers, alternate, and flags.

---

### Q15. What is a common cause of memory leaks in React?
- A) Using too many components
- B) Forgetting to return cleanup from `useEffect` (uncleared subscriptions, timers, listeners)
- C) Using JSX
- D) Using too many props

**Answer: B**
```javascript
useEffect(() => {
  const id = setInterval(fn, 1000);
  // Missing: return () => clearInterval(id);
}, []);
```
The interval runs forever, retaining closure references to the fiber and state.

---

### Q16. How does React clean up fiber trees after unmount?
- A) Immediate garbage collection
- B) Nullifies `return`, `child`, `stateNode`, `memoizedState`, `dependencies` to break references
- C) Moves fibers to a pool
- D) Uses `WeakRef`

**Answer: B**
React explicitly sets fiber fields to `null` during deletion. This ensures the fiber and all objects it references become eligible for garbage collection.

---

### Q17. What is the performance anti-pattern of creating objects in render?
- A) It's fine
- B) New objects mean new references, defeating memo/bailout optimizations and causing unnecessary child re-renders
- C) It causes memory leaks
- D) It's slower than strings

**Answer: B**
```jsx
<Child style={{ color: 'red' }} />  // New object each render!
```
Each render creates a new `style` object. Even with `React.memo`, `shallowEqual` sees different references. Fix: hoist constants or use `useMemo`.

---

### Q18. What does `AbortController` help with in React?
- A) Aborting component rendering
- B) Cancelling fetch requests in `useEffect` cleanup to prevent setState on unmounted components
- C) Aborting state updates
- D) Cancelling animations

**Answer: B**
```javascript
useEffect(() => {
  const controller = new AbortController();
  fetch(url, { signal: controller.signal }).then(r => setData(r));
  return () => controller.abort();
}, [url]);
```

---

### Q19. What is the `act()` utility designed for?
- A) Animation control
- B) Ensuring all state updates, effects, and renders complete before test assertions
- C) Action creators
- D) Accessibility testing

**Answer: B**
`act(() => { button.click(); })` flushes all pending updates synchronously, ensuring the DOM reflects the final state before assertions.

---

### Q20. What does React's `Profiler` measure with `actualDuration`?
- A) Time the component was mounted
- B) Time spent rendering this Profiler's subtree (including all descendants) in the current commit
- C) Time until the next update
- D) Network request duration

**Answer: B**
`actualDuration` measures the total rendering time (beginWork + completeWork) for the Profiler's subtree. It's only the render phase, not commit or effects.

---

### Q21. What is `baseDuration` in the Profiler?
- A) The base case rendering time
- B) The estimated time to render the subtree without memoization (every component re-renders)
- C) The initial mount duration
- D) The minimum possible duration

**Answer: B**
`baseDuration` sums the most recent render duration of every fiber in the subtree, representing the worst-case render time without any bailouts or memoization.

---

### Q22. What is React's approach to accessibility (a11y)?
- A) React automatically makes apps accessible
- B) React supports standard HTML accessibility attributes and warns about missing alt text, but developers must implement a11y
- C) React has a built-in screen reader
- D) Accessibility is handled by React DevTools

**Answer: B**
React passes through all `aria-*` attributes to the DOM. It warns about known issues (like missing `<img>` alt). But proper a11y requires developer effort.

---

### Q23. What does `React.StrictMode` do in production?
- A) Enables strict rendering
- B) Nothing — StrictMode has zero production runtime cost
- C) Throws errors on violations
- D) Logs warnings

**Answer: B**
All StrictMode behaviors (double render, double effects) only occur in development. In production, StrictMode is completely inert — no performance impact.

---

### Q24. What is the `onRecoverableError` option in `createRoot`?
- A) Error logging configuration
- B) A callback for errors React recovered from (like hydration mismatches) that the developer should know about
- C) Error boundary configuration
- D) Auto-recovery settings

**Answer: B**
`createRoot(el, { onRecoverableError: (error) => log(error) })` is called for errors React handled internally (like hydration mismatch recovery) but that may indicate issues.

---

### Q25. What is the maximum recommended component render time for smooth 60fps?
- A) 50ms
- B) 16ms (one frame at 60fps)
- C) 5ms
- D) 100ms

**Answer: B**
At 60fps, each frame is ~16.67ms. A component render should ideally be well under this to leave time for browser layout, paint, and other JavaScript. React's 5ms time slice helps.

---

### Q26. What does `React.cache()` (experimental) do?
- A) Caches DOM nodes
- B) Memoizes a function's return value per request during server rendering
- C) Browser caching configuration
- D) Component caching

**Answer: B**
`React.cache(fn)` creates a memoized version of `fn`. During a single server request, repeated calls with the same args return cached results. Cache is per-request, not global.

---

### Q27. What is the `thenableState` in Suspense internals?
- A) The state of the then keyword
- B) Internal state tracking which thenables (Promises) have been thrown and their resolution status
- C) The conditional state
- D) The pending state

**Answer: B**
React tracks thrown thenables to know which Promises are pending, resolved, or rejected. This state helps React decide when to retry rendering.

---

### Q28. What is an "update" object in React's state system?
- A) A DOM mutation instruction
- B) An object `{ lane, action, next, ... }` representing a queued state change
- C) An API response
- D) A component lifecycle event

**Answer: B**
Each `setState(action)` creates an update: `{ lane, action, hasEagerState, eagerState, next }`. Updates form a circular linked list on the hook's queue.

---

### Q29. What is the circular linked list structure of update queues?
- A) First points to last, last points to first
- B) `queue.pending` points to the LAST update; `last.next` points to the FIRST update
- C) All updates point to each other
- D) It's a doubly-linked list

**Answer: B**
`pending` points to the most recently added update (tail). `tail.next` points to the head (oldest). This makes it O(1) to append (insert after tail, before head).

---

### Q30. What does `ensureRootIsScheduled` check before scheduling?
- A) If the root DOM exists
- B) If there are pending lanes AND if an existing Scheduler task already covers the needed priority
- C) If the browser is idle
- D) If the component tree is valid

**Answer: B**
If `getNextLanes(root)` returns `NoLanes`, no scheduling needed. If a task exists at the right priority (`root.callbackPriority`), no re-scheduling needed. Otherwise, cancel old and schedule new.

---

### Q31. What is React's "render-phase side effects" rule?
- A) Side effects are allowed during render
- B) The render phase must be pure — no DOM mutations, no subscriptions, no state mutations, no API calls
- C) Side effects are logged during render
- D) Side effects are batched during render

**Answer: B**
Render phase must be pure because React may call component functions multiple times, abandon results, or restart rendering. Side effects belong in effects or event handlers.

---

### Q32. What is `fiber.updateQueue` for a `HostRoot` fiber?
- A) Child fibers to process
- B) A queue of `ReactDOM.render()`/`root.render()` payloads (the element to render)
- C) DOM update instructions
- D) Event handlers

**Answer: B**
The HostRoot's update queue processes `root.render(<App />)` calls. The element (e.g., `<App />`) is enqueued as an update and processed during reconciliation.

---

### Q33. What does the React Compiler's HIR stand for?
- A) High-level Intermediate Representation
- B) Hook Integration Runtime
- C) Hierarchical Instance Registry
- D) Host Interface Resolution

**Answer: A**
The React Compiler transforms component code into HIR (High-level Intermediate Representation) — a control flow graph used for dependency analysis and automatic memoization.

---

### Q34. What is `useMemoCache` used for?
- A) Caching API responses
- B) The React Compiler's internal cache for auto-memoized values on the fiber
- C) Browser cache management
- D) Memoizing component trees

**Answer: B**
`useMemoCache(size)` allocates a fixed-size cache array on the fiber. The compiler stores memoized values and checks dependencies against cache slots.

---

### Q35. What is the `Rules of React` that the compiler validates?
- A) ESLint rules
- B) Components must be pure, hooks must follow call-order rules, no mutation during render, idempotent rendering
- C) Naming conventions
- D) File organization rules

**Answer: B**
The compiler validates and relies on: pure components (no side effects), stable hook order, no prop/state mutation during render, and idempotent rendering.

---

### Q36. What is the `SuspenseException` in React internals?
- A) An error thrown when Suspense fails
- B) A special exception object used internally to distinguish thrown Promises from thrown Errors
- C) A JavaScript built-in exception
- D) A debugging exception

**Answer: B**
React wraps thrown thenables in a `SuspenseException` to distinguish them from regular errors in the catch logic of `renderRootConcurrent`/`renderRootSync`.

---

### Q37. What is the `ping` function in Suspense?
- A) A network ping
- B) A function attached to a thrown Promise's `.then()` that re-schedules rendering when data is ready
- C) A health check
- D) An animation frame callback

**Answer: B**
`promise.then(ping)` → when Promise resolves, `ping` calls `markRootPinged(root, wakeable, pingedLanes)` → `ensureRootIsScheduled(root)` → retry render.

---

### Q38. What is `commitRootImpl`?
- A) The root of the commit implementation
- B) The main function that orchestrates all three commit sub-phases (before mutation, mutation, layout) plus passive effect scheduling
- C) A DOM implementation
- D) A test utility

**Answer: B**
`commitRootImpl` is the core commit function. It calls `commitBeforeMutationEffects`, `commitMutationEffects`, does the tree swap, calls `commitLayoutEffects`, and schedules passive effects.

---

### Q39. What does the `NoFlags` constant represent?
- A) All flags are set
- B) No side effects — the fiber doesn't need any commit-phase processing
- C) The fiber is disabled
- D) The fiber is in error state

**Answer: B**
`NoFlags = 0b0` means no Placement, no Update, no Deletion, no Ref changes — the fiber can be skipped during commit.

---

### Q40. What is `workInProgressRoot`?
- A) The DOM root element
- B) A module-level variable tracking which FiberRoot is currently being rendered
- C) The root of the WIP tree
- D) A configuration object

**Answer: B**
`workInProgressRoot` points to the `FiberRoot` currently being processed. If a render is interrupted and a new root needs rendering, React checks this to determine if it should restart.

---

### Q41. What is `executionContext` in React?
- A) The JavaScript execution context
- B) A bitmask tracking what React is currently doing (rendering, committing, batching, etc.)
- C) The DOM context
- D) The server context

**Answer: B**
`executionContext` tracks whether React is in `RenderContext`, `CommitContext`, `BatchedContext`, etc. This determines how updates are processed (batched vs immediate).

---

### Q42. What is the purpose of `ReactCurrentBatchConfig`?
- A) Configuring batch sizes
- B) Tracking whether the current execution is inside a `startTransition` call
- C) Configuring render batches
- D) Setting up batch processing

**Answer: B**
`ReactCurrentBatchConfig.transition` is set to a non-null value inside `startTransition`, signaling that `requestUpdateLane` should return a `TransitionLane` instead of `DefaultLane`.

---

### Q43. What does `markRootUpdated(root, lane, eventTime)` do?
- A) Marks a DOM element as updated
- B) Adds the lane to `root.pendingLanes` and records the event time for expiration calculations
- C) Marks the root for garbage collection
- D) Updates the root's CSS

**Answer: B**
After creating an update, `markRootUpdated` sets `root.pendingLanes |= lane` and stores `root.eventTimes[laneIndex] = eventTime` for starvation prevention.

---

### Q44. What is the `HostConfig` in react-reconciler?
- A) Server configuration
- B) A set of platform-specific functions (createInstance, appendChild, etc.) that the reconciler calls
- C) Browser configuration
- D) Build configuration

**Answer: B**
The host config is the abstraction layer between the platform-agnostic reconciler and the platform-specific renderer. React DOM, React Native, and custom renderers each provide their own.

---

### Q45. What host config function creates a DOM element?
- A) `createElement`
- B) `createInstance(type, props, rootContainer, hostContext, internalInstanceHandle)`
- C) `new Element(type)`
- D) `document.create(type)`

**Answer: B**
`createInstance` is the host config function called during `completeWork`. For React DOM, it calls `document.createElement(type)` or `document.createElementNS(namespace, type)`.

---

### Q46. What is `fiber.effectTag` in older React versions?
- A) A CSS effect tag
- B) The predecessor to `fiber.flags` — a bitmask of side effects
- C) An HTML tag for effects
- D) A debugging label

**Answer: B**
`effectTag` was renamed to `flags` in newer React versions. The concept is the same: a bitmask indicating what commit-phase work is needed.

---

### Q47. What is React's approach to animations?
- A) Built-in animation system
- B) React doesn't have built-in animation — it provides the rendering model; use libraries like Framer Motion or React Spring
- C) CSS-only animations
- D) WebGL-based animations

**Answer: B**
React focuses on rendering UI. For animations, use CSS transitions/animations, Framer Motion, React Spring, or GSAP. React's concurrent features can help with smooth transitions.

---

### Q48. What is `requestIdleCallback` and why doesn't React use it?
- A) A browser API for scheduling idle work; React doesn't use it because it has 50ms max timeout and inconsistent timing
- B) A React internal function
- C) A deprecated browser API
- D) A Node.js API

**Answer: A**
`requestIdleCallback` fires during browser idle time but has a 50ms max budget, doesn't fire during user interaction, and has inconsistent cross-browser behavior. React's Scheduler gives more control.

---

### Q49. What are the benefits of React 18's `createRoot` over `ReactDOM.render`?
- A) No benefits
- B) Enables concurrent features, automatic batching everywhere, improved Suspense, selective hydration
- C) Only changes the API syntax
- D) Faster initial render

**Answer: B**
`createRoot` is the gateway to React 18's concurrent features. `ReactDOM.render` (legacy root) forces synchronous rendering and legacy batching behavior.

---

### Q50. What is the significance of the number 31 in React's lane system?
- A) The number of React hooks
- B) JavaScript bitwise operators work on 32-bit integers; bit 31 is the sign bit, leaving 31 usable lane bits
- C) The number of React component types
- D) The maximum tree depth

**Answer: B**
JavaScript's bitwise operators operate on 32-bit signed integers. The sign bit (bit 31) can't be used for lanes, leaving 31 bits (positions 0-30) for lane assignments.
