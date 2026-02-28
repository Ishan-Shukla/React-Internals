# React.Children Utilities

## What Are React.Children?

`React.Children` provides utilities for working with the opaque
`props.children` data structure. Children can be a single element, an array,
a string, a number, null, undefined, a boolean, or deeply nested
combinations of all of these. `React.Children` normalizes this chaos.

```jsx
// children can be anything:
<Parent>hello</Parent>              // string
<Parent>{42}</Parent>               // number
<Parent><Child /></Parent>          // single element
<Parent><A /><B /><C /></Parent>    // multiple elements (array)
<Parent>{null}</Parent>             // null
<Parent>{[<A />, [<B />, <C />]]}</Parent>  // nested arrays
<Parent>
  <>                                // fragment
    <A />
    <B />
  </>
</Parent>
```

## React.Children.map

The most commonly used utility. It flattens nested children and maps over
each child, assigning correct keys:

```javascript
// Usage:
React.Children.map(children, (child, index) => {
  return cloneElement(child, { extraProp: true });
});
```

### Internal Implementation

```javascript
// Simplified from packages/react/src/ReactChildren.js

function mapChildren(children, func, context) {
  if (children == null) return children;

  const result = [];
  let count = 0;

  mapIntoArray(children, result, '', '', function(child) {
    return func.call(context, child, count++);
  });

  return result;
}

function mapIntoArray(children, array, escapedPrefix, nameSoFar, callback) {
  const type = typeof children;

  if (type === 'undefined' || type === 'boolean') {
    children = null;  // Treat as empty
  }

  let invokeCallback = false;

  if (children === null) {
    invokeCallback = true;
  } else {
    switch (type) {
      case 'string':
      case 'number':
      case 'bigint':
        invokeCallback = true;
        break;
      case 'object':
        switch (children.$$typeof) {
          case REACT_ELEMENT_TYPE:
          case REACT_PORTAL_TYPE:
            invokeCallback = true;
            break;
        }
    }
  }

  if (invokeCallback) {
    // ═══ LEAF NODE: invoke the callback ═══
    const child = children;
    let mappedChild = callback(child);

    const childKey = nameSoFar === ''
      ? SEPARATOR + getElementKey(child, 0)
      : nameSoFar;

    if (isArray(mappedChild)) {
      // Callback returned an array → flatten recursively
      let escapedChildKey = '';
      if (childKey != null) {
        escapedChildKey = escapeUserProvidedKey(childKey) + '/';
      }
      mapIntoArray(mappedChild, array, escapedChildKey, '', c => c);
    } else if (mappedChild != null) {
      if (isValidElement(mappedChild)) {
        // ★ CLONE WITH NEW KEY ★
        // React.Children.map reassigns keys to maintain uniqueness
        mappedChild = cloneAndReplaceKey(
          mappedChild,
          escapedPrefix +
            (mappedChild.key && (!child || child.key !== mappedChild.key)
              ? escapeUserProvidedKey('' + mappedChild.key) + '/'
              : '') +
            childKey,
        );
      }
      array.push(mappedChild);
    }
    return 1;
  }

  // ═══ NOT A LEAF: recurse into children ═══

  let subtreeCount = 0;
  const nextNamePrefix = nameSoFar === '' ? SEPARATOR : nameSoFar + SUBSEPARATOR;

  if (isArray(children)) {
    // Array of children → recurse into each
    for (let i = 0; i < children.length; i++) {
      const child = children[i];
      const nextName = nextNamePrefix + getElementKey(child, i);
      subtreeCount += mapIntoArray(
        child, array, escapedPrefix, nextName, callback
      );
    }
  } else {
    // Iterable (e.g., Set, Map, generator)
    const iteratorFn = getIteratorFn(children);
    if (typeof iteratorFn === 'function') {
      const iterator = iteratorFn.call(children);
      let step;
      let ii = 0;
      while (!(step = iterator.next()).done) {
        const child = step.value;
        const nextName = nextNamePrefix + getElementKey(child, ii++);
        subtreeCount += mapIntoArray(
          child, array, escapedPrefix, nextName, callback
        );
      }
    }
  }

  return subtreeCount;
}
```

## Key Remapping: Why Children.map Changes Keys

```jsx
function Wrapper({ children }) {
  return React.Children.map(children, child => (
    <li>{child}</li>
  ));
}

<Wrapper>
  <span key="a">A</span>
  <span key="b">B</span>
</Wrapper>

// Output elements have MODIFIED keys:
// <li key=".0/.$a">  ← composite key
//   <span key="a">A</span>
// </li>
// <li key=".1/.$b">  ← composite key
//   <span key="b">B</span>
// </li>
```

```
Key format: {parent_path}/{child_key}

This ensures uniqueness even when:
- Multiple Children.map calls nest
- The callback returns arrays (which get flattened)
- Original keys collide across different mapping levels

Key escaping:
  . → prefix for auto-generated keys (index-based)
  $ → prefix for user-provided keys
  / → separator between nesting levels
  = → escape character for special chars in keys
```

## React.Children.forEach

Like map but doesn't return a result. Used for side effects:

```javascript
function forEach(children, forEachFunc, forEachContext) {
  mapChildren(
    children,
    function() {
      forEachFunc.apply(this, arguments);
      // Return undefined → nothing pushed to result array
    },
    forEachContext,
  );
}
```

## React.Children.count

Counts the number of leaf children (not nested arrays/fragments):

```javascript
function countChildren(children) {
  let n = 0;
  mapChildren(children, () => { n++; });
  return n;
}

// Examples:
React.Children.count(null)                     // 0
React.Children.count('hello')                  // 1
React.Children.count(<A />)                    // 1
React.Children.count([<A />, <B />])           // 2
React.Children.count([<A />, [<B />, <C />]])  // 3 (flattened!)
React.Children.count(<><A /><B /></>)          // 2 (fragment unwrapped)
```

## React.Children.only

Asserts that children is exactly ONE React element and returns it:

```javascript
function onlyChild(children) {
  if (!isValidElement(children)) {
    throw new Error(
      'React.Children.only expected to receive a single React element child.'
    );
  }
  return children;
}

// Usage: components that expect exactly one child
function Wrapper({ children }) {
  const child = React.Children.only(children);
  return cloneElement(child, { extra: true });
}

<Wrapper><Button /></Wrapper>           // ✓ one element
<Wrapper><A /><B /></Wrapper>           // ❌ throws
<Wrapper>text</Wrapper>                 // ❌ throws (string, not element)
```

## React.Children.toArray

Flattens children into a flat array with stable keys:

```javascript
function toArray(children) {
  return mapChildren(children, child => child) || [];
}

// Usage:
const childArray = React.Children.toArray(props.children);
// Now you can .sort(), .filter(), .slice(), etc.

// Example: render only the first N children
function LimitedList({ children, max }) {
  const limited = React.Children.toArray(children).slice(0, max);
  return <ul>{limited.map(child => <li>{child}</li>)}</ul>;
}
```

## Fragment Handling

Fragments (`<>...</>`) are transparent to `React.Children`:

```jsx
<Parent>
  <>
    <A />
    <B />
  </>
  <C />
</Parent>

// props.children = [Fragment, <C />]
// Fragment contains [<A />, <B />]

React.Children.count(props.children)
// = 3 (Fragment is unwrapped: A, B, C)

React.Children.map(props.children, child => child)
// = [<A />, <B />, <C />] (flattened)

// The Fragment itself is NOT a leaf — its children are iterated.
// This is because Fragment's $$typeof is REACT_ELEMENT_TYPE
// and its type is Symbol(react.fragment), but React.Children
// traverses into array children of fragments.

// Actually, correction: React.Children treats <></> as a single element.
// Fragments are NOT automatically unwrapped by React.Children.
// They ARE unwrapped by the reconciler during reconciliation.
//
// React.Children.count(<><A /><B /></>) = 1 (the fragment itself)
// React.Children.count([<A />, <B />]) = 2 (array children)
//
// This is an important distinction!
```

## Why React.Children Over Array Methods?

```javascript
// ❌ props.children is NOT always an array:
props.children.map(...)   // Crashes if children is a single element
props.children.length     // Undefined if children is a string

// React passes:
// - undefined/null for no children
// - a single element/string for one child
// - an array for multiple children

// React.Children handles ALL cases:
React.Children.map(undefined, fn)    // null (no crash)
React.Children.map('text', fn)       // ['text'] (wrapped)
React.Children.map(<A />, fn)        // [fn(<A />)] (wrapped)
React.Children.map([<A />, <B />], fn) // [fn(<A />), fn(<B />)]
```

## Modern Alternative: Children as Array

In modern React, you can often avoid `React.Children` entirely:

```jsx
// React.Children approach (legacy):
function Tabs({ children }) {
  const tabs = React.Children.map(children, (child, i) =>
    cloneElement(child, { isActive: i === activeIndex })
  );
  return <div>{tabs}</div>;
}

// Modern approach — compound components + context:
function Tabs({ children }) {
  const [active, setActive] = useState(0);
  return (
    <TabsContext.Provider value={{ active, setActive }}>
      <div>{children}</div>
    </TabsContext.Provider>
  );
}

function Tab({ index, children }) {
  const { active } = useContext(TabsContext);
  return <div className={active === index ? 'active' : ''}>{children}</div>;
}

// The context approach doesn't need to inspect/clone children,
// making it more flexible and less fragile.
```
