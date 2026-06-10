# React Native Architecture Masterclass
## PHASE 13 & 14: Debugging Architecture & Interview Mastery + Complete Architecture Map

---

## PHASE 13: Debugging Architecture

### 13.1 Common Bugs by Layer

**LAYER 1: JavaScript / React**

**Bug: Component Not Re-rendering**

Symptom:
```
State changes, but UI doesn't update
```

Root Causes:
```
1. Mutating state directly
   ❌ state.count = 5  (doesn't trigger re-render)
   ✅ setState(5)      (triggers re-render)

2. Missing dependencies in useEffect
   ❌ useEffect(() => { ... }, [])
   ✅ useEffect(() => { ... }, [dep])

3. Object reference not changing
   ❌ setUser(user)     (same object)
   ✅ setUser({...user}) (new object)

4. Key issue in lists
   ❌ No keys in FlatList
   ✅ Unique keys per item
```

Debugging:
```
1. Add console.log in render function
   function MyComponent() {
     console.log('Rendering');  // Should log when re-rendering
     return <Text>Hello</Text>;
   }

2. Check state in React DevTools
   ├─ Install React DevTools
   ├─ Inspect component
   ├─ Watch state changes

3. Use React Profiler
   <Profiler id="MyComponent" onRender={onRenderCallback}>
     <MyComponent />
   </Profiler>

4. Test with manual setState
   // Add button to test
   <Button onPress={() => setState(prev => !prev)} />
```

---

**Bug: Memory Leak**

Symptom:
```
App gets slower over time
Memory usage keeps growing
Eventually crashes
```

Root Causes:
```
1. Uncleared intervals/timeouts
   setInterval(() => {
     // This never stops!
   }, 1000);

2. Unsubscribed event listeners
   window.addEventListener('scroll', listener);
   // Never removeEventListener

3. Uncleaned useEffect
   useEffect(() => {
     subscription.subscribe();
     // Missing cleanup!
   });

4. Circular references in state
   const obj = { name: 'John' };
   obj.self = obj;
   setState(obj); // Can't be garbage collected

5. Large cached objects
   const cache = {};
   data.forEach(item => {
     cache[item.id] = largeData; // Grows forever
   });
```

Debugging:
```
1. Use React DevTools Profiler
   ├─ Record component mount/unmount
   ├─ Check for leaks (component not unmounting)
   └─ Identify expensive renders

2. Monitor memory with Android Profiler
   ├─ Open Android Studio Profiler
   ├─ Record memory allocation
   ├─ Identify growing objects
   └─ Find which component holding references

3. Check for missing cleanup
   function MyComponent() {
     useEffect(() => {
       const subscription = subscribe();
       
       // MUST have cleanup
       return () => {
         subscription.unsubscribe();
       };
     }, []);
   }

4. Use Chrome DevTools for JS memory
   ├─ Take heap snapshot
   ├─ Compare snapshots
   ├─ Find retained objects
```

---

**LAYER 2: React Native Bridge/JSI**

**Bug: JSI Type Mismatch**

Symptom:
```
Native crash when calling from JavaScript
"Type mismatch: expected number, got string"
```

Root Causes:
```
1. Passing wrong type
   ❌ NativeModule.add('5', 3)     (string instead of number)
   ✅ NativeModule.add(5, 3)       (correct type)

2. Undefined/null values
   ❌ NativeModule.save(undefined)
   ✅ NativeModule.save(null)

3. Non-serializable objects (Bridge)
   ❌ NativeModule.save({ fn: () => {} })  (function not serializable)
   ✅ NativeModule.save({ value: 5 })

4. Promise not handled
   ❌ NativeModule.fetch().then(...)  (missing .catch)
   ✅ NativeModule.fetch()
       .then(...)
       .catch(error => console.error(error))
```

Debugging:
```
1. Check native error logs
   iOS:
     ├─ Xcode console
     ├─ Product → Scheme → Edit Scheme → Diagnostics
     └─ Enable API Validation

   Android:
     ├─ Android Studio Logcat
     ├─ Filter by error
     └─ Check stack trace

2. Add type validation
   if (typeof value !== 'number') {
     throw new Error('Expected number');
   }
   NativeModule.process(value);

3. Test native function directly
   // In native code
   RCTMathModule.add(5, 3);  // Should work
   
   // From JavaScript
   NativeModules.MathModule.add(5, 3).then(...)

4. Check TypeScript specs
   export interface Spec extends TurboModule {
     add(a: number, b: number): number;
   }
   // Codegen will enforce types
```

---

**Bug: Race Condition in Native Code**

Symptom:
```
Inconsistent results when operations overlap
Sometimes works, sometimes doesn't
Hard to reproduce
```

Root Causes:
```
1. Shared mutable state
   let result = 0;
   
   async function operation() {
     result = await fetch();  // Two ops might overwrite each other
   }

2. No synchronization
   Two native methods access same resource
   No locks → data corruption

3. Callback timing issues
   JS calls native
   Native is slow
   JS calls again before first completes
   Both callbacks might overwrite state
```

Debugging:
```
1. Add logging with timestamps
   function operation(id) {
     console.log(`[${Date.now()}] Start ${id}`);
     nativeCall().then(() => {
       console.log(`[${Date.now()}] End ${id}`);
     });
   }

2. Use Promise-based calls (not callbacks)
   // Better than callbacks
   const result = await NativeModule.operation();

3. Add request IDs
   // Track which request is which
   const requestId = generateId();
   NativeModule.fetch(requestId).then(result => {
     if (requestId === currentRequestId) {
       // This is still valid
       updateUI(result);
     }
     // Else: ignore (outdated request)
   });
```

---

**LAYER 3: Native Code (iOS/Android)**

**Bug: View Not Updating**

Symptom:
```
JavaScript sends update, native view doesn't change
```

Root Causes:
```
1. Property not set correctly
   ❌ view.text = "Hello"  (might be read-only)
   ✅ view.setText("Hello")

2. View not invalidated
   view.text = "Hello";
   // Missing: view.requestLayout() or invalidate()

3. Wrong view type
   Expected UILabel, got UIButton
   Properties don't match

4. Layout not applied
   View has new size but position not updated
```

Debugging:
```
iOS:
  1. Use View Debugger
     ├─ Run app
     ├─ Debug → View Hierarchy
     ├─ Inspect view properties
     └─ Check frame/bounds

  2. Add logging
     override var text: String {
       didSet {
         print("Text changed to: \(text)")
         setNeedsDisplay()  // Redraw
       }
     }

Android:
  1. Use Android Layout Inspector
     ├─ Tools → Layout Inspector
     ├─ Select running app
     ├─ Inspect view hierarchy
     └─ Check properties

  2. Add logging
     fun setText(text: String) {
       Log.d("MyView", "Text changed to: $text")
       invalidate()  // Redraw
     }
```

---

**Bug: Native Crash**

Symptom:
```
App suddenly crashes
No error message from JavaScript
```

Root Causes:
```
1. Null pointer dereference
   NSLog(@"%@", nullObject); // Crash in Objective-C
   nullObject.property;       // Crash in Swift

2. Out of bounds array access
   array[100] when array.size = 10

3. Memory already freed
   Accessing freed memory (use-after-free)

4. Stack overflow
   Too many recursive calls without base case

5. Uncaught exception
   Thrown but not caught
```

Debugging:
```
iOS:
  1. Enable breakpoints on exceptions
     Debug → Breakpoints → Exception Breakpoint

  2. Check console output
     XCTest output shows line number

  3. Use LLDB debugger
     (lldb) po object  // Print object
     (lldb) p variable // Print variable

Android:
  1. Check Logcat (errors and crashes)
     Android Studio → Logcat
     Filter: "FATAL" or "Exception"

  2. Use debugger
     Debug → Debug app
     Set breakpoints in native code

  3. Check ANRs (Application Not Responding)
     Logcat shows ANR details
```

---

### 13.2 Performance Debugging

**Identify Slow Renders:**

```javascript
// Render performance
function MyComponent() {
  console.time('render');
  
  // Component logic
  const result = expensiveCalculation();
  
  console.timeEnd('render');
  
  return <Text>{result}</Text>;
}

// Output:
// render: 250ms
```

**Identify Memory Issues:**

```
Android:
  1. Use Profiler
     ├─ Tools → Profiler
     ├─ Record memory allocation
     ├─ Look for memory spikes
     └─ Identify leak sources

  2. Take heap dumps
     ├─ Memory tab → Dump heap
     ├─ Analyze dump
     ├─ Find large objects
     └─ Check retained references

iOS:
  1. Use Instruments
     ├─ Product → Profile (Cmd+I)
     ├─ Choose "Allocations"
     ├─ Record memory usage
     └─ Find growing allocations

  2. Check Leaks instrument
     ├─ Product → Profile
     ├─ Choose "Leaks"
     ├─ Record and look for red X marks
     └─ Click on leak to see stack trace
```

**Identify Thread Issues:**

```
Android:
  1. Use Profiler → CPU
     ├─ See thread activity
     ├─ Identify blocked threads
     └─ Find heavy computation

iOS:
  1. Use System Trace
     ├─ Product → Profile
     ├─ Choose "System Trace"
     ├─ See all threads
     └─ Identify context switches
```

---

## PHASE 14: Interview Mastery

### 14.1 Interview-Level Explanations by Complexity

**TOPIC: React Component Lifecycle**

**Beginner Explanation:**
```
Components have a lifecycle - they are created, updated, and removed.
When a component is created, we can run setup code.
When props change, we can run update code.
When a component is removed, we can run cleanup code.

Example:
  - Created: Set up event listeners
  - Updated: Fetch new data when props change
  - Removed: Remove event listeners
```

**Intermediate Explanation:**
```
React uses hooks (functional components) for lifecycle:

useEffect(() => {
  // Called after every render
  
  return () => {
    // Cleanup (called on unmount)
  };
}, [dependencies]);

Dependency array controls when effect runs:
  - No dependency: after every render
  - Empty []: once on mount
  - [count]: when count changes

Old class components had:
  - componentDidMount() → useEffect(..., [])
  - componentDidUpdate() → useEffect(..., [deps])
  - componentWillUnmount() → cleanup in useEffect
```

**Senior Engineer Explanation:**
```
React's lifecycle is tied to fiber reconciliation:

1. MOUNT phase (entering component tree):
   - Component instance created
   - Initial render called
   - useEffect with no/empty deps scheduled in microtask queue
   - Component added to DOM

2. UPDATE phase (props/state change):
   - Fiber marked as pending
   - New render generated
   - Reconciliation determines changes
   - useEffect with deps checks dependency array
   - Only runs if deps changed (shallow comparison)
   - Cleanup functions run BEFORE new effect (for same effect)

3. UNMOUNT phase (leaving component tree):
   - Component removed from fiber tree
   - Cleanup functions from ALL useEffects run
   - Event listeners removed
   - Timers cleared
   - Subscriptions unsubscribed

Gotchas:
  - useEffect cleanup runs asynchronously (in microtask)
  - useLayoutEffect runs synchronously (before paint)
  - Dependency array uses Object.is() comparison
  - If dependency is object, even same values trigger re-run
  - Memory leaks happen if cleanup not provided
  - Race conditions if async operations don't track request ID

Performance implications:
  - Each dependency change = effect re-run = potential re-render
  - Memoize dependencies to prevent unnecessary runs
  - Be careful with state in dependencies (stale closures)
```

---

**TOPIC: The Bridge vs JSI**

**Beginner Explanation:**
```
The Bridge is how JavaScript talks to native code.

Without Bridge:
  JavaScript can't access camera, GPS, storage (native features)

With Bridge:
  JavaScript sends message → Native receives → Does work → Sends back result

Example:
  JavaScript: "Please take a photo"
  Bridge: Carries message
  Native: Takes photo
  Bridge: Brings photo back
  JavaScript: Shows photo
```

**Intermediate Explanation:**
```
Bridge uses message passing and serialization:

JavaScript:                 Native:
  Camera.take()    -------->  Receive message
    ↓                         ↓
    Serialize args            Execute camera
    to JSON                   ↓
    ↓                         Serialize result
    Send message              ↓
    ↓<---------- Send response
    Deserialize
    ↓
    Return promise

Key concepts:
  - Data must be JSON-serializable (no functions)
  - Messages are queued (not instant)
  - Asynchronous (promise-based)
  - Works with any JS engine (V8, Hermes, JSC)

Bottlenecks:
  - Serialization time (convert objects to JSON)
  - Message passing delay
  - No direct object access
```

**Senior Engineer Explanation:**
```
Bridge vs JSI architectural differences:

BRIDGE (Old Architecture):
  Data Flow:
    JS Object → JSON.stringify → String → Send → Deserialize → Native Object
  
  Characteristics:
    - Async-only (no blocking calls)
    - Type erasure (all data becomes strings)
    - Buffered (messages queued in BatchedBridge)
    - Batched updates (multiple calls sent together)
    - Latency: 5-20ms per round-trip
    
  Limitations:
    - Serialization overhead (CPU)
    - Batching delay (for throughput)
    - Thread context switching overhead
    - Can't access native objects directly
    - Limited to JSON-serializable types
    
  Bottleneck analysis:
    - Single bridge queue (all modules share)
    - Serialization scales with data size
    - Large objects: 1KB = 1ms overhead

JSI (New Architecture):
  Data Flow:
    JS Object → C++ Binding → Native Object
    (Direct memory access, no conversion)
  
  Characteristics:
    - Sync + Async (can do both)
    - Direct object access
    - No serialization
    - Each module has JSI binding
    - Latency: <1ms
    - Can access C++ objects from JS
    
  Performance:
    - 10-100x faster than Bridge
    - No serialization overhead
    - Lower latency
    - Can hold references to native objects

Implementation:
  - JSI = JavaScript Interface (C++ header only)
  - Runtime provides methods to:
    ├─ Call C++ from JS
    ├─ Call JS from C++
    └─ Exchange objects
  
  - Binding layer (auto-generated by codegen):
    ├─ Type checking
    ├─ Marshalling (converting types)
    └─ Error handling

Memory model:
  - Bridge: Objects duplicated (JS heap + native heap)
  - JSI: Objects can be shared (same memory location)
  - Reduces memory usage

Type safety:
  - Bridge: Runtime errors (wrong type passed)
  - JSI: Compile-time checks (TypeScript spec)
  - Codegen validates types

Thread safety:
  - Bridge: Messages serialized (safe)
  - JSI: Direct access (must synchronize manually)
  - Use DispatchQueue (iOS) or Handler (Android)

Migration implications:
  - Existing Bridge code still works
  - New modules use JSI by default
  - Eventually Bridge deprecated
  - Bridgeless mode removes Bridge entirely
```

---

**TOPIC: React Fiber**

**Beginner Explanation:**
```
Fiber is how React organizes its work.

Old way:
  React must finish ALL work before anything else can happen
  
New way (Fiber):
  React can pause work, let browser do other things, then resume

Benefit:
  App stays responsive while React is thinking
```

**Intermediate Explanation:**
```
Fiber = Unit of work (one component)

React Fiber Process:
  1. Create work queue (all components that need updating)
  2. Process work units one-by-one:
     ├─ Render component
     ├─ Check: Should I keep working or pause?
     ├─ If browser needs to paint: PAUSE
     ├─ Browser paints frame
     └─ Resume work
  3. When all work done: Commit changes to DOM

Code Example:
  // This gets paused/resumed:
  beginWork(fiber) {
    fiber.render();  // Run component
    return fiber.child || fiber.sibling;
  }
  
  completeWork(fiber) {
    reconcile(oldFiber, newFiber);  // Check changes
  }

Benefits:
  - UI stays responsive
  - Animations smooth
  - Can prioritize (high priority = less pausing)
```

**Senior Engineer Explanation:**
```
Fiber Architecture Deep Dive:

Data Structure:
  Fiber Node {
    type: function | string
    key: string
    props: object
    state: any
    return: Fiber          // parent
    child: Fiber           // first child
    sibling: Fiber         // next sibling
    alternate: Fiber       // old version
    memoizedState: any[]   // hooks storage
    pendingProps: object
    memoizedProps: object
    updateQueue: Update[]
    lanes: LanePriority
    childLanes: LanePriority
    flags: WorkTag         // Placement, Update, Deletion
    nextEffect: Fiber      // effect linked list
  }

Linked List Structure:
  Unlike regular tree (node.children = []), 
  Fiber uses (node.child, node.sibling) = O(n) traversal
  
  Benefits:
    - Can pause and resume without recursion
    - No call stack overflow with deep trees
    - Can iterate manually

Work Loop (Simplified):
  function workLoop(deadline) {
    while (nextUnitOfWork && shouldYield(deadline)) {
      nextUnitOfWork = performUnitOfWork(nextUnitOfWork);
    }
    
    if (nextUnitOfWork) {
      // More work, schedule next tick
      scheduleCallback(workLoop);
    } else {
      // All render done, start commit
      commitRoot(workInProgressRoot);
    }
  }

Phases:
  
  RENDER PHASE (can pause):
    - beginWork: process component
    - completeWork: reconcile with old
    - Can pause anytime
    - Can be aborted
    - Can run multiple times for same work
    
    Flow:
      beginWork(App)
        ├─ render() → new JSX
        └─ completeWork:
            ├─ compare old vs new
            ├─ mark changes
            └─ create effect list
      
      beginWork(Header)  [child]
        ├─ render() → new JSX
        └─ completeWork...
      
      [Pause for browser frame]
      
      beginWork(Footer)  [sibling]
        ├─ render() → new JSX
        └─ completeWork...
      
      [All render done]
  
  COMMIT PHASE (cannot pause):
    - Apply all changes immediately
    - Run lifecycle methods
    - Run effects
    - Must complete synchronously
    
    3 Sub-phases:
      1. beforeMutation:
         - Detect DOM mutations
         - Run layout effect cleanup
      
      2. mutation:
         - Create/update/delete native views
         - Update DOM
         - Call imperative methods
      
      3. afterMutation:
         - Run layout effects
         - Schedule passive effects
         - Run other effects

Reconciliation Algorithm:
  
  Rules:
    1. Different type → Replace entire subtree
    2. Same type, different props → Update
    3. Same type, same key → Reuse
    4. Different position, same key → Move
  
  Optimization:
    - Siblings compared linearly
    - First mismatch stops searching
    - Keys help identify moved elements

Example:
  Old: [<Item key="a" />, <Item key="b" />, <Item key="c" />]
  New: [<Item key="c" />, <Item key="a" />, <Item key="b" />]
  
  Without keys:
    - Item 1: update content (a → c)
    - Item 2: update content (b → a)
    - Item 3: update content (c → b)
    Total: 3 updates + potential DOM thrashing
  
  With keys:
    - Item c: move position
    - Item a: move position
    - Item b: move position
    Total: 3 moves (optimal)

Priority Lanes:
  
  Different updates have different priorities:
    - Synchronous: user input, controlled inputs
    - Default: most setState calls
    - Transition: less important updates
    - Deferred: lowest priority
  
  High priority can interrupt low priority:
    - While rendering low priority
    - High priority event occurs
    - Pause low priority rendering
    - Process high priority first
    - Resume low priority later (or discard)

Performance:
  - Fiber overhead: ~1-2% slower than no fiber
  - But enables responsiveness: ~10-100x perceived faster
  - Good for interactive apps
  - Not needed for static content

Debugging:
  - React DevTools shows fiber tree
  - Can inspect each fiber node
  - See which components re-rendered
  - See effect timings
```

---

**TOPIC: setState() Complete Flow**

**Beginner Explanation:**
```
When you call setState(), React doesn't update immediately.

What happens:
  1. You call setState(newValue)
  2. React remembers the change
  3. React schedules an update
  4. When browser is free, React updates
  5. Component re-renders
  6. UI shows new value

Why not immediately?
  - If you setState 10 times, update 10 times = slow
  - React batches them = update once = fast
```

**Intermediate Explanation:**
```
setState() flow with React:

1. setState(newValue) called
   ├─ Create update object: { action: newValue }
   ├─ Add to fiber's updateQueue
   ├─ Schedule work (requestIdleCallback)
   └─ Return immediately (async)

2. React schedules render:
   ├─ Check browser idle
   ├─ Start render phase
   ├─ Process fiber tree
   ├─ Recalculate state from updateQueue
   ├─ Re-render components
   └─ Mark changes

3. React commits changes:
   ├─ Create native views
   ├─ Update existing views
   ├─ Run useEffect
   └─ Display changes

4. Browser paints
   └─ User sees new UI

Batching:
  setState(1);
  setState(2);
  setState(3);
  
  All three are batched:
    ├─ Schedule render once
    ├─ Render with all three updates
    └─ Commit once
  
  Not three separate renders!
```

**Senior Engineer Explanation:**
```
setState() Complete Implementation:

Dispatch:
  function setState(action) {
    const fiber = currentFiber;
    
    // 1. Determine priority
    const lanes = getCurrentUpdatePriority();
    
    // 2. Create update object
    const update = {
      action,           // new state or function
      eagerReducer,     // try to calculate now
      hasEagerState: false,
      eagerState: null,
      next: null,
    };
    
    // 3. Eager state calculation (if possible)
    if (typeof action === 'function') {
      const currentState = fiber.memoizedState;
      update.eagerState = action(currentState);
    } else {
      update.eagerState = action;
    }
    
    // 4. Enqueue update
    fiber.updateQueue.pending = update;
    
    // 5. Schedule work
    if (isRenderingPhase) {
      // During render, just enqueue
    } else {
      // Not rendering, schedule work
      scheduleCallback(renderUpdate, { lane: lanes });
    }
  }

Render Phase (Process Updates):
  function processUpdateQueue(fiber, updateQueue) {
    let state = fiber.memoizedState;
    let update = updateQueue.firstUpdate;
    
    while (update) {
      // Determine if should process based on priority
      if (shouldProcessUpdate(update.lane)) {
        // Calculate new state
        if (typeof update.action === 'function') {
          state = update.action(state);
        } else {
          state = update.action;
        }
      }
      
      update = update.next;
    }
    
    fiber.memoizedState = state;
    return state;
  }

Batching Mechanism:
  // All updates within event go to same updateQueue
  
  onClick={() => {
    setState(1);  // Update 1 added
    setState(2);  // Update 2 added
    setState(3);  // Update 3 added
  }}
  
  // All 3 process in single render:
  // state = 1
  // state = 2
  // state = 3
  // Final: state = 3

State Updates:
  // Previous state: { count: 0 }
  // Update 1: setCount(c => c + 1)
  // Update 2: setCount(c => c + 1)
  
  Processing:
    state = { count: 0 }
    state = { count: 1 }  // After update 1
    state = { count: 2 }  // After update 2
    Final: { count: 2 }

Commit Phase:
  1. Before mutation (useLayoutEffect cleanup)
  2. Mutation (actually apply state to native views)
  3. After mutation (useEffect)
  4. Passive effects (scheduled asynchronously)

Timing:
  setState() → Schedule (microtask) → Render → Commit → Paint → useEffect
  
  So:
    console.log('1');
    setState(1);
    console.log('2');  // Still see 2 (state not updated yet)
    
  Output: 1, 2

Priorities:
  - Synchronous (user input): 1
  - Default (setState): 8
  - Transition: 32
  - Deferred: 128
  
  Higher priority interrupts lower:
    - Rendering deferred update (128)
    - User clicks button (1)
    - Pause deferred, process click (1)
    - Resume deferred after (128)

Concurrent Features:
  useTransition:
    const [isPending, startTransition] = useTransition();
    
    startTransition(() => {
      setState(value);  // Rendered with Transition priority
    });
    
    If user input during render:
      ├─ Pause transition render
      ├─ Process input
      └─ Resume transition
    
    Result:
      - UI responsive
      - Update completes eventually
```

---

### 14.2 System Design Interview Patterns

**Pattern 1: Build a Native Module**

Question:
```
Design a File Upload module for React Native.
How would you implement it?
```

Answer Framework:
```
1. REQUIREMENTS
   ├─ Upload file to server
   ├─ Progress callback
   ├─ Error handling
   ├─ Cancel support
   └─ Background support (iOS/Android)

2. ARCHITECTURE
   TypeScript Spec:
     export interface FileUploadModule extends TurboModule {
       upload(filePath: string, options: UploadOptions): Promise<UploadResult>;
     }
   
   iOS Implementation:
     - Use URLSession for upload
     - Implement delegates for progress
     - Handle background app refresh
   
   Android Implementation:
     - Use OkHttp or similar
     - Implement progress callbacks
     - Handle background limitations

3. THREADING
   JS: Schedule upload
   JS Thread: Idle (not blocked)
   Native: Perform upload in background thread
   Native: Call JS callback with progress (via JSI)
   JS: Update UI based on progress

4. DATA FLOW
   JS: {
     filePath: '/path/to/file',
     url: 'https://api.example.com/upload',
     headers: { ... }
   } → JSI binding
   
   Native: {
     Create URLRequest
     Attach file stream
     Start upload
     Monitor progress
   } → Callback to JS
   
   JS: { progress: 0.5 } ← Update UI

5. ERROR HANDLING
   Network error: Retry logic
   File not found: Validation before upload
   Timeout: Configurable timeout
   Cancelled: Clean up resources

6. PERFORMANCE
   - Stream file (don't load entire file to memory)
   - Batch callbacks (not every byte)
   - Cancel mid-upload (free resources)
   - Background task (doesn't freeze UI)
```

---

**Pattern 2: Performance Optimization Interview**

Question:
```
Your FlatList with 10,000 items is slow. Optimize it.
```

Answer Framework:
```
1. IDENTIFY BOTTLENECK
   Profile the app:
     - Is JS slow (render)?
     - Is native slow (view creation)?
     - Is scroll stuttering?
   
   Use React DevTools Profiler:
     - Record render times
     - Identify slow components
     - Find unnecessary re-renders

2. OPTIMIZE RENDER
   Problem: renderItem creates new objects every render
   Solution:
     const renderItem = useCallback(({ item }) => {
       return <Item data={item} />;
     }, []);
     
     <FlatList renderItem={renderItem} />

3. OPTIMIZE ITEM COMPONENT
   Problem: Item re-renders unnecessarily
   Solution:
     const Item = React.memo(({ data }) => {
       return <View>...</View>;
     }, (prevProps, nextProps) => {
       return prevProps.data.id === nextProps.data.id;
     });

4. OPTIMIZE DATA STRUCTURE
   Problem: Large objects per item
   Solution:
     - Only pass necessary data to Item
     - Keep large data separate
     - Lazy load images

5. OPTIMIZE FLATLIST SETTINGS
   <FlatList
     initialNumToRender={10}          // Don't render all initially
     maxToRenderPerBatch={10}         // Render in batches
     updateCellsBatchingPeriod={50}   // Delay between batches
     removeClippedSubviews={true}     // Remove off-screen views
     windowSize={10}                  // Items to keep in memory
     getItemLayout={...}              // Skip layout calculation
   />

6. MEASURE IMPROVEMENT
   Before: 30 FPS (stuttering)
   After: 60 FPS (smooth)
   
   Measure with:
     - React Profiler
     - Android Profiler
     - Xcode Instruments
     - Frame rate counter
```

---

## COMPLETE REACT NATIVE ARCHITECTURE MAP

**Every layer from JavaScript to Pixels:**

```
┌────────────────────────────────────────────────────────────────────┐
│ 1. USER INTERACTION (Hardware Level)                               │
│    User finger touches screen                                      │
│    └─ Touch sensor sends signal to OS                              │
└────────────────────────────────────────────────────────────────────┘
                              ↓
┌────────────────────────────────────────────────────────────────────┐
│ 2. OPERATING SYSTEM (iOS / Android)                                │
│    OS detects touch event                                          │
│    └─ Creates touch event with coordinates                         │
│       └─ Sends to app process                                      │
└────────────────────────────────────────────────────────────────────┘
                              ↓
┌────────────────────────────────────────────────────────────────────┐
│ 3. NATIVE LAYER (C++/Swift/Kotlin)                                │
│    Main/UI Thread                                                  │
│    ├─ Native view hierarchy processes touch                        │
│    ├─ Hit testing (which view was touched)                         │
│    ├─ Call onPress callback                                        │
│    └─ JSI binding: Call JavaScript with event                      │
└────────────────────────────────────────────────────────────────────┘
                              ↓
┌────────────────────────────────────────────────────────────────────┐
│ 4. JAVASCRIPT THREAD (Hermes / V8 / JSC)                           │
│    ├─ onPress callback executed                                    │
│    ├─ JavaScript runs: setCount(count + 1)                         │
│    ├─ useState hook reducer updates state                          │
│    ├─ Component marked as needing update                           │
│    └─ scheduleWork() called → Work added to scheduler queue        │
└────────────────────────────────────────────────────────────────────┘
                              ↓
┌────────────────────────────────────────────────────────────────────┐
│ 5. REACT FIBER (Render Phase)                                      │
│    JS Thread - Can Pause & Resume                                  │
│                                                                     │
│    Event Loop:                                                     │
│    ├─ Check microtask queue (promises)                             │
│    ├─ Schedule render work                                         │
│    └─ requestIdleCallback fires when browser idle                  │
│                                                                     │
│    Render Phase:                                                   │
│    ├─ workLoop() processes fiber tree                              │
│    ├─ beginWork: Call component render function                    │
│    │   └─ MyComponent() returns new JSX                            │
│    ├─ completeWork: Compare old vs new JSX                         │
│    │   └─ Determine: Update, Create, Delete                       │
│    ├─ Create new fiber tree (alternate)                            │
│    ├─ Mark changes with flags                                      │
│    ├─ Build effect list (useEffect callbacks)                      │
│    └─ [Check: Browser needs frame? Pause rendering]                │
│                                                                     │
│    Yield Points:                                                   │
│    ├─ After each fiber processed                                   │
│    ├─ Check deadline (time remaining?)                             │
│    ├─ If < 5ms left: Pause                                         │
│    └─ Schedule next chunk                                          │
└────────────────────────────────────────────────────────────────────┘
                              ↓
┌────────────────────────────────────────────────────────────────────┐
│ 6. RECONCILIATION (Diffing Algorithm)                              │
│                                                                     │
│    Old Fiber Tree:              New Fiber Tree:                    │
│      Counter (count: 0)            Counter (count: 1)              │
│        └─ Text: "0"                └─ Text: "1"                    │
│                                                                     │
│    Diff:                                                           │
│    ├─ Counter: Same type                                           │
│    │   └─ Props changed? No → SKIP                                 │
│    ├─ Text: Same type                                              │
│    │   └─ Content changed: "0" → "1" → MARK UPDATE                │
│                                                                     │
│    Result:                                                         │
│    ├─ Counter: No changes                                          │
│    └─ Text: Update content (flag: "Update")                        │
└────────────────────────────────────────────────────────────────────┘
                              ↓
┌────────────────────────────────────────────────────────────────────┐
│ 7. FABRIC SHADOW TREE (C++)                                        │
│                                                                     │
│    React passes new fiber tree to Fabric                           │
│    └─ ShadowNode tree created (C++ representation)                 │
│       ├─ ViewShadowNode                                            │
│       ├─ TextShadowNode                                            │
│       └─ Each has props, layout info, children                     │
└────────────────────────────────────────────────────────────────────┘
                              ↓
┌────────────────────────────────────────────────────────────────────┐
│ 8. YOGA LAYOUT ENGINE (C++)                                        │
│                                                                     │
│    Input: Flexbox properties                                       │
│    ├─ flex, flexDirection, padding, margin, etc.                   │
│                                                                     │
│    Processing (Flexbox Algorithm):                                 │
│    ├─ Root: width 100%, height 100%                                │
│    │   └─ Calculate: (0, 0, 375, 667) on iPhone                    │
│    │                                                                │
│    ├─ Text child: flex: 1                                          │
│    │   └─ Calculate: (0, 0, 375, 667)                              │
│    │                                                                │
│    └─ Measure text "1"                                             │
│        └─ Font metrics: width 10px, height 20px                    │
│                                                                     │
│    Output: ShadowTree with layout                                  │
│    └─ Root: { x: 0, y: 0, width: 375, height: 667 }               │
│    └─ Text: { x: 0, y: 0, width: 375, height: 667 }               │
└────────────────────────────────────────────────────────────────────┘
                              ↓
┌────────────────────────────────────────────────────────────────────┐
│ 9. REACT FIBER (Commit Phase)                                      │
│    JS Thread - Cannot Pause                                        │
│                                                                     │
│    Phase 1: Before Mutation                                        │
│    ├─ Check for DOM mutations                                      │
│    ├─ Run useLayoutEffect cleanup (sync)                           │
│                                                                     │
│    Phase 2: Mutation (Apply Changes)                               │
│    ├─ Loop through effect list                                     │
│    ├─ For Text node (marked "Update"):                             │
│    │   ├─ Call UIManager.updateView() via JSI                      │
│    │   ├─ Pass: viewTag, 'text', { text: '1' }                     │
│    │   └─ No serialization (direct C++)                            │
│    └─ Mark old fiber tree as ready for GC                          │
│                                                                     │
│    Phase 3: After Mutation                                         │
│    ├─ Run useLayoutEffect effects (sync)                           │
│    ├─ Schedule useEffect effects (async microtask)                 │
│    └─ Schedule useCallback/useMemo updates                         │
│                                                                     │
│    Commit Complete                                                 │
│    └─ Old fiber tree freed (GC eligible)                           │
└────────────────────────────────────────────────────────────────────┘
                              ↓
┌────────────────────────────────────────────────────────────────────┐
│ 10. NATIVE LAYER - MOUNTING (Main/UI Thread)                       │
│                                                                     │
│     iOS:                                                           │
│     ├─ UIManager receives updateView() call (JSI)                  │
│     ├─ Find UILabel with viewTag                                   │
│     ├─ Update property: uiLabel.text = "1"                         │
│     ├─ Mark view as needing display: setNeedsDisplay()             │
│     └─ (Does NOT render yet)                                       │
│                                                                     │
│     Android:                                                       │
│     ├─ ReactViewManager receives updateView() call                 │
│     ├─ Find TextView with viewTag                                  │
│     ├─ Update property: textView.text = "1"                        │
│     ├─ Invalidate view: invalidate()                               │
│     └─ (Does NOT render yet)                                       │
└────────────────────────────────────────────────────────────────────┘
                              ↓
┌────────────────────────────────────────────────────────────────────┐
│ 11. VSYNC & FRAME CALLBACK                                         │
│                                                                     │
│     VSyncProvider detects VSYNC signal:                            │
│     ├─ 60 FPS: Every 16.67ms                                       │
│     ├─ 120 FPS: Every 8.33ms                                       │
│     └─ (Hardware signal from display)                              │
│                                                                     │
│     Frame callback triggered:                                      │
│     ├─ Measure views (if needed)                                   │
│     ├─ Update layouts (if dirty)                                   │
│     ├─ Prepare rendering instructions                              │
│     └─ Hand off to graphics system                                 │
│                                                                     │
│     Native thread ready to render                                  │
└────────────────────────────────────────────────────────────────────┘
                              ↓
┌────────────────────────────────────────────────────────────────────┐
│ 12. RENDERING (Native Layer)                                       │
│                                                                     │
│     iOS (Core Graphics / Metal):                                   │
│     ├─ Call drawRect() or render pass                              │
│     ├─ Create rendering context                                    │
│     ├─ For Text view:                                              │
│     │   ├─ Allocate bitmap                                         │
│     │   ├─ Rasterize text to bitmap                                │
│     │   │   ├─ Font: System 16pt                                   │
│     │   │   ├─ Character: "1"                                      │
│     │   │   ├─ Layout: (0, 0, 375, 667)                            │
│     │   │   └─ Anti-alias: smooth edges                            │
│     │   └─ Position bitmap at (0, 0)                               │
│     │                                                                │
│     └─ Create rendering commands                                   │
│                                                                     │
│     Android (Skia / OpenGL):                                       │
│     ├─ Call onDraw()                                               │
│     ├─ Create canvas                                               │
│     ├─ For Text view:                                              │
│     │   ├─ Paint text to canvas                                    │
│     │   ├─ Font: System 16pt                                       │
│     │   ├─ Character: "1"                                          │
│     │   ├─ Layout: (0, 0, 375, 667)                                │
│     │   └─ Anti-alias                                              │
│     │                                                                │
│     └─ Convert to display lists                                    │
└────────────────────────────────────────────────────────────────────┘
                              ↓
┌────────────────────────────────────────────────────────────────────┐
│ 13. GPU RENDERING (GPU)                                            │
│                                                                     │
│     GPU receives commands:                                         │
│     ├─ Vertex shader: Position vertices                            │
│     │   └─ Move text geometry to screen position                   │
│     │                                                                │
│     ├─ Fragment shader: Color each pixel                           │
│     │   └─ Apply texture (text bitmap)                             │
│     │   └─ Blend with background                                   │
│     │                                                                │
│     ├─ Texture sampler: Read pixel data                            │
│     ├─ Rasterization: Convert vectors to pixels                    │
│     └─ Output to frame buffer (off-screen)                         │
│                                                                     │
│     Frame buffer contains final pixel data                         │
│     └─ Back buffer ready for display                               │
└────────────────────────────────────────────────────────────────────┘
                              ↓
┌────────────────────────────────────────────────────────────────────┐
│ 14. DISPLAY SCAN-OUT (Display Controller)                          │
│                                                                     │
│     Display controller reads frame buffer                          │
│     ├─ Scans out pixels row by row                                 │
│     ├─ Left to right, top to bottom                                │
│     ├─ 60 times per second (60 FPS) or 120x (120 FPS)             │
│     └─ Sends pixel data to display panel                           │
│                                                                     │
│     Pixel data sent to physical display                            │
│     ├─ Each pixel has RGB values                                   │
│     └─ Example: Text "1" at (0,0) = 8 × 20 pixels                 │
└────────────────────────────────────────────────────────────────────┘
                              ↓
┌────────────────────────────────────────────────────────────────────┐
│ 15. PHYSICAL DISPLAY (LCD / OLED Panel)                            │
│                                                                     │
│     LCD Panel:                                                     │
│     ├─ For each pixel:                                             │
│     │   ├─ Apply voltage to liquid crystals                        │
│     │   ├─ Crystals rotate to let light through                    │
│     │   └─ Color filters (RGB) blend to final color                │
│     │                                                                │
│     └─ Light from backlight passes through                         │
│                                                                     │
│     OLED Panel:                                                    │
│     ├─ For each pixel:                                             │
│     │   ├─ Apply voltage to OLED                                   │
│     │   ├─ OLED emits light directly                               │
│     │   └─ Intensity controls brightness                           │
│     │                                                                │
│     └─ No backlight (more power efficient)                         │
│                                                                     │
│     Light reaches user's eyes                                      │
│     ├─ Eyes see text "1" on screen                                 │
│     ├─ Brain processes visual information                          │
│     └─ User perceives the change                                   │
└────────────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────────────┐
│ TIMING SUMMARY                                                     │
├────────────────────────────────────────────────────────────────────┤
│                                                                     │
│ User touch                            0ms                          │
│   → OS event dispatching              1ms                          │
│   → Native onPress callback          2ms                           │
│   → JavaScript execution              5ms                          │
│   → setState / setCount call          6ms                          │
│   → React Fiber render phase          12ms                         │
│   → React commit phase                14ms                         │
│   → Native view update via JSI         15ms                         │
│   → Layout calculation by Yoga         16ms                         │
│   → VSYNC signal arrives              16.67ms                       │
│   → Native rendering to frame buffer  18ms                         │
│   → GPU rendering                     19ms                         │
│   → Display scan-out begins           20ms                         │
│   → Display shows new frame           20.67ms                      │
│                                                                     │
│ Total latency: ~20-21ms (≈ 1-2 frames at 60 FPS)                  │
│                                                                     │
│ For comparison:                                                    │
│ ├─ Ideal responsive: < 100ms                                       │
│ ├─ Smooth animation: 16.67ms per frame (60 FPS)                   │
│ ├─ Felt immediate: < 50ms                                          │
│ └─ Slow: > 500ms                                                   │
│                                                                     │
└────────────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────────────┐
│ THREAD ALLOCATION                                                  │
├────────────────────────────────────────────────────────────────────┤
│                                                                     │
│ JavaScript Thread:                                                 │
│ ├─ React component rendering                                       │
│ ├─ Fiber reconciliation                                            │
│ ├─ Event handler execution                                         │
│ ├─ useEffect / useMemo / useCallback                               │
│ └─ Business logic                                                  │
│                                                                     │
│ Native Main/UI Thread:                                             │
│ ├─ OS touch event handling                                         │
│ ├─ Native view hierarchy management                                │
│ ├─ Yoga layout engine calculations                                 │
│ ├─ Native rendering to frame buffer                                │
│ ├─ GPU command submission                                          │
│ └─ VSYNC handling                                                  │
│                                                                     │
│ Background Threads:                                                │
│ ├─ Image loading                                                   │
│ ├─ Network requests                                                │
│ ├─ Heavy computation                                               │
│ ├─ File I/O                                                        │
│ └─ Database operations                                             │
│                                                                     │
└────────────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────────────┐
│ MEMORY ALLOCATION                                                  │
├────────────────────────────────────────────────────────────────────┤
│                                                                     │
│ JavaScript Heap:                                                   │
│ ├─ React fiber tree (~5KB per 100 components)                      │
│ ├─ Component state (user defined)                                  │
│ ├─ JSX objects (during render)                                     │
│ ├─ Event handlers                                                  │
│ └─ Event listener closures                                         │
│                                                                     │
│ Native Heap (iOS/Android):                                         │
│ ├─ ShadowNode tree (C++) (~5KB per 100 components)                 │
│ ├─ Native view objects (~10KB each)                                │
│ ├─ Layout cache (Yoga) (~1KB per 100 components)                   │
│ └─ Rendering resources (shaders, textures)                         │
│                                                                     │
│ Image Cache:                                                       │
│ ├─ Downloaded images in memory (~50-500KB each)                    │
│ ├─ Thumbnails (~10-50KB each)                                      │
│ └─ LRU cache (old images removed)                                  │
│                                                                     │
│ GPU Memory:                                                        │
│ ├─ Textures for rendering                                          │
│ ├─ Framebuffer (screen pixels)                                     │
│ ├─ Shaders compiled code                                           │
│ └─ Vertex/index buffers                                            │
│                                                                     │
└────────────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────────────┐
│ PERFORMANCE CRITICAL PATHS                                         │
├────────────────────────────────────────────────────────────────────┤
│                                                                     │
│ Critical: JS Render Phase < 16.67ms (for 60 FPS)                  │
│ ├─ If takes 50ms: Drops 3 frames                                   │
│ └─ Solution: Optimize components, memoize, virtualize             │
│                                                                     │
│ Critical: Native Rendering < 16.67ms (for 60 FPS)                 │
│ ├─ If takes 50ms: Drops 3 frames                                   │
│ └─ Solution: Simplify views, reduce draw calls                    │
│                                                                     │
│ Critical: Yoga Layout < 16.67ms (for 60 FPS)                      │
│ ├─ If takes 50ms: Layout thrashing                                │
│ └─ Solution: Cache layouts, avoid recomputation                   │
│                                                                     │
│ Critical: JSI Calls < 1ms (for responsiveness)                     │
│ ├─ If takes 10ms: Perceivable lag                                  │
│ └─ Solution: Batch calls, avoid large serialization (Bridge)      │
│                                                                     │
└────────────────────────────────────────────────────────────────────┘
```

---

## Final Summary

**You now understand:**

1. **JavaScript Fundamentals** → How code executes, memory management, event loop
2. **Browser vs React Native** → DOM, Virtual DOM, native views
3. **React Fundamentals** → Components, JSX, Props, State, Re-rendering, Reconciliation
4. **React Fiber** → Work units, render/commit phases, scheduling, priorities
5. **React Native Fundamentals** → JavaScript thread, UI thread, bridge, native modules, Yoga
6. **Old Architecture** → Serialization, message passing, bottlenecks, thread communication
7. **New Architecture** → JSI, Fabric, TurboModules, Shadow Tree, Codegen
8. **Hermes** → Bytecode compilation, AOT, startup optimization
9. **Rendering Pipeline** → Complete flow from setState() to pixels on screen
10. **Native Communication** → How JS calls camera, GPS, storage; thread safety
11. **Performance** → Re-renders, memoization, FlatList virtualization, thread blocking
12. **Advanced Topics** → TurboModules, Fabric, Bridgeless, Concurrent, React Compiler
13. **Debugging** → Common bugs, root causes, debugging techniques
14. **Interview Mastery** → Multi-level explanations, system design patterns

**You can now:**
- Explain every layer from JavaScript to pixels on screen
- Debug performance issues
- Understand why things happen (not just what)
- Design new features architecturally
- Ace React Native architecture interviews
- Mentor other engineers
- Make informed optimization decisions

---

## Study Resources

**To go deeper:**
- React source code: github.com/facebook/react
- React Native source: github.com/facebook/react-native
- Hermes: github.com/facebook/hermes
- Metro bundler: github.com/facebook/metro
- Yoga: github.com/facebook/yoga

**Key files to read:**
- React: `packages/react-reconciler/src/ReactFiber*.js`
- React Native: `Libraries/Renderer/implementations/ReactFabric*.js`
- Hermes: `lib/VM/HermesRuntime.cpp`

Good luck! You've got this! 🚀
