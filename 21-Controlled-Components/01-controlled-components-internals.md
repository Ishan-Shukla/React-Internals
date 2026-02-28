# Controlled Components: How React Manages Form Elements

## The Problem: DOM State vs React State

Form elements like `<input>`, `<textarea>`, and `<select>` maintain their
own internal state in the DOM. This creates two sources of truth:

```
                    ┌───────────────────┐
                    │ React State       │
                    │ value = "Hello"   │
                    └────────┬──────────┘
                             │ props
                             ▼
                    ┌───────────────────┐
                    │ <input>           │
                    │ DOM value = ???   │ ← Does DOM follow React,
                    └───────────────────┘   or does React follow DOM?

Controlled:   React state → DOM.  React is the source of truth.
Uncontrolled: DOM → React reads it.  DOM is the source of truth.
```

## Controlled Input Internals

When you write `<input value={text} onChange={handler} />`, React does
something special during the commit phase:

```javascript
// Simplified from ReactDOMInput.js

function setInitialProperties(domElement, tag, props) {
  switch (tag) {
    case 'input':
      ReactDOMInput.initInput(domElement, props);
      break;
    case 'textarea':
      ReactDOMTextarea.initTextarea(domElement, props);
      break;
    case 'select':
      ReactDOMSelect.initSelect(domElement, props);
      break;
  }
}

// For <input>:
function initInput(element, props) {
  const value = props.value;
  const defaultValue = props.defaultValue;
  const checked = props.checked;

  // Set initial attributes
  if (props.type != null) {
    element.setAttribute('type', props.type);
  }

  if (value != null) {
    // ★ CONTROLLED: React sets the DOM value ★
    element.value = '' + value;
  } else if (defaultValue != null) {
    element.value = '' + defaultValue;
  }

  if (checked != null) {
    element.checked = checked;
  }

  // ★ Track the "controlled" state ★
  // React stores whether this input is controlled
  element._wrapperState = {
    initialChecked: checked ?? defaultValue,
    initialValue: value ?? defaultValue,
    controlled: value != null,  // ← Key flag!
  };
}
```

## The Update Cycle for Controlled Inputs

```
User types "H" in controlled input:

Step 1: DOM Event
  ├── Browser updates input.value = "H" (native behavior)
  ├── Input event fires
  └── React's delegated listener catches it

Step 2: React Event Handler
  ├── Your onChange fires: setText("H")
  ├── Enqueue state update
  └── Schedule render

Step 3: React Render
  ├── Component re-renders with text = "H"
  └── Returns <input value="H" ... />

Step 4: React Commit (commitUpdate)
  ├── updateInput(element, prevProps, nextProps)
  ├── Compare: new value "H" vs DOM value "H"
  └── Already matches → no DOM write needed

BUT if the handler DOESN'T update state:

Step 1: Browser: input.value = "H"
Step 2: onChange fires, but handler does nothing (no setState)
Step 3: No re-render... but DOM shows "H"!

Step 4: ★ REACT RESTORES THE VALUE ★
  React detects this case and RESETS the DOM:

  function restoreControlledInputState(element, props) {
    if (props.value != null) {
      const value = '' + props.value;
      if (element.value !== value) {
        element.value = value;  // ← Force DOM back to React's value
      }
    }
  }

This is why typing in a controlled input with no onChange seems "stuck" —
React immediately resets the DOM value to match the state.
```

```
Full controlled input flow:

User types ──► DOM updates ──► onChange fires ──► setState ──►
    │                                                          │
    │                                                     Re-render
    │                                                          │
    │                                               Commit: set DOM value
    │                                                          │
    │              ◄──── DOM reflects React state ◄────────────┘
    │
    └── If no setState: React RESETS DOM value
        Input appears "frozen"
```

## How React Detects Controlled vs Uncontrolled

```javascript
// During updates in the commit phase:

function updateInput(element, prevProps, nextProps) {
  const prevValue = prevProps.value;
  const nextValue = nextProps.value;

  if (nextValue != null) {
    // ═══ CONTROLLED ═══
    // React owns the value
    const value = '' + nextValue;
    if (element.value !== value) {
      element.value = value;
    }
  } else if (prevValue != null && nextValue == null) {
    // ═══ SWITCHING FROM CONTROLLED TO UNCONTROLLED ═══
    // This is a warning in development
    if (__DEV__) {
      console.error(
        'A component is changing a controlled input to be uncontrolled...'
      );
    }
  }

  // Handle checked for checkboxes/radio
  if (nextProps.checked != null) {
    element.checked = nextProps.checked;
  }
}
```

## Textarea and Select Internals

### Textarea

```javascript
// React treats textarea value as children in JSX but
// converts it to the value property internally

// <textarea value="hello" />
// ↓ internally becomes ↓
// element.value = "hello"
// (NOT element.innerHTML or element.textContent)

function initTextarea(element, props) {
  let initialValue = props.value;

  if (initialValue == null) {
    let { children, defaultValue } = props;
    if (children != null) {
      // <textarea>text here</textarea>
      // In dev, warn that children shouldn't be used with value
      initialValue = children;
    }
    if (defaultValue != null) {
      initialValue = defaultValue;
    }
  }

  element.value = initialValue ?? '';
}
```

### Select

```javascript
// React handles <select value="option2"> by setting
// the matching <option>'s selected property

function updateSelect(element, prevProps, nextProps) {
  const value = nextProps.value;
  const multiple = nextProps.multiple;

  if (value != null) {
    // Set which option(s) are selected
    const options = element.options;
    for (let i = 0; i < options.length; i++) {
      const option = options[i];
      const optionValue = option.value;

      if (multiple) {
        // value is an array
        option.selected = value.indexOf(optionValue) !== -1;
      } else {
        // value is a string
        option.selected = optionValue === value;
      }
    }
  }
}
```

## useInsertionEffect: CSS-in-JS Timing

`useInsertionEffect` runs BEFORE any DOM mutations in the commit phase.
It exists specifically for CSS-in-JS libraries that need to inject
`<style>` tags before React reads layout:

```javascript
import { useInsertionEffect } from 'react';

// Used by CSS-in-JS libraries (styled-components, emotion)
function useCSS(rule) {
  useInsertionEffect(() => {
    // Insert <style> tag BEFORE React does DOM mutations
    // This way, when React inserts elements in the mutation phase,
    // the styles are already available
    const style = document.createElement('style');
    style.textContent = rule;
    document.head.appendChild(style);

    return () => {
      document.head.removeChild(style);
    };
  });
}
```

```
Commit Phase Timing:

  ┌────────────────────────────────────────────────────────────────┐
  │ useInsertionEffect  │ DOM Mutations  │ useLayoutEffect │ Paint │
  │ (inject styles)     │ (insert/update │ (read layout)   │       │
  │                     │  elements)     │                 │       │
  └────────────────────────────────────────────────────────────────┘
  ▲                     ▲                ▲                  ▲
  │                     │                │                  │
  Styles exist          Elements get     Components can     User
  before elements       styled correctly read dimensions    sees
  are inserted                                             result

Without useInsertionEffect, CSS-in-JS could cause a flash of
unstyled content (FOUC) because styles would be injected AFTER
elements are inserted.
```

```javascript
// Effect execution order during commit:
// 1. useInsertionEffect  ← For CSS injection (before DOM changes)
// 2. DOM mutations        ← insertBefore, appendChild, removeChild
// 3. useLayoutEffect      ← For DOM measurements (sync, before paint)
// 4. Browser paint        ← User sees the result
// 5. useEffect            ← For side effects (async, after paint)
```
