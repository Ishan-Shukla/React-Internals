# MCQs: Suspense & Concurrent Features

---

### Q1. What does a component throw to trigger Suspense?
- A) An Error object
- B) A Promise (thenable)
- C) A string message
- D) `undefined`

**Answer: B**
Components signal "I'm not ready" by throwing a Promise. The Suspense boundary catches it, shows the fallback, and retries when the Promise resolves.

---

### Q2. What catches a thrown Promise during rendering?
- A) A try/catch in the component
- B) The nearest `<Suspense>` boundary in the fiber tree
- C) The error boundary
- D) The Scheduler

**Answer: B**
`throwException` walks up the fiber tree looking for a `SuspenseComponent` fiber. When found, it marks the boundary to show its fallback.

---

### Q3. What happens when a thrown Promise resolves?
- A) The component re-renders automatically
- B) React calls a "ping" function attached to the Promise, which schedules a retry render
- C) The page refreshes
- D) Nothing — the developer must manually retry

**Answer: B**
React attaches `.then(ping)` to the thrown Promise. When it resolves, `ping()` calls `ensureRootIsScheduled`, triggering a new render where the component can succeed.

---

### Q4. What happens if a component throws an Error (not a Promise)?
- A) Suspense catches it
- B) The nearest error boundary catches it
- C) React silently ignores it
- D) The entire app crashes

**Answer: B**
`throwException` differentiates: thenables (Promises) → Suspense path; Errors → error boundary path. Only error boundaries (class components with `getDerivedStateFromError`) catch errors.

---

### Q5. Can function components be error boundaries?
- A) Yes, using `useErrorBoundary` hook
- B) No — only class components with `getDerivedStateFromError` or `componentDidCatch`
- C) Yes, using try/catch in the function body
- D) Yes, using `use()` hook

**Answer: B**
Error boundaries must be class components. There is currently no hook equivalent. Libraries like `react-error-boundary` provide function component wrappers around class boundaries.

---

### Q6. What is the `OffscreenComponent` (Activity) fiber?
- A) A fiber for components outside the viewport
- B) An internal fiber used by Suspense to manage hidden/visible content toggling
- C) A fiber for background workers
- D) A fiber for portal content

**Answer: B**
Suspense wraps its content in an `OffscreenComponent` fiber. When suspended, the Offscreen hides the content (preserving state) and the fallback is shown.

---

### Q7. What is `startTransition` used for?
- A) CSS transitions
- B) Marking state updates as non-urgent so they don't block user interaction
- C) Page navigation transitions
- D) Database transactions

**Answer: B**
`startTransition(() => setState(x))` assigns the update to a `TransitionLane`, making it interruptible by urgent updates like typing or clicking.

---

### Q8. What does `isPending` from `useTransition` indicate?
- A) The transition has an error
- B) The transition's render is in progress (hasn't committed yet)
- C) The transition is pending user confirmation
- D) The network request is pending

**Answer: B**
`isPending` is `true` from the moment `startTransition` is called until the transition render commits. Useful for showing loading indicators.

---

### Q9. What happens to a Suspense fallback during a transition update?
- A) The fallback is shown immediately
- B) React avoids showing a NEW fallback; it keeps the previous UI while the transition renders in the background
- C) The fallback is never shown
- D) The fallback replaces the entire page

**Answer: B**
For transition updates (not initial mount), React "recedes" to the previous UI instead of showing a new fallback. This prevents disruptive loading states.

---

### Q10. What is the "Suspense waterfall" problem?
- A) A visual effect where spinners cascade
- B) Sequential data fetching where child components can't start fetching until parent data arrives
- C) Too many Suspense boundaries
- D) Memory leaks in Suspense

**Answer: B**
When a parent component suspends (blocks rendering), child components never get a chance to start their own data fetches. Fetches happen sequentially instead of in parallel.

---

### Q11. How do you fix a Suspense waterfall?
- A) Remove all Suspense boundaries
- B) Use parallel fetching with separate Suspense boundaries, or preload data before rendering
- C) Use `useEffect` instead
- D) Disable concurrent mode

**Answer: B**
Place components that fetch data in separate Suspense boundaries so they can suspend independently. Or preload/prefetch data before the component tree renders.

---

### Q12. What does `useDeferredValue(value)` do?
- A) Delays the value by a fixed time
- B) Returns the previous version of the value during urgent renders, updating later at lower priority
- C) Creates a debounced version of the value
- D) Memoizes the value

**Answer: B**
During an urgent render, `useDeferredValue` returns the old value. React then schedules a deferred render with the new value that can be interrupted.

---

### Q13. What is "tearing" in concurrent React?
- A) Memory corruption
- B) Different components reading different values from the same external source in the same render
- C) DOM elements overlapping
- D) CSS layout breaking

**Answer: B**
If an external mutable store changes between fiber processing (during a concurrent yield), components rendered before and after the change see different values — a torn UI.

---

### Q14. What hook prevents tearing?
- A) `useState`
- B) `useSyncExternalStore`
- C) `useMemo`
- D) `useEffect`

**Answer: B**
`useSyncExternalStore` ensures consistency by checking the snapshot before and after rendering. If it changed, React re-renders synchronously.

---

### Q15. Can React state (useState/useReducer) cause tearing?
- A) Yes, frequently
- B) No — React controls state reads, so components always see consistent state
- C) Only in production builds
- D) Only with class components

**Answer: B**
React state is read from the fiber during render. Since React controls when state updates are applied, all components in the same render see the same state values.

---

### Q16. What is `workLoopConcurrent`?
- A) A loop that runs in a Web Worker
- B) The render work loop that checks `shouldYield()` between processing fibers
- C) A loop for handling concurrent users
- D) An infinite loop

**Answer: B**
```javascript
while (workInProgress !== null && !shouldYield()) {
  performUnitOfWork(workInProgress);
}
```
It processes fibers one at a time, checking if it should yield control to the browser between each.

---

### Q17. What determines whether React uses `workLoopSync` or `workLoopConcurrent`?
- A) The component type
- B) The priority of the lanes being rendered
- C) The browser's performance
- D) A configuration flag

**Answer: B**
`SyncLane` and expired lanes use `workLoopSync` (synchronous, uninterruptible). Transition and deferred lanes use `workLoopConcurrent` (interruptible).

---

### Q18. What happens to a half-completed WIP tree when a higher-priority update arrives?
- A) It's committed immediately
- B) It's abandoned — React starts fresh with `prepareFreshStack()`
- C) It's saved for later completion
- D) It's merged with the new update

**Answer: B**
The abandoned WIP tree becomes unreachable and is garbage collected. React calls `prepareFreshStack()` to create a new WIP tree incorporating the higher-priority update.

---

### Q19. Why is it safe to abandon a partially-rendered WIP tree?
- A) The tree is backed up
- B) The render phase has no side effects — no DOM was touched, no effects ran
- C) React rolls back any changes
- D) The browser caches the partial state

**Answer: B**
React's render phase is pure — it only builds fiber objects. No DOM mutations, no effect execution. Abandoning the WIP tree is just letting objects become garbage collected.

---

### Q20. What is selective hydration?
- A) Hydrating only visible components
- B) Prioritizing hydration of components the user is interacting with
- C) Selecting which components to hydrate
- D) Hydrating in a random order

**Answer: B**
When a user clicks on un-hydrated content, React boosts that Suspense boundary's hydration priority, pausing other hydration work to make the clicked area interactive first.

---

### Q21. What happens to events on un-hydrated DOM elements?
- A) They are lost
- B) They are captured and replayed after hydration completes
- C) They are forwarded to the server
- D) They trigger a page refresh

**Answer: B**
React's event system captures events on un-hydrated areas and stores them in a replay queue. After the target boundary is hydrated, the events are replayed.

---

### Q22. What is `Fizz` in React?
- A) A CSS animation library
- B) React's streaming server renderer
- C) A testing framework
- D) A bundle optimizer

**Answer: B**
Fizz is the streaming SSR renderer (`renderToPipeableStream`). It sends HTML incrementally as components resolve, using inline scripts to swap Suspense fallbacks with real content.

---

### Q23. What does the `$RC` function in streaming SSR do?
- A) React Component initialization
- B) Replaces a Suspense fallback's HTML with the resolved content in the browser
- C) Remote Component fetching
- D) React Context propagation

**Answer: B**
`$RC(boundaryId, contentId)` is an inline script sent by Fizz. It finds the fallback placeholder and replaces it with the resolved component's HTML.

---

### Q24. What happens when an error occurs during SSR inside a Suspense boundary?
- A) The server crashes
- B) Fizz sends the fallback HTML; the client attempts to render the component from scratch
- C) The error is silently ignored
- D) The boundary is removed

**Answer: B**
The server sends fallback HTML for the error boundary's Suspense wrapper. On the client, React attempts client-side rendering. If that also fails, the error boundary shows its fallback.

---

### Q25. What is the Flight wire format?
- A) A format for airplane booking APIs
- B) The serialization format for React Server Components (RSC) — a streaming protocol for component trees
- C) A binary encoding for React elements
- D) A compression format for HTML

**Answer: B**
Flight serializes server component output into a streaming format that includes references to client components, serialized props, and the component tree structure.

---

### Q26. In RSC, what does `'use client'` mean?
- A) The file runs on the client only
- B) The file marks a boundary — components in this file and its imports are client components
- C) The file uses client-side APIs
- D) The file is dynamically imported

**Answer: B**
`'use client'` at the top of a file declares that its exports are client components. Server components can render them, but they're included in the client bundle.

---

### Q27. Can a client component import a server component?
- A) Yes, always
- B) No — client components can only RECEIVE server components as children/props, not import them
- C) Only with dynamic import
- D) Only in development

**Answer: B**
Client components can't import server components because server components don't exist in the client bundle. However, server components can be passed as `children` prop.

---

### Q28. What is the `ping` mechanism in Suspense?
- A) Network latency checking
- B) A callback attached to thrown Promises that triggers re-rendering when data is ready
- C) A heartbeat for server connections
- D) A notification sound

**Answer: B**
When Suspense catches a thrown Promise, React attaches `promise.then(ping)`. The `ping` function schedules a retry render on the Suspense boundary.

---

### Q29. What is the `SuspenseList` component (experimental)?
- A) A list that can be suspended
- B) Coordinates loading states for multiple Suspense boundaries (reveal order: forwards, backwards, together)
- C) A list of suspended components
- D) A debugging tool

**Answer: B**
`SuspenseList` controls how multiple Suspense boundaries within it reveal their content. `revealOrder="forwards"` shows them top-to-bottom, preventing out-of-order loading.

---

### Q30. How does React handle nested Suspense boundaries?
- A) Only the outermost boundary works
- B) The innermost boundary (closest to the suspending component) catches the thrown Promise
- C) All boundaries show their fallbacks
- D) Nested boundaries are not supported

**Answer: B**
Like try/catch, the nearest Suspense boundary catches the thrown Promise. Outer boundaries only activate if no inner boundary exists.

---

### Q31. What is the purpose of `React.lazy`?
- A) Making components render lazily
- B) Code-splitting — loading component code on demand via dynamic import
- C) Delaying state updates
- D) Lazy evaluation of props

**Answer: B**
`React.lazy(() => import('./Component'))` enables code-splitting. The component's code is loaded only when first rendered, triggering Suspense while loading.

---

### Q32. What does `use(context)` do in React 19?
- A) Creates a new context
- B) Reads the current value of the context (same as `useContext` but callable conditionally)
- C) Uses the context for styling
- D) Subscribes to context changes

**Answer: B**
`use(ThemeContext)` reads the current context value, just like `useContext(ThemeContext)`. The difference: `use()` can be called conditionally.

---

### Q33. What does `use(promise)` do when the promise is rejected?
- A) Returns `undefined`
- B) Throws the rejection error, which can be caught by an error boundary
- C) Returns the error object
- D) Retries indefinitely

**Answer: B**
When the Promise rejects, `use()` throws the rejection reason. This flows through the normal error boundary mechanism, showing the error fallback.

---

### Q34. What is a "wakeable" in React's Suspense internals?
- A) A component that can be woken up
- B) An object with a `.then()` method (thenable) thrown during render
- C) A notification system
- D) An alarm timer

**Answer: B**
"Wakeable" is React's internal term for the thrown Promise/thenable. React calls `wakeable.then(resolve, reject)` to attach ping callbacks.

---

### Q35. What happens when a `startTransition` update causes Suspense?
- A) Fallback is shown immediately
- B) React keeps the old UI and shows it with `isPending=true`, retrying when data is ready
- C) The transition is cancelled
- D) React falls back to synchronous rendering

**Answer: B**
Transitions avoid showing new fallbacks. React displays the previous committed UI (potentially with dimmed opacity via `isPending`) while the transition retries in the background.

---

### Q36. What is the `RetryLane` used for in Suspense?
- A) Retrying failed network requests
- B) Scheduling the retry render after a Suspense boundary's promise resolves
- C) Retrying error boundary recovery
- D) Retrying hydration

**Answer: B**
When a Suspense ping fires, the retry update is scheduled on `RetryLane`. This allows React to batch multiple Suspense retries together.

---

### Q37. What is `renderToPipeableStream` used for?
- A) Rendering to a video stream
- B) Server-side streaming rendering that sends HTML progressively
- C) Rendering to a file stream
- D) Rendering to WebSocket

**Answer: B**
`renderToPipeableStream(element, options)` returns a Node.js stream. It sends the initial HTML shell immediately, then sends resolved Suspense content as it becomes available.

---

### Q38. What is `renderToReadableStream` used for?
- A) Same as `renderToPipeableStream` but for Web Streams (edge runtimes like Cloudflare Workers)
- B) Reading component source code
- C) Rendering to a readable file
- D) Rendering markdown

**Answer: A**
`renderToReadableStream` is the Web Streams API equivalent of `renderToPipeableStream`, designed for edge environments that use the WHATWG Streams standard.

---

### Q39. Can Suspense boundaries be used without lazy loading?
- A) No, Suspense only works with `React.lazy`
- B) Yes — any component that throws a Promise triggers Suspense
- C) Only in experimental builds
- D) Only on the server

**Answer: B**
Any thrown Promise triggers Suspense. Data fetching libraries (like Relay, SWR) throw Promises to integrate with Suspense. `React.lazy` is just one use case.

---

### Q40. What is the `fallback` prop on `<Suspense>`?
- A) A function to call on error
- B) The React element to display while children are suspended
- C) A default value for missing data
- D) A backup component

**Answer: B**
`<Suspense fallback={<Spinner />}>` displays `<Spinner />` while any child component in the boundary is suspended (threw a pending Promise).

---

### Q41. What is the difference between concurrent rendering and parallel rendering?
- A) They are the same
- B) Concurrent = interleaving work on one thread; Parallel = simultaneous work on multiple threads
- C) Concurrent is faster
- D) Parallel uses Web Workers

**Answer: B**
React's concurrent rendering is single-threaded — it interleaves rendering work with browser tasks. It doesn't use multiple threads or Web Workers for rendering.

---

### Q42. Can a concurrent render be committed partially?
- A) Yes, React can commit partial trees
- B) No — a render either commits entirely or is abandoned entirely
- C) Only for Suspense boundaries
- D) Only in development mode

**Answer: B**
React commits the entire finishedWork tree atomically. There's no partial commit. If a render is interrupted, none of it is committed.

---

### Q43. What does `suppressHydrationWarning` do?
- A) Suppresses all React warnings
- B) Suppresses the hydration mismatch warning for that specific element
- C) Prevents hydration entirely
- D) Suppresses SSR errors

**Answer: B**
`<div suppressHydrationWarning>` tells React not to warn about mismatches for that element's text content. The DOM is still patched to match the client render.

---

### Q44. What is "progressive hydration"?
- A) Hydrating one component at a time
- B) Hydrating the most critical parts first, deferring less critical parts via Suspense boundaries
- C) Progressively loading CSS
- D) Hydrating based on scroll position

**Answer: B**
Using Suspense boundaries during hydration allows React to hydrate critical UI first and defer less important sections, making the page interactive sooner.

---

### Q45. What does React do when hydration discovers a mismatch in a Suspense boundary?
- A) Crashes the page
- B) Falls back to full client-side rendering for that boundary only
- C) Ignores the mismatch
- D) Retries hydration

**Answer: B**
React client-renders the mismatched Suspense boundary from scratch while keeping the rest of the page hydrated. This is a more graceful recovery than re-rendering the entire page.

---

### Q46. What is `useFormStatus` used for?
- A) Checking form validation
- B) Reading the pending state of the parent `<form>` with a server action
- C) Managing form field status
- D) Tracking form submission count

**Answer: B**
`const { pending, data, method, action } = useFormStatus()` gives status information about the closest parent `<form>` that has a server action.

---

### Q47. What is `useOptimistic` used for?
- A) Performance optimization
- B) Showing optimistic UI state that reverts when an async action settles
- C) Optimistic rendering of lists
- D) Caching optimistic values

**Answer: B**
`const [optimistic, addOptimistic] = useOptimistic(state, reducer)` lets you show a temporary "optimistic" state while an async operation is pending. It reverts to actual state when done.

---

### Q48. What is the `<form action={fn}>` feature in React 19?
- A) A way to define REST API actions
- B) Allows forms to trigger async server actions with automatic pending state management
- C) A shortcut for `onSubmit`
- D) A way to set the form's HTTP action attribute

**Answer: B**
React 19 enhances `<form>` to accept a function as `action`. The form submission triggers the function (which can be a server action), with React managing the pending state.

---

### Q49. What is `useActionState`?
- A) Redux-like action state
- B) A hook that wraps an action function, returning `[state, formAction, isPending]` for form handling
- C) A hook for managing action creators
- D) A hook for state machines

**Answer: B**
`const [state, formAction, isPending] = useActionState(action, initialState)`. The `formAction` can be passed to `<form action>`. `state` is the action's return value. `isPending` tracks execution.

---

### Q50. What determines if Suspense shows the fallback on initial mount vs update?
- A) Always shows fallback
- B) Initial mount: always shows fallback. Transition update: keeps previous UI (avoids new fallback)
- C) Never shows fallback during transition
- D) Shows fallback only if the promise takes >500ms

**Answer: B**
On initial mount (no previous UI exists), Suspense must show the fallback. During transition updates, React "recedes" to keep the previous committed UI instead of showing a new fallback.
