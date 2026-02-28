# Edge Cases: Suspense & SSR/Hydration

## 1. Nested Suspense Boundaries

```jsx
<Suspense fallback={<PageSkeleton />}>          {/* Outer */}
  <Header />
  <Suspense fallback={<ContentSpinner />}>       {/* Inner */}
    <AsyncContent />
  </Suspense>
  <Suspense fallback={<SidebarSkeleton />}>      {/* Inner */}
    <AsyncSidebar />
  </Suspense>
</Suspense>
```

```
Scenario: Both AsyncContent and AsyncSidebar suspend

  t=0ms: Render starts
    → Header renders fine
    → AsyncContent throws Promise → Inner boundary catches it
    → AsyncSidebar throws Promise → Inner boundary catches it
    → Outer boundary is NOT triggered (inner boundaries handled it)

  Screen shows:
    ┌──────────────────────┐
    │ Header               │
    │ [ContentSpinner]     │ ← inner fallback
    │ [SidebarSkeleton]    │ ← inner fallback
    └──────────────────────┘

  t=500ms: AsyncContent's promise resolves
    → Inner boundary retries → AsyncContent renders
    → AsyncSidebar still suspended

  Screen shows:
    ┌──────────────────────┐
    │ Header               │
    │ Actual Content       │ ← resolved
    │ [SidebarSkeleton]    │ ← still loading
    └──────────────────────┘

  If there were NO inner boundaries:
    → Both suspensions bubble to outer boundary
    → Screen shows: <PageSkeleton /> (everything hidden!)
    → Neither Header, Content, nor Sidebar visible

  Key: Suspense boundaries are like try/catch for promises.
  The NEAREST boundary catches the thrown promise.
```

## 2. Suspense with Transitions: Avoid Fallback Flash

```jsx
function App() {
  const [tab, setTab] = useState('home');
  const [isPending, startTransition] = useTransition();

  const switchTab = (newTab) => {
    startTransition(() => {
      setTab(newTab);
    });
  };

  return (
    <div style={{ opacity: isPending ? 0.7 : 1 }}>
      <Suspense fallback={<Spinner />}>
        <TabContent tab={tab} />
      </Suspense>
    </div>
  );
}
```

```
WITHOUT transition (urgent update):
  Click tab → setTab('settings')
    → Re-render: TabContent suspends (new data)
    → Suspense shows <Spinner />
    → Old tab content IMMEDIATELY replaced with spinner
    → User experience: jarring flash

WITH transition:
  Click tab → startTransition(() => setTab('settings'))
    → React keeps showing OLD tab content (tab='home')
    → Renders new content in background
    → TabContent('settings') suspends
    → React does NOT show the fallback!
    → Instead, keeps previous UI with isPending=true
    → opacity: 0.7 shows user something is loading

  Why no fallback with transitions?
    → Transitions use a special Suspense behavior:
    → If the UPDATE (not mount) causes suspension,
      React avoids showing a NEW fallback
    → It "recedes" to the previous committed state
    → Only shows fallback if the INITIAL mount suspends
      or if the transition takes too long

  Internal mechanism:
    When a Suspense boundary catches a thrown promise during
    a transition render:
      → showFallback flag NOT set
      → React keeps the current fiber tree
      → Marks the boundary as "suspended but not showing fallback"
      → Retries when promise resolves
```

## 3. Suspense Waterfall Problem

```jsx
// ❌ Waterfall: sequential loading
function Profile({ id }) {
  return (
    <Suspense fallback={<Spinner />}>
      <UserDetails id={id} />          {/* Fetches user → suspends */}
      {/* UserPosts doesn't even START until UserDetails resolves! */}
    </Suspense>
  );
}

function UserDetails({ id }) {
  const user = use(fetchUser(id));       // Suspends
  return (
    <>
      <h1>{user.name}</h1>
      <Suspense fallback={<PostsSpinner />}>
        <UserPosts userId={user.id} />   {/* Can't start until user loads */}
      </Suspense>
    </>
  );
}
```

```
Waterfall timeline:
  t=0ms:    Start rendering
  t=0ms:    UserDetails suspends → fetch user
  t=500ms:  User data arrives → UserDetails renders
  t=500ms:  UserPosts starts → fetch posts → suspends
  t=1000ms: Posts data arrives → UserPosts renders
  Total: 1000ms (sequential!)

Why this happens:
  React renders top-down. When UserDetails suspends,
  React STOPS rendering its children. UserPosts never
  gets a chance to start its fetch.

Fix 1: Parallel fetching with separate boundaries
  function Profile({ id }) {
    return (
      <>
        <Suspense fallback={<UserSkeleton />}>
          <UserDetails id={id} />
        </Suspense>
        <Suspense fallback={<PostsSkeleton />}>
          <UserPosts userId={id} />       {/* Starts immediately! */}
        </Suspense>
      </>
    );
  }

  Timeline: both fetch at t=0, both resolve by t=500ms.
  Total: 500ms (parallel!)

Fix 2: Preload data before render
  const userPromise = fetchUser(id);    // Start fetch
  const postsPromise = fetchPosts(id);  // Start fetch in parallel

  function Profile() {
    const user = use(userPromise);      // May already be resolved
    const posts = use(postsPromise);    // May already be resolved
    return <div>...</div>;
  }
```

## 4. Hydration Mismatch Recovery

```jsx
// Server renders:
function App() {
  return <div>{Date.now()}</div>;  // Different on server vs client!
}

// Server HTML: <div>1700000000000</div>
// Client render: <div>1700000001000</div> ← mismatch!
```

```
Hydration mismatch handling:

  1. React walks server HTML and client VDOM simultaneously
  2. Compares each node:
     - Tag type must match (div === div ✓)
     - Text content compared
     - Attributes compared

  3. On MISMATCH:
     Development:
       → Console error: "Text content did not match"
       → "Server: 1700000000000 Client: 1700000001000"

     Recovery strategy:
       → React DELETES the server-rendered DOM subtree
       → Re-creates it from scratch using client render
       → This is called "client-side recovery"
       → Performance cost: full re-render of mismatched subtree

  4. Suppressable with suppressHydrationWarning:
     <div suppressHydrationWarning>
       {Date.now()}
     </div>
     → No warning, but React still patches the DOM

  Common mismatch causes:
    - Date/time values
    - Math.random()
    - Browser-only APIs (window.innerWidth)
    - Different data on server vs client
    - Invalid HTML nesting (browser auto-corrects)

  React 18+ improvement:
    Mismatches in Suspense boundaries are handled more gracefully:
    → React falls back to client rendering for that boundary
    → Rest of the page stays hydrated
    → Partial recovery instead of full page re-render
```

## 5. Selective Hydration Priority

```html
<!-- Server-rendered HTML with multiple Suspense boundaries -->
<div id="root">
  <nav>...</nav>           <!-- Hydrates first (no boundary) -->
  <!--$-->                  <!-- Suspense boundary A -->
  <div class="sidebar">
    ...large sidebar...
  </div>
  <!--/$-->
  <!--$-->                  <!-- Suspense boundary B -->
  <div class="content">
    ...large content...
  </div>
  <!--/$-->
</div>
```

```
Selective hydration (React 18):

  Scenario: User clicks on content area while sidebar is hydrating

  t=0ms:  Page loads with server HTML (not interactive yet)
  t=1ms:  React starts hydrating
          → nav hydrates immediately (no boundary)
          → Sidebar boundary starts hydrating

  t=5ms:  USER CLICKS on content area!
          → React detects click on un-hydrated content
          → Records the event (replay queue)
          → BOOSTS content boundary's hydration priority

  t=6ms:  React PAUSES sidebar hydration
          → Switches to hydrating content (higher priority)

  t=10ms: Content boundary finishes hydrating
          → Content is now interactive
          → REPLAYS the captured click event
          → User's click actually fires!

  t=11ms: React resumes sidebar hydration

  Internal mechanism:
    queueExplicitHydrationTarget(target):
      → Find the Suspense boundary containing the click target
      → Upgrade its lane to InputContinuousLane (high priority)
      → attemptExplicitHydrationTarget: try hydrating immediately

    dispatchEvent captures events on un-hydrated boundaries:
      → attemptToDispatchEvent fails (no fiber yet)
      → Event stored in replay queue
      → After hydration completes, events are replayed
```

## 6. Streaming SSR: Out-of-Order Completion

```jsx
// Server component tree:
<Suspense fallback={<A_skeleton />}>
  <SlowComponentA />                    {/* Takes 2 seconds */}
</Suspense>
<Suspense fallback={<B_skeleton />}>
  <FastComponentB />                    {/* Takes 200ms */}
</Suspense>
```

```
Fizz streaming behavior:

  t=0ms: Server starts rendering
    → Encounters SlowComponentA → suspends
    → Sends fallback HTML: <template id="B:0">A_skeleton</template>
    → Encounters FastComponentB → suspends

  t=1ms: Server sends initial HTML shell:
    <div id="root">
      <!--$?--><template id="B:0"></template>A_skeleton<!--/$-->
      <!--$?--><template id="B:1"></template>B_skeleton<!--/$-->
    </div>

  t=200ms: FastComponentB resolves FIRST (out of order!)
    → Server sends a REPLACEMENT chunk:
    <div hidden id="S:1">
      <div class="content">Fast B content</div>
    </div>
    <script>
      // This script swaps the skeleton with real content
      $RC("B:1", "S:1");
    </script>

  t=2000ms: SlowComponentA resolves
    → Server sends another replacement chunk:
    <div hidden id="S:0">
      <div>Slow A content</div>
    </div>
    <script>$RC("B:0", "S:0");</script>

  The $RC function:
    → Finds the template marker (B:0 or B:1)
    → Replaces the fallback HTML with the real content
    → This happens in the browser via inline <script>
    → No JavaScript framework needed for this swap!

  Streaming enables:
    - Fastest content appears first
    - No waiting for slowest component
    - Progressive enhancement
    - Works even before React JS bundle loads
```

## 7. Suspense + Error Boundary Interaction

```jsx
<ErrorBoundary fallback={<ErrorUI />}>
  <Suspense fallback={<Loading />}>
    <AsyncComponent />
  </Suspense>
</ErrorBoundary>
```

```
Three possible outcomes for AsyncComponent:

  Case 1: Throws a Promise (normal suspension)
    → Suspense catches it
    → Shows <Loading />
    → Promise resolves → retry → shows content
    → Error boundary NOT involved

  Case 2: Throws an Error
    → Suspense does NOT catch errors
    → Error bubbles up to ErrorBoundary
    → Shows <ErrorUI />
    → Suspense NOT involved

  Case 3: Throws a Promise, which REJECTS
    → Suspense catches the promise
    → Shows <Loading />
    → Promise rejects
    → React retries rendering
    → AsyncComponent throws the rejection error
    → Suspense sees it's an Error, not a Promise
    → Error bubbles to ErrorBoundary
    → Shows <ErrorUI />

  Case 4: Throws a Promise, resolves, then render throws Error
    → Suspense catches promise → <Loading />
    → Promise resolves → retry render
    → Render throws Error (bad data, etc.)
    → Error bubbles to ErrorBoundary
    → Shows <ErrorUI />

  Internal distinction:
    throwException(root, value, fiber, lanes):
      if (typeof value === 'object' && typeof value.then === 'function') {
        // It's a thenable → SUSPENSE path
        const wakeable = value;
        // Find nearest SuspenseComponent boundary
      } else {
        // It's an error → ERROR BOUNDARY path
        // Find nearest class component with getDerivedStateFromError
      }
```

## 8. Server Components and Client Components Boundary

```jsx
// server-component.jsx (runs on server only)
async function ServerList() {
  const data = await db.query('SELECT * FROM items');  // Direct DB access!
  return (
    <ul>
      {data.map(item => (
        <ClientItem key={item.id} item={item} />  // Client component
      ))}
    </ul>
  );
}

// client-item.jsx
'use client';
function ClientItem({ item }) {
  const [expanded, setExpanded] = useState(false);  // Client state OK!
  return <li onClick={() => setExpanded(!expanded)}>{item.name}</li>;
}
```

```
Boundary rules:

  Server → Client: ✓ Always allowed
    Server components can render client components
    Props must be serializable (no functions, no classes)

  Client → Server: ✗ Direct import not allowed
    Client components cannot import server components
    BUT: client components can RENDER server components
    passed as children or props:

    // ✓ This works:
    function ClientWrapper({ children }) {  // 'use client'
      return <div className="wrapper">{children}</div>;
    }

    // Server component passes server component as children:
    <ClientWrapper>
      <ServerComponent />    {/* Rendered on server, passed as React tree */}
    </ClientWrapper>

  Serialization boundary:
    Server components are rendered to a special wire format (Flight):
    {
      type: "$Lclient-item",  // Reference to client component
      props: {
        item: { id: 1, name: "Widget" }  // Serialized props
      }
    }

    What CANNOT cross the boundary:
      - Functions (except Server Actions with 'use server')
      - Class instances
      - Symbols
      - DOM nodes
      - Streams (except as special React types)
```

## 9. Hydration with useId: Server/Client Consistency

```jsx
function Accordion() {
  const id = useId();  // Must match on server and client!

  return (
    <div>
      <button aria-controls={`${id}-panel`}>Toggle</button>
      <div id={`${id}-panel`}>Content</div>
    </div>
  );
}
```

```
Server rendering:
  renderToString(<Accordion />)
  → useId generates ":R1:" (based on tree position)
  → HTML: <button aria-controls=":R1:-panel">Toggle</button>
          <div id=":R1:-panel">Content</div>

Client hydration:
  hydrateRoot(container, <Accordion />)
  → useId generates ":R1:" (SAME tree position = same ID)
  → Matches server HTML ✓
  → No hydration mismatch

If IDs were counter-based (like a global incrementing counter):
  Server: renders components in streaming order → id=1
  Client: renders components in tree traversal order → id=3 (different!)
  → MISMATCH!

Tree-position-based IDs solve this because:
  - Same component in same position → same ID
  - Regardless of rendering order or streaming
  - Works with selective hydration (out-of-order)
  - Works with Suspense boundaries

  identifierPrefix (createRoot option):
    For multiple React roots on the same page:
    createRoot(container, { identifierPrefix: 'app1-' });
    → IDs become "app1-:R1:" instead of ":R1:"
    → Prevents collision between roots
```

## 10. SSR Error Recovery

```jsx
// Server-side rendering with error:
<Suspense fallback={<Placeholder />}>
  <ComponentThatMayFail />
</Suspense>
```

```
Server-side error handling:

  Case 1: Component throws during SSR
    → Fizz catches the error
    → Sends the FALLBACK HTML for that Suspense boundary
    → Client will try to render the component from scratch
    → If client render also fails → error boundary handles it

  Case 2: Streaming error after partial send
    → Server has already sent some HTML
    → Error occurs in a Suspense boundary
    → Server sends an error replacement chunk:
      <script>$RX("B:0", errorDigest)</script>
    → Client receives error and falls back to client rendering

  Case 3: Fatal server error (outside any boundary)
    → renderToPipeableStream's onError callback fires
    → Stream may be aborted
    → Client falls back to full client-side rendering
    → Performance: user sees empty page, then client renders everything

  Error recovery chain:
    Server render fails
      → Try Suspense fallback on server
        → Send fallback HTML
          → Client hydrates
            → Client tries to render the real component
              → Success → user sees content
              → Failure → error boundary catches it

  The key insight: SSR errors don't crash the page.
  They gracefully degrade through multiple fallback layers.
```
