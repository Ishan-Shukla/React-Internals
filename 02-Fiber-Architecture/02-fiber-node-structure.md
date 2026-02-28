# Fiber Node Structure

## Complete Fiber Node Fields

Every fiber node in React contains these fields. This is the actual data
structure from `packages/react-reconciler/src/ReactFiber.js`:

```javascript
function FiberNode(tag, pendingProps, key, mode) {
  // ════════════════════════════════════════════════════
  // IDENTITY
  // ════════════════════════════════════════════════════
  this.tag = tag;                    // FunctionComponent, HostComponent, etc.
  this.key = key;                    // Reconciliation key from JSX
  this.elementType = null;           // The original type (before resolving)
  this.type = null;                  // The resolved type (function, string, etc.)

  // ════════════════════════════════════════════════════
  // TREE STRUCTURE (linked list pointers)
  // ════════════════════════════════════════════════════
  this.return = null;                // Parent fiber
  this.child = null;                 // First child fiber
  this.sibling = null;               // Next sibling fiber
  this.index = 0;                    // Position among siblings

  // ════════════════════════════════════════════════════
  // REFS
  // ════════════════════════════════════════════════════
  this.ref = null;                   // Ref callback or ref object
  this.refCleanup = null;            // Cleanup for ref callback

  // ════════════════════════════════════════════════════
  // PROPS AND STATE
  // ════════════════════════════════════════════════════
  this.pendingProps = pendingProps;   // New props (from element)
  this.memoizedProps = null;          // Props from last render
  this.memoizedState = null;          // State from last render (hooks linked list)
  this.updateQueue = null;            // Queue of pending state updates

  // ════════════════════════════════════════════════════
  // CONTEXT
  // ════════════════════════════════════════════════════
  this.dependencies = null;           // Contexts this fiber reads from

  // ════════════════════════════════════════════════════
  // MODE FLAGS
  // ════════════════════════════════════════════════════
  this.mode = mode;                  // ConcurrentMode, StrictMode, etc.

  // ════════════════════════════════════════════════════
  // EFFECTS
  // ════════════════════════════════════════════════════
  this.flags = NoFlags;              // Side-effect flags (bitmask)
  this.subtreeFlags = NoFlags;       // Aggregated child flags
  this.deletions = null;             // Child fibers to delete

  // ════════════════════════════════════════════════════
  // SCHEDULING (LANES)
  // ════════════════════════════════════════════════════
  this.lanes = NoLanes;              // Pending work priority on this fiber
  this.childLanes = NoLanes;         // Pending work priority in subtree

  // ════════════════════════════════════════════════════
  // DOUBLE BUFFERING
  // ════════════════════════════════════════════════════
  this.alternate = null;             // Points to the other version of this fiber

  // ════════════════════════════════════════════════════
  // HOST INSTANCE
  // ════════════════════════════════════════════════════
  this.stateNode = null;             // DOM node, class instance, or FiberRoot
}
```

## Field-by-Field Explanation

### Identity Fields

```
┌────────────────────────────────────────────────────────────────┐
│  tag          │ Numeric constant from ReactWorkTags.js         │
│               │ Determines how React processes this fiber      │
│               │ e.g., FunctionComponent=0, HostComponent=5     │
├───────────────┼────────────────────────────────────────────────┤
│  key          │ The `key` from JSX, used for list diffing      │
│               │ "user-123" or null if no key                   │
├───────────────┼────────────────────────────────────────────────┤
│  elementType  │ Raw type from JSX (before resolution)          │
│               │ e.g., memo(MyComp) → elementType = memoObject  │
├───────────────┼────────────────────────────────────────────────┤
│  type         │ Resolved type after unwrapping                 │
│               │ For memo(MyComp) → type = MyComp               │
│               │ For 'div' → type = 'div'                       │
│               │ For function App → type = App                  │
└───────────────┴────────────────────────────────────────────────┘
```

### Tree Structure Fields

```
             return (parent)
                ▲
                │
    ┌───────────┴──────────┐
    │     Current Fiber    │
    │                      │
    └──┬───────────────┬───┘
       │               │
       ▼               ▼
    child          sibling
  (1st child)    (next sibling)

Example:
  <div>           return=null, child=A, sibling=null
    <A />         return=div, child=null, sibling=B, index=0
    <B />         return=div, child=null, sibling=C, index=1
    <C />         return=div, child=null, sibling=null, index=2
  </div>

Note: Only the FIRST child is linked via `child`.
      Other children are linked via `sibling` chains.
      All children point back to parent via `return`.
```

### Props and State Fields

```
pendingProps vs memoizedProps:

  ┌──────────────────────┐     ┌──────────────────────┐
  │ Before render:       │     │ After render:        │
  │                      │     │                      │
  │ pendingProps = new   │ ──► │ memoizedProps = new  │
  │ memoizedProps = old  │     │ (copied from pending)│
  └──────────────────────┘     └──────────────────────┘

  If pendingProps === memoizedProps → potential bailout (no re-render)

memoizedState:
  • For CLASS components:  the this.state object
  • For FUNCTION components:  the hooks linked list head
  • For HostRoot:  the element passed to root.render()
```

### stateNode: The Real Instance

```
Fiber tag               stateNode value
──────────────────      ────────────────────────────────
HostRoot (3)            FiberRoot object
HostComponent (5)       DOM Element (e.g., <div>)
HostText (6)            DOM Text Node
ClassComponent (1)      Class instance (this)
FunctionComponent (0)   null (no instance)
```

### Effect Flags (Bitmask)

```javascript
// packages/react-reconciler/src/ReactFiberFlags.js

export const NoFlags          = 0b0000000000000000000000000000;
export const Placement        = 0b0000000000000000000000000010;  // New insertion
export const Update           = 0b0000000000000000000000000100;  // Props changed
export const ChildDeletion    = 0b0000000000000000000000010000;  // Child removed
export const ContentReset     = 0b0000000000000000000000100000;  // Text changed
export const Callback         = 0b0000000000000000000001000000;  // setState cb
export const Ref              = 0b0000000000000000001000000000;  // Ref changed
export const Snapshot         = 0b0000000000000000010000000000;  // getSnapshot
export const Passive          = 0b0000000000000000100000000000;  // useEffect
export const LayoutMask       = Update | Callback | Ref;
export const PassiveMask      = Passive | ChildDeletion;
```

Flags are combined with bitwise OR and checked with bitwise AND:

```javascript
// Mark fiber for placement and ref update
fiber.flags |= Placement | Ref;

// Check if fiber needs placement
if (fiber.flags & Placement) {
  // Insert this node into the DOM
}

// subtreeFlags aggregates ALL descendant flags
// This allows skipping entire subtrees during commit:
if (fiber.subtreeFlags === NoFlags) {
  // No effects in entire subtree — skip it entirely!
}
```

## Memory Layout Visualization

```
FiberNode @0x7f2a (for <div className="app">)
┌─────────────────────────────────────────────────────┐
│ tag: 5 (HostComponent)                              │
│ type: "div"                                         │
│ key: null                                           │
│                                                     │
│ return: ───────────► FiberNode @0x7f10 (App)        │
│ child:  ───────────► FiberNode @0x7f3b (Header)     │
│ sibling: ──────────► null                           │
│                                                     │
│ pendingProps:  { className: "app" }                 │
│ memoizedProps: { className: "app" }                 │
│ memoizedState: null                                 │
│ updateQueue: null                                   │
│                                                     │
│ stateNode: ────────► <div class="app"> (DOM)        │
│ alternate: ────────► FiberNode @0x8a2a (WIP copy)   │
│                                                     │
│ flags: 0b0000 (NoFlags)                             │
│ subtreeFlags: 0b0100 (Update) ◄── child has update  │
│ lanes: NoLanes                                      │
│ childLanes: DefaultLane ◄── child has pending work  │
└─────────────────────────────────────────────────────┘
```
