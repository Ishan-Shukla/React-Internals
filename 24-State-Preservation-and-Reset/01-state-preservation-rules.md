# State Preservation and Reset Rules

## The Fundamental Rule

React preserves component state when **the same component type appears at
the same position in the tree** across renders. If either the type or the
position changes, React **destroys** the old state and starts fresh.

This is one of the most important mental models in React — getting it wrong
causes subtle bugs where state persists when it shouldn't, or resets when
you expect it to stay.

## Rule 1: Same Type + Same Position = State Preserved

```jsx
function App({ showFancy }) {
  return (
    <div>
      {showFancy ? (
        <Counter style="fancy" />    // Position 0 in div's children
      ) : (
        <Counter style="plain" />    // Position 0 in div's children
      )}
    </div>
  );
}
```

```
showFancy = true:
  div
   └── Counter (style="fancy")    ← position 0, type = Counter

showFancy = false:
  div
   └── Counter (style="plain")    ← position 0, type = Counter

SAME type (Counter) at SAME position (0) → STATE PRESERVED!
The count inside Counter does NOT reset when showFancy toggles.

Internally:
  beginWork compares:
    current fiber type === WIP element type?
    Counter === Counter → YES
    → useFiber(current, newProps)  ← REUSE the existing fiber
    → fiber.memoizedState is kept (hooks linked list intact)
    → Only props change (style: "fancy" → "plain")
```

## Rule 2: Different Type = State Destroyed

```jsx
function App({ isLoggedIn }) {
  return (
    <div>
      {isLoggedIn ? (
        <UserProfile />    // Position 0, type = UserProfile
      ) : (
        <LoginForm />      // Position 0, type = LoginForm
      )}
    </div>
  );
}
```

```
isLoggedIn = true:
  div
   └── UserProfile    ← position 0, type = UserProfile

isLoggedIn = false:
  div
   └── LoginForm      ← position 0, type = LoginForm

DIFFERENT type at same position → STATE DESTROYED!
UserProfile fiber is deleted, LoginForm fiber is created fresh.

Internally:
  reconcileSingleElement:
    current fiber type (UserProfile) !== element type (LoginForm)
    → deleteChild(returnFiber, currentFiber)  ← DESTROY old
    → createFiberFromElement(element)          ← CREATE new
    → All state, hooks, effects are gone
```

## Rule 3: Different Position = State Destroyed

```jsx
function App({ showHeader }) {
  return (
    <div>
      {showHeader && <Header />}
      <Counter />
    </div>
  );
}
```

```
showHeader = true:
  div
   ├── Header     ← position 0 (index 0)
   └── Counter    ← position 1 (index 1)

showHeader = false:
  div
   ├── false      ← position 0 (falsy, no fiber)
   └── Counter    ← position 1 (index 1)

Wait — Counter is at position 1 in both cases!
But in the reconciler, the index mapping depends on siblings.

Actually, let's be precise. React reconciles children as an array:
  true:  children = [<Header />, <Counter />]
  false: children = [false, <Counter />]

  Index 0: Header vs false → type mismatch → delete Header
  Index 1: Counter vs Counter → same type → PRESERVE state ✓

Counter state IS preserved here because its array index (1) is stable.

BUT consider this version:

function App({ showHeader }) {
  return (
    <div>
      {showHeader ? <Header /> : null}
      <Counter />
    </div>
  );
}

  true:  children = [<Header />, <Counter />]
  false: children = [null, <Counter />]

  Index 0: Header vs null → delete Header
  Index 1: Counter vs Counter → PRESERVE ✓

Same result. But THIS version is different:

function App({ showHeader }) {
  if (showHeader) {
    return (
      <div>
        <Header />
        <Counter />
      </div>
    );
  }
  return (
    <div>
      <Counter />
    </div>
  );
}

  true:  div children = [<Header />, <Counter />]
  false: div children = [<Counter />]

  Index 0: Header vs Counter → DIFFERENT TYPE → destroy Header, create Counter
  Counter state RESETS because it moved from position 1 to position 0!
```

## Rule 4: Keys Override Position

Keys provide an explicit identity that overrides positional matching:

```jsx
function App({ user }) {
  return <Profile key={user.id} data={user} />;
}
```

```
user = { id: "alice", name: "Alice" }:
  Profile (key="alice")

user = { id: "bob", name: "Bob" }:
  Profile (key="bob")

Different keys → STATE DESTROYED even though same type and position!
React treats key="alice" Profile and key="bob" Profile as
completely different components.

This is the ONLY way to force React to reset state for the same
component type at the same position.
```

## The key Reset Pattern

```jsx
// Force a component to fully remount (reset all state)
// by changing its key:

function ChatRoom({ roomId }) {
  // When roomId changes, the entire MessageList resets
  return <MessageList key={roomId} roomId={roomId} />;
}

// Without key: MessageList state (scroll position, loaded messages)
//              persists across room changes → BUG!
// With key:    MessageList is destroyed and recreated → fresh state ✓
```

## How the Reconciler Decides: The Complete Algorithm

```
Given: current fiber at position N, new element at position N

Step 1: Check key
  ┌─────────────────────────┐
  │ current.key === element.key? │
  └─────────┬───────────────┘
        ┌───┴───┐
       YES      NO
        │        │
        ▼        ▼
  Step 2:     DESTROY current
  Check type  CREATE new fiber
        │     (state reset)
        ▼
  ┌──────────────────────────────┐
  │ current.elementType ===       │
  │ element.type?                 │
  └──────────┬───────────────────┘
        ┌────┴────┐
       YES        NO
        │          │
        ▼          ▼
    PRESERVE     DESTROY current
    STATE        CREATE new fiber
    Update props (state reset)
    Reuse fiber
```

## Visual: State Preservation Decision Tree

```
                    New render produces element at position P
                                     │
                         Is there a current fiber at P?
                              ┌──────┴──────┐
                             YES            NO
                              │              │
                              ▼              ▼
                    Compare keys         CREATE new fiber
                    ┌───────┴───────┐    (initial mount)
                  Same key        Different key
                  (or both null)     │
                    │                ▼
                    ▼           DESTROY old fiber
                Compare type   CREATE new fiber
                ┌───┴───┐     (full state reset)
              Same    Different
              type    type
                │       │
                ▼       ▼
           REUSE    DESTROY old
           fiber    CREATE new
           (keep    (state reset)
            state,
            update
            props)
```

## Host Component Type Changes

```jsx
function App({ useSection }) {
  if (useSection) {
    return (
      <section>
        <Counter />
      </section>
    );
  }
  return (
    <div>
      <Counter />
    </div>
  );
}
```

```
useSection = true:
  section ← type = 'section'
   └── Counter

useSection = false:
  div ← type = 'div'
   └── Counter

'section' !== 'div' → DIFFERENT TYPE → entire subtree destroyed!
Counter state resets, even though Counter itself didn't change.

This is React's Heuristic #1: different element types produce
entirely different trees. React doesn't try to match children
across type changes.

Internally:
  reconcileSingleElement:
    current.elementType ('section') !== element.type ('div')
    → deleteRemainingChildren(returnFiber, child)
    → createFiberFromElement(element)  ← new div fiber
    → Counter gets a fresh fiber too (child of new div)
```

## Component Identity: Function Reference Matters

```jsx
// ❌ BUG: Component defined inside render
function App() {
  // New function reference every render!
  function InnerCounter() {
    const [count, setCount] = useState(0);
    return <button onClick={() => setCount(c + 1)}>{count}</button>;
  }

  return <InnerCounter />;
}

// Every render:
//   InnerCounter (render 1) !== InnerCounter (render 2)
//   Different function reference = different type!
//   → State resets EVERY render!
//   → Count is always 0!

// ✅ FIX: Define component outside
function StableCounter() {
  const [count, setCount] = useState(0);
  return <button onClick={() => setCount(c + 1)}>{count}</button>;
}

function App() {
  return <StableCounter />;  // Same reference across renders
}
```

```
Why this happens at the fiber level:

  beginWork checks: current.elementType === workInProgress.type

  Render 1: element.type = InnerCounter@0x1234
  Render 2: element.type = InnerCounter@0x5678  ← new function!

  0x1234 !== 0x5678 → different type → DESTROY + RECREATE
```

## Preserving State Across Conditional Branches

```jsx
// ❌ State resets when switching branches
function App({ isEditing }) {
  if (isEditing) {
    return <Input key={undefined} />;  // Position in App's return
  }
  return <Display key={undefined} />;
}
// Input and Display are different types → state doesn't transfer

// To SHARE state across branches, lift it up:
function App({ isEditing }) {
  const [text, setText] = useState('');

  if (isEditing) {
    return <Input value={text} onChange={setText} />;
  }
  return <Display value={text} />;
}
// State lives in App, survives the branch switch
```

## Lists: Why Stable Keys Are Critical for State

```jsx
// Items with state (e.g., checkboxes, text inputs)
function TodoList({ todos }) {
  return todos.map((todo, index) => (
    <TodoItem key={todo.id} data={todo} />
    //        ^^^^^^^^ stable key preserves item state correctly
  ));
}
```

```
todos = [A(id:1), B(id:2), C(id:3)]

Delete B:
  todos = [A(id:1), C(id:3)]

With key={todo.id}:
  key="1": fiber exists → REUSE (A state preserved) ✓
  key="2": no matching element → DELETE (B state destroyed) ✓
  key="3": fiber exists → REUSE (C state preserved) ✓

With key={index}:
  key=0: was A, still A → REUSE (correct)
  key=1: was B, now C → type matches → REUSE B's fiber with C's props
         → B's internal state (checkbox, input) now lives on C! ✗
  key=2: was C, now gone → DELETE
         → C's state is destroyed, but it shouldn't have been ✗

This is the #1 key-related bug in React applications.
```

## Fragment Children and Position

```jsx
// Fragments are transparent to positioning:

function App() {
  return (
    <div>
      <>
        <A />    {/* position 0 inside div */}
        <B />    {/* position 1 inside div */}
      </>
      <C />      {/* position 2 inside div */}
    </div>
  );
}

// Fragment fiber exists in the fiber tree but is transparent
// for child reconciliation — children are "flattened" for
// position calculation.

// This means <>...</> does NOT create an extra nesting level
// for state preservation purposes. A, B, C positions are
// relative to div, not to the fragment.
```

## Summary Table

```
Scenario                                    State Preserved?
─────────────────────────────────────────   ────────────────
Same type, same position, no key            YES ✓
Same type, same position, same key          YES ✓
Same type, same position, different key     NO ✗ (reset!)
Same type, different position               NO ✗
Different type, same position               NO ✗
Different type, different position          NO ✗
Component defined inside render             NO ✗ (every render)
Parent type changes (div → section)         NO ✗ (subtree destroyed)
Conditional rendering with same type        YES ✓ (if position stable)
List item with stable key                   YES ✓ (even if reordered)
List item with index key                    WRONG STATE (key collision)
```
