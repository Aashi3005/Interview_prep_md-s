# React Native Architecture Masterclass
## PHASE 3 & 4: React Fundamentals & React Fiber

---

## PHASE 3: React Fundamentals

### 3.1 Components

**What is a Component?**

A reusable piece of UI. JavaScript function that returns JSX (which becomes HTML/native views).

```javascript
function Greeting() {
  return <Text>Hello</Text>;
}
```

**Component = Function**

```
Input: Props
  ↓
Processing: Logic
  ↓
Output: JSX
  ↓
React renders to screen
```

**Class Component vs Functional Component:**

```javascript
// CLASS COMPONENT (Old, still works)
class Greeting extends React.Component {
  render() {
    return <Text>Hello</Text>;
  }
}

// FUNCTIONAL COMPONENT (Modern, with hooks)
function Greeting() {
  return <Text>Hello</Text>;
}
```

**Component Lifecycle (Class):**

```
┌───────────────────────────────────┐
│  MOUNTING PHASE                   │
├───────────────────────────────────┤
│  constructor() → render() →       │
│  componentDidMount()              │
└───────────────────────────────────┘
           ↓
┌───────────────────────────────────┐
│  UPDATING PHASE                   │
├───────────────────────────────────┤
│  componentDidUpdate()             │
│  (runs when props/state change)   │
└───────────────────────────────────┘
           ↓
┌───────────────────────────────────┐
│  UNMOUNTING PHASE                 │
├───────────────────────────────────┤
│  componentWillUnmount()           │
│  (cleanup before removal)         │
└───────────────────────────────────┘
```

---

### 3.2 JSX (JavaScript XML)

**What is JSX?**

```javascript
// THIS IS JSX:
const element = <View style={styles.container}>
  <Text>Hello World</Text>
</View>;
```

**JSX is NOT valid JavaScript!**

Browsers don't understand JSX. React compiles it.

**JSX Compilation:**

```javascript
// BEFORE (JSX):
<View style={styles.container}>
  <Text>Hello</Text>
</View>

// AFTER (Compiled to JavaScript):
React.createElement(View, { style: styles.container },
  React.createElement(Text, null, 'Hello')
);
```

**What is React.createElement?**

```javascript
React.createElement(type, props, ...children);

// Returns:
{
  type: View,
  props: {
    style: styles.container,
    children: [
      {
        type: Text,
        props: {
          children: 'Hello'
        }
      }
    ]
  }
}
```

**This is a JavaScript object!** Not a view yet.

```
JSX → Babel Compiler → React.createElement() calls → Objects → React processes → Views rendered
```

**Example with Props:**

```javascript
// JSX:
<Text color="blue" size={20}>Hello</Text>

// Compiled:
React.createElement(Text, {
  color: 'blue',
  size: 20
}, 'Hello');

// Result object:
{
  type: Text,
  props: {
    color: 'blue',
    size: 20,
    children: 'Hello'
  }
}
```

---

### 3.3 Props (Properties)

**What are Props?**

Data passed from parent to child component. **Read-only**.

```javascript
// PARENT
function App() {
  return <Button title="Click Me" onPress={() => {}} />;
}

// CHILD
function Button(props) {
  return (
    <TouchableOpacity onPress={props.onPress}>
      <Text>{props.title}</Text>
    </TouchableOpacity>
  );
}
```

**Props Flow (One-way):**

```
Parent
  ↓
Props passed down
  ↓
Child receives props
  ↓
Child renders based on props
  ↓
If parent changes props
  ↓
Child re-renders with new props
```

**Props are Immutable:**

```javascript
function MyComponent(props) {
  // ❌ WRONG - Cannot modify props
  props.title = 'New Title';
  
  // ✅ RIGHT - Use state instead
  const [title, setTitle] = useState(props.title);
}
```

**Destructuring Props:**

```javascript
// Instead of:
function Button(props) {
  return <Text>{props.title}</Text>;
}

// Use destructuring:
function Button({ title, onPress }) {
  return (
    <TouchableOpacity onPress={onPress}>
      <Text>{title}</Text>
    </TouchableOpacity>
  );
}

// Call:
<Button title="Click" onPress={() => console.log('clicked')} />
```

---

### 3.4 State

**What is State?**

Data that component manages internally. Can change. Causes re-render when changed.

```javascript
function Counter() {
  // State: count
  // Setter: setCount
  const [count, setCount] = useState(0);
  
  return (
    <View>
      <Text>{count}</Text>
      <Button title="+" onPress={() => setCount(count + 1)} />
    </View>
  );
}
```

**Props vs State:**

| Props | State |
|-------|-------|
| Passed from parent | Managed locally |
| Read-only | Can change |
| Child cannot modify | Child can modify via setState |
| New props = re-render | setState = re-render |

**State Update Process:**

```
User clicks button
  ↓
onPress() called
  ↓
setCount(count + 1) called
  ↓
React marks component as "needs update"
  ↓
Component re-renders
  ↓
New JSX created with count = 1
  ↓
Virtual representation updated
  ↓
UI updates
```

**State is NOT Updated Synchronously:**

```javascript
function Counter() {
  const [count, setCount] = useState(0);
  
  const handlePress = () => {
    setCount(count + 1);
    console.log(count); // Still 0! (update scheduled, not done yet)
  };
  
  return <Button onPress={handlePress} />;
}
```

**Why? Batching for performance.**

---

### 3.5 Re-rendering

**What Causes a Re-render?**

1. Parent re-renders (child re-renders too)
2. Component's state changes
3. Component's props change

**Re-rendering Doesn't Always Update UI:**

```
Component re-renders
  ↓
New JSX created
  ↓
React checks: "Did anything actually change?"
  ↓
If YES: Update UI
If NO: Skip UI update (optimization)
```

**Example:**

```javascript
function MyComponent() {
  const [count, setCount] = useState(0);
  
  const handlePress = () => {
    setCount(0); // Updating to SAME value
  };
  
  // React detects no change, skips UI update
  return <Text>Count: {count}</Text>;
}
```

**Performance Problem:**

```javascript
function Parent() {
  const [count, setCount] = useState(0);
  
  return (
    <View>
      <Button onPress={() => setCount(count + 1)} />
      <Child />        {/* Re-renders unnecessarily */}
      <AnotherChild /> {/* Re-renders unnecessarily */}
    </View>
  );
}
```

When count changes:
1. Parent re-renders
2. Parent's children re-render (even if their props didn't change!)
3. Wasted renders = slower app

**Solution: Memoization (later sections)**

---

### 3.6 Reconciliation & Diffing Algorithm

**The Problem:**

When component re-renders, which DOM elements change?

```
Old JSX:
<View>
  <Text>Hello</Text>
  <Button title="Click" />
</View>

New JSX:
<View>
  <Text>Hello World</Text>
  <Button title="Click" />
</View>

Question: Which element changed?
Answer: Only the Text changed. Button is same.
```

**Reconciliation = Process of figuring out what changed**

**Diffing Algorithm:**

React compares old and new tree element by element.

```
OLD TREE:
View
  ├─ Text: "Hello"
  └─ Button

NEW TREE:
View
  ├─ Text: "Hello World"
  └─ Button

DIFF:
View → Same type, no change
  ├─ Text → Type same, but content different → UPDATE
  └─ Button → Same type & props, no change → SKIP
```

**Keys in Lists (Critical):**

```javascript
// ❌ WITHOUT KEYS - Can cause bugs
{items.map(item => (
  <Text>{item.name}</Text>
))}

// ✅ WITH KEYS - Correct
{items.map(item => (
  <Text key={item.id}>{item.name}</Text>
))}
```

**Why Keys Matter:**

```
Data: [{ id: 1, name: 'A' }, { id: 2, name: 'B' }]

Render:
<Text key={1}>A</Text>
<Text key={2}>B</Text>

User adds new item at beginning:
Data: [{ id: 0, name: 'X' }, { id: 1, name: 'A' }, { id: 2, name: 'B' }]

WITHOUT KEY:
1. First Text: 'A' → 'X' (UPDATE)
2. Second Text: 'B' → 'A' (UPDATE)
3. Add new Text: 'B' (CREATE)
Result: 3 updates instead of 1 insert!

WITH KEY:
1. Text with key=0: CREATE 'X'
2. Text with key=1: KEEP 'A' (no change)
3. Text with key=2: KEEP 'B' (no change)
Result: 1 insert (optimal!)
```

**Diffing Algorithm Rules:**

```
1. Different types → Replace entire tree
   <View> → <Text> means remove View, add Text

2. Same type, different props → Update props
   <View style={a}> → <View style={b}> means update style

3. Same type, different content → Update content
   <Text>A</Text> → <Text>B</Text> means update text

4. Same position, same key → Keep element
   <Text key="1"> in old and new = reuse

5. Different position, same key → Move element
   <Text key="1"> moves → <Text key="1"> follows it
```

---

### 3.7 Component Lifecycle (Hooks Era)

**Functional Components with Hooks:**

```javascript
function MyComponent() {
  // MOUNTING
  useEffect(() => {
    console.log('Component mounted');
    
    // CLEANUP (UNMOUNTING)
    return () => {
      console.log('Component will unmount');
    };
  }, []); // Empty dependency = run once on mount
  
  // UPDATING
  useEffect(() => {
    console.log('Props or state changed');
  }, [props.value]); // Run when props.value changes
  
  return <Text>Hello</Text>;
}
```

**Lifecycle Diagram:**

```
┌──────────────────────────────────────┐
│  1. MOUNTING                         │
│  └─ Component created, inserted DOM  │
│     └─ useEffect(fn, []) runs       │
├──────────────────────────────────────┤
│  2. RENDERING                        │
│  └─ Return JSX                       │
├──────────────────────────────────────┤
│  3. UPDATING                         │
│  └─ Props/state change               │
│     └─ useEffect(fn, [deps]) runs   │
├──────────────────────────────────────┤
│  4. UNMOUNTING                       │
│  └─ Component removed from DOM       │
│     └─ Cleanup function runs         │
└──────────────────────────────────────┘
```

---

### 3.8 useState Internals

**How does useState work?**

The magic: **Closures + Module-level state**

```javascript
// React internally stores state like this:
let componentState = [];
let currentIndex = 0;

function useState(initialValue) {
  const index = currentIndex;
  currentIndex++;
  
  // Initialize if not set
  if (componentState[index] === undefined) {
    componentState[index] = initialValue;
  }
  
  const state = componentState[index];
  
  const setState = (newValue) => {
    componentState[index] = newValue;
    // Schedule re-render
    scheduleRender();
  };
  
  return [state, setState];
}
```

**Critical Issue: Hook Order**

```javascript
// ❌ WRONG - Hook called conditionally
function MyComponent() {
  if (someCondition) {
    const [count, setCount] = useState(0); // WRONG!
  }
}

// Why? currentIndex depends on call order
// If sometimes hook not called, indices mismatch

// Render 1: call 1 for count
// Render 2: condition false, don't call count
// Render 3: call 1 for count again
// Now index 1 is wrong!

// ✅ RIGHT - Hook always called
function MyComponent() {
  const [count, setCount] = useState(0); // Always runs
  
  if (someCondition) {
    // Use count here
  }
}
```

**useState with Objects:**

```javascript
// ❌ WRONG - Modifying state directly
const [user, setUser] = useState({ name: 'John' });
user.name = 'Jane'; // Direct mutation - React doesn't detect change

// ✅ RIGHT - Create new object
const [user, setUser] = useState({ name: 'John' });
setUser({ ...user, name: 'Jane' }); // New object, React detects change
```

**Why? React checks if reference changed, not deep equality.**

---

### 3.9 useEffect Internals

**useEffect Timing:**

```
Component renders
  ↓
Browser paints screen
  ↓
useEffect runs (AFTER paint)
```

**Important: useEffect runs AFTER rendering, not before.**

```javascript
function MyComponent() {
  const [count, setCount] = useState(0);
  
  useEffect(() => {
    console.log('After render, count =', count);
  }, [count]);
  
  console.log('During render, count =', count);
  
  return <Text>{count}</Text>;
}

// If count changes:
// OUTPUT:
// "During render, count = 1"
// (Screen updated with "1")
// "After render, count = 1"
```

**Dependency Array:**

```javascript
// No dependency array → Run after every render
useEffect(() => {
  console.log('After every render');
});

// Empty array [] → Run once on mount
useEffect(() => {
  console.log('On mount');
}, []);

// With dependencies [count] → Run when count changes
useEffect(() => {
  console.log('Count changed to', count);
}, [count]);

// Multiple dependencies [count, name]
useEffect(() => {
  console.log('Either count or name changed');
}, [count, name]);
```

**Cleanup Function:**

```javascript
useEffect(() => {
  const subscription = API.subscribe(listener);
  
  // Cleanup function
  return () => {
    subscription.unsubscribe();
  };
}, []);

// Timeline:
// Mount → Subscribe
// Unmount → Run cleanup → Unsubscribe
```

---

### 3.10 Context API Internals

**What is Context?**

A way to pass data deep in component tree without manually passing through every component (prop drilling).

```javascript
// Create context
const ThemeContext = React.createContext();

// Provider at top
function App() {
  return (
    <ThemeContext.Provider value={{ color: 'blue' }}>
      <Screen1 />
    </ThemeContext.Provider>
  );
}

// Screen1 doesn't need to care about theme
function Screen1() {
  return <Screen2 />;
}

// Screen2 doesn't need to care about theme
function Screen2() {
  return <Screen3 />;
}

// Only Screen3 uses theme
function Screen3() {
  const theme = useContext(ThemeContext);
  return <Text style={{ color: theme.color }}>Hello</Text>;
}
```

**Without Context (Prop Drilling):**

```javascript
function App() {
  const theme = { color: 'blue' };
  return <Screen1 theme={theme} />; // Pass theme
}

function Screen1({ theme }) {
  return <Screen2 theme={theme} />; // Pass again
}

function Screen2({ theme }) {
  return <Screen3 theme={theme} />; // Pass again
}

function Screen3({ theme }) {
  return <Text style={{ color: theme.color }}>Hello</Text>;
}
```

**Context Triggering Re-renders:**

```javascript
const ThemeContext = React.createContext();

function MyComponent() {
  const [theme, setTheme] = useState('light');
  
  return (
    <ThemeContext.Provider value={{ theme }}>
      <Child />
    </ThemeContext.Provider>
  );
}

function Child() {
  const { theme } = useContext(ThemeContext);
  // When theme changes, Child re-renders
  // ALL consumers of this context re-render
}
```

**Performance Issue with Context:**

```javascript
// ❌ WRONG - Creates new object every render
<ThemeContext.Provider value={{ color: 'blue' }}>

// ✅ RIGHT - Memoize value
const value = useMemo(() => ({ color: 'blue' }), []);
<ThemeContext.Provider value={value}>
```

If you don't memoize, new object every render → all consumers re-render unnecessarily.

---

## PHASE 4: React Fiber

### 4.1 Why Fiber Was Created

**The Old Problem (Stack Reconciler):**

```
setState() called
  ↓
React starts processing old reconciler
  ↓
Processes entire component tree recursively
  ↓
Cannot pause or stop
  ↓
If tree is huge, blocks JS thread
  ↓
UI thread blocked, drops frames
  ↓
Janky animation/interactions
```

**Example:**

```javascript
function App() {
  const [count, setCount] = useState(0);
  
  return (
    <View>
      {/* This renders 1000 items */}
      {Array.from({ length: 1000 }).map(i => (
        <ExpensiveComponent key={i} />
      ))}
    </View>
  );
}

// User clicks button to increment count
// React has to reconcile 1000 components
// Takes 200ms
// During those 200ms, JS thread is blocked
// Animation frame dropped
// Animation looks janky
```

**Fiber Architecture:**

Allows React to:
1. Pause work
2. Prioritize work
3. Reuse work
4. Abort work if needed

```
Old Reconciler:
  Start → Process Entire Tree → Finish (can't stop)
  
Fiber Reconciler:
  Start → Process Part → Pause → Paint → Resume → Process Part → ...
```

---

### 4.2 What is a Fiber Node?

**Fiber = Unit of Work**

Each component becomes a fiber node.

```javascript
<App>
  <Header />
  <List>
    <Item />
    <Item />
  </List>
</App>

// Creates fiber nodes:
Fiber(App)
  ├─ Fiber(Header)
  ├─ Fiber(List)
  │   ├─ Fiber(Item)
  │   └─ Fiber(Item)
```

**Fiber Node Structure (Simplified):**

```javascript
{
  // Identity
  type: MyComponent,
  key: 'list-0',
  
  // Data
  props: { title: 'Hello' },
  state: { count: 5 },
  hooks: [ /* hook values */ ],
  
  // Tree
  return: parentFiber,
  child: firstChildFiber,
  sibling: nextSiblingFiber,
  
  // Work
  pendingProps: { title: 'Hello' },
  memoizedProps: { title: 'Hello' },
  memoizedState: { count: 5 },
  
  // Scheduling
  lanes: 5,        // Priority
  childLanes: 3,
  
  // Effects
  flags: 'Update', // What changed
  nextEffect: nextFiberWithEffect,
  
  // Old tree reference
  alternate: oldFiber,
}
```

**Fiber Tree:**

```
App Fiber
  ├─ return: null (it's the root)
  ├─ child: Header Fiber
  │           ├─ return: App Fiber
  │           ├─ sibling: List Fiber
  │
  ├─ Header Fiber
  │   └─ child: null
  │
  ├─ List Fiber
  │   ├─ return: App Fiber
  │   ├─ child: Item Fiber
  │
  ├─ Item Fiber
  │   ├─ return: List Fiber
  │   ├─ sibling: Item Fiber

// This is not a tree! It's a linked list!
// Each node points to child, sibling, and parent
```

---

### 4.3 Fiber Tree vs Regular Tree

**Difference:**

```
REGULAR TREE:
  Node has array of children

FIBER TREE:
  Node has:
    - child (first child only)
    - sibling (next sibling)
    - return (parent)
```

**Why This Structure?**

Can traverse without recursion (no call stack overflow).

```javascript
// Traversal without recursion:
let currentFiber = rootFiber;

while (currentFiber) {
  // Process currentFiber
  
  if (currentFiber.child) {
    currentFiber = currentFiber.child;
  } else if (currentFiber.sibling) {
    currentFiber = currentFiber.sibling;
  } else {
    // Go back up and find next sibling
    currentFiber = currentFiber.return;
    if (currentFiber) {
      currentFiber = currentFiber.sibling;
    }
  }
}
```

**Can pause without losing position!**

---

### 4.4 Work Units & Scheduling

**What is a Work Unit?**

A unit of work = Processing one fiber.

```
Render Phase:
  Work Unit 1: Process App Fiber
  → Can pause here
  Work Unit 2: Process Header Fiber
  → Can pause here
  Work Unit 3: Process List Fiber
  → Can pause here
  ...
```

**Scheduling:**

React uses `requestIdleCallback` or `MessageChannel` to schedule work.

```javascript
// React's scheduling (simplified)
function scheduleWork(callback) {
  requestIdleCallback(callback, { timeout: 5000 });
}

// requestIdleCallback says:
// "When the browser is idle (no input, animation, etc.),
//  call my callback. But timeout if nothing happens in 5s"

requestIdleCallback((deadline) => {
  while (deadline.timeRemaining() > 1) {
    // Do 1 unit of work
    workLoop();
  }
  
  if (moreWork) {
    scheduleWork(continueWork);
  }
});
```

**Priority Lanes:**

React assigns priority to different updates.

```
Synchronous (highest):
  ├─ User input
  ├─ Controlled input changes
  └─ Results in immediate UI update

Default:
  ├─ Regular setState
  └─ Results in batched update

Low Priority (lowest):
  ├─ Suspense
  ├─ Transitions
  └─ Can be interrupted
```

**How setState() Works (Complete Flow):**

```
setState(newValue) called
  ↓
Create update object with newValue
  ↓
Determine priority lane (depends on context)
  ↓
Add update to fiber's update queue
  ↓
Schedule work on root fiber
  ↓
requestIdleCallback schedules render cycle
  ↓
[RENDER PHASE]
  ├─ Process fiber tree
  ├─ Call render functions
  ├─ Determine what changed
  ├─ Create new fiber tree
  └─ Can be paused/resumed
  ↓
[COMMIT PHASE]
  ├─ Apply all changes
  ├─ Update DOM/Views
  ├─ Run useEffect
  ├─ Cannot be paused
  └─ Ensures consistency
  ↓
Browser paints
  ↓
User sees update
```

---

### 4.5 Render Phase

**What Happens:**

1. Traverse fiber tree
2. Call component functions
3. Compare old and new JSX
4. Mark what changed
5. **Do NOT update anything yet**

**Example:**

```javascript
function Counter() {
  const [count, setCount] = useState(0);
  
  console.log('Rendering Counter');
  return <Text>{count}</Text>;
}

// User clicks button: setCount(1)

// Render Phase happens:
// "Rendering Counter" logged to console
// New JSX created: <Text>1</Text>
// Compared with old: <Text>0</Text>
// Mark: Text content changed

// Nothing is actually updated yet!
```

**Render Phase Can be Paused:**

```
Process App → Pause → Paint → Resume → Process Header → Pause → Paint → Resume ...

This allows UI to stay responsive!
```

**Pure Functions in Render:**

Render phase can happen multiple times. It must be pure:

```javascript
// ❌ WRONG - Side effect in render
function MyComponent() {
  API.call(); // Side effect! Called multiple times
  return <Text>Hello</Text>;
}

// ✅ RIGHT - Use useEffect for side effects
function MyComponent() {
  useEffect(() => {
    API.call(); // Called once on mount
  }, []);
  
  return <Text>Hello</Text>;
}
```

---

### 4.6 Commit Phase

**What Happens:**

1. All changes from render phase are applied
2. DOM/Views are updated
3. useEffect cleanup runs
4. useEffect effects run
5. **Cannot pause**

**Timeline:**

```
Commit Phase Starts
  ↓
Before Mutation (useLayoutEffect cleanup)
  ↓
Mutation (actually update views)
  ↓
After Mutation (useEffect cleanup, useEffect)
  ↓
Commit Phase Ends
  ↓
Browser paints
```

**Commit is Synchronous:**

```javascript
function MyComponent() {
  useLayoutEffect(() => {
    console.log('Before mutation');
  }, []);
  
  useEffect(() => {
    console.log('After mutation');
  }, []);
  
  return <Text>Hello</Text>;
}

// Output:
// "Before mutation" (synchronously)
// (View updated)
// (Browser paints)
// "After mutation" (asynchronously in microtask)
```

---

### 4.7 Complete setState() Flow with Fiber

**Detailed Flow:**

```
USER CLICKS BUTTON
  ↓
onClick handler executed
  ↓
setState(newValue) called
  ↓
╔═══════════════════════════════════╗
║ UPDATE SCHEDULING                 ║
╠═══════════════════════════════════╣
║                                   ║
║ 1. Create Update Object:          ║
║    {                              ║
║      action: newValue,            ║
║      eagerReducer: calculateNew   ║
║    }                              ║
║                                   ║
║ 2. Add to Fiber's Update Queue:   ║
║    fiber.updateQueue = [update1]  ║
║                                   ║
║ 3. Schedule Work:                 ║
║    scheduleCallback(() => render) ║
║                                   ║
╚═══════════════════════════════════╝
  ↓
requestIdleCallback waiting for browser idle
  ↓
Browser finishes current task
  ↓
Browser is idle
  ↓
requestIdleCallback fires
  ↓
╔═══════════════════════════════════╗
║ RENDER PHASE (Can pause)          ║
╠═══════════════════════════════════╣
║                                   ║
║ 1. beginWork(rootFiber)           ║
║    ├─ Process App fiber           ║
║    ├─ setState() detected         ║
║    ├─ Process update queue        ║
║    ├─ Calculate new state         ║
║    ├─ Call render function        ║
║    ├─ Create new JSX              ║
║                                   ║
║ 2. completeWork(appFiber)         ║
║    ├─ Compare old vs new JSX      ║
║    ├─ Mark Text as "Updated"      ║
║                                   ║
║ 3. beginWork(textFiber)           ║
║    ├─ Text has no state           ║
║    ├─ Return JSX                  ║
║                                   ║
║ 4. completeWork(textFiber)        ║
║    ├─ No changes needed           ║
║                                   ║
║ [BROWSER NEEDS TO PAINT]          ║
║ Pause here!                       ║
║ Return control to browser         ║
║                                   ║
║ [BROWSER PAINTS]                  ║
║                                   ║
║ [BROWSER IDLE AGAIN]              ║
║                                   ║
║ 5. Continue render phase...       ║
║    (if more work)                 ║
║                                   ║
╚═══════════════════════════════════╝
  ↓
[ALL WORK DONE]
  ↓
╔═══════════════════════════════════╗
║ COMMIT PHASE (Cannot pause)       ║
╠═══════════════════════════════════╣
║                                   ║
║ 1. beforeMutation:                ║
║    ├─ Check for DOM mutations     ║
║    ├─ Run useLayoutEffect cleanup ║
║                                   ║
║ 2. mutation:                      ║
║    ├─ Update actual native views  ║
║    ├─ For Text: update text value ║
║                                   ║
║ 3. afterMutation:                 ║
║    ├─ Run useEffect cleanup       ║
║    ├─ Run useEffect               ║
║    ├─ Run other effects           ║
║                                   ║
║ 4. passive (microtask):           ║
║    ├─ Schedule useEffect callback ║
║    ├─ Will run in microtask queue ║
║                                   ║
╚═══════════════════════════════════╝
  ↓
COMMIT PHASE COMPLETE
  ↓
Browser paints updated view
  ↓
User sees "1" on screen
```

**Memory Allocation During This Process:**

```
┌──────────────────────────────────────────┐
│ HEAP MEMORY                              │
├──────────────────────────────────────────┤
│                                          │
│ Update Object (created):                 │
│   { action: 1, eagerReducer: fn }       │
│                                          │
│ New Fiber Tree (created):                │
│   App Fiber { ... }                      │
│   Text Fiber { ... }                     │
│                                          │
│ New JSX Object (created):                │
│   { type: Text, props: { children: "1" } }│
│                                          │
│ old Fiber reference (kept):              │
│   Needed for comparison during diffing   │
│                                          │
│ After Commit Complete:                   │
│   Old Fiber tree can be GC'd             │
│   (no longer needed)                     │
│                                          │
└──────────────────────────────────────────┘
```

---

### 4.8 Common Misconceptions

**Misconception 1: setState() updates state synchronously**

```javascript
const [count, setCount] = useState(0);

const handlePress = () => {
  setCount(1);
  console.log(count); // 0 (not 1!)
  // state update is scheduled, not immediate
};
```

**Misconception 2: Every re-render updates the DOM**

```javascript
function MyComponent() {
  const [count, setCount] = useState(0);
  
  console.log('Render');
  
  const handlePress = () => {
    setCount(count + 1);
  };
  
  return <Button onPress={handlePress} />;
}

// setCount(same value)
// "Render" logged (component re-rendered)
// But DOM didn't update (React detects no change)
```

**Misconception 3: useEffect runs before render**

```javascript
function MyComponent() {
  console.log('Render');
  
  useEffect(() => {
    console.log('Effect');
  });
  
  return <Text>Hello</Text>;
}

// OUTPUT:
// "Render"
// (View painted)
// "Effect"

// useEffect runs AFTER rendering
```

---

## Summary of Phase 3 & 4

**React Fundamentals (Phase 3):**
- Components are functions that return JSX
- JSX is syntactic sugar for React.createElement()
- Props flow down, state is local
- Re-rendering is the mechanism for updates
- Reconciliation algorithm finds what changed
- Keys help identify list items
- useState stores state using closures and indices
- useEffect runs after render (paint) for side effects
- Context passes data without prop drilling

**React Fiber (Phase 4):**
- Fiber is a unit of work that can be paused
- Fiber nodes form a linked list tree, not a regular tree
- Render phase: create new fiber tree, mark changes (can pause)
- Commit phase: actually update views, run effects (cannot pause)
- setState() complete flow: Schedule → Render → Commit → Paint
- Updates are scheduled, not immediate
- Proper use of dependencies prevents unnecessary work

**Next Phase:** React Native Fundamentals - understand the bridge, native modules, and how JavaScript connects to native code.
