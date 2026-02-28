# Commit Phase: Applying Changes to the DOM

## The Three Sub-Phases

After the render phase completes (all beginWork + completeWork), React enters
the **commit phase**. This phase is **synchronous and uninterruptible** — once
it starts, it runs to completion. It has three sub-phases:

```
                        COMMIT PHASE
    ┌──────────────────────────────────────────────────────┐
    │                                                      │
    │  ┌─────────────┐  ┌────────────┐  ┌────────────────┐ │
    │  │  BEFORE     │  │  MUTATION  │  │    LAYOUT      │ │
    │  │  MUTATION   │  │   PHASE    │  │    PHASE       │ │
    │  │             │  │            │  │                │ │
    │  │getSnapshot  │  │ DOM writes │  │useLayoutEffect │ │
    │  │BeforeUpdate │  │ insert,    │  │componentDidMt  │ │
    │  │             │  │ update,    │  │componentDidUpd │ │
    │  │             │  │ delete     │  │ref attachment  │ │
    │  └─────────────┘  └────────────┘  └────────────────┘ │
    │                                                      │
    └──────────────────────────────────────────────────────┘
                           │
                    Browser Paints
                           │
                    ┌──────▼───────┐
                    │   PASSIVE    │
                    │   EFFECTS    │
                    │              │
                    │  useEffect   │
                    │  (async)     │
                    └──────────────┘
```

## commitRoot Entry Point

```javascript
// Simplified from ReactFiberWorkLoop.js

function commitRoot(root, finishedWork, lanes) {
  // Check for passive effects to schedule
  const hasPassiveEffects =
    (finishedWork.subtreeFlags & PassiveMask) !== NoFlags ||
    (finishedWork.flags & PassiveMask) !== NoFlags;

  if (hasPassiveEffects) {
    // Schedule passive effects to run AFTER paint
    scheduleCallback(NormalPriority, () => {
      flushPassiveEffects();
      return null;
    });
  }

  // ═══ PHASE 1: BEFORE MUTATION ═══
  commitBeforeMutationEffects(root, finishedWork);

  // ═══ PHASE 2: MUTATION ═══
  commitMutationEffects(root, finishedWork, lanes);

  // ★ SWAP THE TREES ★
  // The work-in-progress tree becomes the current tree
  root.current = finishedWork;

  // ═══ PHASE 3: LAYOUT ═══
  commitLayoutEffects(finishedWork, root, lanes);

  // Tell the scheduler to paint
  requestPaint();

  // Check for remaining work
  ensureRootIsScheduled(root);
}
```

## Phase 1: Before Mutation

```javascript
function commitBeforeMutationEffects(root, firstChild) {
  let fiber = firstChild;
  while (fiber !== null) {
    // Recurse into children first (depth-first)
    if (fiber.child !== null && (fiber.subtreeFlags & BeforeMutationMask)) {
      commitBeforeMutationEffects(root, fiber.child);
    }

    if (fiber.flags & Snapshot) {
      // CLASS COMPONENTS: getSnapshotBeforeUpdate
      // Called BEFORE DOM changes — last chance to read the DOM
      const snapshot = fiber.stateNode.getSnapshotBeforeUpdate(
        fiber.memoizedProps,   // prev props (before this update)
        fiber.memoizedState,   // prev state
      );
      fiber.stateNode.__reactInternalSnapshotBeforeUpdate = snapshot;
    }

    fiber = fiber.sibling;
  }
}
```

```
getSnapshotBeforeUpdate use case: Capturing scroll position

class Chat extends React.Component {
  getSnapshotBeforeUpdate(prevProps) {
    // Called BEFORE new messages are inserted into DOM
    if (prevProps.messages.length < this.props.messages.length) {
      const list = this.listRef.current;
      return list.scrollHeight - list.scrollTop;  // Distance from bottom
    }
    return null;
  }

  componentDidUpdate(prevProps, prevState, snapshot) {
    // Called AFTER DOM update
    if (snapshot !== null) {
      const list = this.listRef.current;
      list.scrollTop = list.scrollHeight - snapshot;  // Restore position
    }
  }
}
```

## Phase 2: Mutation (DOM Writes)

This is where the actual DOM is modified:

```javascript
function commitMutationEffects(root, finishedWork, lanes) {
  commitMutationEffectsOnFiber(finishedWork, root);
}

function commitMutationEffectsOnFiber(finishedWork, root) {
  const flags = finishedWork.flags;

  // Process deletions first
  if (flags & ChildDeletion) {
    const deletions = finishedWork.deletions;
    for (let i = 0; i < deletions.length; i++) {
      commitDeletionEffects(root, finishedWork, deletions[i]);
    }
  }

  // Recurse into children
  if (finishedWork.subtreeFlags & MutationMask) {
    let child = finishedWork.child;
    while (child !== null) {
      commitMutationEffectsOnFiber(child, root);
      child = child.sibling;
    }
  }

  // Process this fiber's mutations
  switch (finishedWork.tag) {
    case HostComponent: {
      const instance = finishedWork.stateNode;

      if (flags & Placement) {
        // INSERT this node into the DOM
        commitPlacement(finishedWork);
      }

      if (flags & Update) {
        // UPDATE props on existing DOM node
        const updatePayload = finishedWork.updateQueue;
        // updatePayload = ['className', 'new-class', 'style', {color: 'red'}]
        commitUpdate(instance, updatePayload);
      }

      if (flags & ContentReset) {
        instance.textContent = '';
      }
      break;
    }

    case HostText: {
      if (flags & Update) {
        const newText = finishedWork.memoizedProps;
        finishedWork.stateNode.nodeValue = newText;
      }
      break;
    }

    case FunctionComponent: {
      if (flags & Update) {
        // Run useInsertionEffect (for CSS-in-JS libraries)
        commitHookEffectListMount(HookInsertion, finishedWork);
      }
      break;
    }
  }

  // Detach refs from old DOM nodes
  if (flags & Ref) {
    commitDetachRef(finishedWork);
  }
}
```

### Placement: Inserting DOM Nodes

```javascript
function commitPlacement(finishedWork) {
  // 1. Find the nearest HOST parent (actual DOM parent)
  const parentFiber = getHostParentFiber(finishedWork);
  const parentDOM = parentFiber.stateNode;

  // 2. Find the DOM node to insert BEFORE
  const before = getHostSibling(finishedWork);

  // 3. Insert
  if (before) {
    parentDOM.insertBefore(getHostNode(finishedWork), before);
  } else {
    parentDOM.appendChild(getHostNode(finishedWork));
  }
}
```

```
Finding the host sibling (getHostSibling) is complex because
React components don't produce DOM nodes:

Fiber tree:              DOM tree:
  div                     <div>
   ├── Wrapper (func)       ├── <p>  ← Wrapper's child
   │    └── p               ├── <span>
   ├── span                 └── ...
   └── ...

If inserting NEW <p>, its host sibling is <span>,
but in fiber tree, span is Wrapper's sibling, not p's.
React must walk the fiber tree to find the correct DOM sibling.
```

### Deletion: Removing DOM Nodes

```javascript
function commitDeletionEffects(root, returnFiber, deletedFiber) {
  // Walk the deleted subtree
  let node = deletedFiber;
  while (true) {
    switch (node.tag) {
      case HostComponent:
        // Remove from DOM
        removeChild(hostParent, node.stateNode);
        break;

      case FunctionComponent:
        // Run useLayoutEffect cleanups (sync)
        commitHookEffectListUnmount(HookLayout, node);
        // Schedule useEffect cleanups (async)
        schedulePassiveEffectCleanup(node);
        break;

      case ClassComponent:
        // Call componentWillUnmount
        node.stateNode.componentWillUnmount();
        break;
    }

    // Recurse into children
    if (node.child !== null) {
      node = node.child;
      continue;
    }
    // ... walk siblings and back up
  }
}
```

## Phase 3: Layout

```javascript
function commitLayoutEffects(finishedWork, root, lanes) {
  commitLayoutEffectsOnFiber(finishedWork, root);
}

function commitLayoutEffectsOnFiber(finishedWork, root) {
  // Recurse into children first
  if (finishedWork.subtreeFlags & LayoutMask) {
    let child = finishedWork.child;
    while (child !== null) {
      commitLayoutEffectsOnFiber(child, root);
      child = child.sibling;
    }
  }

  switch (finishedWork.tag) {
    case FunctionComponent: {
      // ★ Run useLayoutEffect setup functions ★
      commitHookEffectListMount(HookLayout | HookHasEffect, finishedWork);
      break;
    }

    case ClassComponent: {
      const instance = finishedWork.stateNode;
      if (finishedWork.flags & Update) {
        if (current === null) {
          // ★ componentDidMount ★
          instance.componentDidMount();
        } else {
          // ★ componentDidUpdate ★
          instance.componentDidUpdate(
            prevProps,
            prevState,
            instance.__reactInternalSnapshotBeforeUpdate,
          );
        }
      }

      // Flush setState callbacks
      commitUpdateQueue(finishedWork);
      break;
    }

    case HostComponent: {
      // Handle autoFocus
      if (finishedWork.flags & Update) {
        commitHostComponentMount(finishedWork);
      }
      break;
    }
  }

  // Attach refs (DOM nodes are now in the document)
  if (finishedWork.flags & Ref) {
    commitAttachRef(finishedWork);
  }
}
```

## Passive Effects (useEffect)

Runs asynchronously AFTER the browser paints:

```javascript
function flushPassiveEffects() {
  // 1. Run ALL destroy functions (cleanups) from previous render
  commitPassiveUnmountEffects(root.current);

  // 2. Run ALL create functions (setups) from current render
  commitPassiveMountEffects(root, root.current);
}
```

## Complete Commit Timeline

```
setState()
   │
   ▼
Render Phase (interruptible)
   │  beginWork → completeWork for each fiber
   │
   ▼
Commit Phase Begins (synchronous, uninterruptible)
   │
   ├── Before Mutation ─── getSnapshotBeforeUpdate()
   │                       (read DOM before changes)
   │
   ├── Mutation ────────── DOM insertions/updates/deletions
   │                       useInsertionEffect (setup)
   │                       detach old refs
   │
   ├── ★ root.current = finishedWork (tree swap)
   │
   ├── Layout ─────────── useLayoutEffect (cleanup then setup)
   │                       componentDidMount / componentDidUpdate
   │                       attach new refs
   │                       setState callbacks
   │
   ▼
requestPaint() → Browser Layout & Paint
   │
   ▼
Passive Effects (async, via scheduler)
   │
   ├── useEffect cleanups (ALL components, depth-first)
   └── useEffect setups (ALL components, depth-first)
```
