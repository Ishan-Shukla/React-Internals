# MCQs: Hooks Internals

---

### Q1. How are hooks stored internally on a fiber?
- A) As an array on `fiber.hooks`
- B) As a linked list on `fiber.memoizedState`
- C) As a Map on `fiber.hookState`
- D) As properties on the component function

**Answer: B**
Hooks form a singly-linked list: `fiber.memoizedState → hook1 → hook2 → ... → null`. Each hook node has `{ memoizedState, baseState, baseQueue, queue, next }`.

---

### Q2. How many hook dispatchers does React use?
- A) 1
- B) 2
- C) 3 (mount, update, rerender)
- D) 4

**Answer: C**
`HooksDispatcherOnMount` (first render), `HooksDispatcherOnUpdate` (subsequent renders), and `HooksDispatcherOnRerender` (render-phase state updates). DEV builds add more with validation.

---

### Q3. What function wraps the execution of a function component?
- A) `callComponent()`
- B) `renderWithHooks()`
- C) `executeHooks()`
- D) `runFunctionComponent()`

**Answer: B**
`renderWithHooks` sets up the hooks dispatcher, resets the cursor, calls the component function, then validates the hook state afterward.

---

### Q4. What does `mountWorkInProgressHook()` do?
- A) Updates an existing hook
- B) Creates a new hook object and appends it to the linked list
- C) Mounts a hook to the DOM
- D) Initializes the work loop

**Answer: B**
During mount, each hook call creates a new hook object `{ memoizedState: null, baseState: null, baseQueue: null, queue: null, next: null }` and links it to the previous hook.

---

### Q5. What does `updateWorkInProgressHook()` do?
- A) Creates a new hook
- B) Advances the cursor to the next hook in the linked list and clones it from the current fiber
- C) Updates the DOM
- D) Runs the hook's side effect

**Answer: B**
During updates, `updateWorkInProgressHook` moves to the next hook from the current fiber's list, clones it onto the WIP fiber, and returns it. The cursor must advance in the same order as mount.

---

### Q6. What is the "eager state" optimization in `useState`?
- A) State is computed lazily
- B) React computes the new state at dispatch time; if equal to current, it skips scheduling
- C) State updates are applied eagerly during render
- D) State is pre-computed on mount

**Answer: B**
When `dispatch(action)` is called and no other updates are pending, React immediately computes `reducer(currentState, action)`. If `Object.is(eagerState, currentState)`, no render is scheduled.

---

### Q7. What comparison function does React use for hook dependencies?
- A) `===` (strict equality)
- B) `==` (loose equality)
- C) `Object.is()`
- D) `JSON.stringify` comparison

**Answer: C**
`areHookInputsEqual` compares each dependency with `Object.is`. Unlike `===`, `Object.is(NaN, NaN)` is `true` and `Object.is(+0, -0)` is `false`.

---

### Q8. What is the internal structure of a `useEffect` hook's `memoizedState`?
- A) A boolean indicating if it should run
- B) An effect object: `{ tag, create, destroy, deps, next }`
- C) The cleanup function
- D) The dependency array

**Answer: B**
Each effect has `tag` (bitmask of effect type), `create` (setup function), `destroy` (cleanup function), `deps` (dependency array), and `next` (circular linked list pointer).

---

### Q9. How are effects connected on a fiber?
- A) In a linear linked list
- B) In a circular linked list via `fiber.updateQueue.lastEffect`
- C) In an array
- D) In a tree structure

**Answer: B**
Effects form a circular linked list. `fiber.updateQueue.lastEffect` points to the last effect, and `lastEffect.next` points to the first. This makes it easy to iterate all effects.

---

### Q10. What is the `HookHasEffect` tag?
- A) Indicates the hook exists
- B) Indicates the effect should run this commit (dependencies changed or first mount)
- C) Indicates the effect has an error
- D) Indicates the hook is memoized

**Answer: B**
When effect dependencies change (or on mount), the effect is tagged with `HookHasEffect`. During commit, only effects with this tag are executed.

---

### Q11. What does `useCallback(fn, deps)` store in `hook.memoizedState`?
- A) Just the function `fn`
- B) A tuple `[fn, deps]`
- C) A wrapper function
- D) The memoized return value of `fn`

**Answer: B**
`useCallback` stores `[callback, deps]` in `memoizedState`. On update, if deps haven't changed, the stored callback is returned. It's equivalent to `useMemo(() => fn, deps)`.

---

### Q12. What does `useMemo(create, deps)` store in `hook.memoizedState`?
- A) Just the computed value
- B) A tuple `[computedValue, deps]`
- C) The create function
- D) A cache Map

**Answer: B**
`useMemo` stores `[value, deps]`. On update, if deps match, `value` is returned directly. If deps differ, `create()` is called and the new `[value, deps]` stored.

---

### Q13. What does `useRef(initialValue)` store in `hook.memoizedState`?
- A) The `initialValue` directly
- B) A `{ current: initialValue }` object
- C) A DOM reference
- D) A frozen object

**Answer: B**
`useRef` creates `{ current: initialValue }` on mount and stores it. On update, it simply returns the same object — it's never re-created or compared.

---

### Q14. Can `useRef` trigger a re-render when `ref.current` changes?
- A) Yes, always
- B) Yes, if the ref is used in JSX
- C) No, refs are completely invisible to React's rendering system
- D) Only in strict mode

**Answer: C**
Mutating `ref.current` never triggers a re-render. React doesn't track ref mutations. Refs are escape hatches from the reactive model.

---

### Q15. What happens when you call `useState` with a function as the initial value?
- A) The function is stored as the state
- B) The function is called once (lazy initializer) and its return value becomes the state
- C) React throws an error
- D) The function is called every render

**Answer: B**
`useState(() => expensiveComputation())` calls the function only during mount. The return value is stored as the initial state. On subsequent renders, the initializer is completely ignored.

---

### Q16. What is the `basicStateReducer` used by `useState`?
- A) `(state, action) => state + action`
- B) `(state, action) => typeof action === 'function' ? action(state) : action`
- C) `(state, action) => { ...state, ...action }`
- D) `(state, action) => action`

**Answer: B**
If the action is a function, it's called with the current state (functional updater pattern). Otherwise, the action itself becomes the new state.

---

### Q17. What is `dispatchReducerAction`?
- A) A Redux middleware
- B) The internal function bound to a hook's update queue that enqueues state updates
- C) A function that dispatches DOM events
- D) A function that runs the reducer synchronously

**Answer: B**
When you call `setCount(1)` or `dispatch(action)`, you're calling `dispatchReducerAction` bound to the fiber and the hook's queue. It creates an update object and schedules rendering.

---

### Q18. What does the `dispatch` function identity guarantee?
- A) It changes every render
- B) It's stable — the same function reference across all renders
- C) It's stable only with `useCallback`
- D) It changes when state changes

**Answer: B**
The `dispatch` function is bound once during mount with `bind(null, fiber, queue)`. The queue object persists across renders, so `dispatch` is always the same reference.

---

### Q19. What happens if `useEffect` has no dependency array?
- A) It runs only on mount
- B) It runs after every render
- C) It never runs
- D) It runs once then cleans up

**Answer: B**
With no deps argument, the effect runs after every render. React compares `undefined` deps with the previous `undefined`, and since there's nothing to compare, it always re-runs.

---

### Q20. What happens if `useEffect` has an empty dependency array `[]`?
- A) It runs after every render
- B) It runs only once (on mount) and cleanup runs on unmount
- C) It never runs
- D) It runs on the first two renders

**Answer: B**
Empty deps means "depends on nothing" — the effect fires on mount and never again. Its cleanup fires on unmount.

---

### Q21. In what order do effect cleanups and setups run during an update?
- A) Setup → Cleanup
- B) All cleanups first, then all setups
- C) Interleaved: cleanup1 → setup1 → cleanup2 → setup2
- D) Random order

**Answer: B**
React 18 runs ALL cleanup functions first (for all components that need it), then runs ALL setup functions. They're not interleaved.

---

### Q22. When does `useLayoutEffect` cleanup run?
- A) After browser paint
- B) Synchronously during the commit phase, before the new setup runs
- C) Before the render phase
- D) During garbage collection

**Answer: B**
Layout effect cleanups run synchronously in the commit phase's layout sub-phase, before the new layout effect setup runs and before the browser paints.

---

### Q23. What happens when `setState` is called in a `useLayoutEffect`?
- A) The update is batched with the next event
- B) React immediately and synchronously re-renders before the browser paints
- C) The update is deferred to the next frame
- D) React logs a warning and ignores it

**Answer: B**
`setState` in `useLayoutEffect` triggers a synchronous re-render. The DOM is updated again before the browser has a chance to paint, preventing visual flicker.

---

### Q24. What does `useInsertionEffect` do?
- A) Inserts elements into the DOM
- B) Runs synchronously before DOM mutations — designed for CSS-in-JS libraries
- C) Inserts hooks into the linked list
- D) Inserts components into the tree

**Answer: B**
`useInsertionEffect` fires before any DOM mutations in the commit phase. It's specifically designed for CSS-in-JS libraries to inject `<style>` tags before `useLayoutEffect` tries to measure.

---

### Q25. What is the execution order: useInsertionEffect → useLayoutEffect → useEffect?
- A) useEffect → useLayoutEffect → useInsertionEffect
- B) useInsertionEffect → useLayoutEffect → useEffect
- C) useLayoutEffect → useInsertionEffect → useEffect
- D) All run simultaneously

**Answer: B**
`useInsertionEffect` fires first (before DOM mutations), then `useLayoutEffect` (after mutations, before paint), then `useEffect` (after paint).

---

### Q26. How does `useId` generate deterministic IDs?
- A) Using a global counter
- B) Using Math.random() with a seed
- C) Based on the fiber's position in the tree
- D) Using a UUID library

**Answer: C**
`useId` generates IDs from the fiber's tree position, encoded as a compact string. This ensures the same component at the same position gets the same ID on server and client.

---

### Q27. Why is `useId` tree-position-based instead of counter-based?
- A) For better randomness
- B) To ensure server and client generate matching IDs (counters differ due to rendering order)
- C) For shorter IDs
- D) For security

**Answer: B**
Server and client may render components in different orders (streaming, selective hydration). Tree position is the same on both, ensuring hydration compatibility.

---

### Q28. What does `useSyncExternalStore` do?
- A) Synchronizes state between components
- B) Subscribes to an external store with tear-safe consistency in concurrent mode
- C) Syncs state to localStorage
- D) Creates a synchronized mutex for state access

**Answer: B**
`useSyncExternalStore(subscribe, getSnapshot)` ensures external store reads are consistent across all components in a concurrent render, preventing tearing.

---

### Q29. How does `useSyncExternalStore` prevent tearing?
- A) By locking the store
- B) By checking if the snapshot changed after rendering; if so, it retries synchronously
- C) By copying the store state
- D) By disabling concurrent rendering

**Answer: B**
After rendering, React calls `getSnapshot()` again. If the value differs from what was used during render, React falls back to synchronous rendering to guarantee consistency.

---

### Q30. What does `useTransition` return?
- A) `[transitionState, startTransition]`
- B) `[isPending, startTransition]`
- C) `[transition, endTransition]`
- D) `[isLoading, setLoading]`

**Answer: B**
`useTransition()` returns `[isPending, startTransition]`. `isPending` is `true` while the transition is rendering. `startTransition(callback)` wraps state updates.

---

### Q31. What is the key difference between `useTransition` and `useDeferredValue`?
- A) They are identical
- B) `useTransition` wraps setState calls; `useDeferredValue` wraps a value (useful when you can't control the setState)
- C) `useTransition` is sync; `useDeferredValue` is async
- D) `useTransition` is for class components

**Answer: B**
`useTransition` gives you `startTransition` to control which setState is deferred. `useDeferredValue(value)` defers a value you receive (e.g., from props), useful when you don't control the update.

---

### Q32. What does `useDeferredValue(value)` return during an urgent render?
- A) The new value immediately
- B) The previous value (deferred), then updates in a background render
- C) `undefined`
- D) A Promise resolving to the new value

**Answer: B**
During the urgent render, `useDeferredValue` returns the old value. A second render at lower priority is scheduled with the new value. The component renders twice with different values.

---

### Q33. What is the purpose of `useDebugValue`?
- A) Debugging state values in production
- B) Displaying a label in React DevTools for custom hooks
- C) Logging values to the console
- D) Validating hook values

**Answer: B**
`useDebugValue(value)` shows a debug label in React DevTools next to the custom hook. It's only for development tooling and has no runtime effect.

---

### Q34. What is `useImperativeHandle` used for?
- A) Imperative DOM manipulation
- B) Customizing the value exposed via a `ref` when using `forwardRef`
- C) Handling keyboard shortcuts
- D) Managing imperative animations

**Answer: B**
`useImperativeHandle(ref, createHandle, deps)` lets you customize what value the parent receives through a `ref`. Instead of exposing the DOM node, you expose a custom object.

---

### Q35. What happens if you call `useState` inside a `useEffect`?
- A) It works normally
- B) React throws an error — hooks can only be called during render
- C) The state is ignored
- D) It creates a memory leak

**Answer: B**
Hooks must be called during render (at the top level of a component function). Calling `useState` inside `useEffect` violates the rules of hooks and throws an error.

---

### Q36. What does `use()` (React 19) do when given a resolved Promise?
- A) Suspends
- B) Returns the resolved value immediately without suspending
- C) Throws an error
- D) Returns the Promise object

**Answer: B**
If the Promise is already resolved (status fulfilled), `use()` returns the value synchronously. Only pending Promises cause suspension.

---

### Q37. Can `use()` be called conditionally?
- A) No, it follows the same rules as other hooks
- B) Yes, `use()` is unique — it can be called inside conditions and loops
- C) Only inside `useEffect`
- D) Only inside class components

**Answer: B**
`use()` is special — it can be called conditionally, inside loops, or after early returns. This is because it doesn't store state in the hooks linked list the same way.

---

### Q38. What does `useOptimistic` do?
- A) Optimizes rendering performance
- B) Provides an optimistic state value that reverts to actual state when an async action completes
- C) Caches API responses
- D) Pre-fetches data

**Answer: B**
`useOptimistic(state, updateFn)` returns an optimistic state that can be temporarily modified during an async action. When the action completes, the optimistic value reverts to the actual state.

---

### Q39. What does `useFormStatus` return?
- A) The form's validation state
- B) `{ pending, data, method, action }` for the parent `<form>` with a server action
- C) The form's submit event
- D) The number of form fields

**Answer: B**
`useFormStatus()` returns the status of the nearest parent `<form>` with an action. `pending` is `true` while the action is executing.

---

### Q40. What does `useActionState` replace?
- A) `useState`
- B) `useFormState` (renamed in React 19)
- C) `useReducer`
- D) `useContext`

**Answer: B**
`useActionState` was previously named `useFormState`. It manages form action state: `const [state, formAction, isPending] = useActionState(action, initialState)`.

---

### Q41. What is the `HookPassive` tag on an effect?
- A) The effect is passive (does nothing)
- B) The effect uses `useEffect` (runs asynchronously after paint)
- C) The effect has been skipped
- D) The effect is read-only

**Answer: B**
`HookPassive` identifies the effect as a passive effect (`useEffect`). `HookLayout` identifies layout effects (`useLayoutEffect`). These tags determine when the effect runs during commit.

---

### Q42. What happens when a component with hooks throws during render?
- A) Hooks state is lost and the error boundary catches it
- B) Hooks state is preserved and the error is silently ignored
- C) React retries the render with the same hooks state
- D) The entire application crashes

**Answer: A**
If a component throws during render, the current render is abandoned. The error bubbles to the nearest error boundary. The component's hooks state is not committed and is lost.

---

### Q43. What is the `numberOfReRenders` limit for render-phase updates?
- A) 10
- B) 25
- C) 50
- D) 100

**Answer: B**
If `setState` is called during render (render-phase update), React re-runs the component. If this happens more than 25 times, React throws "Too many re-renders" to prevent infinite loops.

---

### Q44. What dispatcher does React use during render-phase re-renders?
- A) `HooksDispatcherOnMount`
- B) `HooksDispatcherOnUpdate`
- C) `HooksDispatcherOnRerender`
- D) `HooksDispatcherOnError`

**Answer: C**
`HooksDispatcherOnRerender` is a third dispatcher specifically for render-phase state updates. It processes queued updates without creating new hooks.

---

### Q45. What does `mountState` return?
- A) Just the state value
- B) `[initialState, dispatch]` — the state and its updater function
- C) A state object
- D) A reducer

**Answer: B**
`mountState(initialState)` initializes the hook, creates the update queue, binds the dispatch function, and returns `[hook.memoizedState, dispatch]`.

---

### Q46. What does `Object.is(NaN, NaN)` return?
- A) `false`
- B) `true`
- C) `undefined`
- D) Throws an error

**Answer: B**
`Object.is(NaN, NaN)` returns `true`, unlike `NaN === NaN` which is `false`. This matters for hook dependency comparison — React won't re-run effects when deps go from `NaN` to `NaN`.

---

### Q47. What does `Object.is(+0, -0)` return?
- A) `true`
- B) `false`
- C) `undefined`
- D) Throws an error

**Answer: B**
`Object.is(+0, -0)` returns `false`, unlike `+0 === -0` which is `true`. This rarely matters in practice but is a subtle difference in React's dependency comparison.

---

### Q48. What does calling `setCount(count)` with the same value do?
- A) Always triggers a re-render
- B) May skip the re-render via eager state bailout if `Object.is(newState, currentState)`
- C) Throws an error
- D) Resets the count to 0

**Answer: B**
If `Object.is(action, currentState)` is `true` and no other updates are pending, React may bail out entirely — no render is scheduled.

---

### Q49. Where are `useEffect` cleanup functions stored?
- A) In the component function's closure
- B) In `effect.destroy` on the effect object in the hooks linked list
- C) In a global cleanup registry
- D) On the DOM node

**Answer: B**
The setup function returns a cleanup function. React stores it as `effect.destroy`. On the next render (or unmount), React calls `effect.destroy()` before running the new setup.

---

### Q50. What does `flushPassiveEffects()` do?
- A) Flushes the DOM update queue
- B) Runs all pending `useEffect` cleanups and setups that were deferred after commit
- C) Flushes the event queue
- D) Runs `useLayoutEffect`

**Answer: B**
`flushPassiveEffects` is called asynchronously after commit (via `MessageChannel`). It runs all pending passive effect cleanups, then all setups, in tree order.
