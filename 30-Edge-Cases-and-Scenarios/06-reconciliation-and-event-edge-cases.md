# Edge Cases: Reconciliation & Event System

## 1. Key Changes Force Complete Remount

```jsx
function App({ userId }) {
  return (
    <div>
      {/* Changing key destroys and recreates the entire subtree */}
      <UserProfile key={userId} userId={userId} />
    </div>
  );
}
```

```
What happens when userId changes from "alice" to "bob":

  Before: <UserProfile key="alice" userId="alice" />
  After:  <UserProfile key="bob" userId="bob" />

  Reconciliation:
    1. reconcileSingleElement compares keys
    2. "alice" !== "bob" → keys don't match
    3. Mark old fiber (key="alice") for DELETION
    4. Create BRAND NEW fiber (key="bob")

  Consequences:
    - All state inside UserProfile is DESTROYED
    - All useEffect cleanups run (unmount)
    - DOM nodes removed and recreated
    - All child components also destroyed and recreated
    - useEffect setups run fresh (mount)
    - useState initializers run again

  This is INTENTIONAL and useful:
    - Reset form state when switching users
    - Force re-initialization of effects
    - Clear stale state from previous entity

  Performance note:
    Key change = full unmount + remount
    This is MORE expensive than a re-render with new props
    Only use when you actually want state reset
```

## 2. List Reconciliation: The Two-Pass Algorithm

```jsx
// Before: [A, B, C, D, E]
// After:  [B, E, C, A]  (D removed, reordered)
```

```
Pass 1: Compare old and new lists in order

  oldFiber: A → B → C → D → E
  newChildren: [B, E, C, A]

  Compare index 0: old=A, new=B → keys differ → BREAK
  (Stop pass 1 as soon as keys don't match)

Pass 2: Map remaining old fibers by key

  existingChildren = Map {
    "A" → fiberA,
    "B" → fiberB,
    "C" → fiberC,
    "D" → fiberD,
    "E" → fiberE
  }

  Walk new children:
    new[0] = B → map.get("B") → found fiberB → REUSE, mark moved
    new[1] = E → map.get("E") → found fiberE → REUSE, mark moved
    new[2] = C → map.get("C") → found fiberC → REUSE, mark moved
    new[3] = A → map.get("A") → found fiberA → REUSE, mark moved

  Remaining in map: { "D" → fiberD }
    → fiberD is not in new list → mark for DELETION

  Placement algorithm (lastPlacedIndex):
    Track the last old index that was "in place"

    B: oldIndex=1, lastPlacedIndex=0 → 1 >= 0 → in place, lastPlacedIndex=1
    E: oldIndex=4, lastPlacedIndex=1 → 4 >= 1 → in place, lastPlacedIndex=4
    C: oldIndex=2, lastPlacedIndex=4 → 2 < 4 → NEEDS MOVE (Placement flag)
    A: oldIndex=0, lastPlacedIndex=4 → 0 < 4 → NEEDS MOVE (Placement flag)

  Result: B and E stay, C and A are moved, D is deleted
  DOM operations: move C, move A, remove D
```

## 3. Index Keys: The Anti-Pattern

```jsx
// ❌ Using index as key
{items.map((item, index) => (
  <Input key={index} defaultValue={item.text} />
))}

// Items: ["Apple", "Banana", "Cherry"]
// Remove "Banana" (index 1)
// New items: ["Apple", "Cherry"]
```

```
What goes wrong with index keys:

  Before:
    key=0 → Input with "Apple"  (state: user typed "A!")
    key=1 → Input with "Banana" (state: user typed "B!")
    key=2 → Input with "Cherry" (state: user typed "C!")

  After removing "Banana":
    key=0 → Input with "Apple"  → matches old key=0 → REUSE fiber → state "A!" ✓
    key=1 → Input with "Cherry" → matches old key=1 → REUSE fiber → state "B!" ✗
    key=2 → nothing → old key=2 deleted

  Result:
    "Apple"  shows state "A!" ← correct
    "Cherry" shows state "B!" ← WRONG! Got Banana's state!

  The state for "Cherry" is "B!" because:
    - key=1 matched the old key=1 (which was Banana)
    - React reused Banana's fiber (with Banana's state)
    - Only the PROPS changed (defaultValue), not the STATE
    - uncontrolled input keeps its old state

  With stable keys:
    key="apple"  → matches old "apple"  → state "A!" ✓
    key="cherry" → matches old "cherry" → state "C!" ✓
    key="banana" → no match → deleted (state "B!" gone) ✓
```

## 4. Component Type Change at Same Position

```jsx
function App({ isAdmin }) {
  return (
    <div>
      {isAdmin ? <AdminPanel /> : <UserPanel />}
    </div>
  );
}
```

```
When isAdmin changes from true to false:

  Old tree: div → AdminPanel (FunctionComponent, type=AdminPanel)
  New tree: div → UserPanel (FunctionComponent, type=UserPanel)

  Reconciliation at position 0 of div's children:
    oldFiber.type === AdminPanel
    newElement.type === UserPanel
    AdminPanel !== UserPanel → DIFFERENT TYPE

  React's behavior:
    1. Delete AdminPanel fiber and entire subtree
    2. Create new UserPanel fiber from scratch
    3. All state inside AdminPanel is LOST
    4. All state inside UserPanel starts fresh

  Even if AdminPanel and UserPanel have identical structure:
    function AdminPanel() { return <div><Input /></div>; }
    function UserPanel() { return <div><Input /></div>; }
    → Still destroyed and recreated! Type check is by reference.

  Important subtlety:
    function App() {
      const Panel = isAdmin ? AdminPanel : UserPanel;
      return <Panel />;
    }
    → Same behavior! React compares Panel function references.

  But if you define component inline:
    function App() {
      // ❌ New function reference every render!
      const Panel = () => <div>Content</div>;
      return <Panel />;
    }
    → Panel is a NEW function each render
    → React sees different type each time
    → Destroys and recreates EVERY render
    → Terrible performance and state always lost
```

## 5. Fragment Reconciliation

```jsx
// Fragments are transparent to reconciliation:
function App() {
  return (
    <div>
      <>
        <A />
        <B />
      </>
      <C />
    </div>
  );
}
```

```
Fiber tree (Fragment is a real fiber node):

  div (HostComponent)
   ├── Fragment (tag=7)
   │    ├── A (position 0 within fragment)
   │    └── B (position 1 within fragment)
   └── C (position 1 of div's children)

Reconciliation processes Fragment as a regular child:
  div has children: [Fragment, C]
  Fragment has children: [A, B]

  If Fragment changes to a single element:
    Before: <>{<A /><B />}</>
    After:  <>{<A />}</>
    → B is deleted, A is updated
    → Fragment fiber stays (same type)

  If Fragment is removed entirely:
    Before: <><A /><B /></>
    After:  <A />
    → Fragment (type=Symbol(react.fragment)) vs A (type=A function)
    → Different types → Fragment deleted
    → A created fresh at position 0

  Keyed fragments:
    <Fragment key="group1">...</Fragment>
    → Fragment participates in key-based reconciliation
    → Useful when fragments are in lists
```

## 6. Event Delegation Through Portals

```jsx
function App() {
  const [clicks, setClicks] = useState(0);

  return (
    <div onClick={() => setClicks(c => c + 1)}>
      Clicks: {clicks}
      {createPortal(
        <button>Click Me</button>,
        document.getElementById('portal-root')  // Different DOM parent!
      )}
    </div>
  );
}
```

```
DOM tree:
  <div id="root">
    <div>                    ← has onClick handler
      Clicks: 0
    </div>
  </div>
  <div id="portal-root">
    <button>Click Me</button> ← rendered here by portal
  </div>

Fiber tree:
  App
   └── div (onClick handler)
        ├── "Clicks: 0"
        └── Portal
             └── button ("Click Me")

When button is clicked:

  DOM event bubbling:
    button → #portal-root → body → html → document
    → The click does NOT reach the div in #root via DOM bubbling!

  React event handling:
    → Click captured at root (#root) via delegation
    → React finds the fiber for button
    → Walks UP the fiber tree: button → Portal → div → App
    → Encounters onClick on div fiber
    → onClick fires! setClicks(c => c + 1) ✓

  Key insight:
    React events bubble through the FIBER tree, not the DOM tree.
    Portals maintain fiber parentage even though DOM parentage differs.
    This is usually what you want — the portal is logically part of
    the component that created it.

  To STOP React's fiber-tree bubbling:
    e.stopPropagation() on the portal's content
    → Stops React event propagation through fiber tree
    → onClick on parent div won't fire
```

## 7. SyntheticEvent Pooling (Legacy) and Modern Behavior

```jsx
function App() {
  const handleClick = (e) => {
    console.log(e.type);  // "click" ✓

    // React 16 (pooling):
    setTimeout(() => {
      console.log(e.type);  // null ❌ (event was pooled!)
    }, 0);

    // React 17+ (no pooling):
    setTimeout(() => {
      console.log(e.type);  // "click" ✓ (event persists)
    }, 0);
  };
}
```

```
React 16 — Event Pooling:
  → SyntheticEvent objects were reused (pooled)
  → After event handler returns, all properties nullified
  → Accessing event properties async → null
  → Fix: e.persist() to keep the event

React 17+ — No Pooling:
  → SyntheticEvent objects are NOT reused
  → Event properties persist after handler returns
  → e.persist() is a no-op (exists for compat)
  → Safe to use events in setTimeout, promises, etc.

Why pooling was removed:
  - Caused widespread confusion
  - Didn't provide measurable performance benefits
  - Modern browsers allocate objects cheaply
  - The persist() workaround was needed too often

SyntheticEvent still exists in React 17+:
  → Wraps native event for cross-browser normalization
  → e.nativeEvent gives access to the real DOM event
  → e.stopPropagation() affects React event propagation
  → e.preventDefault() calls native preventDefault
```

## 8. Event Priority Mapping

```
React maps DOM events to different scheduler priorities:

  DiscreteEventPriority (SyncLane):
    → click, keydown, keyup, input, change, focus, blur
    → These need immediate response
    → Rendered synchronously (not concurrent)

  ContinuousEventPriority (InputContinuousLane):
    → mousemove, mouseenter, mouseleave, scroll, drag, touchmove
    → Frequent events that shouldn't block discrete events
    → Can be batched together

  DefaultEventPriority (DefaultLane):
    → Everything else (load, error, animation events)
    → Normal priority

  How this works internally:
    function dispatchEvent(domEventName, ...) {
      const eventPriority = getEventPriority(domEventName);

      // The priority determines which lane the update gets
      switch (eventPriority) {
        case DiscreteEventPriority:
          // Updates from click handlers → SyncLane
          // Processed synchronously
          break;
        case ContinuousEventPriority:
          // Updates from mousemove → InputContinuousLane
          // Can be interrupted by discrete events
          break;
        case DefaultEventPriority:
          // Normal lane assignment
          break;
      }
    }

  Edge case: setTimeout inside click handler
    onClick = () => {
      setState(1);  // SyncLane (inside discrete event)
      setTimeout(() => {
        setState(2);  // DefaultLane (outside event context!)
      }, 0);
    };
    → Two different lanes for updates in the "same" handler!
```

## 9. Reconciling Children: Single vs Multiple

```jsx
// Going from one child to many:
// Before: <div><span>only</span></div>
// After:  <div><span>first</span><span>second</span></div>
```

```
React optimizes for the common case of a single child:

  reconcileChildFibers checks:
    if (typeof newChild === 'object' && newChild !== null) {
      switch (newChild.$$typeof) {
        case REACT_ELEMENT_TYPE:
          // SINGLE ELEMENT → reconcileSingleElement
          // Fast path: compare with existing child
          break;
      }

      if (isArray(newChild)) {
        // MULTIPLE ELEMENTS → reconcileChildrenArray
        // Two-pass algorithm (see edge case #2 above)
      }
    }

  Single → Single:
    reconcileSingleElement:
      → Compare keys, then type
      → Same? Update props, reuse fiber
      → Different? Delete old, create new

  Single → Multiple:
    → Old: single child fiber at position 0
    → New: array of children
    → reconcileChildrenArray processes:
      → First element may match old fiber (if same key/type)
      → Remaining elements create new fibers

  Multiple → Single:
    → reconcileSingleElement
    → Tries to find a match in old children list
    → If found: reuse that fiber, DELETE all others
    → If not found: delete all old, create new

  Multiple → Zero (null):
    → deleteRemainingChildren(parent, firstChild)
    → Walks sibling list, marks each for deletion
```

## 10. Passive Event Listeners and React

```jsx
function ScrollHandler() {
  useEffect(() => {
    const handler = (e) => {
      e.preventDefault();  // ❌ May not work!
    };

    // React registers wheel/touch as PASSIVE by default
    window.addEventListener('wheel', handler);
    // Equivalent to: { passive: true } in modern browsers
    // preventDefault() is ignored in passive listeners!

    // Fix: explicitly set passive: false
    window.addEventListener('wheel', handler, { passive: false });

    return () => window.removeEventListener('wheel', handler);
  }, []);
}
```

```
React's passive event handling:

  React registers these as PASSIVE on the root:
    - touchstart
    - touchmove
    - wheel

  Why? Performance. Passive listeners let the browser scroll
  without waiting for JavaScript to call preventDefault().
  This makes scrolling smoother.

  But this means:
    <div onWheel={(e) => e.preventDefault()}>
      {/* preventDefault does NOTHING because the listener is passive */}
    </div>

  Workaround:
    Use a ref and add the event listener manually:
    useEffect(() => {
      const el = ref.current;
      const handler = (e) => e.preventDefault();
      el.addEventListener('wheel', handler, { passive: false });
      return () => el.removeEventListener('wheel', handler);
    }, []);

  React's event delegation setup (simplified):
    rootNode.addEventListener('wheel', onWheel, {
      passive: !eventSystemFlags.includes(IS_NON_DELEGATED),
      capture: false
    });

  Non-delegated events (registered directly on element):
    - scroll (doesn't bubble)
    - focus/blur (React uses focusin/focusout for delegation)
    - These are NOT affected by the passive issue
```
