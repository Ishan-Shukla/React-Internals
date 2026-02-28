# React Internals: A Contributor-Level Deep Dive

A comprehensive guide to React's internal architecture, algorithms, and data
structures. After reading this, you will understand how React works at the
source-code level — enough to navigate the codebase and contribute.

## Reading Order

| # | Section | What You'll Learn |
|---|---------|-------------------|
| 00 | [Introduction](./00-Introduction/) | Architecture overview, source code layout, setup |
| 01 | [React Element & JSX](./01-React-Element-and-JSX/) | JSX compilation, ReactElement, $$typeof security, component types |
| 02 | [Fiber Architecture](./02-Fiber-Architecture/) | Fiber nodes, tree structure, traversal algorithm, double buffering |
| 03 | [Reconciliation](./03-Reconciliation/) | Diffing algorithm, beginWork, completeWork, update queue |
| 04 | [Scheduler](./04-Scheduler/) | Priority queue, time slicing, shouldYield, MessageChannel |
| 05 | [Lanes](./05-Lanes/) | Bitmask priority model, lane assignment, starvation prevention |
| 06 | [Hooks](./06-Hooks/) | Hooks linked list, dispatcher, useState/useEffect/useMemo internals |
| 07 | [Event System](./07-Event-System/) | Event delegation, SyntheticEvent, priority mapping |
| 08 | [Commit Phase](./08-Commit-Phase/) | Before mutation, mutation, layout sub-phases, passive effects |
| 09 | [Suspense & Concurrent](./09-Suspense-and-Concurrent/) | Thrown promises, concurrent rendering, transitions |
| 10 | [Server Rendering](./10-Server-Rendering/) | SSR, Fizz streaming, hydration, React Server Components |
| 11 | [Error Handling](./11-Error-Handling/) | Error boundaries, fiber unwinding, recovery |
| 12 | [DevTools & Profiling](./12-DevTools-and-Profiling/) | DevTools hook, Profiler, StrictMode |
| 13 | [Complete System Diagram](./13-Complete-System-Diagram/) | Full data flow, architecture map, timing diagrams |
| 14 | [React Compiler](./14-React-Compiler/) | Auto-memoization, HIR, useMemoCache, Rules of React |
| 15 | [Context Propagation](./15-Context-Propagation/) | Context stack, propagateContextChange, deep scan, bypassing memo |
| 16 | [Batching & flushSync](./16-Batching-and-flushSync/) | Automatic batching, React 17 vs 18, flushSync escape hatch |
| 17 | [External Stores & Tearing](./17-External-Stores-and-Tearing/) | Tearing, useSyncExternalStore, consistency checks |
| 18 | [Custom Renderers](./18-Custom-Renderers/) | Host config interface, renderer-agnostic architecture |
| 19 | [Portals](./19-Portals/) | createPortal, fiber vs DOM tree, event bubbling through portals |
| 20 | [React 19 APIs](./20-React-19-APIs/) | use() hook, Actions, useActionState, useOptimistic, Server Actions |
| 21 | [Controlled Components](./21-Controlled-Components/) | Form element internals, useInsertionEffect, CSS-in-JS timing |
| 22 | [Testing Internals](./22-Testing-Internals/) | act(), test renderer, async flushing, StrictMode in tests |
| 23 | [Memory & Performance](./23-Memory-and-Performance/) | GC, fiber cleanup, memory leaks, performance anti-patterns |
| 24 | [State Preservation & Reset](./24-State-Preservation-and-Reset/) | When state survives, position vs type vs key, component identity rules |
| 25 | [Resource Loading](./25-Resource-Loading/) | preload/preinit, stylesheet precedence, document metadata hoisting |
| 26 | [HTML Sanitization & Security](./26-HTML-Sanitization-and-Security/) | XSS prevention, dangerouslySetInnerHTML, $$typeof, URL sanitization |
| 27 | [SVG & Namespace Handling](./27-SVG-and-Namespace-Handling/) | Namespace context stack, createElementNS, SVG attribute mapping |
| 28 | [React.Children Utilities](./28-React-Children-Utilities/) | mapChildren flattening, key remapping, count/toArray/only |
| 29 | [IndeterminateComponent](./29-Indeterminate-Component-Resolution/) | First-render tag resolution, function vs class detection, SimpleMemoComponent |
| 30 | [Edge Cases & Scenarios](./30-Edge-Cases-and-Scenarios/) | Render/commit/concurrent/hooks/Suspense/SSR/event edge cases |
| 31 | [Interview Questions](./31-Interview-Questions/) | 40 frequently asked React internals interview questions with answers |
| 32 | [MCQs](./32-MCQs/) | 500 multiple-choice questions covering all React internals topics |

## Quick Reference

### The Update Lifecycle (5-Phase Summary)

```
Phase 0: EVENT        → Synthetic event dispatch, handler runs, setState called
Phase 1: SCHEDULE     → Update created, lane assigned, render scheduled
Phase 2: RENDER       → Fiber tree traversal (beginWork/completeWork), diffing
Phase 3: COMMIT       → DOM mutations, layout effects, tree swap
Phase 4: PAINT        → Browser layout, paint, composite
Phase 5: PASSIVE      → useEffect cleanup and setup (async)
```

### Key Source Files

| File | Purpose |
|------|---------|
| `ReactElement.js` | createElement / jsx |
| `ReactFiber.js` | Fiber node creation |
| `ReactFiberWorkLoop.js` | Work loop, render entry |
| `ReactFiberBeginWork.js` | Top-down fiber processing |
| `ReactFiberCompleteWork.js` | Bottom-up, DOM creation |
| `ReactFiberCommitWork.js` | Commit phase |
| `ReactFiberHooks.js` | All hook implementations |
| `ReactFiberLane.js` | Lane priority system |
| `ReactChildFiber.js` | Child reconciliation |
| `Scheduler.js` | Task scheduling |
| `DOMPluginEventSystem.js` | Event delegation |
