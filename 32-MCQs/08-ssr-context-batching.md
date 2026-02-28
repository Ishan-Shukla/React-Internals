# MCQs: SSR, Context, Batching & Advanced Topics

---

### Q1. What is the difference between `renderToString` and `renderToPipeableStream`?
- A) `renderToString` is async; `renderToPipeableStream` is sync
- B) `renderToString` blocks until complete; `renderToPipeableStream` sends HTML progressively
- C) `renderToString` is for Node.js; `renderToPipeableStream` is for browsers
- D) They are identical

**Answer: B**
`renderToString` waits for the entire tree to render before returning HTML. `renderToPipeableStream` starts sending HTML immediately, streaming resolved Suspense content as it becomes available.

---

### Q2. How does React's Context system store values during rendering?
- A) In a global variable
- B) On a value stack that pushes on Provider entry and pops on exit
- C) In localStorage
- D) On each fiber's props

**Answer: B**
React uses a stack-based approach. When entering a Provider during `beginWork`, the value is pushed. During `completeWork`, it's popped. Consumers read from `context._currentValue`.

---

### Q3. What does `propagateContextChange` do?
- A) Propagates CSS changes
- B) Walks the subtree to find fibers that consume the changed context and marks them for update
- C) Propagates state changes to children
- D) Sends context changes to the server

**Answer: B**
When a Provider's value changes, `propagateContextChange` scans the entire subtree, checking each fiber's `dependencies` list. Matching consumers get their `lanes` marked for re-render.

---

### Q4. Does `propagateContextChange` bypass `React.memo`?
- A) No, memo prevents context updates
- B) Yes — it marks consumer fibers directly, bypassing any memoization checks
- C) Only for PureComponent
- D) Only in concurrent mode

**Answer: B**
`propagateContextChange` directly sets `fiber.lanes |= renderLanes` on consumers. This happens before any memo/shouldComponentUpdate check, so context changes always reach consumers.

---

### Q5. What is the performance implication of a large context value?
- A) Large values are compressed automatically
- B) Any change to the context value re-renders ALL consumers, even if they only use part of the value
- C) Large values cause memory leaks
- D) No performance impact

**Answer: B**
Context has no selector mechanism. Changing `{ theme: 'dark', user: {...} }` re-renders all consumers even if only `theme` changed. Splitting contexts is the recommended optimization.

---

### Q6. What is automatic batching in React 18?
- A) Automatically splitting large components
- B) Batching all state updates into a single re-render, regardless of where they originate
- C) Batching DOM operations
- D) Batching network requests

**Answer: B**
React 18 batches setState calls from timeouts, promises, native events, and React events into a single render. React 17 only batched inside React event handlers.

---

### Q7. How does React 18 implement automatic batching?
- A) Using `requestAnimationFrame`
- B) Using microtask scheduling — `ensureRootIsScheduled` deduplicates via `scheduleCallback` at microtask timing
- C) Using `setTimeout(fn, 0)`
- D) Using Web Workers

**Answer: B**
React 18 uses microtask-level deduplication. Multiple `setState` calls within the same microtask batch into one render because `ensureRootIsScheduled` detects the existing callback.

---

### Q8. How do you opt out of batching in React 18?
- A) `ReactDOM.unstable_batchedUpdates(fn)` with `noBatch` flag
- B) `flushSync(() => setState(x))` — forces synchronous rendering
- C) `React.unbatch(() => setState(x))`
- D) You can't opt out

**Answer: B**
`flushSync(() => setState(x))` forces the update to render and commit synchronously before continuing, bypassing batching.

---

### Q9. What did `unstable_batchedUpdates` do in React 17?
- A) Nothing — batching was always automatic
- B) Manually enabled batching outside of React event handlers (e.g., in setTimeout)
- C) Disabled batching
- D) Batched DOM mutations

**Answer: B**
In React 17, updates in `setTimeout` weren't batched. `unstable_batchedUpdates(() => { setState(a); setState(b); })` forced them to batch. In React 18, this is automatic.

---

### Q10. What is `flushSync` used for?
- A) Flushing the event queue
- B) Forcing a synchronous render and commit before continuing execution
- C) Flushing the browser cache
- D) Synchronizing state across components

**Answer: B**
`flushSync(() => { setState(x) })` schedules the update on `SyncLane` and immediately calls `performSyncWorkOnRoot`, ensuring DOM is updated before `flushSync` returns.

---

### Q11. What is the `identifierPrefix` option in `createRoot`?
- A) A CSS class prefix
- B) A prefix for `useId`-generated IDs to prevent collision between multiple React roots
- C) A namespace prefix
- D) A logging prefix

**Answer: B**
`createRoot(el, { identifierPrefix: 'app1' })` makes `useId` generate IDs like `app1-:R1:` instead of `:R1:`, preventing collisions when multiple React roots exist on one page.

---

### Q12. What is hydration's "client-side recovery"?
- A) Recovering from server errors
- B) When a mismatch is detected, React deletes the server DOM subtree and re-creates it from client render
- C) Recovering network connections
- D) Restoring browser state

**Answer: B**
On mismatch, React falls back to full client-side rendering for the affected subtree. It deletes the server-rendered DOM and creates new DOM from the client VDOM.

---

### Q13. What are HTML comments like `<!--$-->` and `<!--/$-->` in React SSR output?
- A) Debug comments
- B) Suspense boundary markers — React uses them during hydration to identify boundary positions
- C) Source map references
- D) Performance markers

**Answer: B**
`<!--$-->` marks the start of a Suspense boundary, `<!--/$-->` marks the end. `<!--$?-->` with `<template>` markers indicate pending Suspense boundaries for streaming.

---

### Q14. How does React Custom Renderer work?
- A) By extending ReactDOM
- B) By providing a "host config" — a set of ~30 functions that define how to create/modify/delete host instances
- C) By writing a new browser engine
- D) By monkey-patching React internals

**Answer: B**
`react-reconciler` accepts a host config object with functions like `createInstance`, `appendChildToContainer`, `commitUpdate`, etc. The reconciler is host-agnostic.

---

### Q15. Name a React custom renderer besides React DOM.
- A) React Native
- B) react-three-fiber (3D/WebGL)
- C) ink (terminal UI)
- D) All of the above

**Answer: D**
React Native renders to native mobile views, react-three-fiber to Three.js 3D scenes, and ink to terminal output. All use `react-reconciler` with custom host configs.

---

### Q16. What is `createPortal(children, container)` used for?
- A) Creating CSS portals
- B) Rendering children into a DOM node outside the parent component's DOM hierarchy
- C) Creating network portals
- D) Teleporting components to the server

**Answer: B**
Portals render children into a different DOM container while maintaining fiber tree parentage. Common use: modals, tooltips, popups rendered at `document.body` level.

---

### Q17. How do events behave with Portals?
- A) Events don't work in portals
- B) React events bubble through the fiber tree (to the component that created the portal), not the DOM tree
- C) Events bubble through the DOM tree only
- D) Events are duplicated in both trees

**Answer: B**
A click inside a portal's content bubbles up through the fiber tree to the portal's parent component. DOM-wise, the click bubbles through the actual DOM ancestry.

---

### Q18. What does `shallowEqual` compare?
- A) Only the first level of object keys, comparing values with `Object.is`
- B) Deep equality of all nested objects
- C) Reference equality only
- D) String comparison of JSON.stringify output

**Answer: A**
`shallowEqual({a:1, b:2}, {a:1, b:2})` returns `true` (same keys, same values via `Object.is`). But `{a: {x:1}}` vs `{a: {x:1}}` returns `false` (different object references for `a`).

---

### Q19. What is the SVG namespace issue React handles?
- A) SVG elements must be created with `createElementNS` using the SVG namespace, not `createElement`
- B) SVG elements need a special React import
- C) SVG requires a different version of React
- D) There is no namespace issue

**Answer: A**
`document.createElement('circle')` creates an HTML element (broken). `document.createElementNS('http://www.w3.org/2000/svg', 'circle')` creates the correct SVG element. React handles this via host context tracking.

---

### Q20. How does React track the current namespace (HTML/SVG/MathML)?
- A) Via a global variable
- B) Via the host context — pushed when entering `<svg>`/`<math>`, popped when leaving
- C) Via CSS classes
- D) By checking the tag name against a list

**Answer: B**
React pushes SVG_NAMESPACE when entering `<svg>`, MATH_NAMESPACE for `<math>`, and switches back to HTML_NAMESPACE inside `<foreignObject>`. The context stack handles this automatically.

---

### Q21. What does `<foreignObject>` do in SVG?
- A) Embeds foreign languages
- B) Switches back to HTML namespace inside SVG, allowing HTML elements within SVG
- C) Imports external SVG files
- D) Creates foreign key relationships

**Answer: B**
`<foreignObject>` inside `<svg>` tells the browser (and React) to switch back to HTML namespace. React's host context tracks this transition.

---

### Q22. How does React handle `className` for SVG elements?
- A) Same as HTML — `element.className = value`
- B) Uses `element.setAttribute('class', value)` because SVG's `className` is an `SVGAnimatedString`
- C) SVG doesn't support classes
- D) Uses `classList.add()`

**Answer: B**
SVG elements have `className` as `SVGAnimatedString` (not a simple string). React detects SVG namespace and uses `setAttribute('class', value)` instead.

---

### Q23. What is `React.Children.toArray` useful for?
- A) Converting children to typed arrays
- B) Flattening nested children into a flat array with stable keys, enabling `.sort()`, `.filter()`, etc.
- C) Converting children to DOM arrays
- D) Creating array components

**Answer: B**
`React.Children.toArray(children)` normalizes any children structure (single, array, nested) into a flat array with proper keys, making it safe to use array methods.

---

### Q24. Why shouldn't you use `props.children.map()` directly?
- A) It's slower than React.Children.map
- B) `props.children` isn't always an array — it could be a single element, string, null, or undefined
- C) It doesn't work in JSX
- D) It's deprecated

**Answer: B**
`props.children` is only an array when there are multiple children. With one child, it's the element directly. With zero, it's `undefined`. `React.Children.map` handles all cases safely.

---

### Q25. What does `act()` do in React testing?
- A) Creates actors for testing
- B) Wraps code that causes state updates, flushing all effects and re-renders before assertions
- C) Mocks API calls
- D) Creates test snapshots

**Answer: B**
`act(() => { trigger(); })` ensures all pending state updates, effects, and commits are flushed synchronously, so assertions run against the fully-updated UI.

---

### Q26. What is the `Profiler` component used for?
- A) Profiling network requests
- B) Measuring render performance — `onRender` callback receives timing information for the subtree
- C) Profiling memory usage
- D) CPU profiling

**Answer: B**
`<Profiler id="Nav" onRender={callback}>` measures `actualDuration`, `baseDuration`, `startTime`, `commitTime` for its subtree's renders.

---

### Q27. What is StrictMode's double-render behavior checking for?
- A) Performance issues
- B) Impure render functions — if a component has side effects during render, double-rendering exposes them
- C) Security vulnerabilities
- D) Accessibility issues

**Answer: B**
By rendering twice, StrictMode detects if a component produces different results between calls (indicating impurity/side effects during render).

---

### Q28. What does StrictMode's double-effect-mounting check for?
- A) Memory leaks
- B) Missing cleanup functions — if effects don't clean up properly, the remount will break
- C) Performance issues
- D) Syntax errors

**Answer: B**
StrictMode mounts → unmounts → remounts effects to simulate component reuse. If cleanup is missing or incomplete, the second mount will fail or behave differently.

---

### Q29. What is `preload()` in React 19?
- A) A lifecycle method
- B) A function to start loading a resource (font, stylesheet, script) before it's needed
- C) Pre-loading component state
- D) Pre-loading route data

**Answer: B**
`preload(url, { as: 'style' })` injects a `<link rel="preload">` into the document head, telling the browser to start downloading the resource early.

---

### Q30. What is `preinit()` in React 19?
- A) Pre-initializing component state
- B) Loading AND evaluating a resource (e.g., inserting a `<script>` or `<link rel="stylesheet">` that blocks rendering)
- C) Pre-initializing the DOM
- D) Initializing SSR

**Answer: B**
`preinit(url, { as: 'style' })` not only loads the stylesheet but inserts it into the page, potentially blocking rendering until loaded. Stronger than `preload`.

---

### Q31. How does React handle stylesheet precedence?
- A) First-come, first-served
- B) Via the `precedence` prop on `<link>` elements — React sorts stylesheets by precedence groups
- C) By CSS specificity
- D) Alphabetically

**Answer: B**
React 19's `<link rel="stylesheet" precedence="high">` lets React manage insertion order of stylesheets, ensuring correct cascade order regardless of component render order.

---

### Q32. What does `dangerouslySetInnerHTML` accept?
- A) A string directly
- B) An object with a `__html` key: `{ __html: htmlString }`
- C) An array of HTML strings
- D) A DOM element

**Answer: B**
The awkward API `{ __html: string }` is intentional — it forces you to acknowledge the danger. React does NOT sanitize the HTML; you must ensure it's safe.

---

### Q33. How does React prevent XSS through text content?
- A) HTML entity encoding
- B) By using `textContent` (or `createTextNode`) which auto-escapes, not `innerHTML`
- C) Content Security Policy
- D) Input validation

**Answer: B**
When rendering `{userInput}`, React uses `textContent` or `createTextNode`, which treat the string as text, not HTML. `<script>` tags become literal text, not executable.

---

### Q34. What is the purpose of `useInsertionEffect`?
- A) Inserting elements into arrays
- B) Injecting `<style>` tags before any DOM reads — designed for CSS-in-JS libraries
- C) Inserting hooks into the linked list
- D) Inserting components into the tree

**Answer: B**
`useInsertionEffect` runs before DOM mutations, allowing CSS-in-JS to inject styles before `useLayoutEffect` reads computed styles. Timing: insertion → mutation → layout → passive.

---

### Q35. What is the difference between controlled and uncontrolled components?
- A) Controlled are faster
- B) Controlled: React manages the value via state+onChange. Uncontrolled: DOM manages the value, React reads via ref
- C) Controlled are for forms only
- D) Uncontrolled components don't re-render

**Answer: B**
Controlled: `<input value={state} onChange={handler} />`. Uncontrolled: `<input defaultValue="x" ref={ref} />`. React internally tracks and restores values for controlled inputs.

---

### Q36. What is "value restoration" for controlled inputs?
- A) Restoring from localStorage
- B) React re-sets `input.value` after commit to prevent the DOM from diverging from React state
- C) Undo/redo for form values
- D) Restoring default values

**Answer: B**
After committing prop changes, React explicitly sets `input.value = fiber.memoizedProps.value` to ensure the DOM input always reflects React's controlled value.

---

### Q37. What is the `act()` warning in React tests?
- A) A performance warning
- B) "An update was not wrapped in act(...)" — indicating state updates happened outside `act`, risking stale assertions
- C) A security warning
- D) A deprecation warning

**Answer: B**
This warning means an asynchronous state update happened after `act()` returned but before assertions. The test may be checking stale UI. Wrap all triggers in `act()`.

---

### Q38. What does `ReactCurrentActQueue` do?
- A) Manages the current animation queue
- B) Stores effects and flushes them synchronously within `act()` for predictable testing
- C) Manages the action queue for forms
- D) Tracks the current active component

**Answer: B**
When `act()` is active, `ReactCurrentActQueue` captures effects that would normally run asynchronously. `act()` flushes them synchronously before returning.

---

### Q39. What is `fiber.return === null` used to detect?
- A) The root of the fiber tree
- B) A detached (unmounted) fiber — `return` is nullified during deletion
- C) A component that returns null
- D) A missing parent component

**Answer: B**
When a fiber is deleted, its `return` pointer is set to `null`. If `setState` is called on this fiber, React detects `return === null`, can't find a root, and silently ignores the update.

---

### Q40. What is the React DevTools `__REACT_DEVTOOLS_GLOBAL_HOOK__`?
- A) A security vulnerability
- B) A global object injected by the DevTools extension for React to register itself
- C) A debugging console command
- D) A performance profiling hook

**Answer: B**
DevTools injects `__REACT_DEVTOOLS_GLOBAL_HOOK__` on `window`. React checks for it and calls `hook.inject(internals)` to expose fiber tree access and update notifications.

---

### Q41. What is `__DEV__` in React source code?
- A) A runtime check for developer tools
- B) A compile-time constant replaced by the bundler — `true` in dev, `false` in production (dead-code eliminated)
- C) A global variable
- D) An environment variable

**Answer: B**
`__DEV__` is replaced at build time. Development warnings, extra checks, and validation are wrapped in `if (__DEV__)` and completely removed in production builds.

---

### Q42. What is `React.forwardRef` used for?
- A) Forwarding events
- B) Passing a `ref` through a component to a child DOM element
- C) Forwarding context
- D) Forwarding state

**Answer: B**
`forwardRef((props, ref) => <input ref={ref} />)` makes the `ref` available inside the component, allowing parent components to directly reference a child's DOM node.

---

### Q43. In React 19, is `forwardRef` still necessary?
- A) Yes, always
- B) No — React 19 passes `ref` as a regular prop, making `forwardRef` unnecessary for new code
- C) Only for class components
- D) Only for SVG elements

**Answer: B**
React 19 passes `ref` as a regular prop to function components. `forwardRef` still works for backward compatibility but is no longer required.

---

### Q44. What does `React.cloneElement` do internally?
- A) Deep clones the DOM tree
- B) Creates a new React element with the same type and key, merging new props over old props
- C) Clones the fiber tree
- D) Creates a deep copy of the component instance

**Answer: B**
`cloneElement(element, newProps)` creates a new element: same `type`, same `key` (unless overridden), props merged (`{...element.props, ...newProps}`).

---

### Q45. What is the purpose of `React.memo`'s second argument?
- A) A fallback component
- B) A custom comparison function `(prevProps, nextProps) => boolean` to replace default shallow comparison
- C) A list of props to ignore
- D) A dependency array

**Answer: B**
`React.memo(Component, areEqual)` — `areEqual(prevProps, nextProps)` should return `true` if rendering would produce the same result (skip re-render). Default uses `shallowEqual`.

---

### Q46. What is the mutation mode vs persistent mode in custom renderers?
- A) Mutation: renders to DOM; Persistent: renders to Canvas
- B) Mutation: modifies host instances in place (DOM). Persistent: creates new instances instead of mutating (used by React Native Fabric)
- C) Mutation: allows state changes; Persistent: is immutable
- D) They are the same

**Answer: B**
Mutation mode renderers (like React DOM) modify existing DOM nodes. Persistent mode renderers create new host instances instead of mutating, which suits immutable view systems.

---

### Q47. What does `fiber.effectTag` (now `fiber.flags`) control?
- A) CSS effects
- B) Which side effects (Placement, Update, Deletion, Ref, etc.) to apply during commit
- C) Audio/visual effects
- D) Animation effects

**Answer: B**
`fiber.flags` is a bitmask of side effects the fiber needs during commit. The commit phase checks these flags to determine which operations to perform.

---

### Q48. What is the `LegacyHiddenComponent`?
- A) A deprecated hidden input component
- B) An internal component used by Suspense in legacy mode to hide content while showing fallback
- C) A component for hiding elements with CSS
- D) A component for hidden form fields

**Answer: B**
`LegacyHiddenComponent` (now replaced by `OffscreenComponent`) was used to hide Suspense content while showing the fallback.

---

### Q49. What does the `StrictEffectsMode` bit enable?
- A) Strict typing for effects
- B) Double-firing of effects (mount → unmount → remount) in development for testing cleanup
- C) Preventing effects from running
- D) Strict dependency checking

**Answer: B**
`StrictEffectsMode` (part of StrictMode) causes effects to fire twice: setup → cleanup → setup. This simulates component reuse (e.g., for future features like offscreen components).

---

### Q50. What is `fiber.type` for a `<div>` element?
- A) `HTMLDivElement`
- B) The string `'div'`
- C) A function
- D) A Symbol

**Answer: B**
For host elements, `fiber.type` is the tag name string: `'div'`, `'span'`, `'input'`, etc. For components, it's the function or class reference.
