# React 19: Resource Loading and Document Metadata

## The Problem

Before React 19, managing `<title>`, `<meta>`, `<link>`, and `<style>` tags
required third-party libraries (react-helmet, next/head) or manual
`document.head` manipulation in effects. Stylesheet loading was
uncoordinated — components had no way to declare style dependencies that
block rendering until loaded.

```jsx
// Before React 19 — manual, error-prone:
function ProductPage({ product }) {
  useEffect(() => {
    document.title = product.name;
    const link = document.createElement('link');
    link.rel = 'stylesheet';
    link.href = '/styles/product.css';
    document.head.appendChild(link);
    return () => document.head.removeChild(link);
  }, [product.name]);

  return <div>...</div>;
}

// After React 19 — declarative:
function ProductPage({ product }) {
  return (
    <>
      <title>{product.name}</title>
      <link rel="stylesheet" href="/styles/product.css" precedence="default" />
      <div>...</div>
    </>
  );
}
```

## Document Metadata Hoisting

React 19 automatically hoists `<title>`, `<meta>`, and `<link>` elements
from anywhere in the component tree into `<head>`:

```jsx
function BlogPost({ post }) {
  return (
    <article>
      {/* These render into <head>, not here in the DOM */}
      <title>{post.title} — My Blog</title>
      <meta name="description" content={post.excerpt} />
      <meta property="og:title" content={post.title} />
      <link rel="canonical" href={`/posts/${post.slug}`} />

      {/* This renders in place */}
      <h1>{post.title}</h1>
      <p>{post.body}</p>
    </article>
  );
}
```

### How Hoisting Works Internally

```
Component tree:                    DOM output:

<App>                              <html>
  <Layout>                           <head>
    <BlogPost>                         <title>My Post — My Blog</title>
      <title>My Post — My Blog</      <meta name="description" ...>
      <meta name="description"..>      <link rel="canonical" ...>
      <h1>My Post</h1>                <link rel="stylesheet" href=...>
      <p>Content...</p>              </head>
    </BlogPost>                      <body>
  </Layout>                            <div id="root">
</App>                                   <article>
                                           <h1>My Post</h1>
                                           <p>Content...</p>
                                         </article>
                                       </div>
                                     </body>
                                   </html>

React detects these special elements during the commit phase:
- <title> → hoisted to document.head, deduped (last wins)
- <meta> → hoisted to document.head
- <link rel="canonical"> → hoisted to document.head
- <link rel="stylesheet"> → special handling (see below)
```

### Deduplication Rules

```
<title>:
  Multiple <title> tags → LAST ONE WINS
  React deduplicates by replacing the previous title

  <Layout>
    <title>Default Title</title>     ← rendered first
    <Page>
      <title>Page Title</title>      ← rendered second, WINS
    </Page>
  </Layout>
  Result: <title>Page Title</title>

<meta>:
  Deduped by 'name' or 'property' attribute
  <meta name="description" content="old" />  ← replaced
  <meta name="description" content="new" />  ← wins

<link rel="canonical">:
  Only one canonical link — last wins
```

## Stylesheet Loading with precedence

React 19 introduces coordinated stylesheet loading. Stylesheets with
`precedence` are:
1. Deduplicated (same href loaded only once)
2. Ordered by precedence in the document
3. Can block rendering until loaded (via Suspense)

```jsx
function Component() {
  return (
    <>
      {/* Stylesheet with precedence — React manages it */}
      <link rel="stylesheet" href="/base.css" precedence="default" />
      <link rel="stylesheet" href="/theme.css" precedence="high" />
      <link rel="stylesheet" href="/component.css" precedence="default" />

      <div className="styled-component">Content</div>
    </>
  );
}
```

### How precedence Works

```
precedence determines insertion ORDER in <head>, not CSS specificity.
Stylesheets with the same precedence are grouped together.

User writes:                       React inserts into <head>:

Component A:                       <head>
  <link href="/a.css"                ← precedence groups ordered:
    precedence="low" />
                                     <!-- "high" group -->
Component B:                         <link href="/theme.css" />
  <link href="/theme.css"
    precedence="high" />             <!-- "default" group -->
                                     <link href="/base.css" />
Component C:                         <link href="/component.css" />
  <link href="/base.css"
    precedence="default" />          <!-- "low" group -->
  <link href="/component.css"        <link href="/a.css" />
    precedence="default" />        </head>

Within a group: insertion order.
Between groups: precedence name ordering (alphabetical as default,
but the first encountered precedence establishes the order).
```

### Stylesheet and Suspense Integration

```jsx
function App() {
  return (
    <Suspense fallback={<Loading />}>
      <PageWithStyles />
    </Suspense>
  );
}

function PageWithStyles() {
  return (
    <>
      {/* This stylesheet must load before the content reveals */}
      <link rel="stylesheet" href="/page.css" precedence="default" />
      <div className="page">
        This content won't flash unstyled!
      </div>
    </>
  );
}
```

```
Timeline:

1. React renders <PageWithStyles />
2. Encounters <link rel="stylesheet" precedence="default">
3. Inserts <link> into <head>
4. If stylesheet not yet loaded:
   → Suspense boundary shows fallback (<Loading />)
   → Browser downloads /page.css
   → Stylesheet loads
   → Suspense reveals content
5. Content appears WITH styles applied (no FOUC!)

This is different from a regular <link> without precedence:
  Without precedence: inserted in-place, no loading coordination
  With precedence: hoisted to <head>, loading blocks Suspense reveal
```

## Preloading APIs

React 19 provides imperative APIs for resource preloading:

```javascript
import { preload, preinit, prefetchDNS, preconnect } from 'react-dom';

function App() {
  // Preload a resource (hint — browser decides when to fetch)
  preload('/font.woff2', { as: 'font', type: 'font/woff2' });

  // Preinit a resource (fetch AND execute/apply immediately)
  preinit('/critical.css', { as: 'style' });
  preinit('/analytics.js', { as: 'script' });

  // DNS prefetch (resolve domain early)
  prefetchDNS('https://api.example.com');

  // Preconnect (DNS + TCP + TLS handshake early)
  preconnect('https://cdn.example.com');

  return <div>...</div>;
}
```

### What Each API Produces

```
preload('/font.woff2', { as: 'font' })
→ <link rel="preload" href="/font.woff2" as="font" crossorigin />
  (hint: browser should fetch this soon)

preinit('/critical.css', { as: 'style' })
→ <link rel="stylesheet" href="/critical.css" />
  (fetch AND apply immediately — stronger than preload)

preinit('/app.js', { as: 'script' })
→ <script src="/app.js" async />
  (fetch AND execute immediately)

prefetchDNS('https://api.example.com')
→ <link rel="dns-prefetch" href="https://api.example.com" />
  (resolve DNS early)

preconnect('https://cdn.example.com')
→ <link rel="preconnect" href="https://cdn.example.com" />
  (DNS + TCP + TLS early)
```

### When to Call Preload APIs

```javascript
// During render — most common
function ImageGallery({ images }) {
  // Preload the next image while viewing current
  preload(images[currentIndex + 1].src, { as: 'image' });
  return <img src={images[currentIndex].src} />;
}

// During event handlers
function SearchResults() {
  function handleHover(result) {
    // User hovering over a result — preload the detail page resources
    preload(`/api/details/${result.id}`, { as: 'fetch' });
    preload(`/styles/detail-page.css`, { as: 'style' });
  }
  // ...
}

// During module initialization (top-level)
preinit('/global-styles.css', { as: 'style' });
// Executed when the module loads — before any component renders

// Inside loaders (e.g., React Router)
async function loader({ params }) {
  preload(`/api/user/${params.id}`, { as: 'fetch' });
  const data = await fetchUser(params.id);
  return data;
}
```

## SSR Integration

During server-side rendering, these APIs inject tags into the streamed HTML:

```
Server renders:
  1. Component calls preload('/font.woff2', { as: 'font' })
  2. Fizz (streaming SSR) collects this hint
  3. Before streaming the shell HTML, includes:

<html>
  <head>
    <link rel="preload" href="/font.woff2" as="font" crossorigin />
    <link rel="stylesheet" href="/critical.css" />
    <!-- Preloaded resources appear BEFORE the body -->
    <!-- Browser starts fetching immediately while parsing -->
  </head>
  <body>
    <div id="root">
      <!-- Component HTML streams here -->
    </div>
  </body>
</html>

This means:
  - Fonts start downloading before the page content is even parsed
  - Critical CSS is available before the first paint
  - DNS/connections are established early
  → Faster Time to First Paint (TTFP) and Largest Contentful Paint (LCP)
```

## Async Scripts (React 19)

React 19 also deduplicates and hoists `<script async>`:

```jsx
function Component() {
  return (
    <>
      <script async src="/analytics.js" />
      <script async src="/widget.js" />
      <div>Content</div>
    </>
  );
}

// Multiple components can declare the same script:
function A() { return <script async src="/shared.js" />; }
function B() { return <script async src="/shared.js" />; }

// React deduplicates: /shared.js is only inserted ONCE in <head>
```
