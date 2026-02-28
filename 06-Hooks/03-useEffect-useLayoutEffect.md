# useEffect and useLayoutEffect Internals

## Effect Types

Both hooks create **effect objects** but differ in WHEN they execute:

```
Commit Phase Timeline:
─────────────────────────────────────────────────────────►
  │                    │                    │
  │  Mutation Phase    │  Layout Phase      │  After Paint
  │  (DOM writes)      │  (sync, before     │  (async)
  │                    │   browser paint)   │
  │                    │                    │
  │                    │  useLayoutEffect   │  useEffect
  │                    │  runs HERE         │  runs HERE
  │                    │  (synchronous!)    │  (asynchronous)
```

## The Effect Object

```javascript
const effect = {
  tag: tag,           // HookHasEffect | HookPassive | HookLayout
  create: create,     // The effect function you pass
  destroy: destroy,   // Return value of create (cleanup function)
  deps: deps,         // Dependency array
  next: null,         // Circular linked list of effects on this fiber
};
```

Effect tags (bitmask):

```javascript
const HookHasEffect  = 0b0001;  // Effect needs to fire (deps changed)
const HookPassive    = 0b1000;  // useEffect (async, after paint)
const HookLayout     = 0b0100;  // useLayoutEffect (sync, before paint)
```

## mountEffect (First Render)

```javascript
function mountEffect(create, deps) {
  return mountEffectImpl(
    PassiveEffect | PassiveStaticEffect,   // Fiber flags
    HookPassive,                           // Hook effect tag
    create,
    deps,
  );
}

function mountLayoutEffect(create, deps) {
  return mountEffectImpl(
    UpdateEffect,    // Fiber flags
    HookLayout,      // Hook effect tag
    create,
    deps,
  );
}

function mountEffectImpl(fiberFlags, hookFlags, create, deps) {
  const hook = mountWorkInProgressHook();

  const nextDeps = deps === undefined ? null : deps;

  // Mark the fiber for effect processing in commit phase
  currentlyRenderingFiber.flags |= fiberFlags;

  // Create effect and store on hook
  hook.memoizedState = pushEffect(
    HookHasEffect | hookFlags,   // Always has effect on mount
    create,
    createEffectInstance(),      // { destroy: undefined }
    nextDeps,
  );
}
```

## pushEffect: Building the Effect List

Effects are stored as a **circular linked list** on the fiber's `updateQueue`:

```javascript
function pushEffect(tag, create, inst, deps) {
  const effect = {
    tag: tag,
    create: create,
    inst: inst,
    deps: deps,
    next: null,
  };

  let componentUpdateQueue = currentlyRenderingFiber.updateQueue;

  if (componentUpdateQueue === null) {
    componentUpdateQueue = { lastEffect: null, events: null };
    currentlyRenderingFiber.updateQueue = componentUpdateQueue;
    componentUpdateQueue.lastEffect = effect.next = effect;  // Circular
  } else {
    const lastEffect = componentUpdateQueue.lastEffect;
    const firstEffect = lastEffect.next;
    lastEffect.next = effect;
    effect.next = firstEffect;
    componentUpdateQueue.lastEffect = effect;
  }

  return effect;
}
```

```
fiber.updateQueue.lastEffect (circular list):

  effect1 (useState) ──► effect2 (useEffect) ──► effect3 (useLayoutEffect)
      ▲                                              │
      └──────────────────────────────────────────────┘

lastEffect points to effect3
effect3.next points to effect1 (circular)
```

## updateEffect (Re-renders)

```javascript
function updateEffect(create, deps) {
  return updateEffectImpl(
    PassiveEffect,
    HookPassive,
    create,
    deps,
  );
}

function updateEffectImpl(fiberFlags, hookFlags, create, deps) {
  const hook = updateWorkInProgressHook();
  const nextDeps = deps === undefined ? null : deps;
  const currentEffect = currentHook.memoizedState;
  const inst = currentEffect.inst;

  if (nextDeps !== null) {
    const prevDeps = currentEffect.deps;

    if (areHookInputsEqual(nextDeps, prevDeps)) {
      // ═══ DEPS UNCHANGED ═══
      // Push effect WITHOUT HookHasEffect tag
      // This means the effect will be in the list but WON'T fire
      hook.memoizedState = pushEffect(hookFlags, create, inst, nextDeps);
      return;
    }
  }

  // ═══ DEPS CHANGED (or no deps array) ═══
  currentlyRenderingFiber.flags |= fiberFlags;
  hook.memoizedState = pushEffect(
    HookHasEffect | hookFlags,  // HAS effect → will fire in commit
    create,
    inst,
    nextDeps,
  );
}
```

## Dependency Comparison

```javascript
function areHookInputsEqual(nextDeps, prevDeps) {
  if (prevDeps === null) return false;  // No prev deps → always re-run

  for (let i = 0; i < prevDeps.length && i < nextDeps.length; i++) {
    if (Object.is(nextDeps[i], prevDeps[i])) {
      continue;    // Same → skip
    }
    return false;  // Different → deps changed
  }

  return true;     // All deps same
}
```

```
useEffect(() => fetch(url), [url])

Render 1: deps = ['/api/users']     → HAS effect (mount)
Render 2: deps = ['/api/users']     → Object.is match → NO effect
Render 3: deps = ['/api/products']  → Object.is mismatch → HAS effect

useEffect(() => log())              → No deps array → effect EVERY render
useEffect(() => init(), [])         → Empty deps → effect ONLY on mount
```

## Commit Phase: Running Effects

### useLayoutEffect (Synchronous)

```javascript
// During commitLayoutEffects (after DOM mutation, before paint)

function commitHookLayoutEffects(finishedWork, hookFlags) {
  const updateQueue = finishedWork.updateQueue;
  const lastEffect = updateQueue.lastEffect;

  if (lastEffect !== null) {
    const firstEffect = lastEffect.next;
    let effect = firstEffect;

    do {
      if ((effect.tag & hookFlags) === hookFlags) {
        if (effect.tag & HookHasEffect) {
          // 1. Run cleanup from PREVIOUS render
          const inst = effect.inst;
          const destroy = inst.destroy;
          if (destroy !== undefined) {
            inst.destroy = undefined;
            destroy();  // Call cleanup function
          }

          // 2. Run the new effect
          const create = effect.create;
          inst.destroy = create();
          //             ^^^^^^^^
          //  Return value becomes the cleanup function
        }
      }
      effect = effect.next;
    } while (effect !== firstEffect);
  }
}
```

### useEffect (Asynchronous)

```javascript
// useEffect runs AFTER the browser paints

function commitPassiveEffects(root, finishedWork) {
  // Phase 1: Run all CLEANUP functions (destroy)
  commitPassiveUnmountEffects(finishedWork);

  // Phase 2: Run all SETUP functions (create)
  commitPassiveMountEffects(root, finishedWork);
}

// Scheduled via:
function schedulePassiveEffects(finishedWork) {
  // This uses the scheduler to run effects after paint
  scheduleCallback(NormalPriority, () => {
    flushPassiveEffects();
  });
}
```

## Effect Execution Order

```
Component tree:
  App
   ├── Parent
   │    ├── ChildA
   │    └── ChildB
   └── Sidebar

useLayoutEffect execution order (depth-first, children before parents):
  1. ChildA layoutEffect cleanup
  2. ChildA layoutEffect setup
  3. ChildB layoutEffect cleanup
  4. ChildB layoutEffect setup
  5. Parent layoutEffect cleanup
  6. Parent layoutEffect setup
  7. Sidebar layoutEffect cleanup
  8. Sidebar layoutEffect setup
  9. App layoutEffect cleanup
  10. App layoutEffect setup

useEffect execution order (ALL cleanups first, then ALL setups):
  Phase 1 - Cleanups (depth-first):
    1. ChildA cleanup
    2. ChildB cleanup
    3. Parent cleanup
    4. Sidebar cleanup
    5. App cleanup

  Phase 2 - Setups (depth-first):
    6. ChildA setup
    7. ChildB setup
    8. Parent setup
    9. Sidebar setup
    10. App setup
```

## Visual: useEffect Complete Lifecycle

```
Render 1 (Mount):
  ┌────────────────────────────────────────────────────────┐
  │ Render Phase  │ Commit Phase            │ After Paint  │
  │               │                         │              │
  │ Component()   │ DOM mutations           │ useEffect    │
  │ useEffect     │ useLayoutEffect runs    │ setup()  ────┤
  │ pushEffect    │                         │  return      │
  │ (HookHasEffect│                         │  cleanup fn  │
  │  tag set)     │                         │              │
  └────────────────────────────────────────────────────────┘

Render 2 (deps changed):
  ┌────────────────────────────────────────────────────────┐
  │ Render Phase   │ Commit Phase            │ After Paint │
  │                │                         │             │
  │ Component()    │ DOM mutations           │ cleanup()   │
  │ useEffect      │ useLayoutEffect runs    │ from prev   │
  │ deps differ    │                         │ render      │
  │ → HookHasEffect│                         │    │        │
  │   tag set      │                         │    ▼        │
  │                │                         │ setup() new │
  └────────────────────────────────────────────────────────┘

Render 3 (deps same):
  ┌────────────────────────────────────────────────────────┐
  │ Render Phase  │ Commit Phase            │ After Paint  │
  │               │                         │              │
  │ Component()   │ DOM mutations           │ (nothing!)   │
  │ useEffect     │                         │ Effect NOT   │
  │ deps same     │                         │ re-run       │
  │ → NO HookHas  │                         │              │
  │   Effect tag  │                         │              │
  └────────────────────────────────────────────────────────┘

Unmount:
  ┌─────────────────────────────────────────────────────────┐
  │ Commit Phase (deletion)                  │ After Paint  │
  │                                          │              │
  │ useLayoutEffect cleanup runs (sync)      │ useEffect    │
  │                                          │ cleanup()    │
  │                                          │ runs         │
  └─────────────────────────────────────────────────────────┘
```
