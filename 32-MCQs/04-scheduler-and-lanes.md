# MCQs: Scheduler & Lanes Priority System

---

### Q1. What data structure does the React Scheduler use for its task queue?
- A) FIFO queue (array)
- B) Linked list
- C) Min-heap (priority queue)
- D) Stack

**Answer: C**
The Scheduler uses a min-heap sorted by `expirationTime`. The task with the smallest (earliest) expiration time is at the top, ensuring highest-priority tasks run first.

---

### Q2. How many priority levels does the Scheduler define?
- A) 3
- B) 5
- C) 7
- D) 31

**Answer: B**
The Scheduler has 5 priority levels: ImmediatePriority (1), UserBlockingPriority (2), NormalPriority (3), LowPriority (4), and IdlePriority (5).

---

### Q3. What mechanism does the Scheduler use to schedule tasks?
- A) `setTimeout(fn, 0)`
- B) `requestAnimationFrame`
- C) `MessageChannel.postMessage`
- D) `queueMicrotask`

**Answer: C**
The Scheduler uses `MessageChannel` because `postMessage` has near-zero delay (unlike `setTimeout` which has a minimum ~4ms delay) and yields to the browser's event loop between messages.

---

### Q4. What is the default time slice for React's work loop?
- A) ~1ms
- B) ~5ms
- C) ~16ms
- D) ~100ms

**Answer: B**
React yields to the browser approximately every 5ms (`yieldInterval = 5`). This allows the browser to handle user input and painting without noticeable lag.

---

### Q5. What does `shouldYieldToHost()` check?
- A) If the network is idle
- B) If the elapsed time since the current task started exceeds the yield interval (~5ms)
- C) If the CPU usage is high
- D) If there are pending DOM mutations

**Answer: B**
`shouldYieldToHost` compares `performance.now()` against the deadline (`startTime + yieldInterval`). If the current time exceeds the deadline, it returns `true` to yield.

---

### Q6. What is the timeout for `ImmediatePriority` in the Scheduler?
- A) 0ms
- B) -1ms (already expired)
- C) 250ms
- D) 5000ms

**Answer: B**
`ImmediatePriority` has a timeout of `-1`, meaning the task is immediately expired and runs synchronously without yielding.

---

### Q7. What is the timeout for `NormalPriority`?
- A) 250ms
- B) 1000ms
- C) 5000ms
- D) 10000ms

**Answer: C**
`NormalPriority` has a 5000ms (5 second) timeout. After 5 seconds, the task becomes expired and forces synchronous execution.

---

### Q8. How many lane bits does React's lane model use?
- A) 8
- B) 16
- C) 31
- D) 64

**Answer: C**
React uses 31 bits (JavaScript's bitwise operators work on 32-bit integers, but bit 31 is the sign bit). Each bit represents a lane priority level.

---

### Q9. What is `SyncLane`?
- A) Lane for synchronized components
- B) The highest-priority lane (bit 1) for updates that must render synchronously
- C) A lane for CSS sync operations
- D) A lane for server synchronization

**Answer: B**
`SyncLane` (bit position 1) is the highest-priority lane. Updates assigned to `SyncLane` render synchronously via `performSyncWorkOnRoot` — they cannot be interrupted.

---

### Q10. What lane does a `click` event handler's `setState` receive?
- A) `SyncLane`
- B) `InputContinuousLane`
- C) `DefaultLane`
- D) `TransitionLane1`

**Answer: A**
Click is a discrete event, so `setState` inside a click handler gets `SyncLane` — the highest priority. The update renders synchronously.

---

### Q11. What lane does `startTransition(() => setState(x))` assign?
- A) `SyncLane`
- B) `DefaultLane`
- C) `TransitionLane` (one of 16 transition lanes)
- D) `IdleLane`

**Answer: C**
Updates inside `startTransition` get one of the 16 `TransitionLane` bits (lanes 6-21). These are low priority and can be interrupted by urgent updates.

---

### Q12. What is the purpose of `getNextLanes(root, wipLanes)`?
- A) Get the next available network lane
- B) Determine which lanes to process in the next render based on priority
- C) Get the next fiber to process
- D) Compute the next animation frame

**Answer: B**
`getNextLanes` examines `root.pendingLanes`, checks expiration times, considers entangled lanes, and returns the highest-priority set of lanes to render next.

---

### Q13. How are lanes combined (batched)?
- A) Addition: `lane1 + lane2`
- B) Bitwise OR: `lane1 | lane2`
- C) Array concatenation: `[lane1, lane2]`
- D) String join: `lane1 + ',' + lane2`

**Answer: B**
Lanes use bitwise OR to combine: `mergeLanes(a, b) = a | b`. Multiple updates can be batched together in a single render by ORing their lanes.

---

### Q14. How do you check if a lane is included in a set of lanes?
- A) `lanes.includes(lane)`
- B) `(lanes & lane) !== 0`
- C) `lanes === lane`
- D) `lanes.has(lane)`

**Answer: B**
Bitwise AND checks membership: `includesSomeLane(set, subset) = (set & subset) !== 0`. If any bits overlap, the lane is included.

---

### Q15. What is "lane starvation" in React?
- A) Running out of available lane bits
- B) Low-priority updates never completing because higher-priority updates keep interrupting
- C) Memory leaks in the lane system
- D) Network bandwidth exhaustion

**Answer: B**
If urgent updates continuously arrive, transitions and other low-priority work get perpetually interrupted and restarted, never reaching commit.

---

### Q16. How does React prevent lane starvation?
- A) By limiting the number of urgent updates
- B) By assigning expiration times — expired lanes are upgraded to synchronous rendering
- C) By randomly boosting priority
- D) By queuing all updates

**Answer: B**
Each pending lane has an expiration time. `markStarvedLanesAsExpired` checks if any lane has been pending longer than its timeout. Expired lanes use `workLoopSync`, making them uninterruptible.

---

### Q17. What is the default expiration timeout for `TransitionLane`?
- A) 250ms
- B) 1000ms
- C) 5000ms
- D) Never expires

**Answer: C**
Transition lanes have a 5000ms (5 second) expiration. After 5 seconds without completing, the transition is forced to render synchronously.

---

### Q18. What does `IdleLane` represent?
- A) The lane for the most important updates
- B) The lowest-priority lane for work that can wait indefinitely
- C) The lane used when the CPU is idle
- D) The lane for background sync operations

**Answer: B**
`IdleLane` (bit 29) is the lowest priority. Updates on `IdleLane` have no expiration timeout, meaning they can be starved indefinitely by higher-priority work.

---

### Q19. What are "entangled lanes"?
- A) Lanes that have been merged into one
- B) Lanes that must be rendered together in the same render pass
- C) Lanes that conflict with each other
- D) Lanes from different React roots

**Answer: B**
Entangled lanes are forced to render together via `root.entanglements`. When `getNextLanes` selects one entangled lane, it includes all lanes entangled with it.

---

### Q20. What does `ensureRootIsScheduled(root)` do?
- A) Ensures the root DOM element exists
- B) Checks pending lanes and schedules a Scheduler task if work is needed
- C) Ensures the root fiber has been created
- D) Validates the root container

**Answer: B**
`ensureRootIsScheduled` examines `getNextLanes(root)`, determines the appropriate Scheduler priority, and schedules a task (or reuses an existing one) to start the render.

---

### Q21. How does React deduplicate scheduled renders?
- A) By using a Set of pending renders
- B) `ensureRootIsScheduled` checks if an existing task with sufficient priority already exists
- C) By cancelling all previous renders
- D) By using a semaphore

**Answer: B**
If a Scheduler task already exists for this root with the same or higher priority, `ensureRootIsScheduled` doesn't schedule a new one. If a lower-priority task exists, it cancels and reschedules.

---

### Q22. What is the difference between `workLoopSync` and `workLoopConcurrent`?
- A) `workLoopSync` processes one fiber; `workLoopConcurrent` processes all
- B) `workLoopSync` never yields; `workLoopConcurrent` checks `shouldYield()` between fibers
- C) `workLoopSync` is for class components; `workLoopConcurrent` is for function components
- D) They are identical

**Answer: B**
`workLoopSync` runs `while (wip !== null)` — never yields. `workLoopConcurrent` runs `while (wip !== null && !shouldYield())` — can pause between fibers.

---

### Q23. What does a Scheduler "continuation" mean?
- A) A promise that continues later
- B) When a task's callback returns a function, the task is re-added to run the returned function
- C) A linked list continuation
- D) A goto statement

**Answer: B**
If the work loop yields mid-render, the `performConcurrentWorkOnRoot` callback returns itself. The Scheduler sees the returned function and reuses the task to continue later.

---

### Q24. What is the `callbackNode` on the FiberRoot?
- A) A DOM node for callbacks
- B) A reference to the currently scheduled Scheduler task for this root
- C) A callback function
- D) The root's event handler

**Answer: B**
`root.callbackNode` holds the Scheduler task reference. `ensureRootIsScheduled` uses it to check if work is already scheduled and to cancel outdated tasks.

---

### Q25. What lane does a `mousemove` event handler's setState receive?
- A) `SyncLane`
- B) `InputContinuousLane`
- C) `DefaultLane`
- D) `TransitionLane`

**Answer: B**
`mousemove` is a continuous event, so updates get `InputContinuousLane` — higher than default but lower than sync. This allows batching of rapid mouse events.

---

### Q26. What is `requestUpdateLane(fiber)` responsible for?
- A) Requesting a new lane be created
- B) Determining which lane to assign to a new update based on the current execution context
- C) Releasing a used lane
- D) Validating the lane assignment

**Answer: B**
`requestUpdateLane` checks the current context: inside a transition? Use `TransitionLane`. Inside a discrete event? Use `SyncLane`. Default context? Use `DefaultLane`.

---

### Q27. How many transition lanes does React provide?
- A) 1
- B) 4
- C) 16
- D) 31

**Answer: C**
React provides 16 transition lanes (bits 6-21). Multiple concurrent transitions can use different lanes, allowing React to batch or separate them.

---

### Q28. When two different `startTransition` calls are pending, what happens?
- A) They always share the same lane
- B) They may get different TransitionLane bits, allowing independent rendering
- C) The second one is dropped
- D) They are queued sequentially

**Answer: B**
React assigns transition lanes in a round-robin fashion via `claimNextTransitionLane`. Different transitions may get different lane bits, allowing them to be processed independently.

---

### Q29. What is `markRootUpdated(root, updateLane)`?
- A) Marks the root DOM as updated
- B) Sets `root.pendingLanes |= updateLane` and records the event time
- C) Updates the root fiber's props
- D) Triggers a DOM repaint

**Answer: B**
`markRootUpdated` adds the update's lane to `root.pendingLanes` and stores the event time for expiration calculations.

---

### Q30. What happens when `includesExpiredLane(root, lanes)` returns true?
- A) The lanes are discarded
- B) React uses `workLoopSync` instead of `workLoopConcurrent`, making the render uninterruptible
- C) The lanes get lower priority
- D) React logs a performance warning

**Answer: B**
Expired lanes force synchronous rendering. This ensures starved updates eventually complete, even at the cost of blocking the main thread.

---

### Q31. What is `InputContinuousLane` used for?
- A) Keyboard input
- B) Continuous events like mousemove, scroll, drag, touchmove
- C) Continuous rendering loops
- D) Streaming data input

**Answer: B**
`InputContinuousLane` handles high-frequency events. Multiple `mousemove` updates can be batched on this lane without each one forcing a separate synchronous render.

---

### Q32. What does `removeLanes(set, lane)` compute?
- A) `set - lane`
- B) `set & ~lane` (bitwise AND with complement)
- C) `set | lane`
- D) `set ^ lane`

**Answer: B**
`removeLanes(set, subset) = set & ~subset` — clears the bits in `subset` from `set`. Used to mark lanes as completed after commit.

---

### Q33. What is `root.pendingLanes`?
- A) Lanes that have been committed
- B) Bitmask of all lanes that have pending work across the entire fiber tree
- C) Lanes waiting for user input
- D) Lanes being rendered currently

**Answer: B**
`pendingLanes` tracks which lanes have unprocessed updates. After a render completes and commits, the processed lanes are removed.

---

### Q34. What does `markRootFinished(root, remainingLanes)` do?
- A) Marks the root as unmounted
- B) Clears completed lanes from `pendingLanes` and cleans up related data
- C) Finishes the root's initialization
- D) Commits the root to the DOM

**Answer: B**
After commit, `markRootFinished` computes `noLongerPendingLanes = pendingLanes & ~remainingLanes` and clears event times, expiration times, and other per-lane data for completed lanes.

---

### Q35. Can a single fiber have updates on multiple lanes simultaneously?
- A) No, one fiber can only have one lane
- B) Yes, `fiber.lanes` is a bitmask that can include multiple lane bits
- C) Only if the fiber is a class component
- D) Only during concurrent rendering

**Answer: B**
`fiber.lanes` is a bitmask. A single fiber can have pending updates on different lanes (e.g., a click update on `SyncLane` AND a transition on `TransitionLane`).

---

### Q36. What is the Scheduler's `taskQueue` vs `timerQueue`?
- A) `taskQueue` is for sync tasks, `timerQueue` for async
- B) `taskQueue` for ready tasks (sorted by expiration); `timerQueue` for delayed tasks (sorted by start time)
- C) They are the same queue with different views
- D) `taskQueue` for React tasks, `timerQueue` for user tasks

**Answer: B**
Delayed tasks (with a future `startTime`) go in `timerQueue`. When their start time arrives, they're moved to `taskQueue`. Both are min-heaps.

---

### Q37. What does `Scheduler.unstable_scheduleCallback(priority, callback)` return?
- A) A Promise
- B) A task object (used for cancellation via `cancelCallback`)
- C) A task ID number
- D) Nothing (void)

**Answer: B**
Returns a task object: `{ id, callback, priorityLevel, startTime, expirationTime, sortIndex }`. This object can be passed to `cancelCallback` to cancel the task.

---

### Q38. How does `flushSync` affect lane assignment?
- A) It assigns `DefaultLane`
- B) It forces the update onto `SyncLane` and immediately processes it
- C) It assigns `TransitionLane`
- D) It doesn't affect lanes

**Answer: B**
`flushSync(() => setState(x))` assigns `SyncLane` and calls `performSyncWorkOnRoot` immediately, forcing synchronous render and commit before returning.

---

### Q39. What is `DefaultLane` used for?
- A) The default lane for all updates
- B) Updates from non-React contexts (setTimeout, promises, native event handlers in React 18)
- C) The lowest priority lane
- D) Server-rendered content

**Answer: B**
`DefaultLane` (bit 5) is assigned to updates outside React's event handler context — like `setTimeout`, `fetch().then()`, and native event handlers. It's lower priority than `SyncLane` but higher than transitions.

---

### Q40. What is the relationship between Scheduler priorities and React Lanes?
- A) They are the same system
- B) React maps lanes to Scheduler priorities: SyncLane→Immediate, InputContinuous→UserBlocking, Default/Transition→Normal, Idle→Idle
- C) Lanes replaced the Scheduler entirely
- D) They operate independently with no connection

**Answer: B**
React translates its lane-based priority to Scheduler priority levels when scheduling tasks via `ensureRootIsScheduled`. Higher-priority lanes get higher Scheduler priority.

---

### Q41. What does `getCurrentUpdatePriority()` return?
- A) The current Scheduler priority
- B) The event priority for the currently executing event handler
- C) The highest pending lane
- D) The CPU priority level

**Answer: B**
`getCurrentUpdatePriority` returns the priority associated with the current execution context (e.g., `DiscreteEventPriority` during a click handler), used by `requestUpdateLane`.

---

### Q42. Why doesn't React use `requestIdleCallback` for scheduling?
- A) It doesn't exist in all browsers
- B) It has inconsistent timing, 50ms max budget, and doesn't fire during user interaction
- C) It's too fast
- D) It was deprecated

**Answer: B**
`requestIdleCallback` has a 50ms max timeout, doesn't fire reliably during active user interaction, and has inconsistent behavior across browsers. `MessageChannel` gives React more control.

---

### Q43. What is the `OffscreenLane`?
- A) A lane for components not visible on screen
- B) A lane for the Activity API (formerly Offscreen), lower priority than idle
- C) A lane for server-rendered content
- D) A lane for error recovery

**Answer: B**
`OffscreenLane` (bit 30) is the lowest-priority lane, used for pre-rendering hidden content via the Activity API.

---

### Q44. What does `isSubsetOfLanes(set, subset)` check?
- A) If `set` contains all bits in `subset`
- B) If `set` is smaller than `subset`
- C) If they have any bits in common
- D) If `set` is a proper subset

**Answer: A**
`isSubsetOfLanes(set, subset) = (set & subset) === subset` — checks that every bit in `subset` is also set in `set`.

---

### Q45. What is `RetryLane` used for?
- A) Retrying failed network requests
- B) Retrying Suspense boundaries after their promises resolve
- C) Retrying failed renders
- D) Retrying event handlers

**Answer: B**
When a Suspense boundary's promise resolves, the retry is scheduled on `RetryLane`. This allows React to batch multiple Suspense retries together.

---

### Q46. What happens in the Scheduler when a higher-priority task is added while a lower-priority task is running?
- A) The lower-priority task is immediately killed
- B) After the current work-loop yield point, the higher-priority task runs first
- C) Both run simultaneously
- D) The higher-priority task waits

**Answer: B**
The Scheduler can only preempt at yield points. When `shouldYield()` returns true and the work loop yields, the Scheduler picks the highest-priority task from the heap for the next iteration.

---

### Q47. What does `DiscreteEventPriority` map to in lanes?
- A) `DefaultLane`
- B) `SyncLane`
- C) `TransitionLane`
- D) `InputContinuousLane`

**Answer: B**
Discrete events (click, keydown, input) map to `SyncLane` — the highest priority. These updates render synchronously for immediate responsiveness.

---

### Q48. What is `ContinuousEventPriority`?
- A) Priority for events that fire once
- B) Priority for events that fire rapidly and continuously (mousemove, scroll, drag)
- C) Priority for continuous rendering
- D) Priority for streaming data

**Answer: B**
Continuous events fire at high frequency. Their updates get `InputContinuousLane` — high enough for responsiveness but allowing batching of rapid-fire updates.

---

### Q49. How does the Scheduler handle a task that exceeds its time slice?
- A) Kills the task
- B) The task returns a continuation function; the Scheduler requeues it
- C) Increases the time slice
- D) Moves it to a lower priority

**Answer: B**
When `shouldYield()` returns true, the React work loop stops and returns the callback function. The Scheduler sees the returned function and keeps the task in the queue to continue later.

---

### Q50. What is the purpose of `root.callbackPriority`?
- A) The priority of the root component
- B) The Scheduler priority of the currently scheduled render task, used to avoid redundant scheduling
- C) The callback function's priority in the event queue
- D) The rendering quality level

**Answer: B**
`ensureRootIsScheduled` compares the new lanes' priority with `callbackPriority`. If they match, no re-scheduling is needed. If the new priority is higher, the old task is cancelled and a new one scheduled.
