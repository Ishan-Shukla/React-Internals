# MCQs: JSX, React Elements & Component Types

---

### Q1. What does JSX compile to in the modern React 17+ transform?
- A) `React.createElement()`
- B) `jsx()` / `jsxs()` from `react/jsx-runtime`
- C) `document.createElement()`
- D) `ReactDOM.render()`

**Answer: B**
The new JSX transform (React 17+) compiles to `jsx()` for single child and `jsxs()` for multiple children, imported automatically from `react/jsx-runtime`. The classic transform used `React.createElement()`.

---

### Q2. What is the `$$typeof` field on a React element?
- A) A string identifying the element tag name
- B) A Symbol used to prevent XSS via JSON injection
- C) A numeric hash of the component function
- D) A reference to the fiber node

**Answer: B**
`$$typeof: Symbol.for('react.element')` prevents forged React elements from JSON responses. Symbols cannot be serialized to JSON, so any JSON payload from an API cannot contain a valid `$$typeof`.

---

### Q3. What value does `$$typeof` fall back to in environments without Symbol support?
- A) `0xdead`
- B) `0xeac7`
- C) `0xbeef`
- D) `null`

**Answer: B**
`0xeac7` (which visually resembles "React") is used as a fallback magic number when Symbols are unavailable.

---

### Q4. Which of the following is NOT a valid return value from a React component?
- A) `null`
- B) A string
- C) `undefined`
- D) An array of elements

**Answer: C**
Returning `undefined` from a component causes an error in development: "Nothing was returned from render." `null`, strings, numbers, booleans, arrays, and elements are all valid return types.

---

### Q5. What is the difference between `jsx()` and `jsxs()`?
- A) `jsxs()` is for server components, `jsx()` for client
- B) `jsxs()` is used when there are multiple children (passes a static children array)
- C) `jsxs()` supports Suspense, `jsx()` does not
- D) There is no difference; they are aliases

**Answer: B**
`jsxs()` is used when the element has multiple static children. It receives `children` as an array and sets a flag indicating the children are static (for potential optimization).

---

### Q6. What type does `typeof MyClassComponent` return in JavaScript?
- A) `'class'`
- B) `'object'`
- C) `'function'`
- D) `'component'`

**Answer: C**
JavaScript classes are syntactic sugar over functions. `typeof` returns `'function'` for both class and function components, which is why React needs additional checks (like `isReactComponent` on the prototype) to distinguish them.

---

### Q7. How does React distinguish a class component from a function component?
- A) By checking `typeof component === 'class'`
- B) By checking `component.prototype.isReactComponent`
- C) By checking if the component has a `render` method before calling it
- D) By checking the component's `displayName` property

**Answer: B**
`React.Component` sets `Component.prototype.isReactComponent = {}`. React's `shouldConstruct()` function checks for this property to determine if `new` should be used.

---

### Q8. What does a React Element contain?
- A) A reference to the DOM node it represents
- B) The component's state and lifecycle methods
- C) `type`, `key`, `ref`, `props`, and `$$typeof`
- D) The fiber node with its linked list of hooks

**Answer: C**
A React Element is a plain JavaScript object with `$$typeof`, `type`, `key`, `ref`, and `props`. It is a lightweight description, not a DOM node or fiber.

---

### Q9. What is the `key` used for in React elements?
- A) To encrypt element data for security
- B) To identify elements uniquely among siblings during reconciliation
- C) To set the CSS `key-frames` animation name
- D) To define the rendering priority of the element

**Answer: B**
Keys help React's reconciler match old and new elements among siblings, enabling efficient reordering, insertion, and deletion without destroying state unnecessarily.

---

### Q10. In the classic JSX transform, what must be in scope for JSX to work?
- A) `ReactDOM`
- B) `React`
- C) `jsx`
- D) `createElement`

**Answer: B**
The classic transform compiles `<div />` to `React.createElement('div')`, so `React` must be imported. The new transform (React 17+) auto-imports `jsx` from `react/jsx-runtime`, removing this requirement.

---

### Q11. What happens when you pass `key` as a prop to a component?
- A) The component receives `key` in `props.key`
- B) `key` is extracted by React and NOT available in `props`
- C) React throws an error
- D) `key` is renamed to `_key` in props

**Answer: B**
`key` (and `ref` in older versions) are special props extracted by React during element creation. They are not passed to the component via `props`. Accessing `props.key` returns `undefined`.

---

### Q12. What does `React.createElement('div', null, 'a', 'b', 'c')` produce for `children`?
- A) `'a'`
- B) `['a', 'b', 'c']`
- C) `'abc'`
- D) `{ 0: 'a', 1: 'b', 2: 'c' }`

**Answer: B**
When multiple children are passed after the props argument, `createElement` collects them into an array: `props.children = ['a', 'b', 'c']`.

---

### Q13. What is the `type` field for `<MyComponent />`?
- A) The string `'MyComponent'`
- B) A reference to the `MyComponent` function/class
- C) A Symbol representing the component
- D) The fiber tag number

**Answer: B**
For custom components, `type` is the function or class reference itself. For host elements like `<div>`, `type` is the string `'div'`.

---

### Q14. What `$$typeof` value does a Portal element have?
- A) `Symbol.for('react.element')`
- B) `Symbol.for('react.portal')`
- C) `Symbol.for('react.fragment')`
- D) `Symbol.for('react.suspense')`

**Answer: B**
Portals have `$$typeof: REACT_PORTAL_TYPE` (`Symbol.for('react.portal')`), not the standard element type. This is how the reconciler knows to handle them differently.

---

### Q15. What does `React.isValidElement(obj)` check?
- A) Whether `obj` is a DOM element
- B) Whether `obj` is a non-null object with `$$typeof === REACT_ELEMENT_TYPE`
- C) Whether `obj` passes prop-type validation
- D) Whether `obj` has a valid `render` method

**Answer: B**
`isValidElement` checks that the argument is an object with `$$typeof` equal to `Symbol.for('react.element')`.

---

### Q16. What is the `type` of a Fragment (`<>...</>`)?
- A) `'fragment'`
- B) `Symbol.for('react.fragment')`
- C) `null`
- D) `React.Fragment` (which is the string `'Fragment'`)

**Answer: B**
`<>...</>` compiles to `jsx(Fragment, { children })` where `Fragment` is `Symbol.for('react.fragment')`.

---

### Q17. Which of these correctly describes the React Element to DOM pipeline?
- A) Element → DOM node → Fiber
- B) Element → Fiber → DOM node
- C) Fiber → Element → DOM node
- D) DOM node → Element → Fiber

**Answer: B**
JSX creates Elements, Elements are used to create/update Fibers during reconciliation, and Fibers produce DOM nodes during completeWork/commit.

---

### Q18. What `$$typeof` does `React.memo()` return?
- A) `Symbol.for('react.element')`
- B) `Symbol.for('react.memo')`
- C) `Symbol.for('react.lazy')`
- D) `Symbol.for('react.forward_ref')`

**Answer: B**
`React.memo(Component)` returns `{ $$typeof: REACT_MEMO_TYPE, type: Component, compare: null }`.

---

### Q19. What does `React.lazy()` accept as its argument?
- A) A component function
- B) A Promise that resolves to a component
- C) A function that returns a Promise resolving to a module with a `default` export
- D) A string path to a module

**Answer: C**
`React.lazy(() => import('./MyComponent'))` — the argument is a function returning a dynamic import Promise. The resolved module must have a `default` export containing the component.

---

### Q20. When a component returns `false`, what does React render?
- A) The string `"false"`
- B) Nothing (same as returning `null`)
- C) A text node with empty string
- D) An error is thrown

**Answer: B**
Returning `false` from a component is treated the same as `null` — nothing is rendered. React filters out `null`, `undefined`, `true`, and `false` as empty children.

---

### Q21. What happens when children include the number `0`?
- A) Nothing is rendered
- B) The string `"0"` is rendered as a text node
- C) React throws a type error
- D) A falsy check makes it behave like `null`

**Answer: B**
`0` is a number and React renders numbers as text. This is a common gotcha: `{items.length && <List />}` renders `"0"` when `items` is empty. Fix: `{items.length > 0 && <List />}`.

---

### Q22. What is `React.forwardRef` used for?
- A) Forwarding context to deeply nested components
- B) Forwarding a `ref` prop to a child DOM element or component
- C) Forwarding events from child to parent
- D) Forwarding state between sibling components

**Answer: B**
`forwardRef` creates a component that receives a `ref` attribute and passes it down. It wraps a render function that receives `(props, ref)` as arguments.

---

### Q23. What does a `forwardRef` component's `$$typeof` resolve to?
- A) `Symbol.for('react.element')`
- B) `Symbol.for('react.forward_ref')`
- C) `Symbol.for('react.memo')`
- D) `Symbol.for('react.context')`

**Answer: B**
`forwardRef(renderFn)` returns `{ $$typeof: REACT_FORWARD_REF_TYPE, render: renderFn }`.

---

### Q24. What is the purpose of `React.Children.map`?
- A) To iterate over DOM child nodes
- B) To safely iterate and transform `props.children` regardless of its type
- C) To create a map data structure from children
- D) To memoize children elements

**Answer: B**
`props.children` can be a single element, array, string, or null. `React.Children.map` normalizes all cases, flattens nested arrays, and remaps keys for uniqueness.

---

### Q25. What fiber tag does a host element like `<div>` receive?
- A) `FunctionComponent (0)`
- B) `ClassComponent (1)`
- C) `HostComponent (5)`
- D) `HostRoot (3)`

**Answer: C**
Native DOM elements (`div`, `span`, `input`, etc.) get the `HostComponent` tag (5). `HostRoot` (3) is the root of the fiber tree, not a regular DOM element.

---

### Q26. What fiber tag is assigned to a text node like `"Hello"`?
- A) `HostComponent (5)`
- B) `HostText (6)`
- C) `Fragment (7)`
- D) `IndeterminateComponent (2)`

**Answer: B**
Text content creates fibers with `HostText` tag (6). These correspond to DOM `TextNode` instances.

---

### Q27. Which statement about `React.createElement` is TRUE?
- A) It creates a DOM element
- B) It creates a fiber node
- C) It creates a plain JavaScript object describing the UI
- D) It is only available in class components

**Answer: C**
`createElement` returns a plain object (React Element): `{ $$typeof, type, key, ref, props }`. No DOM or fiber creation happens at this point.

---

### Q28. What does `React.cloneElement` do?
- A) Deep clones the DOM node for the element
- B) Creates a new React element using the original as a template, merging new props
- C) Creates a copy of the fiber node
- D) Duplicates the component's state

**Answer: B**
`cloneElement(element, newProps, ...children)` creates a new element with the same `type` and `key`, merging `newProps` over original props.

---

### Q29. What is the `ref` property on a React element used for?
- A) A unique identifier for reconciliation
- B) A callback or object to receive the underlying DOM node or component instance
- C) A reference to the parent element
- D) A pointer to the element's position in the virtual DOM

**Answer: B**
`ref` provides access to the underlying DOM node (for host components) or the class instance (for class components). It can be a callback function, a `useRef` object, or a string (legacy).

---

### Q30. In React 19, what new capability do ref callbacks have?
- A) They can be async
- B) They can return a cleanup function
- C) They receive the fiber node instead of the DOM node
- D) They are called during the render phase

**Answer: B**
React 19 allows ref callbacks to return a cleanup function, similar to `useEffect`. The cleanup runs when the ref is detached (element unmounts or ref changes).

---

### Q31. What is the output of `typeof <div />`?
- A) `'object'`
- B) `'function'`
- C) `'element'`
- D) `'symbol'`

**Answer: A**
`<div />` compiles to `jsx('div', {})` which returns a plain JavaScript object (React Element). `typeof` an object is `'object'`.

---

### Q32. Which of the following is TRUE about React Elements?
- A) They are mutable and React updates them in place
- B) They are immutable — once created, their props cannot change
- C) They hold a reference to their corresponding DOM node
- D) They are created during the commit phase

**Answer: B**
React Elements are immutable. Once created, you cannot change `element.props`. To update UI, you create new elements (via re-rendering) and React reconciles the differences.

---

### Q33. What is the `children` field in a React Element?
- A) A separate field on the element object
- B) A property inside `element.props`
- C) A linked list of sibling elements
- D) A reference to child fiber nodes

**Answer: B**
Children are passed as `element.props.children`. There is no separate `children` field on the element object — it's a regular prop.

---

### Q34. What does `React.Children.count(<><A /><B /></>)` return?
- A) `0`
- B) `1`
- C) `2`
- D) `3`

**Answer: B**
`React.Children` treats a Fragment (`<>...</>`) as a single element, NOT unwrapping its children. Fragments are unwrapped by the reconciler, not by `React.Children`. The count is `1` (the fragment itself).

---

### Q35. What is the purpose of `React.Children.only`?
- A) Returns the first child
- B) Asserts exactly one React element child exists and returns it
- C) Filters children to only valid elements
- D) Returns `null` if there are multiple children

**Answer: B**
`React.Children.only(children)` throws if `children` is not exactly one valid React element. It's used by components that require a single child.

---

### Q36. How does `React.Children.map` handle keys?
- A) It preserves original keys unchanged
- B) It strips all keys
- C) It remaps keys with composite paths for uniqueness
- D) It assigns random keys

**Answer: C**
`React.Children.map` creates composite keys like `.0/.$originalKey` to ensure uniqueness across nesting levels and multiple mapping passes.

---

### Q37. What happens when you render `{[<A key="x" />, <B key="x" />]}`?
- A) Both render normally
- B) React warns about duplicate keys and behavior is undefined
- C) Only the first element renders
- D) React throws an error and stops rendering

**Answer: B**
Duplicate keys among siblings cause a DEV warning. At runtime, the second key overwrites the first in the Map-based lookup, potentially causing the first element to be incorrectly deleted.

---

### Q38. What is the `type` of `React.createContext(defaultValue)`?
- A) A plain object with `Provider` and `Consumer`
- B) A Symbol
- C) A function component
- D) A class component

**Answer: A**
`createContext` returns an object: `{ Provider, Consumer, _currentValue, _currentValue2, ... }`. `Provider` and `Consumer` are special component types.

---

### Q39. What `$$typeof` does a Context Provider have?
- A) `Symbol.for('react.provider')`
- B) `Symbol.for('react.context')`
- C) `Symbol.for('react.element')`
- D) Context Providers don't have `$$typeof`

**Answer: A**
The Provider is `{ $$typeof: REACT_PROVIDER_TYPE, _context: contextObject }`. The Consumer is `{ $$typeof: REACT_CONTEXT_TYPE }`.

---

### Q40. What happens if you use `<MyComponent />` where `MyComponent` is `undefined`?
- A) Nothing renders
- B) React renders an empty div
- C) React throws an error about element type being invalid
- D) React treats it as a Fragment

**Answer: C**
React throws: "Element type is invalid: expected a string (for built-in components) or a class/function (for composite components) but got: undefined."

---

### Q41. Which is more performant for static content: `createElement` or JSX?
- A) `createElement` because it skips the compilation step
- B) JSX because the compiler can optimize static children with `jsxs()`
- C) They are identical at runtime
- D) Neither — React uses template literals internally

**Answer: B**
The new JSX transform uses `jsxs()` for static children, which sets a flag allowing React to skip certain reconciliation work. `createElement` always processes children dynamically.

---

### Q42. What is `React.Fragment` equivalent to?
- A) `<div style={{display: 'contents'}}>`
- B) `<>` (empty JSX tags)
- C) `null`
- D) `document.createDocumentFragment()`

**Answer: B**
`<React.Fragment>` and `<>` compile to the same thing. The short syntax `<>` can't accept a `key` prop; for keyed fragments, use `<React.Fragment key={k}>`.

---

### Q43. What is the fiber tag for a `React.memo` component?
- A) `FunctionComponent (0)`
- B) `MemoComponent (14)`
- C) `SimpleMemoComponent (15)`
- D) Either B or C depending on the comparison function

**Answer: D**
`React.memo` with no custom comparison function gets the faster `SimpleMemoComponent` (15) tag. With a custom `compare` function, it uses `MemoComponent` (14).

---

### Q44. What does `React.lazy` return?
- A) A Promise
- B) The resolved component
- C) An object with `$$typeof: REACT_LAZY_TYPE` and a `_init`/`_payload` pair
- D) A Suspense boundary

**Answer: C**
`React.lazy` returns `{ $$typeof: REACT_LAZY_TYPE, _payload: { _status, _result }, _init: lazyInitializer }`. The component is resolved on first render.

---

### Q45. When a lazy component is first rendered, what happens?
- A) React renders `null` and waits
- B) React throws the import Promise, which Suspense catches
- C) React renders a placeholder div
- D) React blocks the thread until the import resolves

**Answer: B**
On first render, `React.lazy` triggers the dynamic import. If the Promise hasn't resolved, it throws the Promise. The nearest Suspense boundary catches it and shows the fallback.

---

### Q46. What does `PureComponent.prototype.isPureReactComponent` control?
- A) Prevents the component from re-rendering entirely
- B) Enables automatic shallow comparison in `shouldComponentUpdate`
- C) Makes the component a function component internally
- D) Enables concurrent rendering for the component

**Answer: B**
When React sees `isPureReactComponent = true`, it performs `shallowEqual(oldProps, newProps) && shallowEqual(oldState, newState)` — returning `false` only if there's a shallow difference.

---

### Q47. What is the relationship between `React.memo` and `PureComponent`?
- A) They are identical
- B) `React.memo` is for function components, `PureComponent` is for class components; both do shallow comparison
- C) `React.memo` does deep comparison, `PureComponent` does shallow
- D) `PureComponent` is deprecated in favor of `React.memo`

**Answer: B**
Both perform shallow prop comparison to skip unnecessary re-renders. `React.memo` wraps function components, `PureComponent` is a base class for class components.

---

### Q48. What does the `IndeterminateComponent` tag (2) mean?
- A) The component has an error
- B) The component hasn't been rendered yet and React doesn't know if it's a function or class
- C) The component is lazy-loading
- D) The component is suspended

**Answer: B**
On first encounter, a function-typed component gets tag `IndeterminateComponent` because React hasn't called it yet to determine if it returns JSX (function component) or a class instance (legacy pattern).

---

### Q49. After the first render, what happens to an `IndeterminateComponent` fiber?
- A) It stays as `IndeterminateComponent` forever
- B) Its tag is permanently changed to `FunctionComponent` or `ClassComponent`
- C) It is deleted and recreated
- D) It becomes a `HostComponent`

**Answer: B**
After calling the function and inspecting the return value, React permanently resolves the tag to `FunctionComponent` (0) or `ClassComponent` (1). Subsequent renders use the resolved tag directly.

---

### Q50. What does `React.Children.toArray` do?
- A) Converts children to a typed array for performance
- B) Flattens nested children into a flat array with stable keys
- C) Returns an array of DOM nodes
- D) Converts children into fiber nodes

**Answer: B**
`toArray` flattens all nested children structures into a single flat array, assigning stable composite keys. You can then use standard array methods (`.sort()`, `.filter()`, `.slice()`).
