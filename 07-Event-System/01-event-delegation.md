# Event System: Delegation and Synthetic Events

## How React Handles Events

React does NOT attach event listeners to individual DOM nodes. Instead, it
uses **event delegation** — a single listener on the root container captures
all events:

```
Traditional DOM:                        React's Approach:
<div>                                   <div id="root">  ← ALL listeners here
  <button onclick={fn1}>               │
    Click me                            │  ┌─ button (no listener)
  </button>                             │  │   "Click me"
  <input onchange={fn2} />             │  │
  <a onclick={fn3}>Link</a>            │  ├─ input (no listener)
</div>                                  │  │
                                        │  └─ a (no listener)
3 listeners on 3 elements              1 listener captures everything
```

## Why Event Delegation?

1. **Memory efficiency** — 1 listener vs thousands for large lists
2. **Dynamic elements** — New elements automatically work (no attach/detach)
3. **Consistent behavior** — React controls the entire event flow
4. **Priority mapping** — React maps events to lanes for scheduling

## Root Listener Setup

When you call `createRoot(container)`, React attaches listeners:

```javascript
// Simplified from DOMPluginEventSystem.js

function listenToAllSupportedEvents(rootContainerElement) {
  // For every event React supports...
  allNativeEvents.forEach(domEventName => {
    // Attach both bubble and capture listeners
    if (!nonDelegatedEvents.has(domEventName)) {
      // BUBBLE phase listener
      listenToNativeEvent(domEventName, false, rootContainerElement);
      // CAPTURE phase listener
      listenToNativeEvent(domEventName, true, rootContainerElement);
    }
  });
}

function listenToNativeEvent(domEventName, isCapturePhase, target) {
  let listener;

  // Map event to priority
  const eventPriority = getEventPriority(domEventName);

  switch (eventPriority) {
    case DiscreteEventPriority:    // click, keydown, focus
      listener = dispatchDiscreteEvent.bind(null, domEventName, ...);
      break;
    case ContinuousEventPriority:  // drag, scroll, mousemove
      listener = dispatchContinuousEvent.bind(null, domEventName, ...);
      break;
    default:                       // everything else
      listener = dispatchEvent.bind(null, domEventName, ...);
      break;
  }

  if (isCapturePhase) {
    target.addEventListener(domEventName, listener, true);
  } else {
    target.addEventListener(domEventName, listener, false);
  }
}
```

## Event Priority Mapping

```javascript
function getEventPriority(domEventName) {
  switch (domEventName) {
    // Discrete events → SyncLane (highest priority)
    case 'click':
    case 'keydown':
    case 'keyup':
    case 'mousedown':
    case 'mouseup':
    case 'focusin':
    case 'focusout':
    case 'change':
    case 'input':
    case 'submit':
      return DiscreteEventPriority;   // → SyncLane

    // Continuous events → InputContinuousLane
    case 'drag':
    case 'dragenter':
    case 'dragleave':
    case 'mousemove':
    case 'mouseout':
    case 'mouseover':
    case 'pointerdown':
    case 'pointermove':
    case 'scroll':
    case 'touchmove':
      return ContinuousEventPriority; // → InputContinuousLane

    default:
      return DefaultEventPriority;    // → DefaultLane
  }
}
```

## Event Dispatch Flow

```
User clicks <button>
        │
        ▼
Native click event bubbles to root container
        │
        ▼
┌─────────────────────────────────┐
│  dispatchDiscreteEvent()        │
│    │                            │
│    ▼                            │
│  Set current update priority    │
│  to DiscreteEventPriority       │
│    │                            │
│    ▼                            │
│  dispatchEvent()                │
│    │                            │
│    ├─ 1. Find the fiber from    │
│    │     the native event target│
│    │     (event.target → fiber) │
│    │                            │
│    ├─ 2. Collect listeners      │
│    │     Walk fiber tree UP     │
│    │     from target to root    │
│    │     Gather onClickCapture  │
│    │     and onClick handlers   │
│    │                            │
│    ├─ 3. Create SyntheticEvent  │
│    │                            │
│    └─ 4. Execute listeners      │
│          in correct order       │
│          (capture down, bubble  │
│           up)                   │
└─────────────────────────────────┘
```

## Finding the Fiber from a DOM Node

React stores a back-pointer from DOM nodes to fibers:

```javascript
// When creating a DOM element in completeWork:
function precacheFiberNode(hostInst, node) {
  node[internalInstanceKey] = hostInst;
  // internalInstanceKey = '__reactFiber$' + randomKey
}

// When looking up the fiber from an event target:
function getClosestInstanceFromNode(targetNode) {
  let targetInst = targetNode[internalInstanceKey];
  if (targetInst) return targetInst;

  // Walk up the DOM tree if no fiber on this node
  let parentNode = targetNode.parentNode;
  while (parentNode) {
    targetInst = parentNode[internalInstanceKey];
    if (targetInst) return targetInst;
    parentNode = parentNode.parentNode;
  }
  return null;
}
```

## Collecting Listeners

```javascript
function accumulateSinglePhaseListeners(targetFiber, reactName, nativeEvent) {
  const captureName = reactName + 'Capture';  // 'onClickCapture'
  const listeners = [];

  let instance = targetFiber;

  // Walk UP the fiber tree from target to root
  while (instance !== null) {
    const { stateNode, tag } = instance;

    if (tag === HostComponent && stateNode !== null) {
      // Check for capture listener
      const captureListener = getListener(instance, captureName);
      if (captureListener != null) {
        listeners.unshift({  // Capture goes to FRONT
          instance, listener: captureListener, currentTarget: stateNode
        });
      }

      // Check for bubble listener
      const bubbleListener = getListener(instance, reactName);
      if (bubbleListener != null) {
        listeners.push({     // Bubble goes to END
          instance, listener: bubbleListener, currentTarget: stateNode
        });
      }
    }

    instance = instance.return;  // Go to parent fiber
  }

  return listeners;
}
```

```
Example: Click on <span> inside <button> inside <div>

Fiber tree:
  div (onClick=handleDivClick, onClickCapture=handleDivCapture)
   └── button (onClick=handleBtnClick)
        └── span ← EVENT TARGET

Listeners collected (walking up from span):
  [
    { listener: handleDivCapture, target: div },    // Capture: div (unshift)
    { listener: handleBtnClick,  target: button },  // Bubble: button (push)
    { listener: handleDivClick,  target: div },     // Bubble: div (push)
  ]

Execution order:
  1. handleDivCapture (capture, top → down)
  2. handleBtnClick   (bubble, bottom → up)
  3. handleDivClick   (bubble, bottom → up)
```

## SyntheticEvent

React wraps native events in a cross-browser SyntheticEvent:

```javascript
function SyntheticEvent(reactName, reactEventType, targetInst, nativeEvent, nativeEventTarget) {
  this._reactName = reactName;
  this.type = reactEventType;
  this.nativeEvent = nativeEvent;
  this.target = nativeEventTarget;
  this.currentTarget = null;  // Set per-listener during dispatch

  // Copy native event properties
  for (const propName in Interface) {
    this[propName] = nativeEvent[propName];
  }

  // Normalize cross-browser quirks
  this.isDefaultPrevented = nativeEvent.defaultPrevented
    ? functionThatReturnsTrue
    : functionThatReturnsFalse;
  this.isPropagationStopped = functionThatReturnsFalse;

  return this;
}

SyntheticEvent.prototype.preventDefault = function() {
  this.isDefaultPrevented = functionThatReturnsTrue;
  this.nativeEvent.preventDefault();
};

SyntheticEvent.prototype.stopPropagation = function() {
  this.isPropagationStopped = functionThatReturnsTrue;
  this.nativeEvent.stopPropagation();
};
```

Note: **Event pooling was removed in React 17.** SyntheticEvents are no
longer reused/nullified after the handler completes. You can safely access
event properties asynchronously.

## React 17+ Change: Root vs Document

```
React 16:                         React 17+:
  document  ← listeners here       document
     │                                │
   <div id="root">                  <div id="root"> ← listeners here
      │                                │
    <App />                          <App />

React 17 moved listeners from document to the root container.
This enables multiple React roots on the same page without conflicts.
```
