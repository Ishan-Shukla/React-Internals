# JSX Transformation

## What JSX Actually Is

JSX is **syntactic sugar** — it is not HTML, not a template language, and not
understood by any JavaScript engine. A compiler (Babel, TypeScript, SWC, esbuild)
transforms JSX into function calls **before** your code ever runs.

## The Two JSX Transforms

### Classic Transform (Pre-React 17)

```jsx
// INPUT: What you write
<div className="app">
  <Header title="Hello" />
  <p>World</p>
</div>

// OUTPUT: What the compiler produces
React.createElement(
  'div',
  { className: 'app' },
  React.createElement(Header, { title: 'Hello' }),
  React.createElement('p', null, 'World')
);
```

This is why you needed `import React from 'react'` at the top of every JSX file —
`React.createElement` had to be in scope.

### New Transform (React 17+)

```jsx
// INPUT: What you write (no import needed!)
<div className="app">
  <Header title="Hello" />
  <p>World</p>
</div>

// OUTPUT: Compiler auto-imports from react/jsx-runtime
import { jsx as _jsx, jsxs as _jsxs } from 'react/jsx-runtime';

_jsxs('div', {
  className: 'app',
  children: [
    _jsx(Header, { title: 'Hello' }),
    _jsx('p', { children: 'World' }),
  ],
});
```

### Differences Between the Two

| Aspect | Classic (`createElement`) | New (`jsx` runtime) |
|--------|--------------------------|---------------------|
| Import required | Yes, `import React` | No, auto-injected |
| Children | Separate args (variadic) | Part of `props.children` |
| Key handling | Part of props in 2nd arg | Separate 3rd argument |
| Function name | `React.createElement` | `jsx` / `jsxs` |
| Performance | Slightly slower | Slightly faster |

`jsx` = single child or no children.
`jsxs` = multiple children (signals static children for optimization).

## How the Compiler Decides

```
JSX Element
    │
    ├── Has children?
    │   ├── No children  → jsx(type, props)
    │   ├── One child    → jsx(type, {...props, children: child})
    │   └── N children   → jsxs(type, {...props, children: [c1, c2, ...]})
    │
    ├── Has key?
    │   └── Yes → jsx(type, props, key)   // key is 3rd arg, NOT in props
    │
    └── Has ref?  (React 19+)
        └── Yes → ref goes into props directly
```

## Fragments

```jsx
// INPUT
<>
  <A />
  <B />
</>

// OUTPUT (new transform)
import { Fragment as _Fragment, jsxs as _jsxs } from 'react/jsx-runtime';

_jsxs(_Fragment, {
  children: [_jsx(A, {}), _jsx(B, {})]
});
```

`Fragment` is a symbol: `Symbol.for('react.fragment')`. It tells the reconciler
"don't create a DOM node, just render the children."

## Source Code: jsx() Function

```javascript
// Simplified from packages/react/src/jsx/ReactJSXElement.js

export function jsx(type, config, maybeKey) {
  let key = null;
  let ref = null;
  let props = {};

  // Extract key if passed as 3rd argument
  if (maybeKey !== undefined) {
    key = '' + maybeKey;
  }
  // Also check config for key (legacy support)
  if (config.key !== undefined) {
    key = '' + config.key;
  }

  // Extract ref from config
  if (config.ref !== undefined) {
    ref = config.ref;
  }

  // Copy remaining config to props
  for (const propName in config) {
    if (propName !== 'key' && propName !== 'ref') {
      props[propName] = config[propName];
    }
  }

  // Apply defaultProps
  if (type && type.defaultProps) {
    const defaultProps = type.defaultProps;
    for (const propName in defaultProps) {
      if (props[propName] === undefined) {
        props[propName] = defaultProps[propName];
      }
    }
  }

  return ReactElement(type, key, ref, props);
}
```

## What Happens at Compile Time vs Runtime

```
┌──────────────────────────────────────────────────────┐
│                    COMPILE TIME                      │
│                                                      │
│  JSX source  ───►  Babel/SWC/TS  ───►  JS with       │
│                    transform           jsx() calls   │
│                                                      │
│  <App />     ───►  ──────────── ───►  jsx(App, {})   │
└──────────────────────────────────────────────────────┘
                          │
                          ▼
┌──────────────────────────────────────────────────────┐
│                    RUNTIME                           │
│                                                      │
│  jsx(App, {})  ───►  ReactElement  ───►  Fiber tree  │
│                      { type, props,      (reconciler │
│                        key, ref }         handles it)│
└──────────────────────────────────────────────────────┘
```
