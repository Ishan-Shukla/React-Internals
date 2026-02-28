# SVG and Namespace Handling

## The Problem: HTML vs SVG vs MathML

HTML, SVG, and MathML are different XML namespaces. Elements must be created
with the correct namespace, or they won't work:

```javascript
// HTML elements:
document.createElement('div');
// Creates: <div> in http://www.w3.org/1999/xhtml

// SVG elements:
document.createElementNS('http://www.w3.org/2000/svg', 'circle');
// Creates: <circle> in http://www.w3.org/2000/svg

// Using createElement for SVG → BROKEN:
document.createElement('circle');
// Creates: <circle> in HTML namespace → browser doesn't render it!
```

React must automatically detect when to switch namespaces.

## Host Context: Namespace Tracking

React uses the **host context** system (part of the reconciler's host config)
to track the current namespace as it traverses the tree:

```javascript
// Simplified from ReactFiberConfigDOM.js

const HTML_NAMESPACE = 'http://www.w3.org/1999/xhtml';
const SVG_NAMESPACE = 'http://www.w3.org/2000/svg';
const MATH_NAMESPACE = 'http://www.w3.org/1998/Math/MathML';

function getChildHostContext(parentHostContext, type) {
  const parentNamespace = parentHostContext;

  // Entering an <svg> element → switch to SVG namespace
  if (type === 'svg') {
    return SVG_NAMESPACE;
  }

  // Entering a <math> element → switch to MathML namespace
  if (type === 'math') {
    return MATH_NAMESPACE;
  }

  // Entering <foreignObject> inside SVG → switch back to HTML
  if (parentNamespace === SVG_NAMESPACE && type === 'foreignObject') {
    return HTML_NAMESPACE;
  }

  // Otherwise, inherit parent namespace
  return parentNamespace;
}

function getRootHostContext(rootContainerInfo) {
  const namespaceURI = rootContainerInfo.namespaceURI;
  const type = rootContainerInfo.tagName;

  // Determine initial namespace from the root container
  return getIntrinsicNamespace(type, namespaceURI);
}
```

## Namespace Context Stack

As React traverses the fiber tree, it pushes and pops namespace context:

```jsx
<div>                              context: HTML_NAMESPACE
  <p>Text</p>                     context: HTML_NAMESPACE
  <svg viewBox="0 0 100 100">     context: SVG_NAMESPACE ← pushed
    <circle cx="50" cy="50" />    context: SVG_NAMESPACE
    <g>                           context: SVG_NAMESPACE
      <rect width="10" />        context: SVG_NAMESPACE
      <foreignObject>             context: HTML_NAMESPACE ← pushed
        <div>HTML inside SVG</div> context: HTML_NAMESPACE
      </foreignObject>            ← popped back to SVG_NAMESPACE
    </g>
  </svg>                          ← popped back to HTML_NAMESPACE
  <math>                          context: MATH_NAMESPACE ← pushed
    <mi>x</mi>                   context: MATH_NAMESPACE
    <mo>=</mo>                   context: MATH_NAMESPACE
  </math>                        ← popped back to HTML_NAMESPACE
</div>
```

```
Stack visualization during traversal:

Processing <div>:        stack: [HTML]
Processing <svg>:        stack: [HTML, SVG]        ← push SVG
Processing <circle>:     stack: [HTML, SVG]        (inherit)
Processing <foreignObject>: stack: [HTML, SVG, HTML] ← push HTML
Processing <div> inside: stack: [HTML, SVG, HTML]  (inherit)
Leaving <foreignObject>: stack: [HTML, SVG]        ← pop
Leaving <svg>:           stack: [HTML]             ← pop
Processing <math>:       stack: [HTML, MATH]       ← push MATH
Processing <mi>:         stack: [HTML, MATH]       (inherit)
Leaving <math>:          stack: [HTML]             ← pop
```

## Element Creation with Namespace

```javascript
// In completeWork, when creating a host component instance:

function createInstance(type, props, rootContainerInstance, hostContext) {
  const namespace = hostContext;  // The current namespace from context

  let domElement;
  if (namespace === HTML_NAMESPACE) {
    // Standard HTML element
    domElement = document.createElement(type);
  } else {
    // SVG or MathML element — must use createElementNS
    domElement = document.createElementNS(namespace, type);
  }

  precacheFiberNode(internalInstanceHandle, domElement);
  updateFiberProps(domElement, props);

  return domElement;
}
```

## SVG Attribute Differences

SVG uses different attribute names than HTML. React handles the mapping:

```javascript
// HTML attributes are lowercase:
<div className="box" tabIndex={0} />
//   className   tabIndex

// SVG attributes use camelCase for multi-word names:
<svg viewBox="0 0 100 100">
  <circle
    cx="50"
    cy="50"
    r="40"
    strokeWidth="2"       // ← React: camelCase
    fillOpacity="0.5"     // ← React: camelCase
    clipPath="url(#clip)"
  />
</svg>

// React maps these to the correct DOM attribute names:
// strokeWidth → stroke-width (kebab-case in DOM)
// fillOpacity → fill-opacity
// clipPath → clip-path
```

### SVG Property Mapping

```javascript
// Simplified from react-dom-bindings

const SVG_ATTRIBUTE_MAP = {
  // React prop name → DOM attribute name
  'accentHeight': 'accent-height',
  'alignmentBaseline': 'alignment-baseline',
  'clipPath': 'clip-path',
  'clipRule': 'clip-rule',
  'colorInterpolation': 'color-interpolation',
  'dominantBaseline': 'dominant-baseline',
  'fillOpacity': 'fill-opacity',
  'fillRule': 'fill-rule',
  'floodColor': 'flood-color',
  'floodOpacity': 'flood-opacity',
  'fontFamily': 'font-family',
  'fontSize': 'font-size',
  'fontWeight': 'font-weight',
  'glyphOrientationHorizontal': 'glyph-orientation-horizontal',
  'letterSpacing': 'letter-spacing',
  'stopColor': 'stop-color',
  'stopOpacity': 'stop-opacity',
  'strokeDasharray': 'stroke-dasharray',
  'strokeDashoffset': 'stroke-dashoffset',
  'strokeLinecap': 'stroke-linecap',
  'strokeLinejoin': 'stroke-linejoin',
  'strokeMiterlimit': 'stroke-miterlimit',
  'strokeOpacity': 'stroke-opacity',
  'strokeWidth': 'stroke-width',
  'textAnchor': 'text-anchor',
  'textDecoration': 'text-decoration',
  'transformOrigin': 'transform-origin',
  'xlinkHref': 'xlink:href',         // ← cross-namespace attribute!
  'xmlLang': 'xml:lang',
  'xmlSpace': 'xml:space',
  // ... many more
};

// For setting SVG attributes:
function setSVGAttribute(node, name, value) {
  const mappedName = SVG_ATTRIBUTE_MAP[name] || name;

  if (name.startsWith('xlink')) {
    // Cross-namespace attribute
    node.setAttributeNS(
      'http://www.w3.org/1999/xlink',
      mappedName,
      value
    );
  } else if (name.startsWith('xml')) {
    node.setAttributeNS(
      'http://www.w3.org/XML/1998/namespace',
      mappedName,
      value
    );
  } else {
    // Regular SVG attribute
    node.setAttribute(mappedName, value);
  }
}
```

## className vs class

```javascript
// HTML:  React uses className (because class is a JS reserved word)
<div className="container" />  → element.className = "container"

// SVG:  React also uses className, but SVG's className is an SVGAnimatedString
// React handles this difference:
function setValueForProperty(node, propertyName, value, attributeName) {
  if (attributeName === 'className') {
    // For SVG elements, use setAttribute instead of .className
    // because SVG className is not a simple string
    if (node.namespaceURI === SVG_NAMESPACE) {
      node.setAttribute('class', value);
    } else {
      node.className = value;
    }
  }
}
```

## Hydration and Namespaces

During SSR hydration, React must validate that server-rendered elements
are in the correct namespace:

```html
<!-- Server-rendered HTML -->
<div>
  <svg>
    <circle cx="50" cy="50" r="40" />
  </svg>
</div>

<!-- Browser parses this correctly because HTML parser
     automatically switches namespace at <svg> -->
```

```javascript
// During hydration, React walks existing DOM nodes
// The namespace is inferred from the existing DOM:

function getNamespace(element) {
  return element.namespaceURI;
  // <circle> in SVG → 'http://www.w3.org/2000/svg'
  // <div> in HTML → 'http://www.w3.org/1999/xhtml'
}

// If a mismatch occurs (e.g., server rendered SVG element
// but client creates HTML element), React logs a hydration
// mismatch warning and recreates the element.
```

## Common SVG Pitfalls in React

```jsx
// ❌ Using HTML-style boolean attributes
<svg>
  <text hidden>Secret</text>    // 'hidden' is an HTML attribute
</svg>                          // SVG uses display="none" or visibility="hidden"

// ❌ Using innerHTML for SVG (use dangerouslySetInnerHTML carefully)
<svg dangerouslySetInnerHTML={{ __html: svgString }} />
// This works but the SVG string must be well-formed SVG

// ❌ Forgetting namespace for programmatic SVG creation
useEffect(() => {
  const circle = document.createElement('circle');  // ❌ HTML namespace!
  svgRef.current.appendChild(circle);               // Won't render!

  const circle2 = document.createElementNS(         // ✓ SVG namespace
    'http://www.w3.org/2000/svg', 'circle'
  );
  svgRef.current.appendChild(circle2);              // Renders correctly
}, []);

// ✅ Let React handle it — just use JSX
<svg ref={svgRef}>
  <circle cx="50" cy="50" r="40" />   {/* React uses correct namespace */}
</svg>
```
