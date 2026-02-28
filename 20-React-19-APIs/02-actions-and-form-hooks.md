# React 19: Actions, useActionState, useFormStatus, useOptimistic

## Actions: The Concept

Actions are async functions that handle form submissions and data mutations.
React 19 introduces first-class support for them with automatic pending
state management, error handling, and optimistic updates.

```javascript
// An "action" is any async function used with form or startTransition
async function submitForm(formData) {
  const name = formData.get('name');
  await saveToDatabase(name);
  return { success: true };
}
```

## useActionState

Replaces the manual pattern of `useState` + `useEffect` + `try/catch` for
form actions:

```javascript
import { useActionState } from 'react';

function Form() {
  const [state, formAction, isPending] = useActionState(
    async (previousState, formData) => {
      // This is the action function
      const name = formData.get('name');
      const error = await submitToServer(name);
      if (error) {
        return { error: error.message };
      }
      return { success: true };
    },
    { error: null, success: false }  // Initial state
  );

  return (
    <form action={formAction}>
      <input name="name" />
      <button disabled={isPending}>
        {isPending ? 'Submitting...' : 'Submit'}
      </button>
      {state.error && <p className="error">{state.error}</p>}
    </form>
  );
}
```

### useActionState Internals

```javascript
// Simplified internal implementation

function mountActionState(action, initialState, permalink) {
  // Create state hook (like useState)
  const stateHook = mountWorkInProgressHook();
  stateHook.memoizedState = initialState;

  // Create pending hook (tracks in-flight action)
  const pendingHook = mountWorkInProgressHook();
  pendingHook.memoizedState = false;  // isPending

  // Create the action queue
  const actionQueue = {
    state: initialState,
    dispatch: null,
    action: action,
    pending: null,
  };

  // Create the dispatch function
  const dispatch = dispatchActionState.bind(
    null, currentlyRenderingFiber, actionQueue,
  );
  actionQueue.dispatch = dispatch;
  stateHook.queue = actionQueue;

  // Return [state, dispatch, isPending]
  return [initialState, dispatch, false];
}

function dispatchActionState(fiber, actionQueue, actionInput) {
  // 1. Set isPending = true (at TransitionLane)
  startTransition(() => {
    // isPending becomes true during this transition

    // 2. Call the action
    const returnValue = actionQueue.action(actionQueue.state, actionInput);

    if (isThenable(returnValue)) {
      // Async action → handle promise
      returnValue.then(
        newState => {
          // 3. Update state with result
          actionQueue.state = newState;
          // 4. isPending = false (end transition)
          dispatchSetState(fiber, pendingQueue, false);
          dispatchSetState(fiber, stateQueue, newState);
        },
        error => {
          // Error handling
          dispatchSetState(fiber, pendingQueue, false);
        }
      );
    }
  });
}
```

```
useActionState lifecycle:

  Initial: state = { error: null }, isPending = false

  User clicks Submit:
  ┌────────────────────────────────────────────────────┐
  │ 1. dispatchActionState fires                       │
  │ 2. startTransition → isPending = true              │
  │    → Re-render: button shows "Submitting..."       │
  │                                                    │
  │ 3. action(previousState, formData) called          │
  │    → async: await saveToServer(...)                │
  │                                                    │
  │ 4a. Success: action returns { success: true }      │
  │     → state = { success: true }                    │
  │     → isPending = false                            │
  │     → Re-render: show success UI                   │
  │                                                    │
  │ 4b. Error: action returns { error: "Failed" }      │
  │     → state = { error: "Failed" }                  │
  │     → isPending = false                            │
  │     → Re-render: show error message                │
  └────────────────────────────────────────────────────┘
```

## useFormStatus

Reads the status of a parent `<form>` from inside a child component.
Uses React's context system internally:

```javascript
import { useFormStatus } from 'react-dom';

function SubmitButton() {
  const { pending, data, method, action } = useFormStatus();
  //      ^^^^^^^ true while form is submitting

  return (
    <button disabled={pending}>
      {pending ? 'Saving...' : 'Save'}
    </button>
  );
}

function Form() {
  return (
    <form action={submitAction}>
      <input name="email" />
      <SubmitButton />  {/* Reads form status via useFormStatus */}
    </form>
  );
}
```

### How useFormStatus Works Internally

```
When a <form> submits with an action:

1. React wraps the form submission in a transition
2. Sets form status context:
   { pending: true, data: formData, method: 'POST', action: fn }

3. Any descendant calling useFormStatus reads this context
   (it's a built-in context provided by react-dom)

4. When the action completes:
   → pending = false
   → Re-render propagates

Important: useFormStatus reads from the NEAREST parent <form>.
           It does NOT work for the same component's own form.
           It must be called from a CHILD component of the form.
```

## useOptimistic

Shows a temporary "optimistic" value while an async action is in progress:

```javascript
import { useOptimistic } from 'react';

function TodoList({ todos, addTodoAction }) {
  const [optimisticTodos, addOptimisticTodo] = useOptimistic(
    todos,
    // Reducer: how to merge optimistic update
    (currentTodos, newTodoText) => [
      ...currentTodos,
      { id: 'temp', text: newTodoText, sending: true },
    ]
  );

  async function handleSubmit(formData) {
    const text = formData.get('todo');

    // Show optimistic update immediately
    addOptimisticTodo(text);

    // Actually submit (takes time)
    await addTodoAction(text);
    // When this completes, the real `todos` prop updates
    // and the optimistic value is automatically replaced
  }

  return (
    <form action={handleSubmit}>
      <ul>
        {optimisticTodos.map(todo => (
          <li key={todo.id} style={{ opacity: todo.sending ? 0.5 : 1 }}>
            {todo.text}
          </li>
        ))}
      </ul>
      <input name="todo" />
      <button>Add</button>
    </form>
  );
}
```

### useOptimistic Internals

```javascript
// Simplified implementation

function mountOptimistic(passthrough, reducer) {
  const hook = mountWorkInProgressHook();

  hook.memoizedState = passthrough;  // Start with real value
  hook.queue = {
    pending: null,
    dispatch: null,
    lastRenderedReducer: reducer,
  };

  const dispatch = dispatchOptimisticUpdate.bind(
    null, currentlyRenderingFiber, hook.queue
  );
  hook.queue.dispatch = dispatch;

  return [passthrough, dispatch];
}

function updateOptimistic(passthrough, reducer) {
  const hook = updateWorkInProgressHook();

  // During a transition (action in progress):
  //   Apply optimistic updates on top of passthrough
  //   return reduced value

  // After transition completes:
  //   Optimistic updates are cleared
  //   return passthrough (real value from props)

  return [resolvedValue, hook.queue.dispatch];
}
```

```
useOptimistic timeline:

  Real todos: [A, B]
  User submits "C"

  ┌──────────────────────────────────────────────┐
  │ Immediately:                                  │
  │   addOptimisticTodo("C")                      │
  │   optimisticTodos = [A, B, C(sending)]        │
  │   UI shows: A, B, C(grayed)                   │
  ├──────────────────────────────────────────────┤
  │ While awaiting server:                        │
  │   optimisticTodos = [A, B, C(sending)]        │
  │   UI shows: A, B, C(grayed)                   │
  ├──────────────────────────────────────────────┤
  │ Server responds, parent re-renders with:      │
  │   todos = [A, B, C]  (real prop)              │
  │   Optimistic state automatically cleared      │
  │   optimisticTodos = [A, B, C]                 │
  │   UI shows: A, B, C  (fully opaque)           │
  └──────────────────────────────────────────────┘
```

## Server Actions

Server Actions are functions that run on the server, callable from client
components:

```javascript
// server-action.js
'use server';

export async function addTodo(formData) {
  const text = formData.get('text');
  await db.todos.create({ text });
  revalidatePath('/todos');
}

// client-component.jsx
'use client';
import { addTodo } from './server-action';

function AddTodoForm() {
  return (
    <form action={addTodo}>
      <input name="text" />
      <button>Add</button>
    </form>
  );
}
```

### How Server Actions Work

```
Client                                    Server
──────                                    ──────

1. <form action={addTodo}>
   User clicks Submit

2. React serializes FormData
   POST /__action
   Body: { actionId: "abc123", formData: {...} }
                                    ────►
                                          3. Deserialize request
                                          4. Call addTodo(formData)
                                          5. Execute server-side logic
                                             await db.todos.create(...)
                                          6. revalidatePath('/todos')
                                          7. Re-render affected RSC
                                          8. Return Flight payload
                                    ◄────
9. React receives Flight payload
10. Reconcile new server component tree
11. Update client components with new props
12. Transition completes, isPending = false

The action ID is generated at build time by the RSC bundler.
The client never has the server function code — just the ID.
```

## Ref Callback Cleanup (React 19)

```javascript
// React 19: ref callbacks can return a cleanup function

function Component() {
  return (
    <input ref={(node) => {
      if (node) {
        // Setup: node mounted
        const observer = new ResizeObserver(() => { ... });
        observer.observe(node);

        // NEW: Return cleanup function
        return () => {
          observer.disconnect();
        };
      }
    }} />
  );
}

// Before React 19, you had to handle cleanup manually:
// ref={(node) => { if (node) setup(); else cleanup(); }}
// The null call for cleanup was confusing and error-prone
```
