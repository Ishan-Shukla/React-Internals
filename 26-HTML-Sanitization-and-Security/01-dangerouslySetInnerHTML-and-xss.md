# HTML Sanitization, dangerouslySetInnerHTML, and XSS Prevention

## React's Default XSS Protection

By default, React escapes ALL string content before inserting it into the DOM.
This is the single most important security feature in React:

```jsx
function UserComment({ text }) {
  return <div>{text}</div>;
}

// If text = '<script>alert("hacked")</script>'
// React does NOT create a <script> element.
// It creates a text node with the literal string:
// "<script>alert("hacked")</script>"
```

### How Escaping Works Internally

```javascript
// When React sets text content, it uses DOM text nodes:

function setTextContent(node, text) {
  if (text) {
    // Uses textContent, NOT innerHTML
    node.textContent = text;
    // textContent automatically escapes HTML:
    // '<script>' becomes the literal text "<script>"
    // No HTML parsing occurs
  } else {
    node.textContent = '';
  }
}

// For child text in JSX:
// <div>{'<b>bold</b>'}</div>
// React creates: document.createTextNode('<b>bold</b>')
// Output: <div>&lt;b&gt;bold&lt;/b&gt;</div>
// The user sees: <b>bold</b> (as visible text, not bold)
```

### What React Escapes

```
Expression                        DOM Output                Safe?
────────────────────────────────  ──────────────────────    ─────
<div>{"<script>alert(1)"}</div>   Text node: <script>...   YES ✓
<div>{userInput}</div>            Text node: escaped        YES ✓
<div>{`Hello ${name}`}</div>      Text node: escaped        YES ✓
<a href={userUrl}>Link</a>        href attribute: raw       PARTIAL ⚠
<div style={userStyle}>...</div>  Inline styles: filtered   YES ✓
```

## dangerouslySetInnerHTML: The Escape Hatch

When you NEED raw HTML (e.g., from a CMS, markdown renderer, or rich text
editor), React provides `dangerouslySetInnerHTML`:

```jsx
function RichContent({ htmlContent }) {
  return <div dangerouslySetInnerHTML={{ __html: htmlContent }} />;
}

// The name is deliberately scary — it's dangerous!
```

### How It Works Internally

```javascript
// During initial property setup (setInitialDOMProperties):

function setInitialDOMProperties(domElement, nextProps) {
  for (const propKey in nextProps) {
    const nextProp = nextProps[propKey];

    if (propKey === 'dangerouslySetInnerHTML') {
      const nextHtml = nextProp ? nextProp.__html : undefined;
      if (nextHtml != null) {
        // ★ DIRECTLY SETS innerHTML — NO ESCAPING ★
        domElement.innerHTML = nextHtml;
        // This is why it's dangerous:
        // Any HTML, including <script>, is parsed and executed
      }
    } else if (propKey === 'children') {
      if (typeof nextProp === 'string' || typeof nextProp === 'number') {
        // Safe: uses textContent (escaped)
        domElement.textContent = '' + nextProp;
      }
    }
    // ... other props
  }
}

// During updates (commitUpdate):
function updateDOMProperties(domElement, updatePayload) {
  for (let i = 0; i < updatePayload.length; i += 2) {
    const propKey = updatePayload[i];
    const propValue = updatePayload[i + 1];

    if (propKey === 'dangerouslySetInnerHTML') {
      domElement.innerHTML = propValue.__html;
    }
    // ...
  }
}
```

### The __html Key Requirement

```javascript
// You MUST use the { __html: '...' } wrapper:
<div dangerouslySetInnerHTML={{ __html: htmlString }} />

// This is intentional — the __html key serves as a "speed bump"
// that forces you to explicitly acknowledge the danger.

// You CANNOT accidentally pass raw HTML:
<div dangerouslySetInnerHTML={htmlString} />     // ❌ Error
<div dangerouslySetInnerHTML={{ htmlString }} />  // ❌ Wrong key
<div dangerouslySetInnerHTML={{ __html: htmlString }} /> // ✓ Explicit
```

### dangerouslySetInnerHTML vs children: Mutual Exclusion

```javascript
// React enforces that you can't use both:

<div dangerouslySetInnerHTML={{ __html: '<b>Bold</b>' }}>
  Some text
</div>
// ❌ Error: Can only set one of `children` or `dangerouslySetInnerHTML`

// Internally:
function validateDOMNesting(tag, props) {
  if (props.dangerouslySetInnerHTML != null && props.children != null) {
    throw new Error(
      'Can only set one of `children` or `dangerouslySetInnerHTML`.'
    );
  }
}
```

## $$typeof: Preventing JSON Injection

As covered in the ReactElement section, `$$typeof: Symbol.for('react.element')`
prevents XSS via JSON data injection:

```javascript
// Attack scenario without $$typeof:
// Server returns untrusted JSON that looks like a React element:
const malicious = JSON.parse(`{
  "type": "div",
  "props": {
    "dangerouslySetInnerHTML": {
      "__html": "<img src=x onerror=alert(1)>"
    }
  }
}`);

// If React rendered this as an element → XSS!

// But React checks $$typeof:
if (element.$$typeof !== Symbol.for('react.element')) {
  // NOT a real React element — treat as plain data
  // Rendered as "[object Object]" text, not as HTML
}

// JSON.parse cannot produce Symbols, so malicious JSON
// can never have the correct $$typeof value.
```

## URL Sanitization (javascript: Protocol)

React blocks dangerous URL protocols in href and src attributes:

```javascript
// Starting in React 16.9+:

<a href="javascript:alert('xss')">Click</a>
// ⚠ Warning: A future version of React will block javascript: URLs
// React logs a warning but currently still allows it

// React's internal URL validation:
function sanitizeURL(url) {
  // Block javascript: and data: protocols
  const isJavaScript = /^[\u0000-\u001F ]*j[\r\n\t]*a[\r\n\t]*v[\r\n\t]*a[\r\n\t]*s[\r\n\t]*c[\r\n\t]*r[\r\n\t]*i[\r\n\t]*p[\r\n\t]*t[\r\n\t]*:/i;

  if (__DEV__) {
    if (isJavaScript.test(url)) {
      console.error(
        'A future version of React will block javascript: URLs as a security precaution.'
      );
    }
  }
}

// Applies to:
// <a href="javascript:...">     ⚠
// <form action="javascript:..."> ⚠
// <iframe src="javascript:...">  ⚠
```

## Style Injection Prevention

React sanitizes style objects to prevent CSS-based attacks:

```javascript
// React only accepts OBJECT styles, not string styles:
<div style={{ color: 'red' }} />    // ✓ Object → safe
<div style="color: red" />          // ❌ String → dev warning

// Why objects are safer:
// 1. No CSS injection via string concatenation
// 2. React controls which properties are set
// 3. Values are assigned via element.style[prop] = value
//    (not via cssText which could contain injection)

// React's internal style handling:
function setValueForStyles(node, styles) {
  const style = node.style;
  for (const styleName in styles) {
    const value = styles[styleName];

    if (styleName === 'float') {
      style.cssFloat = value;
    } else if (isCustomProperty(styleName)) {
      style.setProperty(styleName, value);
    } else {
      // Direct property assignment — no CSS parsing
      style[styleName] = value;
      // This prevents: style["background"] = "url(javascript:...)"
      // from being interpreted as CSS injection
    }
  }
}
```

## Server-Side Rendering Security

During SSR, React must escape content differently because it's generating
HTML strings, not manipulating the DOM:

```javascript
// SSR escaping (in Fizz/react-dom/server):

function escapeHtml(string) {
  // Escape characters that have special meaning in HTML:
  const str = '' + string;
  let match = /["'&<>]/.exec(str);
  if (!match) return str;

  let html = '';
  let index = 0;
  let lastIndex = 0;

  for (index = match.index; index < str.length; index++) {
    switch (str.charCodeAt(index)) {
      case 34: // "
        html += '&quot;';
        break;
      case 38: // &
        html += '&amp;';
        break;
      case 39: // '
        html += '&#x27;';
        break;
      case 60: // <
        html += '&lt;';
        break;
      case 62: // >
        html += '&gt;';
        break;
    }
  }

  return html;
}

// SSR output for <div>{'<script>alert(1)</script>'}</div>:
// <div>&lt;script&gt;alert(1)&lt;/script&gt;</div>
// The browser displays the literal text, not executable script
```

## Security Summary

```
Attack Vector            React's Defense                     Safe?
──────────────────────   ─────────────────────────────────   ─────
XSS via text content     textContent (auto-escaped)          YES ✓
XSS via JSON injection   $$typeof Symbol check               YES ✓
XSS via innerHTML        Requires dangerouslySetInnerHTML    USER ⚠
XSS via javascript: URL  Warning (full block planned)        PARTIAL ⚠
CSS injection            Object styles, no string styles     YES ✓
SSR HTML injection       escapeHtml on all string content    YES ✓
Event handler injection  Only function refs, no strings      YES ✓
Attribute injection      Individual setAttribute calls       YES ✓

⚠ = React warns but developer must handle sanitization
✓ = React handles automatically
```
