# Server-Side Rendering and Hydration

## SSR Overview

Server-Side Rendering generates HTML on the server so the user sees content
immediately, without waiting for JavaScript to download and execute.

```
WITHOUT SSR:                           WITH SSR:
┌──────────────────────┐               ┌──────────────────────┐
│ Browser loads HTML    │ (blank page)  │ Browser loads HTML    │ (content!)
│ <div id="root"></div>│               │ <div id="root">      │
│                      │               │   <h1>Hello</h1>     │
│ Downloads JS bundle  │ (still blank) │   <p>Content...</p>  │
│                      │               │ </div>               │
│ React renders        │ (content!)    │                      │
│                      │               │ Downloads JS bundle  │ (interactive)
│ Interactive!         │               │ React hydrates       │
└──────────────────────┘               │ Interactive!         │
                                       └──────────────────────┘
Time to content: slow                  Time to content: fast
```

## The SSR APIs

```javascript
// React 18 Streaming API (recommended)
import { renderToPipeableStream } from 'react-dom/server';

app.get('/', (req, res) => {
  const { pipe, abort } = renderToPipeableStream(<App />, {
    bootstrapScripts: ['/bundle.js'],
    onShellReady() {
      res.statusCode = 200;
      res.setHeader('Content-type', 'text/html');
      pipe(res);  // Start streaming HTML
    },
    onError(error) {
      console.error(error);
      res.statusCode = 500;
    },
  });

  setTimeout(() => abort(), 10000); // 10 second timeout
});
```

## Fizz: The Streaming SSR Renderer

Fizz (`react-dom/server`) renders React trees to HTML streams:

```
                    ┌─────────────────────┐
                    │       FIZZ          │
                    │                     │
  <App /> ────────► │  Walk component tree│
                    │  Generate HTML      │
                    │  Stream chunks      │
                    │                     │
                    │  Suspense?          │
                    │  → Send placeholder │
                    │  → Continue others  │
                    │  → When ready, send │
                    │    replacement HTML │
                    │    + <script> to    │
                    │    swap in-place    │
                    └─────────────────────┘
```

### Streaming with Suspense

```jsx
<Layout>
  <Header />                          {/* renders immediately */}
  <Suspense fallback={<Skeleton />}>
    <SlowContent />                    {/* data not ready yet */}
  </Suspense>
  <Footer />                          {/* renders immediately */}
</Layout>
```

Stream output:

```html
<!-- Chunk 1: sent immediately (shell) -->
<html>
<body>
  <div id="root">
    <header>Header content</header>
    <div id="B:0">
      <!-- Suspense fallback placeholder -->
      <div>Loading skeleton...</div>
    </div>
    <footer>Footer content</footer>
  </div>
  <script src="/bundle.js" async></script>

<!-- Chunk 2: sent when SlowContent data is ready -->
<div hidden id="S:0">
  <article>
    <h1>Actual Content</h1>
    <p>This was loaded from the database...</p>
  </article>
</div>
<script>
  // Swap the placeholder with actual content
  $RC("B:0", "S:0");
</script>
```

```
Timeline:

Server:
t=0ms:    Start rendering <App />
t=5ms:    Header, Footer ready → stream shell HTML
t=5ms:    SlowContent suspends → send fallback HTML
          ──── network ────
Browser:
t=100ms:  Receives shell → displays Header + Skeleton + Footer
          (user sees content immediately!)

Server:
t=500ms:  SlowContent data ready → render to HTML
t=501ms:  Stream replacement HTML + swap script
          ──── network ────
Browser:
t=600ms:  Receives replacement → $RC swaps content inline
          (skeleton replaced with real content, no full re-render!)

t=800ms:  JS bundle downloads → hydration begins
t=900ms:  App is fully interactive
```

## Hydration: Making Server HTML Interactive

Hydration attaches React's event system and component state to
server-rendered HTML without recreating DOM nodes.

```javascript
// Client-side hydration
import { hydrateRoot } from 'react-dom/client';
hydrateRoot(document.getElementById('root'), <App />);
```

### How Hydration Works

```
Server HTML:                    React Fiber Tree:
<div id="root">                 HostRoot
  <h1>Hello</h1>                 └─ App (FunctionComponent)
  <button>Click</button>              ├─ h1 (HostComponent)
</div>                                │   └─ "Hello" (HostText)
                                      └─ button (HostComponent)
                                          └─ "Click" (HostText)

During hydration, React walks BOTH trees simultaneously:

Step 1: App fiber → no DOM node (FunctionComponent)
Step 2: h1 fiber → match with <h1> DOM node
        → fiber.stateNode = existing <h1>  (ADOPT, don't create)
Step 3: "Hello" text → match with text node
        → fiber.stateNode = existing text node
Step 4: button fiber → match with <button>
        → fiber.stateNode = existing <button>
        → attach onClick listener
Step 5: "Click" text → match with text node
```

### Hydration Mismatch

```javascript
// Server renders:  <div>Server Time: 12:00:00</div>
// Client renders:  <div>Server Time: 12:00:01</div>
//                                          ^^^ MISMATCH!

// React 18 behavior:
// 1. Log a warning in development
// 2. Attempt to patch the mismatch (client wins)
// 3. In severe cases, throw away server HTML and re-render client-side
```

## Selective Hydration

React 18 can hydrate Suspense boundaries independently and prioritize
based on user interaction:

```jsx
<App>
  <Header />                              {/* hydrates first (shell) */}
  <Suspense fallback={<Skeleton />}>
    <HeavyComponent />                     {/* hydrates later */}
  </Suspense>
  <Suspense fallback={<Skeleton />}>
    <AnotherHeavyComponent />              {/* hydrates later */}
  </Suspense>
</App>
```

```
t=0ms:    HTML arrives, user sees full page
t=100ms:  JS loads, hydration starts
t=101ms:  Header hydrates → interactive
t=102ms:  HeavyComponent starts hydrating...

t=105ms:  ★ User clicks on AnotherHeavyComponent!
          React PRIORITIZES hydrating AnotherHeavyComponent
          → Pauses HeavyComponent hydration
          → Hydrates AnotherHeavyComponent first
          → User's click is replayed after hydration
          → Then continues HeavyComponent hydration
```

## React Server Components (RSC) — Flight

RSC is a separate paradigm where components run ONLY on the server:

```
            Server                          Client
┌─────────────────────────┐    ┌─────────────────────────┐
│  Server Components      │    │  Client Components      │
│  (async, no state,      │    │  (useState, useEffect,  │
│   direct DB access)     │    │   event handlers)       │
│                         │    │                         │
│  async function Page() {│    │  'use client';          │
│    const data = await   │    │  function Counter() {   │
│      db.query(...);     │    │    const [c, setC] =    │
│    return <div>         │    │      useState(0);       │
│      <Counter data={data}│   │    return <button       │
│    />;                  │    │      onClick={...}>     │
│  }                      │    │      {c}               │
│                         │    │    </button>;           │
│  Serialized to "Flight" │───►│  Deserialized and      │
│  wire format            │    │  rendered on client     │
└─────────────────────────┘    └─────────────────────────┘
```

Flight wire format (simplified):

```
// Server Component output (NOT HTML — a serialized React tree)
0:["$","div",null,{"children":[
  ["$","$L1",null,{"data":{"items":["a","b","c"]}}]
]}]
1:I["./Counter.js","Counter"]

// $L1 means "lazy client component reference"
// The client knows to import Counter.js and render it
// with the provided props (data was fetched on server)
```
