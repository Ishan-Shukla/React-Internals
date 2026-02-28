# Portals: Rendering Outside the DOM Hierarchy

## What Is a Portal?

A portal renders children into a different DOM node than its parent,
while keeping them in the same **React fiber tree**:

```javascript
import { createPortal } from 'react-dom';

function Modal({ children }) {
  return createPortal(
    <div className="modal">{children}</div>,
    document.getElementById('modal-root')  // Different DOM node!
  );
}

function App() {
  return (
    <div id="app">
      <h1>App</h1>
      <Modal>
        <p>I'm in the modal!</p>
      </Modal>
    </div>
  );
}
```

```
DOM TREE:                              FIBER TREE:
                                       (React's internal tree)
<body>
 ├── <div id="root">                   HostRoot
 │    └── <div id="app">                └── App
 │         └── <h1>App</h1>                  ├── div#app
 │                                           │    └── h1
 ├── <div id="modal-root">                   └── Modal ← still child of App!
 │    └── <div class="modal">     ◄──────────    └── div.modal ← HostPortal
 │         └── <p>I'm in modal</p>                     └── p
 │
 │  DOM parent ≠ Fiber parent!
```

## createPortal Internals

```javascript
// packages/react-dom/src/ReactPortal.js

export function createPortal(children, containerInfo, key) {
  return {
    $$typeof: REACT_PORTAL_TYPE,
    key: key == null ? null : '' + key,
    children: children,
    containerInfo: containerInfo,  // The target DOM node
    implementation: null,
  };
}

// This returns a special element with $$typeof = REACT_PORTAL_TYPE
// The reconciler recognizes this and creates a HostPortal fiber
```

## How the Reconciler Handles Portals

```javascript
// In beginWork, when encountering a portal element:

function updatePortalComponent(current, workInProgress, renderLanes) {
  const nextChildren = workInProgress.pendingProps;
  const containerInfo = workInProgress.stateNode.containerInfo;

  // Push the portal container as the current host context
  pushHostContainer(workInProgress, containerInfo);

  // Reconcile children normally
  reconcileChildren(current, workInProgress, nextChildren, renderLanes);

  // Pop host context
  popHostContainer(workInProgress);

  return workInProgress.child;
}
```

During the commit phase, DOM operations use the **portal's container**
instead of the parent's DOM node:

```javascript
// In commitPlacement:

function getHostParentFiber(fiber) {
  let parent = fiber.return;
  while (parent !== null) {
    if (isHostParent(parent)) {
      return parent;
    }
    parent = parent.return;
  }
}

function isHostParent(fiber) {
  return (
    fiber.tag === HostComponent ||
    fiber.tag === HostRoot ||
    fiber.tag === HostPortal  // ← Portals are valid host parents!
  );
}

// When inserting a child of a portal:
// getHostParentFiber → finds the HostPortal fiber
// Gets containerInfo from portal's stateNode
// Appends to portal's container instead of the normal parent
```

## Event Bubbling Through Portals

The crucial behavior: **events bubble through the FIBER tree, not the DOM tree**.

```jsx
function Parent() {
  return (
    <div onClick={() => console.log('Parent clicked!')}>
      <Modal>
        <button>Click me</button>
      </Modal>
    </div>
  );
}
```

```
DOM tree:                          Fiber tree:
  <div id="root">                    div (onClick=handler)
    <div onClick={handler}>            └── Modal
  </div>                                    └── HostPortal
  <div id="modal-root">                         └── button
    <button>Click me</button>
  </div>

When button is clicked:
  1. Native click event fires on <button>
  2. Bubbles up DOM: button → modal-root → body (DOES NOT hit div#root)
  3. React's root listener catches it
  4. React walks UP the FIBER tree from button:
     button → HostPortal → Modal → div (onClick!) → App
  5. Parent's onClick handler FIRES!

This means: onClick on a parent component catches clicks from
portal children, even though they're in different DOM subtrees.
This is the correct behavior — React preserves the component
hierarchy's event flow.
```

## Portal Use Cases

### 1. Modals/Dialogs

```jsx
function Dialog({ isOpen, children }) {
  if (!isOpen) return null;
  return createPortal(
    <div className="dialog-overlay">
      <div className="dialog">{children}</div>
    </div>,
    document.body  // Render at body level (above all z-index stacking)
  );
}
```

### 2. Tooltips

```jsx
function Tooltip({ target, children }) {
  const rect = target.getBoundingClientRect();
  return createPortal(
    <div style={{ position: 'fixed', top: rect.bottom, left: rect.left }}>
      {children}
    </div>,
    document.body  // Avoids overflow:hidden clipping
  );
}
```

### 3. Rendering into iframes

```jsx
function IframeContent({ children }) {
  const [iframeBody, setIframeBody] = useState(null);
  return (
    <iframe ref={node => {
      if (node) setIframeBody(node.contentDocument.body);
    }}>
      {iframeBody && createPortal(children, iframeBody)}
    </iframe>
  );
}
```

## Portal Lifecycle

```
Mount:
  1. Portal fiber created during reconciliation
  2. Children reconciled normally
  3. During commit mutation phase:
     → DOM nodes appended to portal's containerInfo
     → NOT to the fiber parent's DOM node

Update:
  1. Portal children reconciled (diffed) normally
  2. Updates committed to the portal container DOM

Unmount:
  1. Portal children cleaned up (effects, refs)
  2. DOM nodes removed from portal container
  3. Portal fiber deleted

Context:
  ✓ React context flows through portals (fiber tree)
  ✓ Event bubbling flows through portals (fiber tree)
  ✗ CSS inheritance does NOT flow through portals (DOM tree)
  ✗ DOM focus trapping must be handled manually
```
