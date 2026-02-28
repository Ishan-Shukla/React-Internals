# Custom Renderers and Renderer-Agnostic Architecture

## The Key Insight: Reconciler is Separate from Renderer

React's reconciliation algorithm (Fiber, diffing, scheduling, hooks) knows
**nothing** about the DOM. It works against an abstract "host" interface.
Different renderers plug into this interface:

```
                         ┌─────────────────────────┐
                         │    react-reconciler      │
                         │                          │
                         │  Fiber tree              │
                         │  Work loop               │
                         │  Hooks                   │
                         │  Reconciliation          │
                         │  Scheduler integration   │
                         │                          │
                         │  Calls abstract "host    │
                         │  config" functions:      │
                         │  • createInstance()      │
                         │  • appendChild()         │
                         │  • removeChild()         │
                         │  • commitUpdate()        │
                         │  • etc.                  │
                         └────────────┬─────────────┘
                                      │
                    ┌─────────────────┼──────────────────┐
                    │                 │                   │
                    ▼                 ▼                   ▼
           ┌──────────────┐  ┌──────────────┐  ┌────────────────┐
           │  react-dom    │  │ react-native  │  │ react-three-   │
           │               │  │               │  │ fiber          │
           │ createElement │  │ UIManager     │  │ Three.js       │
           │ appendChild   │  │ .createView() │  │ .add()         │
           │ setAttribute  │  │ .setChildren()│  │ .remove()      │
           │ removeChild   │  │ .manageChild()│  │ .update()      │
           └──────────────┘  └──────────────┘  └────────────────┘
               Browser            iOS/Android         WebGL/3D
```

## The Host Config Interface

To create a custom renderer, you implement a **host config** — a set of
functions that tell the reconciler how to manage your target environment:

```javascript
// Creating a custom renderer
import Reconciler from 'react-reconciler';

const hostConfig = {
  // ═══════════════════════════════════════
  // INSTANCE CREATION
  // ═══════════════════════════════════════

  createInstance(type, props, rootContainer, hostContext, internalHandle) {
    // Create a "node" for your target platform
    // DOM: document.createElement(type)
    // Canvas: new CanvasShape(type)
    // Terminal: new TerminalElement(type)
    // PDF: new PdfElement(type)
  },

  createTextInstance(text, rootContainer, hostContext, internalHandle) {
    // Create a text node for your platform
    // DOM: document.createTextNode(text)
  },

  // ═══════════════════════════════════════
  // TREE OPERATIONS
  // ═══════════════════════════════════════

  appendChild(parentInstance, child) {
    // Append child to parent
    // DOM: parent.appendChild(child)
  },

  appendChildToContainer(container, child) {
    // Append to root container
    // DOM: container.appendChild(child)
  },

  insertBefore(parentInstance, child, beforeChild) {
    // Insert child before another child
    // DOM: parent.insertBefore(child, beforeChild)
  },

  removeChild(parentInstance, child) {
    // Remove child from parent
    // DOM: parent.removeChild(child)
  },

  removeChildFromContainer(container, child) {
    // Remove from root container
  },

  // ═══════════════════════════════════════
  // UPDATES
  // ═══════════════════════════════════════

  prepareUpdate(instance, type, oldProps, newProps, rootContainer, hostContext) {
    // Compute what changed (like diffProperties in react-dom)
    // Return an update payload, or null if nothing changed
    // This is called during completeWork (render phase)
    return computeDiff(oldProps, newProps);
  },

  commitUpdate(instance, updatePayload, type, prevProps, nextProps, internalHandle) {
    // Apply the update payload to the instance
    // This is called during the mutation phase of commit
    // DOM: element.setAttribute(...), element.style = ...
    applyDiff(instance, updatePayload);
  },

  commitTextUpdate(textInstance, oldText, newText) {
    // Update text content
    // DOM: textNode.nodeValue = newText
  },

  // ═══════════════════════════════════════
  // MISC CONFIGURATION
  // ═══════════════════════════════════════

  supportsMutation: true,  // or false for persistent mode

  getRootHostContext(rootContainer) {
    // Context for the root (namespace info, etc.)
    return {};
  },

  getChildHostContext(parentHostContext, type, rootContainer) {
    // Context for children (e.g., SVG namespace inside <svg>)
    return parentHostContext;
  },

  shouldSetTextContent(type, props) {
    // Can this element have direct text content?
    // DOM: true for 'textarea', 'option', etc.
    return false;
  },

  finalizeInitialChildren(instance, type, props, rootContainer, hostContext) {
    // After creating instance with all children appended
    // Return true if it needs focus (commitMount will be called)
    return false;
  },

  prepareForCommit(containerInfo) {
    // Called before the commit phase starts
    // DOM: save selection state
    return null;
  },

  resetAfterCommit(containerInfo) {
    // Called after commit phase completes
    // DOM: restore selection state
  },

  clearContainer(container) {
    // Remove all children from container
    // DOM: container.textContent = ''
  },

  // ═══════════════════════════════════════
  // SCHEDULING
  // ═══════════════════════════════════════

  getCurrentEventPriority() {
    // Return the priority of the current event
    return DefaultEventPriority;
  },

  getInstanceFromNode(node) {
    // Get fiber from platform node (for events)
    return node.__fiber || null;
  },

  prepareScopeUpdate() {},
  getInstanceFromScope() { return null; },

  // Microtask scheduling
  supportsMicrotasks: true,
  scheduleMicrotask: queueMicrotask,

  // For hydration support (optional):
  supportsHydration: false,
};

// Create the renderer
const MyRenderer = Reconciler(hostConfig);
```

## Using Your Custom Renderer

```javascript
// Create a render function
function render(element, container) {
  // Create a root (like createRoot in react-dom)
  const root = MyRenderer.createContainer(
    container,
    0,           // ConcurrentRoot (1) or LegacyRoot (0)
    null,        // hydrationCallbacks
    false,       // isStrictMode
    null,        // concurrentUpdatesByDefaultOverride
    '',          // identifierPrefix
    console.error, // onRecoverableError
    null,        // transitionCallbacks
  );

  // Render the element
  MyRenderer.updateContainer(element, root, null, null);

  return root;
}

// Usage
render(<App />, myContainer);
```

## Real-World Custom Renderers

### react-three-fiber (3D rendering with Three.js)

```javascript
// Host config for Three.js:
createInstance(type, props) {
  // type = 'mesh', 'boxGeometry', 'meshStandardMaterial', etc.
  const instance = new THREE[type.charAt(0).toUpperCase() + type.slice(1)]();
  applyProps(instance, props);
  return instance;
},

appendChild(parent, child) {
  parent.add(child);  // Three.js scene graph
},

commitUpdate(instance, payload, type, prevProps, nextProps) {
  applyProps(instance, nextProps);  // Update Three.js object properties
},
```

```jsx
// Usage:
<Canvas>
  <mesh position={[0, 0, 0]}>
    <boxGeometry args={[1, 1, 1]} />
    <meshStandardMaterial color="orange" />
  </mesh>
</Canvas>

// The reconciler manages the Three.js scene graph
// just like react-dom manages the DOM tree!
```

### ink (Terminal UI)

```javascript
// Host config for terminal:
createInstance(type, props) {
  return { type, props, children: [] };  // Virtual terminal node
},

commitUpdate(instance, payload) {
  Object.assign(instance.props, payload);
  // Re-render the terminal output
  renderToTerminal(rootNode);
},
```

```jsx
// Usage:
import { render, Text, Box } from 'ink';

render(
  <Box flexDirection="column">
    <Text color="green">Hello terminal!</Text>
    <Text>Count: {count}</Text>
  </Box>
);
```

### react-pdf (PDF generation)

```jsx
// Usage:
import { Document, Page, Text, View } from '@react-pdf/renderer';

<Document>
  <Page size="A4">
    <View style={{ margin: 10 }}>
      <Text>Hello PDF!</Text>
    </View>
  </Page>
</Document>
```

## Mutation vs Persistent Mode

```
MUTATION MODE (supportsMutation: true):
  Used by: react-dom, react-native, ink
  Strategy: Mutate existing instances in place
  Functions: appendChild, removeChild, commitUpdate
  Like: Modifying a document in place

PERSISTENT MODE (supportsPersistence: true):
  Used by: react-fabric (new React Native)
  Strategy: Create new immutable instances, replace entire subtrees
  Functions: cloneInstance, createContainerChildSet, replaceContainerChildren
  Like: Functional data structures (create new version, swap pointer)

  cloneInstance(instance, updatePayload, type, oldProps, newProps) {
    // Create a NEW instance with updated props
    // Don't mutate the original
    const clone = cloneNode(instance);
    applyProps(clone, newProps);
    return clone;
  }
```

## How react-dom Implements the Host Config

```javascript
// Simplified from react-dom-bindings/src/client/ReactFiberConfigDOM.js

export function createInstance(type, props, rootContainerInstance) {
  const domElement = document.createElement(type);
  precacheFiberNode(internalInstanceHandle, domElement);
  updateFiberProps(domElement, props);
  return domElement;
}

export function appendChild(parentInstance, child) {
  parentInstance.appendChild(child);
}

export function removeChild(parentInstance, child) {
  parentInstance.removeChild(child);
}

export function commitUpdate(domElement, updatePayload) {
  // updatePayload = ['className', 'new-class', 'style', {...}]
  updateProperties(domElement, updatePayload);
}

export function commitTextUpdate(textInstance, oldText, newText) {
  textInstance.nodeValue = newText;
}

export function shouldSetTextContent(type, props) {
  return (
    type === 'textarea' ||
    type === 'noscript' ||
    typeof props.children === 'string' ||
    typeof props.children === 'number' ||
    (typeof props.dangerouslySetInnerHTML === 'object' && ...)
  );
}
```

## Architecture: Why This Matters

```
The renderer-agnostic design means:

1. ONE reconciliation algorithm serves ALL platforms
   → Bug fixes and optimizations benefit everyone
   → Hooks work identically on DOM, Native, Terminal, etc.

2. New platforms just need a host config (~30 functions)
   → react-three-fiber: 3D scenes with React
   → ink: Terminal UIs with React
   → react-pdf: PDFs with React
   → react-hardware: Arduino/IoT with React
   → react-blessed: Terminal dashboards with React
   → react-figma: Figma plugins with React

3. The "Virtual DOM" is really the "Virtual Host"
   → It's not specific to the DOM at all
   → The fiber tree is a virtual representation of ANY tree structure
```
