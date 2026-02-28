# Suspense Internals

## How Suspense Works: The Thrown Promise Pattern

Suspense is built on a simple mechanism: **components throw promises** when
they can't render yet, and Suspense boundaries catch them.

```javascript
// This is what React.lazy and data fetching libraries do internally:

function SuspendingComponent({ resource }) {
  const data = resource.read();  // Might throw a promise!
  return <div>{data}</div>;
}

// resource.read() implementation:
const resource = {
  _status: 'pending',
  _result: null,
  _promise: null,

  read() {
    switch (this._status) {
      case 'pending':
        throw this._promise;     // ← THROWS a Promise!
      case 'resolved':
        return this._result;     // ← Returns data
      case 'rejected':
        throw this._result;      // ← Throws error
    }
  }
};
```

## How React Catches the Thrown Promise

During `beginWork`, when React calls your component function and it throws:

```javascript
// Simplified from ReactFiberWorkLoop.js — renderRootSync/renderRootConcurrent

function handleThrow(root, thrownValue) {
  if (
    thrownValue !== null &&
    typeof thrownValue === 'object' &&
    typeof thrownValue.then === 'function'
  ) {
    // ═══ IT'S A THENABLE (Promise) ═══
    // This is a Suspense signal, not an error
    const wakeable = thrownValue;

    // Mark the Suspense boundary
    const suspenseBoundary = getNearestSuspenseBoundary(workInProgress);

    if (suspenseBoundary !== null) {
      suspenseBoundary.flags |= ShouldCapture;

      // Attach a listener to re-render when the promise resolves
      attachPingListener(root, wakeable, renderLanes);
    }

    // Unwind the fiber tree back to the Suspense boundary
    workInProgress = suspenseBoundary;
  } else {
    // It's a real error → find error boundary
    // (covered in Error Handling section)
  }
}
```

## The Suspense Boundary Fiber

```
<Suspense fallback={<Loading />}>
  <AsyncContent />
</Suspense>
```

Creates this fiber structure:

```
SuspenseFiber (tag: SuspenseComponent)
├── .memoizedState = {    // null when not suspended
│     dehydrated: null,   // For hydration
│     treeContext: null,
│     retryLane: NoLane,
│   }
│
├── Primary child (Offscreen fiber)
│   └── AsyncContent fiber
│
└── Fallback child (Fragment fiber)
    └── Loading fiber
```

## Suspense State Machine

```
                    ┌──────────────┐
                    │   MOUNTED    │
                    │  (resolved)  │
                    │              │
                    │ Shows:       │
                    │ Primary      │
                    │ children     │
                    └──────┬───────┘
                           │
                    Child throws Promise
                           │
                           ▼
                    ┌──────────────┐
                    │  SUSPENDED   │
                    │  (pending)   │
                    │              │
                    │ Shows:       │
                    │ Fallback     │
                    │ (Loading...) │
                    └──────┬───────┘
                           │
                    Promise resolves
                    (ping → re-render)
                           │
                           ▼
                    ┌──────────────┐
                    │  RESOLVED    │
                    │              │
                    │ Shows:       │
                    │ Primary      │
                    │ children     │
                    │ (with data)  │
                    └──────────────┘
```

## Ping: Re-rendering After Promise Resolution

```javascript
function attachPingListener(root, wakeable, lanes) {
  // Check if we already have a listener for this promise
  let pingCache = root.pingCache;

  // ...

  // Listen for resolution
  const ping = pingSuspendedRoot.bind(null, root, wakeable, lanes);
  wakeable.then(ping, ping);  // Both resolve and reject trigger re-render
}

function pingSuspendedRoot(root, wakeable, pingedLanes) {
  // Mark these lanes as "pinged" (ready to retry)
  root.pingedLanes |= root.suspendedLanes & pingedLanes;

  // Schedule a new render
  ensureRootIsScheduled(root);
}
```

```
Timeline:

t=0:   Render starts
t=1:   <AsyncContent> throws Promise
       → Suspense catches it
       → Shows <Loading /> fallback
       → Attaches .then(ping) to the Promise
       → Commit: DOM shows loading state

t=500: Promise resolves (data fetched)
       → ping() fires
       → root.pingedLanes |= RetryLane
       → ensureRootIsScheduled(root)

t=501: New render starts at RetryLane
       → <AsyncContent> calls resource.read()
       → Status is 'resolved' → returns data
       → Suspense sees primary content rendered successfully
       → Shows primary content, hides fallback

t=502: Commit: DOM shows actual content
```

## Suspense During Updates (Transitions)

When content is already shown and a re-render suspends:

```javascript
// Without transition: shows fallback immediately (bad UX)
setSomeState(newValue);  // If this causes Suspense → loading spinner

// With transition: keeps showing old content until new content is ready
startTransition(() => {
  setSomeState(newValue);  // Suspense → stay on current screen
});
```

```
Without startTransition:
  ┌────────────┐     ┌────────────┐     ┌─────────────┐
  │ Old Content │ ──► │  Loading...│ ──► │ New Content  │
  │             │     │  (fallback)│     │              │
  └────────────┘     └────────────┘     └─────────────┘
                        jarring!

With startTransition:
  ┌────────────┐                         ┌─────────────┐
  │ Old Content │ ──────────────────────► │ New Content  │
  │ (+ pending  │    (old content stays   │              │
  │  indicator) │     while new renders)  └─────────────┘
  └────────────┘
     smooth!
```

## Offscreen Component (Hidden Content)

Suspense uses an Offscreen fiber to manage visibility:

```
When RESOLVED (showing primary):
  SuspenseFiber
   └── OffscreenFiber (mode: "visible")
        └── AsyncContent    ← rendered & visible
   (fallback fiber is not rendered at all)

When SUSPENDED (showing fallback):
  SuspenseFiber
   └── OffscreenFiber (mode: "hidden")
   │    └── AsyncContent    ← still in tree but hidden
   └── Fragment
        └── Loading         ← rendered & visible

The primary content is HIDDEN, not REMOVED.
This preserves state in the suspended subtree.
```

## Nested Suspense Boundaries

```jsx
<Suspense fallback={<PageLoader />}>        // Boundary 1
  <Header />
  <Suspense fallback={<ContentLoader />}>   // Boundary 2
    <AsyncContent />
  </Suspense>
  <Suspense fallback={<SidebarLoader />}>   // Boundary 3
    <AsyncSidebar />
  </Suspense>
</Suspense>
```

```
Scenario: Both AsyncContent and AsyncSidebar suspend

Result:
  ┌───────────────────────────────────┐
  │ <Header /> ← renders fine         │
  │ ┌───────────────────┐             │
  │ │ <ContentLoader /> │ Boundary 2  │
  │ └───────────────────┘             │
  │ ┌───────────────────┐             │
  │ │ <SidebarLoader /> │ Boundary 3  │
  │ └───────────────────┘             │
  └───────────────────────────────────┘

Each Suspense boundary is INDEPENDENT.
Boundary 1 does NOT show its fallback because
inner boundaries caught their respective suspensions.

If we REMOVED Boundary 2 and 3:
  ┌───────────────────────────────────┐
  │ <PageLoader />       Boundary 1   │
  │ (everything replaced)             │
  └───────────────────────────────────┘
```

## SuspenseList (Experimental)

Coordinates reveal order of multiple Suspense boundaries:

```jsx
<SuspenseList revealOrder="forwards">
  <Suspense fallback={<Skeleton />}>
    <Item1 />  {/* Loads in 100ms */}
  </Suspense>
  <Suspense fallback={<Skeleton />}>
    <Item2 />  {/* Loads in 50ms — ready first! */}
  </Suspense>
  <Suspense fallback={<Skeleton />}>
    <Item3 />  {/* Loads in 200ms */}
  </Suspense>
</SuspenseList>

revealOrder="forwards":
  Even though Item2 loads first, React reveals them in order:
  t=100ms: Item1 appears
  t=100ms: Item2 appears (was ready, waited for Item1)
  t=200ms: Item3 appears
```
