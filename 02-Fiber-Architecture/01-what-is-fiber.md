# What Is a Fiber?

## Definition

A **Fiber** is a JavaScript object that represents a **unit of work**. Each
fiber corresponds to a React element (component or DOM node) and holds all the
information React needs to:

1. Track the component's state and props
2. Determine what changed since last render
3. Schedule and prioritize updates
4. Connect to the actual DOM

Think of it this way:
- A **ReactElement** is a description of what to render (immutable, throwaway)
- A **Fiber** is the living, mutable internal representation that persists across renders

## Fiber = Virtual Stack Frame

The key insight behind fibers: they are **virtual stack frames**.

In a normal recursive algorithm, work state lives on the call stack — which the
JS engine controls. You can't pause, serialize, or prioritize it.

A fiber moves that state into **heap-allocated objects** that React controls:

```
CALL STACK FRAME                    FIBER NODE
(engine-controlled)                 (React-controlled)
────────────────────                ────────────────────
• function reference                • type (component fn/class)
• local variables                   • memoizedState (hooks/state)
• return address                    • return (parent fiber)
• arguments                         • pendingProps / memoizedProps
• ← can't pause                    • ← CAN pause, resume, abort
```

## Why "Fiber"?

The name comes from systems programming: a **fiber** is a lightweight thread
of execution that uses **cooperative multitasking** (it explicitly yields
control, as opposed to threads which are preemptively scheduled by the OS).

React's fibers cooperatively yield to the browser's main thread via the
scheduler's `shouldYield()` check.

## One Fiber Per Element

Every element in your React tree has a corresponding fiber:

```jsx
<App>                           FiberNode (tag: FunctionComponent)
  <div>                         FiberNode (tag: HostComponent)
    <Header />                  FiberNode (tag: FunctionComponent)
    <p>Hello</p>                FiberNode (tag: HostComponent)
      "Hello"                   FiberNode (tag: HostText)
    <Counter count={0} />       FiberNode (tag: FunctionComponent)
  </div>
</App>
```

## Fiber Persistence

Unlike elements (which are recreated every render), fibers **persist**.
On re-render, React doesn't throw away the fiber tree. Instead it:

1. Creates new elements (via component render)
2. Diffs new elements against existing fibers
3. Updates fibers in-place (or creates new ones / deletes old ones)

```
Render 1:  Element Tree  ──►  Create Fiber Tree
Render 2:  Element Tree  ──►  Update Fiber Tree (diff against previous)
Render 3:  Element Tree  ──►  Update Fiber Tree (diff against previous)
...

The fiber tree is long-lived. Elements are ephemeral.
```
