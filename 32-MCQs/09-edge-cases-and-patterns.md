# MCQs: Edge Cases, Patterns & Gotchas

---

### Q1. What happens when you call `setState` during render (inside the component body)?
- A) React throws an error
- B) React re-runs the component immediately within the same beginWork call (render-phase update)
- C) The update is deferred to the next render
- D) React ignores it

**Answer: B**
React detects `didScheduleRenderPhaseUpdate = true`, uses `HooksDispatcherOnRerender`, and re-executes the component within the same `beginWork` call. Limited to 25 re-runs.

---

### Q2. What is the maximum number of render-phase re-renders before React throws?
- A) 10
- B) 25
- C) 50
- D) 100

**Answer: B**
If `numberOfReRenders > 25`, React throws "Too many re-renders. React limits the number of renders to prevent an infinite loop."

---

### Q3. What is the "render then bailout" pattern?
- A) Rendering nothing
- B) A component's function IS called, but its children DON'T re-render because hooks detected no state changes
- C) Skipping the entire render
- D) Bailing out during commit

**Answer: B**
When `oldProps !== newProps` (different reference), React must call the component. But if `didReceiveUpdate` stays `false` (no hook changes), children are cloned, not re-rendered.

---

### Q4. What renders on screen when `items = []` and you write `{items.length && <List />}`?
- A) Nothing
- B) `<List />`
- C) The string `"0"` (the number 0 as text)
- D) `false`

**Answer: C**
`items.length` is `0` (falsy number). `0 && <List />` evaluates to `0`. React renders `0` as the text `"0"`. Fix: `{items.length > 0 && <List />}`.

---

### Q5. What happens when two siblings have the same `key`?
- A) Both render normally
- B) Dev warning logged; at runtime, the second key overwrites the first in the reconciler's Map, causing incorrect behavior
- C) React deduplicates them
- D) The second one is ignored

**Answer: B**
Duplicate keys cause the Map-based lookup (Pass 2 of list reconciliation) to overwrite the first fiber. The first may be incorrectly deleted, leading to state corruption.

---

### Q6. What causes the common "each child in a list should have a unique key" warning?
- A) Missing `id` prop
- B) Children in an array/iterator lack a `key` prop
- C) Using the wrong key value
- D) Using numeric keys

**Answer: B**
React requires keys on array children for efficient reconciliation. Without keys, React falls back to index-based matching which can cause state bugs.

---

### Q7. Does a component re-render when its parent re-renders, even if props haven't changed?
- A) No, React checks props automatically
- B) Yes — unless the component is wrapped in `React.memo` or the parent uses `useMemo` for the element
- C) Only class components
- D) Only in development

**Answer: B**
By default, when a parent re-renders, it creates new JSX elements with new props objects. Since `oldProps !== newProps` (different reference), children re-render. `React.memo` adds shallow comparison.

---

### Q8. What is the stale closure problem?
- A) A memory leak
- B) A closure capturing an old state value that never updates, causing handlers to use outdated data
- C) A syntax error
- D) A browser bug

**Answer: B**
When a `useEffect` closure captures a state variable and the effect doesn't re-run (empty deps), the captured value remains the initial value forever. Fix: use functional updaters or include deps.

---

### Q9. What does defining a component inside another component cause?
- A) Nothing unusual
- B) A new function reference each render, causing React to unmount/remount the inner component every time
- C) A syntax error
- D) Better performance

**Answer: B**
```jsx
function Outer() {
  function Inner() { return <div /> }  // New reference each render!
  return <Inner />;  // Type changes every render → unmount/remount
}
```
Always define components outside or use `useMemo` for dynamic component creation.

---

### Q10. What is the correct way to reset a component's state?
- A) Call `setState(initialState)` for every state variable
- B) Change the component's `key` prop
- C) Use `componentWillUnmount`
- D) Use `forceUpdate`

**Answer: B**
Changing a component's `key` causes React to destroy the old instance and create a new one with fresh state. This is the cleanest way to reset all internal state.

---

### Q11. Can `useEffect` cleanups access the latest state?
- A) Yes, always
- B) No — cleanups capture the state from the render in which the effect was created (closure)
- C) Only with `useRef`
- D) Only in class components

**Answer: B**
Cleanup functions are closures over the render in which they were created. They see the state values from that render, not the latest state. Use `useRef` to access current values.

---

### Q12. What is the difference between `key` and `ref` as props?
- A) Both are passed to the component via `props`
- B) Both are "special" — React extracts them during element creation; they're NOT in `props`
- C) `key` is in props, `ref` is not
- D) `ref` is in props, `key` is not

**Answer: B**
Both `key` and `ref` are extracted by React during `createElement`/`jsx`. The component cannot access `props.key` (it's `undefined`). In React 19, `ref` IS passed as a regular prop.

---

### Q13. What happens when a component returns an array?
- A) React throws an error
- B) React creates a Fragment-like structure, rendering each array element as a sibling
- C) Only the first element renders
- D) The array is converted to a string

**Answer: B**
```jsx
function App() { return [<A key="a" />, <B key="b" />]; }
```
This is equivalent to wrapping in a Fragment. Each element needs a `key`.

---

### Q14. Can a component return a string?
- A) No
- B) Yes — it's rendered as a text node
- C) Only class components
- D) Only in React 18+

**Answer: B**
Components can return strings, numbers, booleans (`true`/`false` → nothing), `null` (→ nothing), arrays, elements, or Portals. Strings become `HostText` fibers.

---

### Q15. What is the "key reset" pattern?
- A) Resetting all keys to null
- B) Changing a component's `key` to force destruction and recreation with fresh state
- C) Clearing the key cache
- D) Resetting the key counter

**Answer: B**
```jsx
<UserForm key={userId} userId={userId} />
```
When `userId` changes, `key` changes → React destroys old `UserForm` and creates a new one. All state (form inputs, etc.) is fresh.

---

### Q16. What does `React.memo` NOT protect against?
- A) Props changes
- B) Context changes — context updates bypass memo
- C) State changes inside the component
- D) Both B and C

**Answer: D**
`React.memo` only prevents re-renders from parent prop changes. Internal `useState`/`useReducer` changes and context updates bypass memo and still cause re-renders.

---

### Q17. What is the compound component pattern?
- A) Components made of multiple HTML elements
- B) Related components that share implicit state via Context, working together as a unit
- C) Components that return other components
- D) Nested component definitions

**Answer: B**
```jsx
<Tabs>
  <TabList><Tab>A</Tab></TabList>
  <TabPanels><TabPanel>Content A</TabPanel></TabPanels>
</Tabs>
```
`Tabs` provides Context; `Tab` and `TabPanel` consume it. No `cloneElement` or prop drilling needed.

---

### Q18. Why is `useEffect` with empty deps not exactly the same as `componentDidMount`?
- A) They are exactly the same
- B) `useEffect` runs asynchronously after paint; `componentDidMount` runs synchronously before paint
- C) `componentDidMount` doesn't support cleanup
- D) `useEffect` runs before mount

**Answer: B**
`componentDidMount` runs in the Layout phase (before paint). `useEffect([])` runs in the Passive phase (after paint). For synchronous post-mount work, use `useLayoutEffect`.

---

### Q19. What is the purpose of `useCallback`?
- A) Creating callback functions
- B) Memoizing a function reference to prevent child re-renders when passed as prop
- C) Making functions async
- D) Error handling in callbacks

**Answer: B**
`useCallback(fn, deps)` returns a stable function reference that only changes when deps change. Without it, a new function is created each render, causing children (with memo) to re-render.

---

### Q20. Is `useCallback(fn, deps)` equivalent to `useMemo(() => fn, deps)`?
- A) No, they work differently
- B) Yes — `useCallback` is syntactic sugar for `useMemo` that returns a function
- C) Only in production
- D) Only for pure functions

**Answer: B**
Internally, `useCallback(fn, deps)` and `useMemo(() => fn, deps)` produce the same result. `useCallback` is a convenience API that skips the wrapper arrow function.

---

### Q21. What does `useEffect` return?
- A) A cleanup function or `undefined`
- B) A state value
- C) A promise
- D) Nothing — `useEffect` doesn't return

**Answer: A**
The `useEffect` callback can optionally return a cleanup function. If it returns nothing (or `undefined`), there's no cleanup. Returning a non-function value is an error.

---

### Q22. What happens if `useEffect` returns a Promise (async function)?
- A) Works perfectly
- B) React ignores the Promise
- C) React logs a warning — effect callbacks should not be async (the returned Promise isn't a cleanup function)
- D) React awaits the Promise

**Answer: C**
`useEffect(async () => { ... })` returns a Promise, which React tries to call as a cleanup function. This causes a warning. Instead: `useEffect(() => { async function f() {...} f(); }, [])`.

---

### Q23. What is the "forking" behavior of update queues?
- A) Creating multiple copies of the queue
- B) When an update is skipped (wrong lane), base state "forks" — subsequent updates are rebased on a frozen state
- C) Splitting the queue into parallel streams
- D) Git-like branching of state

**Answer: B**
If update A is skipped (its lane isn't being processed), `baseState` stays before A. When A's lane is eventually processed, all updates from A onward are replayed from `baseState` to ensure consistency.

---

### Q24. What is `React.lazy` combined with `Suspense` used for?
- A) Lazy initialization of state
- B) Code-splitting — loading component code on demand, showing a fallback while loading
- C) Lazy rendering of off-screen content
- D) Deferred state updates

**Answer: B**
```jsx
const LazyComponent = React.lazy(() => import('./Heavy'));
<Suspense fallback={<Spinner />}><LazyComponent /></Suspense>
```
The component's code is loaded only when first rendered.

---

### Q25. What happens when you pass `ref={null}` to an element?
- A) React ignores it
- B) React detaches any previously attached ref (sets it to null)
- C) React throws an error
- D) The element doesn't render

**Answer: A**
Passing `ref={null}` is treated as having no ref. If a previous render had a ref and the new one has `null`, React detaches the old ref.

---

### Q26. What is the `children` prop when a component has no children?
- A) An empty array `[]`
- B) `undefined`
- C) `null`
- D) An empty string `""`

**Answer: B**
`<Component />` with no children means `props.children` is `undefined`. `<Component></Component>` (empty) is also `undefined`. Only explicit content creates a `children` prop.

---

### Q27. What is the `children` prop when a component has exactly one child?
- A) An array with one element
- B) The single child element directly (not wrapped in an array)
- C) A Fragment containing the child
- D) Always undefined

**Answer: B**
With one child, `props.children` IS that child (not an array). With multiple children, it's an array. This is why `React.Children` utilities are needed for safe iteration.

---

### Q28. What does `React.cloneElement` do to the `key`?
- A) Always generates a new key
- B) Preserves the original key unless a new key is explicitly provided in the new props
- C) Always removes the key
- D) Hashes the original key

**Answer: B**
`cloneElement(element, { key: 'new' })` changes the key. `cloneElement(element, { className: 'x' })` preserves the original key.

---

### Q29. Can you use hooks inside a class component?
- A) Yes, directly in the render method
- B) No — hooks only work inside function components (or custom hooks)
- C) Only `useEffect`
- D) Only `useState`

**Answer: B**
Hooks rely on the function component calling convention and the hooks dispatcher. They cannot be used in class component methods.

---

### Q30. What is the `displayName` property used for?
- A) SEO optimization
- B) Setting the component's display name in React DevTools and error messages
- C) Rendering the component name on screen
- D) Registering the component

**Answer: B**
`Component.displayName = 'MyComponent'` helps debugging. React DevTools and error messages use it. Especially useful for HOCs: `withAuth(Component).displayName = 'withAuth(Component)'`.

---

### Q31. What does `React.memo` do when props haven't changed?
- A) Returns null
- B) Returns the previous render result without calling the component function
- C) Still calls the component but skips commit
- D) Caches the DOM nodes

**Answer: B**
When `shallowEqual(prevProps, nextProps)` returns `true`, React skips calling the component function entirely and reuses the previous fiber's children.

---

### Q32. What is the difference between `createElement` and `cloneElement`?
- A) `createElement` makes new elements from scratch; `cloneElement` copies an existing element merging new props
- B) They are identical
- C) `createElement` is for DOM; `cloneElement` is for components
- D) `cloneElement` is deprecated

**Answer: A**
`createElement(type, props, children)` creates a brand new element. `cloneElement(element, newProps)` copies the element's type and key, merging newProps over existing props.

---

### Q33. What happens with `useEffect` on SSR?
- A) Effects run on the server
- B) Effects are skipped entirely on the server — they only run on the client after hydration
- C) Effects throw an error on the server
- D) Effects are converted to synchronous

**Answer: B**
`useEffect` and `useLayoutEffect` callbacks do not run during SSR. They run on the client after hydration. `useLayoutEffect` logs a warning in SSR (use `useEffect` instead).

---

### Q34. What is the `Symbol.for('react.element')` vs `Symbol('react.element')` difference for `$$typeof`?
- A) They're the same
- B) `Symbol.for` creates a global shared Symbol (same across realms); `Symbol()` creates a unique one
- C) `Symbol.for` is faster
- D) `Symbol()` is more secure

**Answer: B**
`Symbol.for('react.element')` is in the global Symbol registry — the same Symbol is returned for the same key string across all code. This ensures React elements from different bundles are recognized.

---

### Q35. What is a "controlled" input in React's internal implementation?
- A) An input with `onChange`
- B) An input where React maintains and restores `value`/`checked` properties to match fiber state after each commit
- C) An input managed by a form library
- D) An input with validation

**Answer: B**
React internally tracks controlled inputs and re-sets their DOM value after commit to ensure they match `fiber.memoizedProps.value`, preventing DOM from diverging from React state.

---

### Q36. What is the `shouldYield` function checking in user-space terms?
- A) If the component should yield state
- B) If ~5ms have elapsed since the current rendering work started, time to let the browser handle events
- C) If the CPU is overloaded
- D) If the network is busy

**Answer: B**
`shouldYield` (called `shouldYieldToHost`) checks if the time budget (~5ms) is exhausted. If yes, React pauses rendering and yields to the browser's event loop.

---

### Q37. What is the `Alt` tag in a fiber?
- A) HTML alt attribute
- B) There is no "Alt" tag — it's `alternate`, a pointer to the fiber's counterpart in the other tree
- C) Alternative rendering mode
- D) Accessibility tag

**Answer: B**
`fiber.alternate` connects current and WIP fibers. "Alt" isn't a tag — `alternate` is a field. `fiber.tag` is a numeric work type identifier.

---

### Q38. Can React render to a `<canvas>` element?
- A) Yes, React DOM renders to canvas natively
- B) Not directly — you need a custom renderer like `react-three-fiber` or `react-canvas`
- C) Only with `dangerouslySetInnerHTML`
- D) Only in React Native

**Answer: B**
React DOM targets the DOM. For Canvas/WebGL, custom renderers built on `react-reconciler` map React components to canvas drawing calls.

---

### Q39. What is the effect of wrapping everything in `React.memo`?
- A) Always improves performance
- B) Adds overhead (shallow comparison on every render) that may WORSEN performance for frequently-changing props
- C) No effect
- D) Causes memory leaks

**Answer: B**
`React.memo` adds a `shallowEqual` comparison cost. If props change frequently, this comparison runs every time AND the component still re-renders. Only memo components with stable props.

---

### Q40. What does `flushSync` do inside a `useEffect`?
- A) Nothing special
- B) Forces synchronous rendering — the DOM is updated before the next line of the effect runs
- C) Throws an error
- D) Defers the update

**Answer: B**
`flushSync` inside `useEffect` forces a synchronous render and commit. Other pending passive effects may not have run yet, but the `flushSync` update completes immediately.

---

### Q41. What is the purpose of the `key` prop on a Fragment?
- A) Fragments can't have keys
- B) Allows Fragments to be used in lists where siblings need unique identification
- C) Improves Fragment rendering performance
- D) Sets a CSS key-frame

**Answer: B**
`<React.Fragment key={id}>` allows fragments in arrays/lists. The short syntax `<>` cannot have keys — use the explicit `<React.Fragment key={...}>` form.

---

### Q42. What does React do when an element changes from `<div>` to `<span>` at the same position?
- A) Updates the existing DOM node's tag
- B) Destroys the `<div>` and all children, creates a new `<span>` from scratch
- C) Wraps both in a fragment
- D) Ignores the change

**Answer: B**
Different types at the same position trigger full subtree destruction and recreation. HTML tags can't be morphed; React must remove the old and insert the new.

---

### Q43. What is the Concurrent React "opt-in" model?
- A) All rendering is concurrent by default
- B) Only updates wrapped in `startTransition` or using deferred values render concurrently; other updates are sync
- C) Developers must enable concurrent mode per component
- D) Concurrent mode is always disabled

**Answer: B**
`createRoot` enables concurrent features, but not all updates use concurrent rendering. Only transitions and deferred values use `workLoopConcurrent`. Click handlers use `workLoopSync`.

---

### Q44. What happens when a Promise thrown by a component rejects?
- A) Suspense retries indefinitely
- B) On retry, the component throws the rejection error, which is caught by the nearest error boundary
- C) The page crashes
- D) React silently catches it

**Answer: B**
When the Promise rejects, React re-renders the component. The `use()` hook or data library re-throws the rejection error, which flows to the error boundary path.

---

### Q45. What is the difference between `React.Children.count` and `children.length`?
- A) They return the same value
- B) `React.Children.count` flattens nested arrays and counts leaf elements; `children.length` only works if children is an array
- C) `React.Children.count` is slower
- D) `children.length` counts fragments

**Answer: B**
`children.length` crashes if children is a single element (not an array) or `undefined`. `React.Children.count` safely handles all cases and flattens nested structures.

---

### Q46. What does the `ref` callback receive when the element unmounts?
- A) The DOM element
- B) `null`
- C) `undefined`
- D) Nothing — it's not called

**Answer: B**
Callback refs are called with the DOM node on mount and with `null` on unmount. In React 19, returning a cleanup function from the callback ref replaces this pattern.

---

### Q47. What is `React.createRef()` used for?
- A) Creating refs for function components
- B) Creating a ref object `{ current: null }` for class components
- C) Creating a reference to a React element
- D) Creating a reference count

**Answer: B**
`React.createRef()` returns `{ current: null }`. Primarily for class components. In function components, `useRef()` is preferred as it persists across renders.

---

### Q48. What are the "Rules of React" that the compiler enforces?
- A) Style rules
- B) Components and hooks must be pure, idempotent, follow hooks rules, and not mutate during render
- C) Naming conventions
- D) File structure rules

**Answer: B**
The React Compiler validates: purity (no side effects in render), idempotency (same input → same output), hooks rules (call order), and no mutation of props/state during render.

---

### Q49. What does `useMemoCache` (React Compiler internal) do?
- A) Caches API responses
- B) Provides a fixed-size cache array on the fiber for the compiler's auto-memoization
- C) Caches DOM nodes
- D) Memoizes the component function

**Answer: B**
`useMemoCache(size)` allocates cache slots on the fiber. The compiler stores memoized values and their dependencies in these slots, checking deps before recomputing.

---

### Q50. Why does React use `MessageChannel` instead of `setTimeout(fn, 0)` for scheduling?
- A) `MessageChannel` is more widely supported
- B) `setTimeout` has a minimum ~4ms delay; `MessageChannel.postMessage` has near-zero delay
- C) `MessageChannel` supports cancellation
- D) `setTimeout` is synchronous

**Answer: B**
Browsers enforce a minimum ~4ms delay for nested `setTimeout` calls. `MessageChannel.postMessage` fires as a macrotask with near-zero delay, giving the Scheduler more precise control.
