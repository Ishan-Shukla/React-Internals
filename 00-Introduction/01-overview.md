# React Internals: Overview

## What This Guide Covers

This guide deconstructs React from the inside out — every data structure, every
algorithm, every scheduling decision. By the end you will understand the complete
lifecycle of a React update, from `setState` to pixels on screen, at the level
needed to contribute to the React codebase itself.

## The 10,000-Foot View

React's job is deceptively simple: **given application state, produce UI**.
Internally, this involves a sophisticated pipeline:

```
User Action / State Change
        │
        ▼
┌─────────────────────┐
│   React Core        │  createElement / JSX → ReactElement objects
│   (react package)   │
└────────┬────────────┘
         │
         ▼
┌─────────────────────┐
│   Scheduler         │  Prioritize work, yield to browser
│   (scheduler pkg)   │
└────────┬────────────┘
         │
         ▼
┌─────────────────────┐
│   Reconciler        │  Fiber tree diffing (render phase)
│   (react-reconciler)│  beginWork → completeWork
└────────┬────────────┘
         │
         ▼
┌─────────────────────┐
│   Commit Phase      │  Apply DOM mutations, run effects
│   (react-dom)       │  mutation → layout → passive
└────────┬────────────┘
         │
         ▼
    Browser Paint
```

## Key Packages in the React Monorepo

| Package | Purpose |
|---------|---------|
| `react` | Public API — `createElement`, hooks, `memo`, `lazy`, `Context` |
| `react-reconciler` | The core algorithm — Fiber tree construction, diffing, work loop |
| `scheduler` | Cooperative scheduling — priority queues, time slicing, `shouldYield` |
| `react-dom` | DOM host config — element creation, attribute diffing, event system |
| `react-dom-bindings` | Event delegation, DOM property operations |
| `react-server` | Server-side rendering foundations |
| `react-server-dom-webpack` | React Server Components (Flight) for webpack |
| `shared` | Constants, type definitions, utilities shared across packages |

## Source Code Layout (facebook/react repo)

```
facebook/react/
├── packages/
│   ├── react/                     # Public API
│   │   └── src/
│   │       ├── React.js           # Entry point
│   │       ├── ReactHooks.js      # Hook dispatcher routing
│   │       ├── ReactElement.js    # createElement
│   │       ├── ReactContext.js    # createContext
│   │       └── ReactLazy.js       # lazy()
│   │
│   ├── react-reconciler/          # Core algorithm
│   │   └── src/
│   │       ├── ReactFiber.js              # Fiber node creation
│   │       ├── ReactFiberWorkLoop.js      # Main work loop
│   │       ├── ReactFiberBeginWork.js     # Enter/diff each fiber
│   │       ├── ReactFiberCompleteWork.js  # Bubble up, create instances
│   │       ├── ReactFiberCommitWork.js    # DOM mutations & effects
│   │       ├── ReactFiberHooks.js         # All hook implementations
│   │       ├── ReactFiberLane.js          # Lane priority model
│   │       ├── ReactChildFiber.js         # Child list reconciliation
│   │       └── ReactFiberSuspenseComponent.js
│   │
│   ├── scheduler/                 # Cooperative scheduler
│   │   └── src/
│   │       ├── Scheduler.js       # Main scheduler
│   │       └── SchedulerMinHeap.js # Priority queue
│   │
│   └── react-dom/                 # DOM renderer
│       └── src/
│           ├── client/
│           │   └── ReactDOMRoot.js
│           └── events/
│               └── DOMPluginEventSystem.js
│
├── scripts/                       # Build scripts, CI
└── fixtures/                      # Test apps
```

## The Two Trees

At any point in time React maintains **two** fiber trees:

```
┌──────────────────┐     ┌──────────────────┐
│   CURRENT TREE   │     │  WORK-IN-PROGRESS│
│  (what's on      │     │     TREE         │
│   screen now)    │     │  (being built)   │
│                  │     │                  │
│  FiberRoot       │     │  FiberRoot       │
│    └─ App        │◄───►│    └─ App        │
│       ├─ Header  │ alt │       ├─ Header  │
│       ├─ Main    │     │       ├─ Main *  │
│       └─ Footer  │     │       └─ Footer  │
└──────────────────┘     └──────────────────┘

* = fiber with pending update

After commit, the work-in-progress tree BECOMES the current tree
(a simple pointer swap on FiberRoot.current).
```

## Reading Order

Follow the numbered directories and files in sequence:

1. **00-Introduction** — You are here
2. **01-React-Element-and-JSX** — How JSX becomes objects
3. **02-Fiber-Architecture** — The Fiber data structure
4. **03-Reconciliation** — The diffing algorithm
5. **04-Scheduler** — Cooperative scheduling
6. **05-Lanes** — Priority model
7. **06-Hooks** — Hook internals
8. **07-Event-System** — Synthetic events and delegation
9. **08-Commit-Phase** — Applying changes to the DOM
10. **09-Suspense-and-Concurrent** — Concurrent features
11. **10-Server-Rendering** — SSR, streaming, RSC
12. **11-Error-Handling** — Error boundaries
13. **12-DevTools-and-Profiling** — Developer tools integration
14. **13-Complete-System-Diagram** — Full architecture & timing diagrams
