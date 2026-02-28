# React 19: The use() Hook

## What Is use()?

`use()` is a new React hook that can read the value of a **Promise** or
**Context** during render. Unlike other hooks, `use()` can be called
conditionally.

```javascript
import { use } from 'react';

// Reading a Promise
function Comments({ commentsPromise }) {
  const comments = use(commentsPromise);  // Suspends until resolved
  return comments.map(c => <Comment key={c.id} data={c} />);
}

// Reading Context (alternative to useContext)
function Button() {
  const theme = use(ThemeContext);  // Same as useContext(ThemeContext)
  return <button className={theme}>Click</button>;
}
```

## How use() Differs from Other Hooks

```
Regular hooks:                         use():
  ❌ Cannot be called conditionally    ✅ CAN be called conditionally
  ❌ Cannot be called in loops         ✅ CAN be called in loops
  ❌ Must be at top level              ✅ Can be inside if/for/while

function Component({ showDetails }) {
  // ❌ ILLEGAL with useContext:
  // if (showDetails) {
  //   const ctx = useContext(DetailContext);
  // }

  // ✅ LEGAL with use:
  if (showDetails) {
    const ctx = use(DetailContext);
    return <Details data={ctx} />;
  }

  return <Summary />;
}
```

## use() with Promises: Internals

```javascript
// Simplified from ReactFiberHooks.js

function use(usable) {
  if (usable !== null && typeof usable === 'object') {
    if (typeof usable.then === 'function') {
      // It's a thenable/Promise
      return useThenable(usable);
    }
    if (usable.$$typeof === REACT_CONTEXT_TYPE) {
      // It's a Context
      return readContext(usable);
    }
  }
  throw new Error('use() argument must be a Promise or Context');
}

function useThenable(thenable) {
  const fiber = currentlyRenderingFiber;

  switch (thenable.status) {
    case 'fulfilled':
      // Promise already resolved → return value immediately
      return thenable.value;

    case 'rejected':
      // Promise rejected → throw the error
      throw thenable.reason;

    default:
      // Promise still pending
      if (typeof thenable.status === 'undefined') {
        // Track the promise (attach status/value/reason)
        const pendingThenable = thenable;
        pendingThenable.status = 'pending';

        pendingThenable.then(
          result => {
            if (pendingThenable.status === 'pending') {
              pendingThenable.status = 'fulfilled';
              pendingThenable.value = result;
            }
          },
          error => {
            if (pendingThenable.status === 'pending') {
              pendingThenable.status = 'rejected';
              pendingThenable.reason = error;
            }
          }
        );
      }

      // ★ SUSPEND: throw the thenable ★
      // Suspense boundary will catch this
      throw thenable;
  }
}
```

## use() with Context

When `use()` receives a context, it simply delegates to `readContext()` —
the same internal function that `useContext()` calls:

```javascript
// use(SomeContext) is equivalent to useContext(SomeContext)
// The difference: use() can be called conditionally

function Component({ loggedIn }) {
  if (loggedIn) {
    const user = use(UserContext);  // Only read context when logged in
    return <Profile user={user} />;
  }
  return <LoginScreen />;
}
```

## Caching Promises

Important: you should NOT create a new Promise on every render.
The Promise should be created outside the component or cached:

```javascript
// ❌ BAD: new Promise every render → infinite suspend loop
function Comments({ postId }) {
  const comments = use(fetch(`/api/comments/${postId}`).then(r => r.json()));
  // New promise each render → never matches previous → always suspends
}

// ✅ GOOD: cache the Promise
const commentsCache = new Map();
function fetchComments(postId) {
  if (!commentsCache.has(postId)) {
    commentsCache.set(postId, fetch(`/api/comments/${postId}`).then(r => r.json()));
  }
  return commentsCache.get(postId);
}

function Comments({ postId }) {
  const comments = use(fetchComments(postId));  // Same Promise across renders
  return comments.map(c => <Comment key={c.id} data={c} />);
}
```

## use() vs useEffect for Data Fetching

```
useEffect (old pattern):                use() (new pattern):

function Component({ id }) {            function Component({ dataPromise }) {
  const [data, setData] = useState();     const data = use(dataPromise);
  const [loading, setLoading] = useState  // Suspense handles loading state
    (true);                               // No manual loading/error state
  const [error, setError] = useState();   // Promise created by parent/router
                                          return <Display data={data} />;
  useEffect(() => {                     }
    fetch(`/api/${id}`)
      .then(r => r.json())              // Parent:
      .then(setData)                    <Suspense fallback={<Loading />}>
      .catch(setError)                    <ErrorBoundary>
      .finally(() =>                        <Component
        setLoading(false));                   dataPromise={fetchData(id)}
  }, [id]);                                 />
                                          </ErrorBoundary>
  if (loading) return <Loading />;      </Suspense>
  if (error) return <Error />;
  return <Display data={data} />;       // Much cleaner!
}
```
