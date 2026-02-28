# ReactElement: The Immutable Description

## What Is a ReactElement?

A ReactElement is a **plain JavaScript object** that describes what you want to
see on the screen. It is:

- **Immutable** — once created, never mutated
- **Lightweight** — just a plain object with a few fields
- **Not a DOM node** — it's a description, not the real thing
- **Not a component instance** — it describes a component, it IS not the component

```javascript
// When you write:
<Button color="blue">Click me</Button>

// React creates this object:
{
  $$typeof: Symbol.for('react.element'),  // Security tag
  type: Button,                           // Component function/class OR string
  key: null,                              // Reconciliation hint
  ref: null,                              // DOM/instance ref
  props: {                                // Everything else
    color: 'blue',
    children: 'Click me'
  },
  _owner: FiberNode | null,               // DEV: fiber that created this element
}
```

## The $$typeof Field: Security

`$$typeof` exists to prevent **XSS attacks via JSON injection**.

**The attack vector (without $$typeof):**

```javascript
// Imagine a server returns user-generated content as JSON:
const serverData = JSON.parse('{"type":"div","props":{"dangerouslySetInnerHTML":{"__html":"<script>alert(1)</script>"}}}');

// If React treated any object as an element, this would execute the script!
return <div>{serverData}</div>;
```

**Why Symbol prevents this:**

```javascript
// JSON.parse CANNOT produce Symbols:
JSON.parse('{"$$typeof": "Symbol(react.element)"}');
// $$typeof would be a STRING, not a Symbol

// React checks:
if (element.$$typeof !== REACT_ELEMENT_TYPE) {
  // Reject! Not a real React element.
}

// REACT_ELEMENT_TYPE = Symbol.for('react.element')
// Only code running in the same JS context can create real Symbols
```

## ReactElement Source Code

```javascript
// Simplified from packages/react/src/jsx/ReactJSXElement.js

function ReactElement(type, key, ref, props) {
  const element = {
    // This tag identifies this as a React Element
    $$typeof: REACT_ELEMENT_TYPE,

    // Built-in properties
    type: type,
    key: key,
    ref: ref,
    props: props,

    // Record the component responsible for creating this element
    _owner: ReactCurrentOwner.current,
  };

  if (__DEV__) {
    // In dev, freeze the element and props to catch mutations
    Object.freeze(element.props);
    Object.freeze(element);
  }

  return element;
}
```

## Element Type Variants

The `type` field determines what kind of thing this element represents:

```
type field value              What it represents
─────────────────────────     ──────────────────────────────
'div'                         Host component (DOM element)
'span'                        Host component (DOM element)
'input'                       Host component (DOM element)
function Button() {}          Function component
class App extends Component   Class component
Symbol(react.fragment)        Fragment (no DOM node)
Symbol(react.portal)          Portal
Symbol(react.suspense)        Suspense boundary
Symbol(react.profiler)        Profiler
React.memo(Component)         Memoized component wrapper
React.lazy(() => import())    Lazy-loaded component wrapper
React.forwardRef(render)      Ref-forwarding wrapper
createContext().Provider       Context provider
createContext().Consumer       Context consumer
```

## Element Tree Example

```jsx
function App() {
  return (
    <div>
      <h1>Title</h1>
      <Counter count={0} />
    </div>
  );
}
```

Calling `App()` produces this element tree:

```javascript
// Root element
{
  $$typeof: Symbol(react.element),
  type: 'div',
  props: {
    children: [
      // Child 1
      {
        $$typeof: Symbol(react.element),
        type: 'h1',
        props: { children: 'Title' }
      },
      // Child 2
      {
        $$typeof: Symbol(react.element),
        type: Counter,     // ◄── reference to the function
        props: { count: 0 }
      }
    ]
  }
}
```

**Crucially:** This tree is just **data**. No components have been instantiated.
No DOM nodes exist. The reconciler will later walk this tree and create fibers.

## Elements vs Fibers vs DOM Nodes

```
ReactElement (description)     Fiber (work unit)           DOM Node (reality)
──────────────────────────     ─────────────────           ──────────────────
{ type: 'div', props: {} }  → FiberNode {                → <div>
                                tag: HostComponent,
                                type: 'div',
                                stateNode: <div>,  ────────┘
                                memoizedProps: {},
                                ...
                              }

{ type: App, props: {} }     → FiberNode {                → (no DOM node)
                                tag: FunctionComponent,
                                type: App,
                                stateNode: null,
                                ...
                              }

Immutable, throwaway           Mutable, persistent,        Actual browser
Created every render           reused across renders       DOM element
```

## Key Takeaways

1. **Elements are cheap** — they're just objects. Creating thousands per render is fine.
2. **Elements are descriptive** — they say "what" not "how."
3. **Elements are immutable** — once created, they represent a snapshot.
4. **The reconciler diffs elements** — compares new elements against current fibers.
5. **$$typeof prevents XSS** — Symbols can't come from JSON.parse.
