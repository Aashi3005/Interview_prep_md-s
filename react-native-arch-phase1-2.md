# React Native Architecture Masterclass
## PHASE 1 & 2: JavaScript Foundations & Browser vs React Native

---

## PHASE 1: JavaScript Foundations

### 1.1 What is JavaScript?

**The Problem Before JavaScript:**
Before JavaScript, browsers were static. Server sent HTML to browser. Browser displayed it. That's it. You clicked a button, page reloaded. Terrible UX.

**Why JavaScript Was Created:**
Brendan Eich created JavaScript in 1995 to run code **inside** the browser on the **client side**. Now you could:
- Validate forms before sending to server
- Change page content without reload
- Handle user interactions immediately

**JavaScript is a Single-Threaded, Event-Driven Language**

```
Your Code → JavaScript Engine (V8, JSC, Hermes) → Machine Code → CPU
```

**Key Properties:**
- Single-threaded (one task at a time)
- Synchronous by default (code runs line by line)
- Asynchronous when needed (callbacks, promises, async/await)
- Dynamically typed
- Garbage collected

---

### 1.2 How JavaScript Executes Code

**The Execution Model (Step by Step)**

```
┌─────────────────────────────────────────────────────────┐
│            JAVASCRIPT EXECUTION                          │
└─────────────────────────────────────────────────────────┘

1. PARSING PHASE
   ├─ Tokenization: "var x = 5;" → tokens [var, x, =, 5, ;]
   ├─ Syntax Analysis: tokens → AST (Abstract Syntax Tree)
   └─ Compilation: AST → Machine Code

2. EXECUTION PHASE
   ├─ Create Global Execution Context
   ├─ Push to Call Stack
   ├─ Allocate memory in Heap
   ├─ Execute code line by line
   └─ Pop from Call Stack

3. GARBAGE COLLECTION
   └─ Remove unreferenced objects from Heap
```

**Real Example:**

```javascript
// Line 1
var x = 5;

// Line 2
function add(a, b) {
  return a + b;
}

// Line 3
var result = add(10, 20);

// Line 4
console.log(result);
```

**What Happens:**
```
PARSING PHASE:
  Code → Tokenization → AST → Compilation to bytecode/machine code

EXECUTION PHASE:
  1. Global Execution Context created
  2. Global object created (window in browser, global in Node)
  3. 'this' bound to global object
  4. Hoisting happens (function declarations move to top)
  5. Code executes line by line
```

**Memory Allocation:**

```
┌──────────────────────────────────────────┐
│          MEMORY LAYOUT                   │
├──────────────────────────────────────────┤
│ STACK (for pointers & primitives)        │
│  ├─ x (pointer to heap address)          │
│  ├─ result (pointer to heap address)     │
│  └─ Call Stack frames                    │
├──────────────────────────────────────────┤
│ HEAP (for objects & complex data)        │
│  ├─ { value: 5 }                         │
│  ├─ function add() { ... }               │
│  └─ { value: 30 }                        │
└──────────────────────────────────────────┘
```

---

### 1.3 Execution Context

**What is Execution Context?**

An Execution Context is a container that holds information about where code is executing.

**Three Types:**
1. **Global Execution Context** - created when JS engine starts
2. **Function Execution Context** - created when function is called
3. **Eval Execution Context** - created when eval() is called (don't use)

**Each Execution Context Contains:**

```
┌─────────────────────────────────────┐
│  EXECUTION CONTEXT                  │
├─────────────────────────────────────┤
│ 1. Variable Environment             │
│    └─ All variables & functions     │
│                                     │
│ 2. Lexical Environment              │
│    └─ Reference to outer scope      │
│                                     │
│ 3. this Binding                     │
│    └─ What 'this' refers to         │
│                                     │
│ 4. Outer Environment Reference      │
│    └─ Scope chain                   │
└─────────────────────────────────────┘
```

**Example:**

```javascript
var globalVar = 'global';

function outer() {
  var outerVar = 'outer';
  
  function inner() {
    var innerVar = 'inner';
    console.log(innerVar);    // 'inner'
    console.log(outerVar);    // 'outer'
    console.log(globalVar);   // 'global'
  }
  
  inner();
}

outer();
```

**Execution Contexts Created:**
```
1. Global EC
   ├─ globalVar
   ├─ outer function
   └─ this = global object

2. outer() EC (when outer called)
   ├─ outerVar
   ├─ inner function
   └─ Outer Reference → Global EC

3. inner() EC (when inner called)
   ├─ innerVar
   └─ Outer Reference → outer() EC
```

---

### 1.4 Call Stack

**What is the Call Stack?**

A data structure (LIFO - Last In First Out) that tracks which function is currently executing.

```
JavaScript is single-threaded, so only ONE function can execute at a time.
The Call Stack keeps track of execution order.
```

**Visual Example:**

```javascript
function restaurant() {
  console.log('Customer arrives');
  takeOrder();
  console.log('Serve food');
}

function takeOrder() {
  console.log('Taking order');
  prepareFood();
  console.log('Order taken');
}

function prepareFood() {
  console.log('Making food');
}

restaurant();
```

**Call Stack Over Time:**

```
Step 1: restaurant() called
┌──────────────┐
│ restaurant() │  ← Currently executing
│ [main]       │
└──────────────┘

Step 2: takeOrder() called from restaurant()
┌──────────────┐
│ takeOrder()  │  ← Currently executing
│ restaurant() │
│ [main]       │
└──────────────┘

Step 3: prepareFood() called from takeOrder()
┌──────────────┐
│ prepareFood()│  ← Currently executing
│ takeOrder()  │
│ restaurant() │
│ [main]       │
└──────────────┘

Step 4: prepareFood() returns
┌──────────────┐
│ takeOrder()  │  ← Back to this
│ restaurant() │
│ [main]       │
└──────────────┘

Step 5: takeOrder() returns
┌──────────────┐
│ restaurant() │  ← Back to this
│ [main]       │
└──────────────┘

Step 6: restaurant() returns
┌──────────────┐
│ [main]       │  ← Done
└──────────────┘
```

**Output Order:**
```
Customer arrives
Taking order
Making food
Order taken
Serve food
```

---

### 1.5 Heap Memory

**Stack vs Heap:**

| Stack | Heap |
|-------|------|
| Small, fast | Large, slower |
| Primitives (numbers, strings) | Objects, arrays, functions |
| Auto cleaned (when function ends) | Manual cleanup (garbage collection) |
| Limited size | No fixed size |
| LIFO order | Random access |

**Heap Allocation Example:**

```javascript
// Primitive - stored in STACK
var age = 25;

// Object - stored in HEAP
var person = {
  name: 'John',
  age: 25
};

// Array - stored in HEAP
var numbers = [1, 2, 3];

// Function - stored in HEAP
function greet() {
  console.log('Hi');
}
```

**Memory Layout:**

```
┌─────────────────────────────────────────────────┐
│                  STACK                          │
├─────────────────────────────────────────────────┤
│ age: 25 (primitive value)                       │
│ person: 0x1000 (address/pointer)                │
│ numbers: 0x2000 (address/pointer)               │
│ greet: 0x3000 (address/pointer)                 │
└─────────────────────────────────────────────────┘
                        ↓ points to
┌─────────────────────────────────────────────────┐
│                  HEAP                           │
├─────────────────────────────────────────────────┤
│ 0x1000: { name: "John", age: 25 }              │
│ 0x2000: [ 1, 2, 3 ]                            │
│ 0x3000: function greet() { ... }               │
└─────────────────────────────────────────────────┘
```

**Why This Matters for React Native:**

In React Native, components are objects in the Heap. Every re-render creates new objects. If you don't manage memory well, you get:
- Memory leaks
- Slow app
- App crashes

---

### 1.6 Event Loop

**The Problem It Solves:**

JavaScript is single-threaded. But you need to handle:
- Click events
- Network requests
- Timers
- File operations

If any of these blocks the main thread, UI freezes.

**Solution: Event Loop + Task Queue**

**How It Works:**

```
┌───────────────────────────────────────────────────┐
│              EVENT LOOP                          │
└───────────────────────────────────────────────────┘

    CALL STACK (executes code)
           ↑
           │
    [Is Call Stack empty?]
           │
    YES → Check MICROTASK QUEUE
           ↓
    [Any microtasks?] YES → Execute 1 microtask
           │                       ↑
           NO                      │
           ↓                   Loop back
    Check MACROTASK QUEUE
           ↓
    [Any macrotasks?] YES → Execute 1 macrotask
           │                       ↑
           NO                      │
           ↓                   Loop back
    [Check MICROTASK QUEUE again after macrotask]
```

**Complete Example:**

```javascript
console.log('1. Start');

setTimeout(() => {
  console.log('2. setTimeout');
}, 0);

Promise.resolve()
  .then(() => {
    console.log('3. Promise');
  });

console.log('4. End');
```

**Execution Order:**

```
OUTPUT:
1. Start        ← Synchronous code
4. End          ← Synchronous code
3. Promise      ← Microtask (Promise.then)
2. setTimeout   ← Macrotask

EXPLANATION:
┌─────────────────────────────────────┐
│ CALL STACK                          │
├─────────────────────────────────────┤
│ Execute:                            │
│   console.log('1. Start')           │
│   setTimeout(...) → Queue macrotask │
│   Promise.resolve().then() → Queue  │
│   microtask                         │
│   console.log('4. End')             │
└─────────────────────────────────────┘
           ↓ Call Stack empty
┌─────────────────────────────────────┐
│ MICROTASK QUEUE                     │
├─────────────────────────────────────┤
│ console.log('3. Promise')           │ ← Execute 1st
└─────────────────────────────────────┘
           ↓ Microtask Queue empty
┌─────────────────────────────────────┐
│ MACROTASK QUEUE                     │
├─────────────────────────────────────┤
│ console.log('2. setTimeout')        │ ← Execute next
└─────────────────────────────────────┘
```

---

### 1.7 Microtasks vs Macrotasks

**Microtasks (High Priority):**
- Promise.then / .catch / .finally
- queueMicrotask()
- MutationObserver
- Process.nextTick (Node.js)

**Macrotasks (Low Priority):**
- setTimeout
- setInterval
- setImmediate (Node.js)
- requestAnimationFrame
- I/O operations
- UI rendering

**Key Rule:**
```
ALL microtasks execute before ANY macrotask.
After each macrotask, check ALL microtasks again.
```

**Practical Example:**

```javascript
console.log('Start');

setTimeout(() => {
  console.log('setTimeout 1');
  Promise.resolve().then(() => console.log('Promise inside setTimeout'));
}, 0);

Promise.resolve()
  .then(() => {
    console.log('Promise 1');
    return Promise.resolve();
  })
  .then(() => {
    console.log('Promise 2');
  });

setTimeout(() => {
  console.log('setTimeout 2');
}, 0);

console.log('End');
```

**Output:**
```
Start
End
Promise 1
Promise 2
setTimeout 1
Promise inside setTimeout
setTimeout 2
```

**Timeline:**
```
EXECUTION TIMELINE:

Synchronous Phase:
  Start → End (in call stack)

Event Loop - Microtask Phase:
  Promise 1 → Promise 2 (all microtasks)

Event Loop - Macrotask Phase #1:
  setTimeout 1 (execute 1st macrotask)
  → New microtask created
  → Microtask Phase: Promise inside setTimeout
  → Back to macrotask

Event Loop - Macrotask Phase #2:
  setTimeout 2 (execute 2nd macrotask)
```

**React Native Impact:**

Microtasks run before rendering. Macrotasks may cause frame drops.

```javascript
// This blocks rendering (macrotask)
setTimeout(() => {
  // Heavy computation
  for (let i = 0; i < 1000000000; i++) {}
}, 0);

// This doesn't block rendering (microtask)
Promise.resolve().then(() => {
  // Heavy computation
});
```

---

### 1.8 Closures

**What is a Closure?**

A function that has access to variables from another function's scope, even after that function has returned.

**The Problem Before Closures:**

```javascript
// OLD WAY - No closure
var count = 0;
function increment() {
  count++;
  console.log(count);
}
increment(); // 1
increment(); // 2

// Problem: count is exposed globally, anyone can modify it
count = 100;
increment(); // 101 - unexpected!
```

**With Closures:**

```javascript
// NEW WAY - With closure
function createCounter() {
  var count = 0; // Private variable
  
  return function increment() {
    count++;
    console.log(count);
  };
}

var counter = createCounter();
counter(); // 1
counter(); // 2
counter(); // 3
// count is private, can't access directly
```

**How Closure Works (Memory):**

```
Function createCounter() creates:
  ├─ count variable (in Heap)
  └─ Returns increment function

increment function keeps reference to:
  └─ count variable in outer scope

Even after createCounter() returns:
  └─ increment() still has access to count

This reference keeps count in memory (not garbage collected)
```

**Lexical Scoping Diagram:**

```
┌─────────────────────────────────────┐
│  GLOBAL SCOPE                       │
│  ├─ createCounter function          │
│  └─ counter variable (points to)    │
│                                     │
│     ┌─────────────────────────┐     │
│     │  createCounter SCOPE    │     │
│     │  ├─ count: 0 (Heap)     │     │
│     │  └─ increment function  │     │
│     │      returns            │     │
│     │                         │     │
│     │    ┌─────────────────┐  │     │
│     │    │ increment() EC  │  │     │
│     │    │ has reference   │  │     │
│     │    │ to outer count  │  │     │
│     │    └─────────────────┘  │     │
│     └─────────────────────────┘     │
└─────────────────────────────────────┘
```

**Multiple Closures (Important):**

```javascript
function createMultipleCounters() {
  var counters = [];
  
  for (var i = 0; i < 3; i++) {
    counters.push(function() {
      console.log(i);
    });
  }
  
  return counters;
}

var fns = createMultipleCounters();
fns[0](); // 3 (not 0!) - CLOSURE GOTCHA
fns[1](); // 3
fns[2](); // 3
```

**Why? All functions share the SAME 'i' variable.**

```
Closure Points To:
┌──────────┐
│ i = 3    │ ← All 3 functions point here
└──────────┘
   ↑
   ├─ fns[0]
   ├─ fns[1]
   └─ fns[2]
```

**Solution:**

```javascript
function createMultipleCounters() {
  var counters = [];
  
  for (let i = 0; i < 3; i++) { // 'let' instead of 'var'
    counters.push(function() {
      console.log(i);
    });
  }
  
  return counters;
}

var fns = createMultipleCounters();
fns[0](); // 0 - Each iteration gets its own 'i'
fns[1](); // 1
fns[2](); // 2
```

**Why? 'let' has block scope, each iteration creates new 'i'.**

```
Each iteration creates new scope:
┌──────────┐
│ i = 0    │ ← fns[0] points here
└──────────┘
┌──────────┐
│ i = 1    │ ← fns[1] points here
└──────────┘
┌──────────┐
│ i = 2    │ ← fns[2] points here
└──────────┘
```

**React Native Impact:**

Closures are used heavily in React hooks:

```javascript
// useState uses closures
function MyComponent() {
  const [count, setCount] = useState(0);
  
  // setCount closure captures the state variable
  const handlePress = () => {
    setCount(count + 1); // Closure accesses count
  };
  
  return <Button onPress={handlePress} />;
}
```

---

### 1.9 Scope Chain

**What is Scope?**

Scope determines which variables are accessible in which part of code.

**Types of Scope:**

```
GLOBAL SCOPE
  └─ Can be accessed anywhere
  
FUNCTION SCOPE
  └─ Variables inside function, local to that function
  
BLOCK SCOPE (let, const)
  └─ Variables inside {}, local to that block
```

**Scope Chain:**

When you access a variable, JavaScript searches:

```
1. Local scope (current execution context)
   ↓ Not found?
2. Enclosing function scope
   ↓ Not found?
3. Global scope
   ↓ Not found?
4. ReferenceError thrown
```

**Example:**

```javascript
var globalVar = 'global';

function outer() {
  var outerVar = 'outer';
  
  function inner() {
    var innerVar = 'inner';
    
    console.log(innerVar);    // Found in local scope
    console.log(outerVar);    // Search: local → outer scope ✓
    console.log(globalVar);   // Search: local → outer → global ✓
  }
  
  inner();
}

outer();
```

**Scope Chain Diagram:**

```
┌────────────────────────────────────┐
│ GLOBAL SCOPE                       │
│ globalVar: 'global'                │
│                                    │
│  ┌──────────────────────────────┐  │
│  │ OUTER() SCOPE                │  │
│  │ outerVar: 'outer'            │  │
│  │ inner: [function]            │  │
│  │                              │  │
│  │  ┌────────────────────────┐  │  │
│  │  │ INNER() SCOPE          │  │  │
│  │  │ innerVar: 'inner'      │  │  │
│  │  │                        │  │  │
│  │  │ Scope Chain:           │  │  │
│  │  │ ├─ innerVar ✓          │  │  │
│  │  │ ├─ outerVar ✓ (outer) │  │  │
│  │  │ └─ globalVar ✓ (global)  │  │
│  │  └────────────────────────┘  │  │
│  └──────────────────────────────┘  │
└────────────────────────────────────┘
```

**Shadowing:**

```javascript
var x = 'global';

function test() {
  var x = 'local'; // Shadows global x
  console.log(x);  // 'local' - local x found first
}

test();
console.log(x); // 'global' - global x unchanged
```

---

### 1.10 Garbage Collection

**The Problem:**

Memory is limited. If you allocate memory but never free it, eventually the device runs out.

**Manual Memory Management (Old Way - C, C++):**

```c
int* ptr = malloc(sizeof(int));  // Allocate
*ptr = 5;
free(ptr);                        // Free (must remember!)
```

Programmer responsible. Easy to forget → memory leak.

**Automatic Garbage Collection (JavaScript Way):**

```javascript
var obj = { name: 'John' }; // Allocate
// Use obj...
// When obj is no longer referenced, GC removes it automatically
obj = null; // Or just stop using it
```

**How Garbage Collection Works:**

**Mark and Sweep Algorithm:**

```
PHASE 1: MARK (Starting from root)
  ├─ Mark all reachable objects
  └─ Objects with no references are unmarked

PHASE 2: SWEEP
  ├─ Go through heap
  ├─ Delete unmarked objects
  └─ Free their memory
```

**Example:**

```javascript
var a = { value: 1 };     // Object A (reachable from 'a')
var b = a;                 // Object A (reachable from 'a' and 'b')
var c = { value: 2 };     // Object C (reachable from 'c')

a = null;                  // Object A still reachable from 'b'
b = null;                  // Object A NOW unreachable - can be GC'd
c = null;                  // Object C unreachable - can be GC'd
```

**Memory Timeline:**

```
TIME 1: a = { value: 1 }
┌──────────────────┐
│ a → Object A     │
└──────────────────┘

TIME 2: b = a
┌──────────────────┐
│ a → Object A ←─┐ │
│ b ─────────────┘ │
└──────────────────┘

TIME 3: a = null
┌──────────────────┐
│ b → Object A     │ (A still referenced)
└──────────────────┘

TIME 4: b = null
┌──────────────────┐
│ GC runs...       │ (A unreachable, deleted)
└──────────────────┘
```

**Memory Leak Example (What to Avoid):**

```javascript
var leakedArray = [];

setInterval(() => {
  leakedArray.push(new Array(1000000)); // Adding data forever
}, 100);

// leakedArray keeps growing, never freed
// Eventually runs out of memory - CRASH!
```

**React Native Memory Impact:**

Components create objects. If not cleaned up:

```javascript
class MyComponent extends React.Component {
  componentDidMount() {
    setInterval(() => {
      console.log('Running...'); // This interval never stops!
    }, 1000);
  }
}

// When component unmounts, interval still running
// Memory keeps growing - LEAK!
```

**Correct Way:**

```javascript
class MyComponent extends React.Component {
  componentDidMount() {
    this.interval = setInterval(() => {
      console.log('Running...');
    }, 1000);
  }
  
  componentWillUnmount() {
    clearInterval(this.interval); // Clean up!
  }
}
```

---

## PHASE 2: Browser vs React Native

### 2.1 How JavaScript Works in Browsers

**Browser Architecture:**

```
┌────────────────────────────────────────────────────────┐
│                    BROWSER                             │
├────────────────────────────────────────────────────────┤
│                                                        │
│  ┌─────────────────────────────────────────────────┐  │
│  │ USER INTERFACE                                  │  │
│  │ ├─ Address bar                                  │  │
│  │ ├─ Back/Forward buttons                         │  │
│  │ ├─ Bookmark                                     │  │
│  └─────────────────────────────────────────────────┘  │
│                                                        │
│  ┌─────────────────────────────────────────────────┐  │
│  │ BROWSER ENGINE                                  │  │
│  │ ├─ Parses HTML → DOM                            │  │
│  │ ├─ Parses CSS → CSSOM                           │  │
│  │ ├─ Combines → Render Tree                       │  │
│  │ └─ Layouts & Paints                             │  │
│  └─────────────────────────────────────────────────┘  │
│                                                        │
│  ┌─────────────────────────────────────────────────┐  │
│  │ RENDERING ENGINE (Webkit, Blink)                │  │
│  │ ├─ DOM                                          │  │
│  │ ├─ Layout Engine                                │  │
│  │ ├─ Rendering                                    │  │
│  │ └─ Display                                      │  │
│  └─────────────────────────────────────────────────┘  │
│                                                        │
│  ┌─────────────────────────────────────────────────┐  │
│  │ JAVASCRIPT ENGINE (V8, SpiderMonkey)            │  │
│  │ ├─ Parser                                       │  │
│  │ ├─ AST                                          │  │
│  │ ├─ Interpreter/Compiler                         │  │
│  │ └─ Executes JS Code                             │  │
│  └─────────────────────────────────────────────────┘  │
│                                                        │
│  ┌─────────────────────────────────────────────────┐  │
│  │ STORAGE (LocalStorage, SessionStorage, Cookies) │  │
│  └─────────────────────────────────────────────────┘  │
│                                                        │
│  ┌─────────────────────────────────────────────────┐  │
│  │ NETWORKING (HTTP, WebSocket, Fetch)            │  │
│  └─────────────────────────────────────────────────┘  │
│                                                        │
└────────────────────────────────────────────────────────┘
```

**Page Load Flow:**

```
1. User types URL in browser
2. Browser sends HTTP request
3. Server responds with HTML
4. HTML Parser starts reading HTML tags
5. When <script> tag found:
   ├─ Download JavaScript file
   ├─ Parse & Compile
   ├─ Execute (blocks HTML parsing) ← IMPORTANT
6. DOM construction continues
7. When <link rel="stylesheet"> found:
   ├─ Download CSS
   ├─ Parse CSS → CSSOM (CSS Object Model)
8. Combine DOM + CSSOM → Render Tree
9. Layout (calculate positions)
10. Paint (draw pixels)
11. Composite (layers)
12. Display on screen
```

**Critical Rendering Path:**

```
HTML Request
      ↓
HTML Parsing → DOM Tree
      ↓
CSS Parsing → CSSOM Tree
      ↓
Combine DOM + CSSOM → Render Tree
      ↓
Layout (Reflow)
      ↓
Paint (Repaint)
      ↓
Composite
      ↓
Display on Screen
```

---

### 2.2 The DOM (Document Object Model)

**What is the DOM?**

A tree representation of the HTML document that JavaScript can interact with.

```
HTML:
<html>
  <head>
    <title>My Page</title>
  </head>
  <body>
    <div id="app">
      <button>Click me</button>
      <p>Hello</p>
    </div>
  </body>
</html>

DOM Tree:
        #document
           │
        <html>
        ├─ <head>
        │  └─ <title>
        │     └─ "My Page"
        │
        └─ <body>
           └─ <div id="app">
              ├─ <button>
              │  └─ "Click me"
              └─ <p>
                 └─ "Hello"
```

**DOM is in Memory:**

```
┌────────────────────────────────┐
│ HEAP (JavaScript Memory)       │
├────────────────────────────────┤
│                                │
│ document                       │
│  └─ Object {                   │
│      documentElement: <html>,  │
│      body: <body>,             │
│      getElementById: fn,       │
│      ...                       │
│     }                          │
│                                │
│ <html> Element Object          │
│ <body> Element Object          │
│ <div> Element Object           │
│ ...                            │
│                                │
└────────────────────────────────┘
```

**JavaScript Can Modify DOM:**

```javascript
// Select element from DOM
var button = document.getElementById('myButton');

// Modify its properties
button.textContent = 'New Text';
button.style.color = 'red';

// When you modify, browser re-renders
// Layout → Paint → Display
```

**DOM Manipulation Triggers Reflow & Repaint:**

```
Reading:
  let width = element.offsetWidth;  // Query current width

Modifying:
  element.style.width = '200px';    // Trigger reflow

Result:
  Layout (Reflow) → Paint → Display
  
  This is SLOW if done repeatedly!
```

---

### 2.3 Virtual DOM

**The Problem Before Virtual DOM:**

```
Each time state changes:
  ├─ Select DOM element
  ├─ Modify it
  └─ Browser reflows & repaints (SLOW!)

If you update 1000 elements:
  └─ 1000 reflows + 1000 repaints = VERY SLOW
```

**Why Facebook Created Virtual DOM:**

Instead of modifying actual DOM directly, React:
1. Creates a virtual representation in JavaScript
2. Compares old vs new virtual representation
3. Calculates minimal changes
4. Updates actual DOM only for changed elements

**Virtual DOM Concept:**

```
┌─────────────────────────────────────────────┐
│ JAVASCRIPT MEMORY                           │
├─────────────────────────────────────────────┤
│                                             │
│ Virtual DOM (JavaScript Object):            │
│ {                                           │
│   type: 'div',                              │
│   props: { id: 'app' },                     │
│   children: [                               │
│     { type: 'button', children: 'Click' },  │
│     { type: 'p', children: 'Hello' }        │
│   ]                                         │
│ }                                           │
│                                             │
└─────────────────────────────────────────────┘
           ↓ Diffing Algorithm (Compare)
┌─────────────────────────────────────────────┐
│ ONLY UPDATE CHANGED ELEMENTS                │
└─────────────────────────────────────────────┘
           ↓
┌─────────────────────────────────────────────┐
│ ACTUAL DOM (Browser Memory)                 │
└─────────────────────────────────────────────┘
           ↓
┌─────────────────────────────────────────────┐
│ BROWSER REFLOW & REPAINT (Once)             │
└─────────────────────────────────────────────┘
```

**Example:**

```javascript
// Initial state
let count = 0;
let vdom = { type: 'div', children: count }; // VDOM: count = 0

// User clicks button
count = 1;
let newVdom = { type: 'div', children: count }; // VDOM: count = 1

// React compares:
// Old: count = 0
// New: count = 1
// Difference: Only "children" changed

// React updates actual DOM:
document.getElementById('app').textContent = 1;
```

**Performance:**

```
WITHOUT Virtual DOM:
  100 updates = 100 DOM operations = 100 reflows = SLOW

WITH Virtual DOM:
  100 updates = Process in JS (fast) = 1 DOM update = 1 reflow = FAST
```

---

### 2.4 Why React Native Has NO DOM

**React Native is NOT for Web**

React Native is for native mobile apps (iOS/Android).

**Mobile devices don't have HTML/CSS/DOM.**

Instead:
- iOS has UIKit (native views)
- Android has Android View System (native views)

```
WEB (React):
  JavaScript → Virtual DOM → Real DOM → Browser renders

MOBILE (React Native):
  JavaScript → Virtual Representation → Native Views → Native OS renders
```

**React Native's Architecture:**

```
┌──────────────────────────────────────┐
│ JAVASCRIPT LAYER                     │
│ ├─ React Components                  │
│ ├─ State & Props                     │
│ └─ Virtual representation (like VDOM)│
└──────────────────────────────────────┘
           ↓
┌──────────────────────────────────────┐
│ BRIDGE (Communication)               │
│ ├─ Serializes JS data                │
│ ├─ Sends to Native                   │
│ └─ Receives Native responses         │
└──────────────────────────────────────┘
           ↓
┌──────────────────────────────────────┐
│ NATIVE LAYER                         │
│ ├─ iOS: UIView, UIButton, etc.       │
│ ├─ Android: View, Button, etc.       │
│ └─ Native OS renders to screen       │
└──────────────────────────────────────┘
```

---

### 2.5 React Web vs React Native (Deep Comparison)

| Aspect | React Web | React Native |
|--------|-----------|--------------|
| **Target** | Web browsers | Mobile (iOS/Android) |
| **DOM** | Has HTML/CSS/DOM | No DOM (uses native views) |
| **Rendering** | Browser engine | Native OS |
| **Components** | `<div>`, `<button>`, etc. | `<View>`, `<Button>`, etc. |
| **Styling** | CSS | StyleSheet (JS object) |
| **Threading** | Single thread | Multiple threads (JS + UI) |
| **Memory** | Shared between JS & DOM | Separated JS & Native |
| **Performance** | DOM operations slow | Direct native rendering faster |
| **Layout** | CSS layout engine | Yoga (React Native layout) |
| **Debugging** | Chrome DevTools | Flipper, React DevTools |

**Code Comparison:**

```javascript
// REACT WEB
function Button() {
  return (
    <button style={{ color: 'blue' }}>
      Click me
    </button>
  );
}

// REACT NATIVE
function Button() {
  return (
    <TouchableOpacity style={styles.button}>
      <Text style={styles.text}>Click me</Text>
    </TouchableOpacity>
  );
}

const styles = StyleSheet.create({
  button: { backgroundColor: 'blue' },
  text: { color: 'white' }
});
```

**Key Difference:**

```
React Web:
  <button> → DOM element → Browser renders HTML button

React Native:
  <TouchableOpacity> → Native View → iOS/Android renders native button
```

---

## Summary of Phase 1 & 2

**JavaScript Fundamentals (Phase 1):**
- Single-threaded execution
- Call Stack & Heap management
- Execution Context creation
- Event Loop (microtasks before macrotasks)
- Closures capture outer scope
- Scope Chain for variable lookup
- Garbage Collection for memory management

**Browser vs React Native (Phase 2):**
- Browsers have DOM (tree representation of HTML)
- Virtual DOM optimizes updates
- React Native has NO DOM (uses native views)
- Mobile rendering is fundamentally different

**Next Phase:** React Fundamentals - understand JSX, Props, State, and Re-rendering mechanics.
