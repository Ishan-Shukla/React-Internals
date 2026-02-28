# Testing Internals: act() and React Test Renderer

## What act() Does

`act()` ensures that all React work (renders, effects, state updates) is
fully flushed before you make assertions. Without it, your test might
assert against stale UI.

```javascript
import { act } from 'react';
import { createRoot } from 'react-dom/client';

// WITHOUT act():
const root = createRoot(container);
root.render(<Counter />);
// State updates and effects might still be PENDING here!
// ❌ Assertion could fail because React hasn't finished

// WITH act():
await act(async () => {
  const root = createRoot(container);
  root.render(<Counter />);
});
// ✓ ALL renders, effects, and state updates are complete
// ✓ Safe to assert against the DOM
```

## act() Internals

```javascript
// Simplified from ReactAct.js

export function act(callback) {
  // 1. Enter act scope
  const prevIsBatchingLegacy = ReactCurrentActQueue.isBatchingLegacy;
  ReactCurrentActQueue.isBatchingLegacy = true;

  // Track all scheduled work
  const prevActQueue = ReactCurrentActQueue.current;
  ReactCurrentActQueue.current = [];

  try {
    // 2. Execute the callback
    const result = callback();

    // 3. If callback returns a thenable (async act)
    if (result !== null && typeof result === 'object' &&
        typeof result.then === 'function') {
      return {
        then(resolve, reject) {
          result.then(() => {
            // Flush all remaining work after async callback
            flushActQueue(resolve, reject);
          }, reject);
        }
      };
    }

    // 4. Synchronous: flush immediately
    flushActQueue();

  } finally {
    ReactCurrentActQueue.isBatchingLegacy = prevIsBatchingLegacy;
    ReactCurrentActQueue.current = prevActQueue;
  }
}

function flushActQueue() {
  // Process ALL pending work:
  // → Sync renders
  // → Microtasks
  // → Passive effects (useEffect)
  // → Scheduled callbacks
  // → Repeat until queue is empty

  while (true) {
    const queue = ReactCurrentActQueue.current;
    if (queue.length === 0) break;

    const callbacks = queue.splice(0);
    for (const callback of callbacks) {
      callback();  // Execute each queued work item
    }
  }

  // Also flush passive effects
  flushPassiveEffects();
}
```

## How act() Captures Async Work

When React is inside an `act()` scope, instead of scheduling work through
the normal scheduler (MessageChannel), it pushes callbacks to the act queue:

```javascript
// In the scheduler, when act is active:

function scheduleCallback(priority, callback) {
  if (ReactCurrentActQueue.current !== null) {
    // Inside act() → push to act queue instead of scheduler
    ReactCurrentActQueue.current.push(callback);
    return fakeCallbackNode;
  }

  // Normal path: schedule with real scheduler
  return Scheduler.scheduleCallback(priority, callback);
}
```

```
Normal (non-test) execution:          Inside act():

setState()                            setState()
  → scheduleCallback                    → scheduleCallback
  → MessageChannel                      → push to actQueue
  → (wait for macrotask)                → (immediately available)
  → render                              → flushActQueue processes it
  → commit                              → render + commit
  → (wait for next tick)                → flush passive effects
  → passive effects                     → ALL DONE synchronously

Tests would be flaky                  Tests are deterministic
without act() because of              because act() flushes
async scheduling                      everything synchronously
```

## React Testing Library and act()

React Testing Library wraps most of its utilities in `act()` automatically:

```javascript
// render() internally calls act()
import { render, fireEvent, waitFor } from '@testing-library/react';

test('counter increments', async () => {
  // render wraps in act()
  const { getByText } = render(<Counter />);

  // fireEvent wraps in act()
  fireEvent.click(getByText('Increment'));

  // After act, DOM is updated
  expect(getByText('Count: 1')).toBeInTheDocument();
});

// For async operations, use waitFor or findBy:
test('loads data', async () => {
  render(<DataComponent />);

  // waitFor polls with act() internally
  await waitFor(() => {
    expect(screen.getByText('Loaded data')).toBeInTheDocument();
  });
});
```

## React Test Renderer

A renderer that outputs a pure JavaScript object tree instead of DOM nodes.
Useful for snapshot testing and testing without a DOM:

```javascript
import TestRenderer from 'react-test-renderer';

function Link({ page, children }) {
  return <a href={page}>{children}</a>;
}

test('Link renders correctly', () => {
  const tree = TestRenderer.create(
    <Link page="https://example.com">Example</Link>
  ).toJSON();

  expect(tree).toMatchSnapshot();
  // Snapshot:
  // {
  //   type: 'a',
  //   props: { href: 'https://example.com' },
  //   children: ['Example'],
  // }
});
```

### Test Renderer Host Config

```javascript
// The test renderer implements a host config that creates
// plain objects instead of DOM nodes:

createInstance(type, props) {
  return {
    type: type,
    props: { ...props },
    children: [],
    // No actual DOM node!
  };
},

appendChild(parentInstance, child) {
  parentInstance.children.push(child);
},

commitUpdate(instance, updatePayload, type, prevProps, nextProps) {
  instance.props = { ...nextProps };
},

// This proves that react-reconciler is truly renderer-agnostic
// The same Fiber tree, hooks, and reconciliation work identically
// whether rendering to DOM, native views, or plain JS objects
```

## Test Renderer Instance API

```javascript
const renderer = TestRenderer.create(<App />);

// Get the rendered tree as a JSON object
const tree = renderer.toJSON();

// Get the underlying fiber root
const instance = renderer.root;

// Find components by type
const buttons = instance.findAllByType('button');
const myComponents = instance.findAllByType(MyComponent);

// Find by props
const submitBtn = instance.findByProps({ type: 'submit' });

// Trigger updates
await act(() => {
  renderer.update(<App newProp={true} />);
});

// Unmount
renderer.unmount();
```

## The act() Warning

If you update state without wrapping in `act()`, React shows this warning:

```
Warning: An update to Counter inside a test was not wrapped in act(...).

When testing, code that causes React state updates should be wrapped into act(...):

act(() => {
  /* fire events that update state */
});
/* assert on the output */
```

This warning exists because:

```javascript
// Without act:
fireEvent.click(button);
// → setState enqueued
// → render scheduled via MessageChannel (async!)
// → your assertion runs HERE, before render completes!
expect(screen.getByText('1')).toBeInTheDocument(); // ❌ Might still show '0'

// With act:
act(() => {
  fireEvent.click(button);
});
// → setState enqueued
// → act flushes ALL work synchronously
// → render + commit + effects complete
expect(screen.getByText('1')).toBeInTheDocument(); // ✓ Always shows '1'
```

## StrictMode in Tests

```javascript
// StrictMode double-invokes in tests to catch bugs:

test('effect cleanup', () => {
  const cleanup = jest.fn();
  const effect = jest.fn(() => cleanup);

  function Component() {
    useEffect(effect);
    return null;
  }

  render(
    <StrictMode>
      <Component />
    </StrictMode>
  );

  // In development + StrictMode:
  // effect called TWICE (mount, unmount, mount)
  // cleanup called ONCE (between the two mounts)
  expect(effect).toHaveBeenCalledTimes(2);
  expect(cleanup).toHaveBeenCalledTimes(1);
});
```
