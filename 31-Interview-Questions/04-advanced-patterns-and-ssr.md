# Interview Questions: Advanced Patterns, SSR & React 19

## Q1. How does server-side rendering (SSR) work in React 18+?

**Answer:**

React 18 uses **Fizz**, a streaming server renderer that works with Suspense:

```
Traditional SSR (React 17):
  renderToString() → blocks until ENTIRE tree is rendered → sends complete HTML

Streaming SSR (React 18):
  renderToPipeableStream() → sends HTML incrementally as components resolve
```

**Streaming flow:**
```
1. Server starts rendering
2. Hits <Suspense> → sends fallback HTML immediately
3. Continues rendering non-suspended parts
4. When suspended data resolves → sends replacement HTML chunk
5. Inline <script> swaps fallback with real content in browser

Client receives:
  <div><!--$?--><template id="B:0"></template>Loading...<!--/$-->
  <!-- ...later... -->
  <div hidden id="S:0">Real content</div>
  <script>$RC("B:0","S:0")</script>  ← swaps in real content
```

**Benefits:**
- Time to First Byte is faster (don't wait for slow data)
- Progressive loading (fast content appears first)
- Works before JS bundle loads (HTML swapping is pure DOM)
- Selective hydration can prioritize interactive areas

---

## Q2. What is hydration and what causes hydration mismatches?

**Answer:**

**Hydration** is the process of attaching React's event listeners and
internal state to server-rendered HTML, making it interactive.

```
Server: Renders HTML string (static)
Client: hydrateRoot(container, <App />)
  → React walks the existing DOM nodes
  → Matches them with the client-rendered fiber tree
  → Attaches event listeners
  → Sets up state and effects
  → Does NOT recreate DOM (reuses server HTML)
```

**Hydration mismatch** occurs when server and client produce different output:

```jsx
// Common causes:
function App() {
  return <div>{Date.now()}</div>;           // Different timestamp
  return <div>{window.innerWidth}</div>;    // No window on server
  return <div>{Math.random()}</div>;        // Different random values
  return <div>{typeof window !== 'undefined' ? 'client' : 'server'}</div>;
}
```

**Recovery:**
- React logs a warning in development
- Deletes the mismatched server DOM subtree
- Re-creates it from scratch using client render
- Performance cost: full re-render of that subtree

**Fix:** Use `useEffect` for client-only values (effects don't run on server):
```jsx
const [width, setWidth] = useState(0);  // Same on server and client
useEffect(() => setWidth(window.innerWidth), []);  // Client only
```

---

## Q3. What are React Server Components (RSC)?

**Answer:**

RSC are components that **run only on the server** and are never shipped
to the client bundle:

```jsx
// server-component.jsx (no 'use client' directive)
async function ProductPage({ id }) {
  const product = await db.query('SELECT * FROM products WHERE id = ?', [id]);
  const reviews = await db.query('SELECT * FROM reviews WHERE product_id = ?', [id]);

  return (
    <div>
      <h1>{product.name}</h1>
      <AddToCart id={id} />       {/* Client component */}
      <Reviews data={reviews} />  {/* Server component */}
    </div>
  );
}
```

**Key characteristics:**
- Can use `async/await` directly (no useEffect/useState needed for data)
- Can access server resources (database, file system, environment variables)
- Output is serialized as **Flight format** (not HTML, not JSON)
- Zero client-side JavaScript for server components
- Cannot use hooks, event handlers, or browser APIs

**Flight wire format (simplified):**
```
0:["$","div",null,{"children":[
  ["$","h1",null,{"children":"Widget Pro"}],
  ["$","$Ladd-to-cart",null,{"id":42}],  ← reference to client component
  ["$","div",null,{"children":"★★★★☆"}]
]}]
```

**Rules:**
- Server → Client: ✓ (server components can render client components)
- Client → Server: ✗ (direct import not allowed)
- Client can receive server components as `children` props

---

## Q4. Explain the `use()` hook in React 19.

**Answer:**

`use()` is unique among hooks — it can be called **conditionally** and works
with both Promises and Contexts:

```jsx
// With Promises (replaces useEffect for data fetching):
function UserProfile({ userPromise }) {
  const user = use(userPromise);  // Suspends until resolved!
  return <h1>{user.name}</h1>;
}

// With Context (replaces useContext):
function ThemeButton() {
  const theme = use(ThemeContext);  // Same as useContext(ThemeContext)
  return <button className={theme}>Click</button>;
}

// Conditionally (UNLIKE other hooks):
function Comments({ shouldLoad, commentsPromise }) {
  if (shouldLoad) {
    const comments = use(commentsPromise);  // ✓ Conditional use()
    return <CommentList comments={comments} />;
  }
  return null;
}
```

**Internal mechanism for Promises:**
```
use(promise):
  → Is promise already resolved?
    YES → return resolved value
    NO  → throw promise (triggers Suspense!)
         → Suspense boundary catches it
         → Shows fallback
         → promise.then() → React retries
         → use(promise) → now resolved → returns value
```

**Differences from useEffect + useState pattern:**
```jsx
// Old pattern (manual):
function Profile({ id }) {
  const [user, setUser] = useState(null);
  useEffect(() => {
    fetchUser(id).then(setUser);
  }, [id]);
  if (!user) return <Spinner />;
  return <h1>{user.name}</h1>;
}

// New pattern (use + Suspense):
function Profile({ userPromise }) {
  const user = use(userPromise);  // Suspends, no loading state needed
  return <h1>{user.name}</h1>;
}
// Parent wraps in <Suspense fallback={<Spinner />}>
```

---

## Q5. What are Server Actions and how do they work?

**Answer:**

Server Actions are functions marked with `'use server'` that can be called
from the client but execute on the server:

```jsx
// actions.js
'use server';

async function addToCart(productId) {
  const cart = await db.query('INSERT INTO cart ...');
  revalidatePath('/cart');
  return { success: true };
}

// client-component.jsx
'use client';
function AddToCartButton({ productId }) {
  const [state, formAction] = useActionState(
    addToCart.bind(null, productId),
    { success: false }
  );

  return (
    <form action={formAction}>
      <button type="submit">Add to Cart</button>
      {state.success && <p>Added!</p>}
    </form>
  );
}
```

**Internal mechanism:**
1. Build tool replaces server function with a **reference ID**
2. Client calls the reference → HTTP request to server
3. Server executes the actual function
4. Response is serialized back as Flight format
5. React applies the response (state update, revalidation)

**Works with forms natively:**
```jsx
<form action={serverAction}>
  {/* Progressive enhancement: works even without JS! */}
  <input name="email" />
  <button type="submit">Subscribe</button>
</form>
```

---

## Q6. How do Portals work internally?

**Answer:**

`createPortal` renders children into a different DOM container while
maintaining the fiber tree hierarchy:

```jsx
createPortal(children, domNode, key?)
```

**Internal representation:**
```javascript
// createPortal returns a special React element:
{
  $$typeof: REACT_PORTAL_TYPE,  // Not REACT_ELEMENT_TYPE!
  key: key,
  children: children,
  containerInfo: domNode,  // Target DOM node
}
```

**Fiber tree vs DOM tree:**
```
Fiber tree:                    DOM tree:
  App                           #root
   └── Modal                      └── div
        └── Portal                     └── "Click counter: 5"
             └── div              #modal-root
                  └── "Hello"       └── div
                                         └── "Hello"
```

**Event bubbling through fiber tree:**
- DOM events on portal content bubble through DOM (to `#modal-root`)
- React events bubble through fiber tree (to `App`)
- A `onClick` on `App` catches clicks inside the Portal

---

## Q7. How does the Context system propagate changes?

**Answer:**

```
Provider value changes:
  1. Provider fiber detects value change (Object.is comparison)
  2. propagateContextChange(fiber, context, renderLanes)
     → Walks the ENTIRE subtree depth-first
     → For each fiber, checks fiber.dependencies linked list
     → If fiber consumes this context:
       a. Mark fiber.lanes |= renderLanes (force update)
       b. Mark all ancestors up to provider (childLanes)
  3. Consumers re-render, reading new context value
```

**Performance implications:**
```
<ThemeContext.Provider value={{ color: 'red', size: 'large' }}>
  {/* Creating new object every render → ALWAYS triggers propagation */}
  {/* Even if color and size haven't changed! */}
</ThemeContext.Provider>

// Fix: memoize the value
const value = useMemo(() => ({ color, size }), [color, size]);
<ThemeContext.Provider value={value}>
```

**Context does NOT support selectors:**
```
// ❌ No way to subscribe to only part of context
const { color } = useContext(ThemeContext);
// If size changes, this component still re-renders!

// Workaround: split into separate contexts
<ColorContext.Provider value={color}>
<SizeContext.Provider value={size}>
```

---

## Q8. What are the key differences between React 17, 18, and 19?

**Answer:**

| Feature | React 17 | React 18 | React 19 |
|---------|----------|----------|----------|
| **Rendering** | Sync only | Sync + Concurrent | Sync + Concurrent |
| **Batching** | Only in events | Automatic everywhere | Automatic everywhere |
| **Root API** | `ReactDOM.render` | `createRoot` | `createRoot` |
| **Suspense** | Basic | Streaming SSR + selective hydration | + `use()` hook |
| **Transitions** | ✗ | `startTransition`, `useTransition` | Enhanced |
| **Server** | `renderToString` | + `renderToPipeableStream` | + RSC, Server Actions |
| **Hooks** | Standard set | + `useId`, `useSyncExternalStore`, `useInsertionEffect` | + `use()`, `useActionState`, `useOptimistic`, `useFormStatus` |
| **Event delegation** | Document root | Container root | Container root |
| **Compiler** | ✗ | ✗ | React Compiler (opt-in) |
| **Ref cleanup** | ✗ | ✗ | Ref callbacks return cleanup fn |
| **Forms** | Manual | Manual | `<form action={fn}>`, `useFormStatus` |

---

## Q9. How does React handle state preservation across renders?

**Answer:**

React preserves state when it can identify a component as the "same"
between renders. The rules are:

```
State PRESERVED when:
  1. Same component TYPE (by reference) at same POSITION in tree
  2. Same KEY (if provided)
  3. Parent component didn't change type

State DESTROYED when:
  1. Component TYPE changes (div → span, ComponentA → ComponentB)
  2. KEY changes (key="a" → key="b")
  3. Component removed and re-added (conditional rendering)
  4. Different position in parent's child list (without keys)
```

**Decision tree:**
```
Is component at same position? ─── No → destroy + recreate
          │
         Yes
          │
Same type? ─── No → destroy + recreate
   │
  Yes
   │
Same key? ─── No → destroy + recreate
   │
  Yes
   │
→ PRESERVE STATE, update props
```

**Key reset pattern:**
```jsx
// Force state reset by changing key:
<UserProfile key={userId} userId={userId} />
// When userId changes: key changes → destroy old → create new → fresh state
```

---

## Q10. What design patterns should a React architect know?

**Answer:**

**1. Compound Components (Context-based):**
```jsx
<Tabs>
  <TabList>
    <Tab index={0}>Home</Tab>
    <Tab index={1}>Settings</Tab>
  </TabList>
  <TabPanels>
    <TabPanel index={0}>Home content</TabPanel>
    <TabPanel index={1}>Settings content</TabPanel>
  </TabPanels>
</Tabs>
// Tabs provides context, children consume it
```

**2. Render Props / Children as Function:**
```jsx
<DataProvider render={(data) => <Chart data={data} />} />
```

**3. Higher-Order Components (HOC):**
```jsx
const EnhancedComponent = withAuth(withTheme(MyComponent));
// Creates wrapper components that inject props
```

**4. Custom Hook Extraction:**
```jsx
function useWindowSize() {
  const [size, setSize] = useState({ width: 0, height: 0 });
  useEffect(() => { /* resize listener */ }, []);
  return size;
}
```

**5. Controlled vs Uncontrolled Components:**
```jsx
// Controlled: React owns the state
<input value={value} onChange={e => setValue(e.target.value)} />

// Uncontrolled: DOM owns the state
<input defaultValue="initial" ref={inputRef} />
```

**6. Optimistic Updates:**
```jsx
const [optimisticItems, addOptimistic] = useOptimistic(items);
async function handleAdd(newItem) {
  addOptimistic(prev => [...prev, { ...newItem, pending: true }]);
  await saveItem(newItem);  // Server confirms
}
```

**Architecture considerations:**
- Colocate state near where it's used
- Lift state up only when sharing is needed
- Use Context for "global" state (theme, auth, locale)
- Use external stores (Redux, Zustand) for complex state machines
- Prefer composition over inheritance
- Keep components pure — side effects in effects/handlers only
