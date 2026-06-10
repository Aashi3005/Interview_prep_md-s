# Complete Frontend & React Native Interview Guide — Aashi Kothari
### Theory → Code → Interview Answers

> **How to use this file:**
> 1. Open in VS Code → `Cmd+Shift+V` (Mac) or `Ctrl+Shift+V` (Windows) to preview
> 2. Read the Theory section of each module first — understand the "why"
> 3. Then read the Code section — see the "how"
> 4. End of each module has Interview Cheatsheet — practice saying these out loud
>
> **Module order:** JS Internals → React Fiber → React Native → TypeScript → Testing → System Design

---

# 📋 Table of Contents

- [MODULE 01 — JavaScript Internals](#module-01--javascript-internals)
  - [Theory 01](#theory-01--concepts--mental-models)
  - [Code 01](#code-01--examples--patterns)
- [MODULE 02 — React Core & Fiber](#module-02--react-core--fiber-architecture)
  - [Theory 02](#theory-02--concepts--mental-models)
  - [Code 02](#code-02--examples--patterns)
- [MODULE 03 — React Native Architecture](#module-03--react-native-architecture)
  - [Theory 03](#theory-03--concepts--mental-models)
  - [Code 03](#code-03--examples--patterns)
- [MODULE 04 — TypeScript](#module-04--typescript-production-patterns)
- [MODULE 05 — Testing](#module-05--testing-strategy)
- [MODULE 06 — System Design](#module-06--system-design-cheatsheet)

---


---

# MODULE 01 — JavaScript Internals

## Theory 01 — Concepts & Mental Models


> Read this first. No code. Just concepts, mental models, and "why" explanations.

---

## Table of Contents

1. [What Kind of Language is JavaScript?](#1-what-kind-of-language-is-javascript)
2. [The Call Stack](#2-the-call-stack)
3. [The Event Loop](#3-the-event-loop)
4. [Microtask Queue vs Macrotask Queue](#4-microtask-queue-vs-macrotask-queue)
5. [Scope](#5-scope)
6. [Closures](#6-closures)
7. [Hoisting](#7-hoisting)
8. [Temporal Dead Zone](#8-temporal-dead-zone)
9. [Prototype Chain](#9-prototype-chain)
10. [Async — The Evolution](#10-async--the-evolution)
11. [Memory & Garbage Collection](#11-memory--garbage-collection)
12. [Type Coercion](#12-type-coercion)

---

## 1. What Kind of Language is JavaScript?

JavaScript is **single-threaded**, **dynamically typed**, and **interpreted** (or JIT-compiled in modern engines).

**Single-threaded** means only one thing runs at a time. There is exactly one call stack. You cannot run two pieces of JS code in parallel. This is a fundamental constraint — everything about async, event loop, and performance flows from this single fact.

**Dynamically typed** means variables don't have types — values do. A variable can hold a string today and a number tomorrow. The engine figures out types at runtime, not at compile time.

**Interpreted / JIT-compiled** means JS engines (V8 in Chrome/Node, Hermes in React Native) take your source code and execute it directly — unlike C++ which compiles to machine code first. Modern engines use Just-In-Time compilation to convert hot code paths to machine code for speed.

---

## 2. The Call Stack

### What it is

The call stack is the engine's way of tracking **where in the program execution currently is**. Every time a function is called, a "frame" is pushed onto the stack. When the function returns, its frame is popped off.

Think of it like a stack of plates. You add plates on top, you remove them from the top. You can never remove from the middle.

### Why it matters

Because JS is single-threaded, there is only ONE call stack. While a function is running, nothing else can run. If a function takes 2 seconds to complete, the entire browser/app is frozen for those 2 seconds — no button presses, no animations, nothing.

This is called **blocking the main thread** and it's one of the most common performance mistakes in JS.

### Stack overflow

If a function keeps calling itself (infinite recursion) with no way to stop, the stack keeps growing until the engine throws a "Maximum call stack size exceeded" error. This is a literal stack overflow.

---

## 3. The Event Loop

### The problem it solves

If JS is single-threaded and only one thing runs at a time, how does it handle things like network requests, timers, and user input without freezing? The answer is the **event loop**.

### What it actually is

The event loop is not a JS feature — it's a feature of the **runtime environment** (browser or Node.js). It's a continuously running loop that does one thing: check if the call stack is empty, and if it is, pull the next task from the queues and push it onto the stack.

The loop looks like this conceptually:

```
while (true) {
  if (call stack is empty) {
    drain the entire microtask queue
    pick ONE task from the macrotask queue and run it
  }
}
```

### The mental model

Imagine a restaurant:
- The **chef** (JS engine) can only cook one dish at a time
- The **order tickets** waiting to be cooked are the macrotask queue
- The **urgent requests from the chef** (like "add salt now before this dish is ruined") are the microtask queue — they happen between orders, not after
- The **event loop** is the kitchen manager who keeps handing the chef new orders

---

## 4. Microtask Queue vs Macrotask Queue

### Why two queues?

Not all async work is equally urgent. Some things need to happen "almost immediately" after the current operation — like a Promise resolving. Others can wait for the next available moment — like a setTimeout callback.

### Microtask Queue (high priority)

Microtasks are **processed immediately after the current task finishes, before the browser renders a frame, before any macrotask runs**. The entire microtask queue is drained in one go. If a microtask adds another microtask, that new one also runs before any macrotask.

**What goes here:** Promise `.then`/`.catch`/`.finally` callbacks, `queueMicrotask()`, `MutationObserver` callbacks.

### Macrotask Queue (lower priority)

Macrotasks are picked **one at a time**. After each macrotask, the microtask queue is fully drained again before the next macrotask runs.

**What goes here:** `setTimeout`, `setInterval`, `setImmediate` (Node), I/O callbacks, UI rendering events.

### The golden rule

> Sync code finishes → ALL microtasks drain → ONE macrotask runs → ALL microtasks drain → ONE macrotask runs → repeat forever

This is why a `Promise.then` callback always runs before a `setTimeout(fn, 0)` callback — even if the timeout is zero. Zero doesn't mean "right now." It means "as soon as possible after the current synchronous code AND all microtasks finish."

### async/await and the event loop

`async/await` is just a cleaner way to write Promises. When you `await` something, the function **pauses** and its continuation (everything after the `await`) is scheduled as a microtask. Control returns to the caller. The rest of the synchronous code runs. Then the microtask runs and the function resumes.

This is why async functions don't block — they pause and yield control, they don't sit and wait.

---

## 5. Scope

### What scope means

Scope is the set of variables that are accessible at a given point in the code. JS uses **lexical scope** (also called static scope), which means scope is determined by where you **write** the code, not where you **call** it from.

### Three levels of scope

**Global scope** — variables declared outside any function or block. Accessible everywhere. Polluting global scope is bad practice (naming conflicts, memory leaks).

**Function scope** — variables declared with `var` inside a function are accessible anywhere within that function, including nested functions, but not outside.

**Block scope** — variables declared with `let` or `const` inside a `{}` block (if, for, while) are only accessible within that block. This is more predictable and is the modern standard.

### Why lexical scope matters

It means you can look at the code and immediately know what a variable refers to — you don't need to trace execution flow. The scope chain is determined at write time and never changes.

---

## 6. Closures

### What a closure is

A closure is when a function **remembers** the variables from the scope where it was **created**, even after that scope has finished executing.

Every function in JavaScript is a closure. When a function is created, it captures a reference to its surrounding scope. Not a snapshot — a live reference. If the outer variable changes, the closure sees the change.

### Why this is powerful

Closures let you create **private state** — data that persists between function calls but isn't accessible from outside. This is how React hooks work internally. `useState` stores state in a closure over the fiber node's state queue. Every time the component renders, the hook function runs, but it "remembers" the previous state through this closure mechanism.

### The biggest mistake with closures

Because closures hold **references** (not copies), if you close over a variable that changes, you might read a stale value. This is called the **stale closure problem**.

In React Native, this most commonly happens in `useEffect` with an empty dependency array — the effect captures the initial value of a state variable and never sees updates because it was created when that value was 0 (or null, or whatever the initial value was). Adding the variable to the dependency array fixes this, or using a functional update (`setState(prev => prev + 1)` instead of `setState(count + 1)`).

---

## 7. Hoisting

### What hoisting means

Before executing any code in a scope, the JavaScript engine makes a first pass and **moves declarations to the top** of their scope. Only the declaration is moved — not the assignment.

### var hoisting

`var` declarations are hoisted to the top of their containing function (not block) and initialized to `undefined`. This is why you can reference a `var` variable before you declare it — it exists, it just has the value `undefined`.

This behavior is confusing and has led to countless bugs. Modern JS avoids `var` entirely.

### Function declaration hoisting

Function declarations (not expressions) are fully hoisted — both the name AND the body. This means you can call a function before you write it in the file. The engine sees the whole function during its first pass.

### let and const — not hoisted the same way

`let` and `const` are technically hoisted, but they are NOT initialized. This leads to the Temporal Dead Zone.

---

## 8. Temporal Dead Zone (TDZ)

### What it is

The **Temporal Dead Zone** is the period between the start of a block scope and the point where a `let` or `const` variable is declared. During this zone, the variable exists in scope (it's been hoisted) but it's in an **uninitialized state**. Accessing it throws a `ReferenceError`.

### Why TDZ exists

`var` hoisting with `undefined` initialization was a design mistake that caused bugs. `let` and `const` were designed to be more predictable — if you access a variable before declaring it, you get an error instead of a silent `undefined`. The error is better because it surfaces the bug immediately.

### In practice

This means with `let` and `const`, your code should flow naturally from top to bottom — declare variables before you use them. If you get a TDZ error, it's telling you that your code is trying to use something before it's ready.

---

## 9. Prototype Chain

### JavaScript's inheritance model

JavaScript does not use classical inheritance like Java or C#. It uses **prototypal inheritance** — objects inherit directly from other objects through a chain of prototypes.

Every JavaScript object has a hidden internal link to another object called its **prototype**. When you access a property on an object and it's not found on the object itself, JS looks at the prototype. Not found there? Looks at the prototype's prototype. Keeps going until it reaches `null` — the end of the chain.

### Why this matters

This is how methods like `.toString()`, `.map()`, `.filter()` work. They're not on every object/array — they're on `Object.prototype` and `Array.prototype`. Your objects inherit them through the chain.

### class syntax

The `class` keyword introduced in ES6 does NOT change how inheritance works in JS. It's **syntactic sugar** over prototypes. Under the hood, `class` still creates a function (the constructor) and sets up prototype links exactly as before. The syntax just looks more familiar to developers coming from classical OOP languages.

Understanding this is important because knowing that `class` is sugar over prototypes helps you debug prototype-related issues and understand why JS inheritance sometimes behaves differently than you'd expect from a classical OOP background.

### The __proto__ vs prototype confusion

- `prototype` is a property on **functions** (constructors) — it's what becomes the prototype of instances created by that function
- `__proto__` is a property on **instances** — it's the actual link to the prototype
- `Object.getPrototypeOf(obj)` is the modern way to access the prototype of an object

---

## 10. Async — The Evolution

### Why async patterns exist

Because JS is single-threaded, you can't just "wait" for a network request to complete — that would freeze everything. So instead of waiting, you **register a callback** to be called when the result is ready. This is the foundation of all async patterns in JS.

### Callbacks — the beginning

The original pattern. You pass a function to be called when the async work is done. Simple, but leads to deeply nested code when you chain multiple async operations — nicknamed "callback hell" or the "pyramid of doom."

The real problem with callbacks isn't nesting — it's **inversion of control**. You hand your callback to someone else's code and trust that they'll call it correctly (once, with the right arguments, handling errors properly). You lose control.

### Promises — taking control back

A Promise is an object that **represents the eventual result** of an async operation. Instead of passing a callback to the async function, the async function gives you a Promise object. You then attach your callbacks to the Promise using `.then()` and `.catch()`.

This restores control — you decide what to do with the result. Promises also chain cleanly (each `.then` returns a new Promise), handle errors consistently through `.catch`, and enable powerful combinators like `Promise.all`.

**Three states of a Promise:**
- **Pending** — initial state, result not yet available
- **Fulfilled** — operation succeeded, has a value
- **Rejected** — operation failed, has a reason (error)

Once a Promise is fulfilled or rejected, it's **settled** and never changes state again.

### async/await — syntactic clarity

`async/await` doesn't introduce any new capability — it's syntax that makes Promise-based code look and read like synchronous code. An `async` function always returns a Promise. `await` pauses the async function and waits for a Promise to settle, then returns its value.

The key insight: `await` only pauses the **async function itself**, not the entire JS thread. While the async function is paused, the event loop can process other tasks. This is why async/await doesn't block.

---

## 11. Memory & Garbage Collection

### The Heap and the Stack

**Stack memory** stores function call frames and primitive values (numbers, booleans, strings). It's managed automatically — when a function returns, its frame is popped and that memory is instantly freed. Stack allocation and deallocation is extremely fast.

**Heap memory** stores objects, arrays, closures — anything that needs a flexible, variable amount of memory. The heap is larger but slower. Memory here is managed by the Garbage Collector.

### Garbage Collection — Mark and Sweep

JS engines use a GC algorithm called **mark-and-sweep**. Periodically, the GC:

1. **Marks** — starts from "roots" (global variables, current call stack) and marks every object it can reach by following references
2. **Sweeps** — frees all memory occupied by objects that were NOT marked (unreachable)

The key insight: **reachability = alive**. If no code can possibly reach an object (no variable holds a reference to it, directly or indirectly), it will be collected.

### Memory leaks

A memory leak happens when objects stay reachable even though your code will never use them again. Common causes:

- **Forgotten event listeners** — an element is removed from the UI but a listener still holds a reference to it
- **Closures over large data** — a function captures a large array/object and is stored somewhere permanently
- **setInterval not cleared** — the callback and everything it closes over stays alive forever
- **React: setState after unmount** — the component is gone but the async callback still holds a reference to its setState function, tries to call it, and React warns you

---

## 12. Type Coercion

### What it is

JavaScript will **automatically convert** one type to another when an operation requires it. This is called implicit type coercion. It follows a complex set of rules that are notoriously confusing.

### Why it exists

JS was designed to be forgiving and easy to write. Rather than throwing an error when you compare a string to a number, it tries to figure out what you meant. This design choice has been controversial — it removes one category of errors but introduces a different category of subtle bugs.

### == vs ===

`==` (loose equality) performs type coercion before comparing. JS will try to convert both values to the same type before checking if they're equal.

`===` (strict equality) never coerces. If the types are different, it immediately returns false.

**Always use `===` in practice.** The only common exception is `value == null`, which checks for both `null` and `undefined` at once — this is an accepted idiom.

### Falsy values

In boolean contexts (if statements, `&&`, `||`, `!`), JS converts values to boolean. These values are "falsy" — they convert to `false`:
- `false`, `0`, `-0`, `0n` (BigInt zero), `""` (empty string), `null`, `undefined`, `NaN`

Everything else is truthy, including `[]`, `{}`, `"0"`, `"false"` — these are all truthy!

### The nullish coalescing operator (??)

`??` is like `||` but only triggers on `null` or `undefined`, not on other falsy values like `0` or `""`. This is important in React Native when you have legitimate falsy values that shouldn't trigger a fallback (e.g., a `count` of `0` should display `0`, not a default).

---

*You've finished Theory 01. Now open `module-01-javascript-internals.md` for all the code examples that illustrate these concepts.*

---

## Code 01 — Examples & Patterns


> **Goal:** Understand how JS actually works under the hood — execution model, memory, closures, prototypes, async. This is the foundation every React/RN interview builds on.

---

## Table of Contents

1. [Execution Model & Call Stack](#1-execution-model--call-stack)
2. [Event Loop — Microtasks & Macrotasks](#2-event-loop--microtasks--macrotasks)
3. [Scope & Closures](#3-scope--closures)
4. [Hoisting & Temporal Dead Zone](#4-hoisting--temporal-dead-zone)
5. [Prototype Chain & Inheritance](#5-prototype-chain--inheritance)
6. [Async Patterns](#6-async-patterns)
7. [Memory Management & Garbage Collection](#7-memory-management--garbage-collection)
8. [Type System & Coercion](#8-type-system--coercion)
9. [Functional JS Patterns](#9-functional-js-patterns)
10. [Interview Cheatsheet](#10-interview-cheatsheet)

---

## 1. Execution Model & Call Stack

### What is it?

JavaScript is **single-threaded** — only one piece of code runs at a time. The engine uses a **call stack** to track which function is currently executing.

```
Call Stack (LIFO — Last In, First Out)
┌─────────────────┐
│  console.log()  │  ← top (currently running)
├─────────────────┤
│  greet()        │
├─────────────────┤
│  main()         │
└─────────────────┘
```

### How it works step by step

```js
function multiply(a, b) {
  return a * b;         // step 3: runs, returns 50
}

function square(n) {
  return multiply(n, n); // step 2: calls multiply
}

const result = square(5); // step 1: square is called
```

**Stack trace:**
1. `square(5)` pushed → stack: `[main, square]`
2. `multiply(5, 5)` pushed → stack: `[main, square, multiply]`
3. `multiply` returns 25 → popped → stack: `[main, square]`
4. `square` returns 25 → popped → stack: `[main]`

### Stack Overflow

Happens when recursion has no base case — stack grows until memory limit.

```js
function infinite() {
  return infinite(); // no base case → stack overflow
}
```

### Interview angle

> "What happens when the call stack is empty?" — The event loop checks the microtask queue, then picks one macrotask. This is how async callbacks get executed.

---

## 2. Event Loop — Microtasks & Macrotasks

### The golden rule

```
Sync code → Microtask queue (drain ALL) → ONE Macrotask → repeat
```

### Three zones

| Zone | What goes here | Examples |
|------|---------------|---------|
| **Call Stack** | Currently executing code | Any running function |
| **Microtask Queue** | High-priority async callbacks | `Promise.then`, `queueMicrotask`, `MutationObserver` |
| **Macrotask Queue** | Lower-priority async callbacks | `setTimeout`, `setInterval`, `setImmediate`, I/O |

### Classic interview question

```js
console.log('1');

setTimeout(() => console.log('2'), 0);

Promise.resolve().then(() => console.log('3'));

console.log('4');

// Output: 1, 4, 3, 2
```

**Why?**
1. `console.log('1')` → sync, runs immediately → output: `1`
2. `setTimeout` → handed to Web API, callback goes to **macrotask** queue
3. `Promise.then` → callback goes to **microtask** queue
4. `console.log('4')` → sync, runs immediately → output: `4`
5. Stack is empty → drain microtasks → `3`
6. Pick one macrotask → `2`

### Promise chain vs setTimeout

```js
Promise.resolve()
  .then(() => console.log('A'))
  .then(() => console.log('B'));

setTimeout(() => console.log('C'), 0);
console.log('D');

// Output: D, A, B, C
```

Chained `.then()` callbacks each queue as a new microtask when the previous resolves. All of them run before `C` even though they were registered before `D` ran.

### async/await is Promise sugar

```js
async function fetchData() {
  console.log('start');   // runs sync
  await Promise.resolve();
  console.log('after await'); // queued as microtask
}

fetchData();
console.log('end');

// Output: start, end, after await
```

Everything **after** `await` is a microtask callback. `await x` literally compiles to `x.then(continuation)`.

### Can microtasks starve the browser?

**Yes.** If a microtask keeps queuing more microtasks, the macrotask queue never gets processed, and the browser never paints.

```js
function starve() {
  Promise.resolve().then(starve); // infinite microtask loop
}
starve(); // browser freezes
```

### Interview angles

- "What is the event loop?" → Single-threaded JS needs a way to handle async. Event loop continuously checks: stack empty? → drain microtasks → run one macrotask → repeat.
- "Why is `setTimeout(fn, 0)` not immediate?" → It's a macrotask. All microtasks (Promises) run first.
- "Difference between microtask and macrotask?" → Priority. Microtasks drain fully before any macrotask runs.

---

## 3. Scope & Closures

### Lexical Scope

Scope is determined by **where code is written**, not where it's called.

```js
const x = 'global';

function outer() {
  const x = 'outer';

  function inner() {
    console.log(x); // 'outer' — lexical scope, not caller's scope
  }

  inner();
}

outer();
```

### Closure

A function that **retains access to its outer scope's variables** even after the outer function has returned.

```js
function makeCounter() {
  let count = 0; // this variable is "closed over"

  return function() {
    count++;
    return count;
  };
}

const counter = makeCounter();
counter(); // 1
counter(); // 2
counter(); // 3
// count is still alive even though makeCounter() finished
```

### Classic closure trap (var in loops)

```js
// WRONG — all callbacks share the same `i`
for (var i = 0; i < 3; i++) {
  setTimeout(() => console.log(i), 100);
}
// Output: 3, 3, 3

// FIX 1 — use let (block-scoped, each iteration gets own i)
for (let i = 0; i < 3; i++) {
  setTimeout(() => console.log(i), 100);
}
// Output: 0, 1, 2

// FIX 2 — IIFE to capture value
for (var i = 0; i < 3; i++) {
  (function(j) {
    setTimeout(() => console.log(j), 100);
  })(i);
}
```

### Stale closure in React (critical!)

```js
function Counter() {
  const [count, setCount] = useState(0);

  useEffect(() => {
    const interval = setInterval(() => {
      console.log(count); // STALE — always logs 0
    }, 1000);
    return () => clearInterval(interval);
  }, []); // empty deps — effect closes over count=0 forever
}

// FIX — add count to deps, or use functional update
useEffect(() => {
  const interval = setInterval(() => {
    setCount(prev => prev + 1); // functional update avoids stale closure
  }, 1000);
  return () => clearInterval(interval);
}, []);
```

### Practical closure uses

```js
// 1. Module pattern (data encapsulation)
function createStore(initialState) {
  let state = initialState;
  return {
    getState: () => state,
    setState: (newState) => { state = { ...state, ...newState }; }
  };
}

// 2. Memoization
function memoize(fn) {
  const cache = new Map();
  return function(...args) {
    const key = JSON.stringify(args);
    if (cache.has(key)) return cache.get(key);
    const result = fn(...args);
    cache.set(key, result);
    return result;
  };
}

// 3. Partial application
function multiply(a) {
  return function(b) {
    return a * b;
  };
}
const double = multiply(2);
double(5); // 10
```

---

## 4. Hoisting & Temporal Dead Zone

### var hoisting

`var` declarations are hoisted to the top of their function and initialized to `undefined`.

```js
console.log(x); // undefined (not ReferenceError)
var x = 5;
console.log(x); // 5

// Engine sees it as:
var x; // declaration hoisted, initialized to undefined
console.log(x); // undefined
x = 5; // assignment stays in place
```

### let/const — Temporal Dead Zone (TDZ)

`let` and `const` are hoisted but **NOT initialized**. Accessing them before declaration throws a `ReferenceError`.

```js
console.log(y); // ReferenceError: Cannot access 'y' before initialization
let y = 5;
```

The gap between the start of the block and the declaration is the **Temporal Dead Zone**.

### Function hoisting

Function declarations are **fully hoisted** (both declaration AND body).

```js
greet(); // works! "Hello"

function greet() {
  console.log("Hello");
}

// Function expressions are NOT fully hoisted
sayBye(); // TypeError: sayBye is not a function
var sayBye = function() { console.log("Bye"); };
```

---

## 5. Prototype Chain & Inheritance

### The chain

Every JS object has a hidden `[[Prototype]]` link to another object. Property lookup walks up the chain until `null`.

```js
const obj = { name: 'Aashi' };
// obj → Object.prototype → null

// Property lookup:
obj.name      // found on obj itself
obj.toString  // not on obj → found on Object.prototype
obj.foo       // not found anywhere → undefined
```

### Constructor functions

```js
function Animal(name) {
  this.name = name; // own property on instance
}

Animal.prototype.speak = function() {
  return `${this.name} makes a noise`;
};

const dog = new Animal('Rex');

// What `new` does:
// 1. Creates empty object: {}
// 2. Sets __proto__ to Animal.prototype
// 3. Runs Animal() with `this` = new object
// 4. Returns the object (unless constructor returns object)

dog.speak();             // "Rex makes a noise" — found on prototype
dog.__proto__ === Animal.prototype  // true
Animal.prototype.__proto__ === Object.prototype  // true
```

### class syntax (same thing, nicer syntax)

```js
class Animal {
  constructor(name) {
    this.name = name; // own property
  }

  speak() {    // goes on Animal.prototype
    return `${this.name} makes a noise`;
  }
}

class Dog extends Animal {
  constructor(name) {
    super(name); // calls Animal constructor
  }

  speak() {   // overrides Animal.prototype.speak
    return `${this.name} barks`;
  }
}

const d = new Dog('Rex');
d.speak(); // "Rex barks"
d instanceof Dog;    // true
d instanceof Animal; // true — prototype chain
```

### hasOwnProperty

```js
const dog = new Animal('Rex');
dog.hasOwnProperty('name');  // true — own property
dog.hasOwnProperty('speak'); // false — on prototype
```

### Interview angles

- "How does inheritance work in JS?" → Prototype chain, not classical inheritance. `class` is syntactic sugar.
- "What does `new` do?" → Creates object, sets `__proto__` to Constructor.prototype, runs constructor with `this`, returns object.
- "Difference between `__proto__` and `prototype`?" → `__proto__` is on instances (the actual link). `prototype` is on functions (what instances link to).

---

## 6. Async Patterns

### Callback (oldest)

```js
function fetchData(url, callback) {
  setTimeout(() => {
    callback(null, { data: 'result' }); // node-style: error first
  }, 1000);
}

fetchData('/api', (err, data) => {
  if (err) return handleError(err);
  console.log(data);
});
```

**Problem:** Callback hell — nested callbacks become unreadable.

### Promise

```js
function fetchData(url) {
  return new Promise((resolve, reject) => {
    setTimeout(() => resolve({ data: 'result' }), 1000);
  });
}

fetchData('/api')
  .then(data => process(data))
  .then(result => save(result))
  .catch(err => handleError(err))
  .finally(() => setLoading(false));
```

### Promise combinators

```js
// All must succeed — fails fast if any rejects
const [user, posts] = await Promise.all([fetchUser(id), fetchPosts(id)]);

// Waits for ALL to settle (success or failure)
const results = await Promise.allSettled([fetchA(), fetchB()]);
results.forEach(r => {
  if (r.status === 'fulfilled') use(r.value);
  else log(r.reason);
});

// First to settle wins (success or failure)
const fastest = await Promise.race([fetchFromCDN1(), fetchFromCDN2()]);

// First to SUCCEED wins (only rejects if ALL reject)
const first = await Promise.any([tryMirror1(), tryMirror2()]);
```

### async/await

```js
async function loadUser(id) {
  try {
    const user = await fetchUser(id);    // pauses, waits
    const posts = await fetchPosts(id);  // pauses, waits
    return { user, posts };
  } catch (err) {
    throw new Error(`Failed: ${err.message}`);
  }
}

// Parallel with async/await (don't await individually if independent)
async function loadAll(id) {
  const [user, posts] = await Promise.all([
    fetchUser(id),
    fetchPosts(id)   // both start simultaneously
  ]);
  return { user, posts };
}
```

### AbortController (cancellation)

```js
const controller = new AbortController();

fetch('/api/data', { signal: controller.signal })
  .then(res => res.json())
  .catch(err => {
    if (err.name === 'AbortError') return; // cancelled, not a real error
    throw err;
  });

// Cancel the request
controller.abort();

// In React: cancel on component unmount
useEffect(() => {
  const controller = new AbortController();
  fetch('/api', { signal: controller.signal }).then(setData);
  return () => controller.abort(); // cleanup
}, []);
```

---

## 7. Memory Management & Garbage Collection

### Heap vs Stack

- **Stack** — function call frames, local primitives. Auto-managed. Fast.
- **Heap** — objects, arrays, closures. Managed by GC.

### Mark and Sweep (V8's GC)

1. **Mark** — starts from GC roots (global, call stack). Marks all reachable objects.
2. **Sweep** — frees everything not marked.

### Common memory leaks

```js
// 1. Global variables
function leak() {
  leakedVar = 'oops'; // no var/let/const → goes on global
}

// 2. Forgotten event listeners
const btn = document.getElementById('btn');
btn.addEventListener('click', heavyHandler);
// Fix: removeEventListener when component unmounts

// 3. Closures holding large data
function createLeak() {
  const bigData = new Array(1000000).fill('data');
  return function() {
    console.log(bigData.length); // bigData never freed
  };
}

// 4. SetInterval not cleared
const id = setInterval(doWork, 1000);
// Fix: clearInterval(id) on cleanup

// 5. React: setState after unmount
useEffect(() => {
  let mounted = true;
  fetchData().then(data => {
    if (mounted) setState(data); // guard against unmounted
  });
  return () => { mounted = false; };
}, []);
```

### WeakMap & WeakRef

```js
// WeakMap: keys are weakly held — GC can collect them
const cache = new WeakMap();

function process(element) {
  if (cache.has(element)) return cache.get(element);
  const result = expensiveComputation(element);
  cache.set(element, result); // if element is removed from DOM, cache entry is GC'd
  return result;
}
```

---

## 8. Type System & Coercion

### typeof

```js
typeof 42          // 'number'
typeof 'hello'     // 'string'
typeof true        // 'boolean'
typeof undefined   // 'undefined'
typeof null        // 'object' ← famous bug in JS
typeof {}          // 'object'
typeof []          // 'object' (use Array.isArray instead)
typeof function(){} // 'function'
typeof Symbol()    // 'symbol'
typeof 42n         // 'bigint'
```

### == vs ===

`==` coerces types before comparing. `===` never coerces.

```js
0 == false    // true (false coerces to 0)
0 === false   // false (different types)
'' == false   // true
null == undefined  // true (special case)
null === undefined // false

// Always use ===, except for null checks:
if (value == null) // catches both null and undefined
```

### Nullish coalescing vs OR

```js
const a = null ?? 'default';    // 'default' — only null/undefined trigger
const b = 0 ?? 'default';       // 0 — 0 is not null/undefined
const c = 0 || 'default';       // 'default' — 0 is falsy, triggers OR
const d = '' ?? 'default';      // '' — empty string is not null/undefined
```

### Optional chaining

```js
const city = user?.address?.city; // undefined instead of throw
const first = arr?.[0];           // safe array access
const result = fn?.();            // call only if fn exists
```

---

## 9. Functional JS Patterns

### Pure Functions

```js
// Pure — same input always gives same output, no side effects
const add = (a, b) => a + b;

// Impure — depends on external state
let total = 0;
const addToTotal = (n) => { total += n; }; // mutates external state
```

### Immutability

```js
// Don't mutate — create new references
const user = { name: 'Aashi', age: 25 };

// Wrong
user.age = 26;

// Right
const updatedUser = { ...user, age: 26 };

// Arrays
const items = [1, 2, 3];
const newItems = [...items, 4];         // add
const filtered = items.filter(x => x !== 2); // remove
const mapped = items.map(x => x * 2);  // transform
```

### Currying

```js
// Normal function
const multiply = (a, b) => a * b;

// Curried — takes arguments one at a time
const curriedMultiply = (a) => (b) => a * b;

const double = curriedMultiply(2);
double(5);  // 10
double(10); // 20

// Real use case
const filterByProp = (prop) => (value) => (arr) =>
  arr.filter(item => item[prop] === value);

const activeUsers = filterByProp('status')('active')(users);
```

### Compose & Pipe

```js
// compose: right to left
const compose = (...fns) => x => fns.reduceRight((v, f) => f(v), x);

// pipe: left to right (more readable)
const pipe = (...fns) => x => fns.reduce((v, f) => f(v), x);

const processUser = pipe(
  validateUser,
  normalizeEmail,
  addTimestamp,
  saveToDb
);
```

### Memoization

```js
function memoize(fn) {
  const cache = new Map();
  return function(...args) {
    const key = JSON.stringify(args);
    if (cache.has(key)) return cache.get(key);
    const result = fn.apply(this, args);
    cache.set(key, result);
    return result;
  };
}

const expensiveCalc = memoize((n) => {
  // heavy computation
  return n * n;
});
```

---

## 10. Interview Cheatsheet

### Output-based questions (must memorize)

```js
// Q1
console.log(typeof null);       // 'object'

// Q2
console.log([] == ![]);         // true
// [] == false → 0 == 0 → true (coercion madness)

// Q3
console.log(0.1 + 0.2 === 0.3); // false (floating point)
console.log((0.1 + 0.2).toFixed(1) === '0.3'); // true

// Q4
var x = 1;
function foo() {
  console.log(x); // undefined (var hoisted inside foo)
  var x = 2;
}
foo();

// Q5
const arr = [1, 2, 3];
arr[10] = 11;
console.log(arr.length); // 11 (sparse array)

// Q6
console.log(1 < 2 < 3);   // true (1 < 2 = true, true < 3 = 1 < 3 = true)
console.log(3 > 2 > 1);   // false (3 > 2 = true, true > 1 = 1 > 1 = false)
```

### Things to always say in interviews

1. "JS is single-threaded, so async is achieved through the event loop"
2. "Closures retain reference to variables, not their values at the time of creation"
3. "Prototype chain is how inheritance works in JS — `class` is syntactic sugar"
4. "`async/await` is syntactic sugar over Promises, which are syntactic sugar over callbacks"
5. "Always prefer `===` over `==` to avoid type coercion surprises"

---

*Next: Module 02 — React Core & Fiber Architecture*

---

# MODULE 02 — React Core & Fiber Architecture

## Theory 02 — Concepts & Mental Models


> Read this first. No code. Just the "why" and "what" before the "how."

---

## Table of Contents

1. [Why React Exists](#1-why-react-exists)
2. [The Virtual DOM — What It Really Is](#2-the-virtual-dom--what-it-really-is)
3. [Why Fiber Was Created](#3-why-fiber-was-created)
4. [What a Fiber Node Is](#4-what-a-fiber-node-is)
5. [The Two Trees — Double Buffering](#5-the-two-trees--double-buffering)
6. [Render Phase — What Happens](#6-render-phase--what-happens)
7. [Commit Phase — What Happens](#7-commit-phase--what-happens)
8. [The Work Loop & Yielding](#8-the-work-loop--yielding)
9. [Priority — The Lane System](#9-priority--the-lane-system)
10. [Reconciliation & Diffing](#10-reconciliation--diffing)
11. [Hooks — The Mental Model](#11-hooks--the-mental-model)
12. [Performance Optimization — The Right Mental Model](#12-performance-optimization--the-right-mental-model)
13. [React 18 Concurrent Mode](#13-react-18-concurrent-mode)

---

## 1. Why React Exists

Before React, building interactive UIs meant manually manipulating the DOM — find the element, change its text, update its class, add a child, remove a sibling. As applications grew complex, keeping the DOM in sync with the application's data became extremely error-prone. Developers had to think about both "what the UI should look like" AND "how to get from the current UI to the new UI."

React's key insight was: **describe what the UI should look like, and let the library figure out how to get there.** You write declarative components — functions that take data (props) and return a description of the UI. React handles updating the actual DOM.

This shifts the burden from the developer to the library, and React can optimize that update process in ways manual DOM manipulation can't.

---

## 2. The Virtual DOM — What It Really Is

The Virtual DOM is not a real thing in the sense of a specific data structure React invented. It's a **programming concept** — React keeps an in-memory description of the UI (a tree of plain JavaScript objects) and compares it to the previous description when something changes. The difference (diff) tells React exactly which parts of the real DOM need to change.

### Why not just update the DOM directly?

The real DOM is **expensive** to work with. Every DOM operation can trigger layout recalculations, style recalculations, and repaints. Working with JavaScript objects (the virtual DOM) is much cheaper. React batches up all the changes, figures out the minimum set of real DOM operations needed, and applies them in one go.

The term "Virtual DOM" is often misunderstood as the source of React's speed. It's not. React can actually be slower than hand-crafted DOM manipulation in some cases. The real value is **developer experience and maintainability** — you think in terms of state and components, not DOM operations. The performance is "good enough" for most applications, and React gives you tools to optimize when needed.

---

## 3. Why Fiber Was Created

The original React reconciler (before React 16) used simple recursion to traverse and update the component tree. When you triggered a re-render, React would walk the entire component tree recursively, comparing old and new, computing updates. This whole process ran **synchronously and couldn't be interrupted**.

**The problem:** On a large, complex application, this recursive walk could take 50–100ms or more. During that entire time, the JS thread was blocked — no user interactions could be processed, no animations could update, the browser couldn't paint a new frame. Users would see janky, unresponsive UIs.

**The solution Fiber provides:** Replace the recursive walk with an iterative, incremental approach that can be paused, resumed, and prioritized. Instead of doing all the work in one synchronous burst, React can do a bit of work, yield control back to the browser for a frame, then continue — all while remaining undetectable to the user.

Fiber is not a new rendering target or a new API. It's a **complete rewrite of React's internal reconciler** to enable this incremental rendering. It's the engine under the hood.

---

## 4. What a Fiber Node Is

A Fiber node is a **plain JavaScript object** that React creates for every component in your application. Think of it as React's internal representation of a component instance — it stores everything React needs to know about that component.

A Fiber node contains:
- **What it is** — what component type (function, class, div, span), what key, what ref
- **Where it is** — pointers to its parent, first child, and next sibling
- **What it has** — its current props, its current state (for hooks, stored as a linked list), any pending state updates
- **What needs to happen** — flags indicating if this node needs to be inserted, updated, or deleted
- **Its twin** — a pointer to its corresponding node in the other tree (the alternate)

### The linked list structure

The most important design choice: Fiber nodes are connected as a **linked list**, not as a recursive tree. Each node has `child` (pointing to its first child), `sibling` (pointing to the next sibling), and `return` (pointing to the parent).

This linked list structure is what makes Fiber interruptible. Instead of recursive function calls (which grow the JS call stack and can't be paused), React traverses the tree with a simple loop that just moves a pointer from one node to the next. At any point, it can stop, save the current pointer, and resume later.

---

## 5. The Two Trees — Double Buffering

React maintains **two Fiber trees at all times**:

**The current tree** — the tree that is currently rendered on the screen. This never changes during an in-progress render.

**The work-in-progress tree** — a clone of the current tree that React is modifying for the next render. This is where all the diffing and updating happens.

Each node in one tree has a pointer (`alternate`) to its corresponding node in the other tree.

### Why two trees?

This is a technique called **double buffering**, borrowed from graphics programming. The key benefit: React can build the new UI completely in the background (work-in-progress tree) without touching what's currently on screen. If something goes wrong, or if higher-priority work arrives, React can throw away the work-in-progress tree entirely — the current tree (and the screen) is untouched.

When the new tree is fully built and ready, React commits it to the screen with a single pointer swap — `root.current` now points to the work-in-progress tree. That old current tree becomes the new work-in-progress tree for the next render, reusing the allocated memory.

---

## 6. Render Phase — What Happens

The render phase is where React figures out **what changed**. It walks the work-in-progress Fiber tree, calling your component functions, comparing new output to previous output, and marking which nodes need to be updated.

### Key characteristics of the render phase

**Pure and side-effect free** — no DOM mutations happen here. React is just doing computation — calling your component functions, running diffing algorithms, building data structures. Nothing visible to the user changes.

**Interruptible** — because it's pure, this phase can be abandoned at any point. If a higher-priority update arrives (like a user typing), React can throw away the in-progress render and start fresh with the new, more important work. Since no DOM was touched, there's nothing to undo.

**May run multiple times** — in React's StrictMode (development), React intentionally double-invokes component functions to help you find side effects in the render phase. This is why you might see console.logs running twice.

### What `beginWork` and `completeWork` mean

As React traverses the Fiber tree depth-first, each node goes through two stages:

- **beginWork** — processing on the way DOWN the tree. This is where your component function is called, children are reconciled (diffed), and flags are set.
- **completeWork** — processing on the way UP the tree (after all children are done). This is where DOM nodes are created (but not yet attached) and effect flags are propagated upward.

---

## 7. Commit Phase — What Happens

The commit phase is where React takes the computed changes and **actually applies them to the real DOM**. This phase is always synchronous — once it starts, it runs to completion.

### Why synchronous?

Because you cannot show a half-updated UI. If React applies some DOM changes and then pauses, the user would see a broken, inconsistent state. The commit phase is a point of no return.

### Three sub-phases of commit

**Before mutation** — React reads the current DOM state before making any changes. This is where `getSnapshotBeforeUpdate` runs. No mutations yet.

**Mutation** — React actually inserts, updates, and removes DOM nodes. After this sub-phase completes, the current tree pointer swaps — the new tree is now "current." This is the moment the screen updates.

**Layout** — `useLayoutEffect` runs here, synchronously, after the DOM has been updated but before the browser has had a chance to paint. This is the right time to read layout information (like element dimensions) without causing visible flicker.

After the commit phase, asynchronously (after the browser paints), `useEffect` runs. This is the passive effects phase.

### useLayoutEffect vs useEffect — the real difference

Both are "effects" but they run at different times in the commit process:

**useLayoutEffect** fires synchronously after DOM mutations, before paint. If you need to measure the DOM (get element height, scroll position) and use that to immediately update something, use `useLayoutEffect`. Anything else → `useEffect`.

**useEffect** fires asynchronously after the browser paints. The user already sees the update. This is right for subscriptions, data fetching, timers — anything that doesn't need to happen before the user sees the screen.

---

## 8. The Work Loop & Yielding

The work loop is React's internal scheduler that decides how much work to do at a time.

In Concurrent Mode, React gives itself a **time budget** — typically around 5 milliseconds per frame. It does as much work as it can within that budget, then yields back to the browser. The browser uses that time to handle input events, run animations, paint frames. Then React resumes.

### Why 5ms?

At 60fps, each frame has about 16ms. If React uses 5ms max, there's still 11ms for the browser to do its work. The user sees smooth animations and responsive input.

### How React yields

React uses `MessageChannel.postMessage()` to schedule its own resumption. This is faster than `setTimeout` and fires at the right time in the event loop — after the browser has had a chance to paint.

### Cooperative scheduling

This whole approach is called **cooperative multitasking**. React voluntarily gives up control rather than being forcibly interrupted. It checks after each unit of work: "Has my time budget run out? Is there something more important to do?" If yes, it yields. If no, it does another unit of work.

---

## 9. Priority — The Lane System

Not all state updates are equally important. Typing in an input must feel instant. Fetching data for a sidebar can wait. React 18 introduced the **Lane system** to express this.

### What a lane is

A lane is a number (actually a bitmask) assigned to an update. Lower numbers = higher priority. React processes higher-priority lanes first.

### The main priorities

**Synchronous** — the highest. Discrete user interactions like clicks and keypresses. Must happen immediately, no yielding.

**Input Continuous** — ongoing interactions like dragging and scrolling. High priority but can be batched.

**Default** — normal async work like network responses and timeouts.

**Transition** — explicitly marked as low-priority work via `startTransition`. Can be interrupted by any higher-priority update.

**Idle** — lowest. Work that should only happen when nothing else is going on.

### Why this matters for React Native

In your Tiffyn app, when a user types into a search field (Sync lane) while a list of 1000 recipes is being filtered (potentially Default/Transition lane), React can interrupt the filtering to keep the input responsive. Using `startTransition` for the filtering explicitly tells React "this is interruptible."

---

## 10. Reconciliation & Diffing

Reconciliation is the process of comparing the old Fiber tree to the new one to figure out what changed. React uses two key heuristics to make this efficient:

### Heuristic 1: Different element type = full rebuild

If the type of an element changes (e.g., `<div>` becomes `<span>`, or `<Counter>` becomes `<Timer>`), React doesn't try to patch it. It tears down the entire subtree and builds a new one from scratch. This means all state in that subtree is lost.

**Practical implication:** Never define component functions inside other component functions. Every render would create a new function reference → React sees a new component type → tears down and rebuilds the child → all child state is lost.

### Heuristic 2: Keys identify list items across renders

When rendering a list, React needs to know which item in the new list corresponds to which item in the old list. Without keys, React uses index — if you insert at the beginning, every item shifts index and React thinks every item changed.

With stable, unique keys, React can track "item with id=5 moved from position 2 to position 0" and move the DOM node rather than destroy and recreate it. This preserves state (like typed input values) and is dramatically faster for large lists.

**Never use array index as key for lists that can be reordered, filtered, or have items added/removed at non-end positions.**

---

## 11. Hooks — The Mental Model

### What hooks really are

Hooks are functions that let functional components "hook into" React's internal state and lifecycle systems. Before hooks, only class components could have state or lifecycle methods. Hooks solved this without introducing classes.

### Why there are rules about hooks

Hooks are stored as a **linked list** on each Fiber node. The first time your component renders, React builds this list: hook 1 is `useState`, hook 2 is `useEffect`, hook 3 is `useMemo`, and so on.

On every subsequent render, React walks this linked list in order, giving each hook call its stored data. Hook call 1 gets state from list position 1, hook call 2 gets state from list position 2, etc.

**If you call hooks conditionally or inside loops**, the order can change between renders. Now hook call 1 might get data that was stored for hook call 2. The wrong data goes to the wrong hook. Everything breaks.

This is why the rule "only call hooks at the top level" exists — it's not arbitrary. It's a direct consequence of how hooks are implemented.

### Each hook's purpose

**useState** — stores a value that persists across renders. Triggers a re-render when updated.

**useEffect** — runs a side effect after render. The cleanup function prevents memory leaks. Dependencies array controls when it re-runs.

**useMemo** — caches an expensive computed value. Only recomputes when dependencies change.

**useCallback** — caches a function reference. Exists so memoized child components don't re-render just because the parent re-rendered and created a new function object.

**useRef** — holds a mutable value that doesn't trigger re-render when changed. Also used to access DOM/native nodes directly.

**useContext** — reads a value from a Context without prop drilling.

---

## 12. Performance Optimization — The Right Mental Model

### Start without optimization

This is the most important principle: **don't optimize prematurely**. React is fast by default for most applications. Premature optimization adds complexity, makes code harder to read, and often doesn't actually improve performance.

**The workflow:**
1. Build the feature
2. Notice a performance problem (measure it, don't guess)
3. Use React DevTools Profiler to identify the bottleneck
4. Apply the minimum optimization needed
5. Verify it helped

### React.memo — what it does and doesn't do

`React.memo` tells React: "only re-render this component if its props have changed." React compares previous props to new props using shallow equality — it checks if each prop is the same reference, not deeply equal.

**The trap:** Objects and functions are compared by reference. A new object `{}` never equals a previous object `{}` even if they contain the same data, because they're different objects in memory. If a parent passes a new object or function on every render, `React.memo` is useless.

This is why `useMemo` and `useCallback` exist — they create stable references that don't change between renders unless their dependencies change.

### The re-render mental model

A component re-renders when:
1. Its own state changes
2. Its parent re-renders (by default)
3. A context it subscribes to changes

Re-renders are not always bad. A re-render is just a function call. If the function is fast and the output is the same (React will see this and skip the DOM update), it's often cheaper to just re-render than to add memoization overhead.

Only memoize when re-renders are measurably expensive.

---

## 13. React 18 Concurrent Mode

### What "concurrent" means

Before React 18, rendering was always synchronous and uninterruptible. A render started and ran to completion no matter what. Concurrent Mode means multiple renders can be "in progress" at the same time (though only one runs at a time on the single JS thread), and renders can be interrupted.

### Automatic batching

In React 17 and earlier, multiple state updates inside event handlers were batched into one render. But state updates inside `setTimeout`, Promises, or native event handlers each triggered their own render.

React 18 batches all state updates everywhere, automatically. Multiple `setState` calls always result in one render, regardless of where they happen.

### startTransition — the most important new API

`startTransition` lets you mark a state update as "non-urgent." React will start the update at low priority and can interrupt it if the user does something (like typing). The UI stays responsive.

The mental model: there are "urgent" updates (what the user is doing right now — typing, clicking) and "transition" updates (what happens as a result — filtering a list, loading new content). `startTransition` is how you tell React which is which.

---

*You've finished Theory 02. Now open `module-02-react-fiber.md` for all the code examples.*

---

## Code 02 — Examples & Patterns


> **Goal:** Understand React's internal engine — Fiber, reconciliation, the two-phase render, hooks internals, and performance optimization. Every senior React/RN interview goes here.

---

## Table of Contents

1. [Why Fiber Exists — The Problem with the Old Reconciler](#1-why-fiber-exists)
2. [Fiber Node — What It Actually Is](#2-fiber-node--what-it-actually-is)
3. [The Two Trees — Current & Work-in-Progress](#3-the-two-trees--current--work-in-progress)
4. [The Two Phases — Render & Commit](#4-the-two-phases--render--commit)
5. [The Work Loop & Yielding](#5-the-work-loop--yielding)
6. [Lane-Based Priority System](#6-lane-based-priority-system)
7. [Reconciliation & Diffing Algorithm](#7-reconciliation--diffing-algorithm)
8. [Hooks — Internal Implementation](#8-hooks--internal-implementation)
9. [Performance Optimization APIs](#9-performance-optimization-apis)
10. [React 18 Concurrent Features](#10-react-18-concurrent-features)
11. [Interview Cheatsheet](#11-interview-cheatsheet)

---

## 1. Why Fiber Exists

### The old reconciler (Stack Reconciler — pre React 16)

React used to reconcile trees using plain recursion.

```js
// Old approach — recursive, cannot be interrupted
function reconcile(vdom) {
  if (isComponent(vdom)) {
    const rendered = render(vdom);
    reconcile(rendered);   // ← goes deeper
  }
  reconcile(vdom.children); // ← can't pause here
}
```

**Problem:** If your component tree is deep, this recursive walk could take 50–100ms, blocking the main thread the entire time. No frame renders during that time → janky UI.

### What Fiber solves

Fiber re-implements the reconciler as an **iterative** linked-list traversal instead of a recursive call. Because it's iterative, React can:

- **Pause** mid-work and yield the thread back to the browser
- **Resume** from exactly where it left off
- **Abort** low-priority work when high-priority work arrives
- **Reuse** completed work

This is what makes React 18's concurrent features possible.

---

## 2. Fiber Node — What It Actually Is

A Fiber is a plain JavaScript object — one per React element/component.

```js
// Simplified FiberNode structure
{
  // Identity
  tag: FunctionComponent | ClassComponent | HostComponent | ...,
  type: App | 'div' | 'span' | null,
  key: null | string,

  // Tree pointers — this is the linked list
  return: FiberNode,    // parent
  child: FiberNode,     // first child
  sibling: FiberNode,   // next sibling (same parent)

  // State
  memoizedState: any,   // current hook state (linked list of hooks)
  memoizedProps: any,   // props from last render
  pendingProps: any,    // props for current render

  // Work
  flags: number,        // bitmask: Placement | Update | Deletion | ...
  lanes: number,        // priority lanes for this fiber
  updateQueue: any,     // queued setState calls

  // Double buffering
  alternate: FiberNode, // pointer to the twin in the other tree

  // DOM
  stateNode: DOM node | class instance | null,

  // Effects
  effectList: FiberNode, // linked list of fibers with side effects
}
```

### The linked list traversal pattern

```
App
├── Header (child of App)
│   └── Logo (child of Header)
└── Main (sibling of Header)
    └── Content (child of Main)

Traversal order (DFS):
App → Header → Logo → (back to Header) → Main → Content → (back to Main) → (back to App)

Pointers:
App.child = Header
Header.sibling = Main
Header.child = Logo
Logo.return = Header
Main.child = Content
Content.return = Main
Main.return = App
```

This linked list structure means React can do `currentFiber = currentFiber.child` or `currentFiber = currentFiber.sibling` — simple pointer operations. No recursive function calls on the JS stack.

---

## 3. The Two Trees — Current & Work-in-Progress

React maintains **two Fiber trees simultaneously**.

```
current tree           workInProgress tree
(on screen right now)  (being built for next render)

     App ←──alternate──→ App'
      |                    |
    Header ←─alternate──→ Header'
      |                    |
    Main ←──alternate──→  Main'
```

- **`current`** — the tree currently rendered to the screen. `root.current` points to it.
- **`workInProgress`** — a clone of current being modified for the next render. Each node has `alternate` pointing to its twin.

### The swap (double buffering)

After the commit phase completes:

```js
root.current = workInProgress; // the swap
// old current becomes the new workInProgress for next render
```

This is why React renders are "atomic" from the user's perspective — the swap is a single pointer assignment. No partial updates visible.

### Why double buffering?

- **Zero allocation** on re-renders — fiber objects are reused, just updated
- **Safe abort** — if workInProgress is abandoned, current tree is untouched
- **Rollback** — you always have the previous tree available

---

## 4. The Two Phases — Render & Commit

### Phase 1: Render Phase (interruptible)

```
beginWork(fiber)       →  completeWork(fiber)
│                           │
├─ Call your component fn   ├─ Create DOM node instances
├─ Reconcile children       ├─ Attach props to DOM nodes
├─ Compute new state        ├─ Collect effect flags
└─ Mark flags on fiber      └─ Build effect list at root
```

**Key rules of the render phase:**
- **Pure** — no DOM mutations, no side effects
- **Interruptible** — React can abandon and restart if higher-priority work arrives
- Can run in multiple chunks across multiple frames

```js
// beginWork simplified
function beginWork(current, workInProgress) {
  switch (workInProgress.tag) {
    case FunctionComponent:
      return updateFunctionComponent(current, workInProgress);
    case HostComponent: // 'div', 'span', etc.
      return updateHostComponent(current, workInProgress);
    // ...
  }
}

// updateFunctionComponent — this is where your component runs
function updateFunctionComponent(current, workInProgress) {
  const Component = workInProgress.type;  // your function
  const props = workInProgress.pendingProps;
  const nextChildren = renderWithHooks(current, workInProgress, Component, props);
  reconcileChildren(current, workInProgress, nextChildren); // diff
  return workInProgress.child;
}
```

### Phase 2: Commit Phase (synchronous — cannot interrupt)

Three sub-phases, all synchronous:

```
1. Before Mutation
   ├─ Read DOM before changes (getSnapshotBeforeUpdate)
   └─ Schedule useEffect cleanups

2. Mutation
   ├─ Insert new DOM nodes (Placement flag)
   ├─ Update existing DOM nodes (Update flag)
   ├─ Remove old DOM nodes (Deletion flag)
   └─ ← current tree swaps to workInProgress here

3. Layout
   ├─ useLayoutEffect fires (sync, DOM updated, no paint yet)
   └─ Update refs (ref.current = DOM node)

4. Passive Effects (async — after paint)
   └─ useEffect fires (async, browser already painted)
```

**Why is commit synchronous?**

Because a half-applied DOM update would show a broken UI. Once you start mutating the DOM, you must finish. Unlike render phase, you can't throw away half a commit.

### useLayoutEffect vs useEffect timing

```js
function Component() {
  const ref = useRef();

  useLayoutEffect(() => {
    // Fires after DOM mutation, BEFORE paint
    // Can read layout (getBoundingClientRect) without flash
    const height = ref.current.offsetHeight;
    console.log('layout height:', height); // accurate
  }, []);

  useEffect(() => {
    // Fires after paint
    // Can't prevent layout flash
    const height = ref.current.offsetHeight;
    console.log('effect height:', height); // also accurate but too late to prevent flash
  }, []);

  return <div ref={ref}>Content</div>;
}
```

**Rule:** Use `useLayoutEffect` only when you need to read or write layout before the user sees the paint (e.g. tooltip positioning, scroll restoration). Everything else → `useEffect`.

---

## 5. The Work Loop & Yielding

```js
// Simplified work loop (Concurrent Mode)
function workLoopConcurrent() {
  while (workInProgress !== null && !shouldYield()) {
    workInProgress = performUnitOfWork(workInProgress);
  }
}

function performUnitOfWork(fiber) {
  const next = beginWork(fiber.alternate, fiber, renderLanes);
  fiber.memoizedProps = fiber.pendingProps;

  if (next === null) {
    // No more children — complete this fiber and go back up
    completeUnitOfWork(fiber);
  }

  return next; // next fiber to process
}
```

### shouldYield — how React yields to the browser

```js
// React Scheduler uses MessageChannel, not setTimeout
// MessageChannel fires before setTimeout and after paint
const channel = new MessageChannel();
channel.port2.onmessage = () => workLoop(); // resume

function shouldYield() {
  const currentTime = performance.now();
  return currentTime >= deadline; // typically 5ms time slice
}
```

React gives itself ~5ms per time slice. After 5ms, it yields, the browser renders a frame, then React resumes via `MessageChannel.postMessage()`. This is why animations stay smooth even during heavy re-renders.

---

## 6. Lane-Based Priority System

Lanes are a bitmask system. Each update gets a "lane" representing its priority.

```js
// Simplified Lane constants
const SyncLane           = 0b0000000000000000000000000000001; // 1
const InputContinuousLane= 0b0000000000000000000000000000100; // 4
const DefaultLane        = 0b0000000000000000000000000010000; // 16
const TransitionLane1    = 0b0000000000000000000000001000000; // 64
const IdleLane           = 0b0100000000000000000000000000000; // huge
```

| Lane | Priority | Triggered by |
|------|----------|-------------|
| `SyncLane` | Highest | `flushSync`, discrete user input (click, keypress) |
| `InputContinuousLane` | High | Drag, scroll, mousemove |
| `DefaultLane` | Normal | `setTimeout`, network response, `startTransition` batched |
| `TransitionLane` | Low | `startTransition()` |
| `IdleLane` | Lowest | Offscreen, prefetch |

### How lanes enable preemption

```js
// User clicks button → SyncLane update queued
// React is mid-render on a TransitionLane update

// React checks: is there higher-priority work?
if (includesSomeLane(root.pendingLanes, SyncLane)) {
  // Yes → abort TransitionLane render
  // → process SyncLane first
  // → then restart TransitionLane
}
```

### startTransition in practice

```js
import { useTransition } from 'react';

function SearchPage() {
  const [query, setQuery] = useState('');
  const [results, setResults] = useState([]);
  const [isPending, startTransition] = useTransition();

  function handleChange(e) {
    // Typing update: SyncLane — always instant
    setQuery(e.target.value);

    // Results update: TransitionLane — can be interrupted
    startTransition(() => {
      setResults(filterItems(allItems, e.target.value));
    });
  }

  return (
    <>
      <input value={query} onChange={handleChange} />
      {isPending && <Spinner />}
      <ResultsList results={results} />
    </>
  );
}
```

---

## 7. Reconciliation & Diffing Algorithm

React's diffing has two main heuristics that make it O(n) instead of O(n³):

### Heuristic 1: Elements of different types → destroy and rebuild

```jsx
// Old tree:
<div><Counter /></div>

// New tree:
<span><Counter /></span>

// div → span: different type
// React destroys entire div subtree (including Counter's state)
// and creates span subtree from scratch
```

### Heuristic 2: Keys tell React which items are "the same"

```jsx
// Without keys — React uses index
// If you insert at the beginning, React re-renders ALL items
<ul>
  <li>Apple</li>   {/* index 0 */}
  <li>Banana</li>  {/* index 1 */}
</ul>

// After inserting Mango at top:
<ul>
  <li>Mango</li>   {/* index 0 — React thinks this is "Apple" updated */}
  <li>Apple</li>   {/* index 1 — React thinks this is "Banana" updated */}
  <li>Banana</li>  {/* index 2 — new item */}
</ul>
// All three re-render even though Apple and Banana didn't change

// With stable keys — React can track by identity
<ul>
  {fruits.map(f => <li key={f.id}>{f.name}</li>)}
</ul>
// React knows: Mango is new (insert), Apple/Banana unchanged (skip)
```

### Why keys can't be array index

```jsx
// Problem: deleting or reordering with index keys
// causes state to be wrongly preserved

const [items, setItems] = useState([
  { id: 1, text: 'First' },
  { id: 2, text: 'Second' },
]);

// With index keys (WRONG):
items.map((item, i) => <Input key={i} defaultValue={item.text} />)
// Deleting first item: Input at index 0 still exists, just gets "Second" as new defaultValue
// But component state (typed input) is preserved → shows wrong value

// With stable id keys (CORRECT):
items.map(item => <Input key={item.id} defaultValue={item.text} />)
// Deleting first item: Input with key=1 is removed, key=2 stays with its state intact
```

---

## 8. Hooks — Internal Implementation

Hooks are stored as a **linked list** on the fiber's `memoizedState`.

```js
// What memoizedState looks like on a FunctionComponent fiber:
{
  memoizedState: {          // Hook 1: useState
    memoizedState: 0,       // current state value
    queue: { ... },         // update queue (setState calls)
    next: {                 // Hook 2: useEffect
      memoizedState: {
        deps: [userId],
        destroy: cleanup,   // previous cleanup function
        create: effect,     // current effect function
      },
      next: {               // Hook 3: useMemo
        memoizedState: [cachedValue, deps],
        next: null
      }
    }
  }
}
```

**This is why hooks can't be called conditionally.** The hook order is the only way React maps hook calls to their stored state. Change the order → wrong state assigned to wrong hook.

### useState internals

```js
// First render: mount
function mountState(initialState) {
  const hook = mountWorkInProgressHook(); // creates hook object in linked list
  hook.memoizedState = initialState;
  const queue = { pending: null, dispatch: null };
  hook.queue = queue;
  const dispatch = queue.dispatch = dispatchSetState.bind(null, currentFiber, queue);
  return [hook.memoizedState, dispatch];
}

// Subsequent renders: update
function updateState() {
  const hook = updateWorkInProgressHook(); // reads from linked list
  // Process any queued setState calls
  // Compute new state
  return [hook.memoizedState, hook.queue.dispatch];
}
```

### useEffect internals

```js
// On render: schedule effect (don't run yet)
function mountEffect(create, deps) {
  const hook = mountWorkInProgressHook();
  hook.memoizedState = {
    tag: HookPassive, // marks as passive (runs after paint)
    create,           // your effect function
    destroy: null,    // cleanup (populated after effect runs)
    deps,
    next: null,
  };
  // Flag fiber for passive effects
  workInProgress.flags |= PassiveEffect;
}

// After commit + paint: run effects
function commitPassiveEffects() {
  // 1. Run all cleanup functions from previous render
  // 2. Run all new effect functions
  // 3. Store returned cleanup function in hook.memoizedState.destroy
}
```

---

## 9. Performance Optimization APIs

### React.memo

Wraps a component. Skips re-render if props haven't changed (shallow comparison).

```jsx
// Without memo: re-renders every time parent renders
const Child = ({ name }) => <Text>{name}</Text>;

// With memo: only re-renders if name prop changes reference
const Child = React.memo(({ name }) => <Text>{name}</Text>);

// Custom comparator (second argument)
const Child = React.memo(
  ({ user }) => <UserCard user={user} />,
  (prev, next) => prev.user.id === next.user.id // deep-ish check
);
```

**When does React.memo FAIL?** When props are created inline in the parent — new object/function reference every render.

```jsx
// BREAKS memo — new object created every parent render
<Child config={{ theme: 'dark' }} />  // {} !== {} always

// BREAKS memo — new function created every parent render
<Child onPress={() => handlePress(id)} />  // () => {} !== () => {}

// FIX: stable references with useMemo and useCallback
const config = useMemo(() => ({ theme: 'dark' }), []);
const onPress = useCallback(() => handlePress(id), [id]);
<Child config={config} onPress={onPress} />
```

### useMemo

Memoizes an **expensive computed value**. Only recomputes when dependencies change.

```jsx
// Without useMemo: recomputed every render
const sorted = [...items].sort((a, b) => a.date - b.date);

// With useMemo: only recomputed when items changes
const sorted = useMemo(
  () => [...items].sort((a, b) => a.date - b.date),
  [items]
);

// useMemo for stable object reference (for React.memo children)
const style = useMemo(() => ({
  color: isActive ? 'blue' : 'gray',
  fontWeight: 'bold',
}), [isActive]);
```

**When NOT to use useMemo:**
- Simple values (strings, numbers) — memoization overhead > recompute cost
- When deps change as often as the component renders
- Primitive returns

### useCallback

Memoizes a **function reference**. Same as `useMemo(() => fn, deps)`.

```jsx
// Without useCallback: new function every render → breaks React.memo on child
function Parent() {
  const handleSubmit = () => submitForm(formData); // new ref every render
  return <MemoizedForm onSubmit={handleSubmit} />;
}

// With useCallback: stable reference → React.memo works
function Parent() {
  const handleSubmit = useCallback(
    () => submitForm(formData),
    [formData]
  );
  return <MemoizedForm onSubmit={handleSubmit} />;
}
```

**useCallback vs useMemo:**
```js
useCallback(fn, deps) === useMemo(() => fn, deps)
// useCallback: memoize the function itself
// useMemo: memoize the result of calling a function
```

### When to reach for these APIs

```
Start without them.
Profile first (React DevTools Profiler).
Add only when:
  1. Component is in a hot render path (renders many times per second)
  2. Expensive computation confirmed in profiler
  3. Child is wrapped in React.memo and parent causes unnecessary re-renders
```

---

## 10. React 18 Concurrent Features

### Automatic Batching

Before React 18, batching only happened inside React event handlers.

```js
// Before React 18 — setTimeout calls batched separately
setTimeout(() => {
  setCount(c => c + 1); // re-render
  setFlag(f => !f);     // re-render
  // TWO renders
});

// React 18 — ALL updates batched everywhere
setTimeout(() => {
  setCount(c => c + 1);
  setFlag(f => !f);
  // ONE render
});
```

### Suspense for Data Fetching

```jsx
// Component "suspends" by throwing a Promise
// React catches it, shows fallback, resumes when Promise resolves

function UserProfile({ userId }) {
  const user = use(fetchUser(userId)); // throws Promise if not ready
  return <div>{user.name}</div>;
}

function App() {
  return (
    <Suspense fallback={<Spinner />}>
      <UserProfile userId={1} />
    </Suspense>
  );
}
```

### useTransition

```jsx
function TabContainer() {
  const [tab, setTab] = useState('about');
  const [isPending, startTransition] = useTransition();

  function switchTab(nextTab) {
    startTransition(() => {
      setTab(nextTab); // TransitionLane — can be interrupted
    });
  }

  return (
    <>
      <TabButtons onSwitch={switchTab} />
      {isPending && <div style={{ opacity: 0.5 }}>Loading...</div>}
      <TabPanel tab={tab} />
    </>
  );
}
```

### useDeferredValue

Like `startTransition` but for values you receive (e.g. from a parent).

```jsx
function SearchResults({ query }) {
  const deferredQuery = useDeferredValue(query);
  // deferredQuery lags behind query — stale content shown while fresh content loads
  // Avoids blocking the input on heavy filtering
  const results = filterItems(allItems, deferredQuery);
  return <List items={results} />;
}
```

---

## 11. Interview Cheatsheet

### Must-know answers

**"What is React Fiber?"**
> A reimplementation of the React reconciler using a linked list of Fiber nodes instead of a recursive call stack. Enables incremental, interruptible rendering — React can pause, resume, or abort render work, making concurrent features possible.

**"What is reconciliation?"**
> The process of diffing the previous Fiber tree against the new one (`beginWork`/`completeWork`) to compute the minimal set of DOM mutations. Uses two heuristics: elements of different types are fully replaced; keys identify which list items are "the same".

**"Render phase vs Commit phase?"**
> Render phase: pure, interruptible, builds work-in-progress tree, no DOM touches.
> Commit phase: synchronous, mutates DOM, runs layout effects, then passive effects. Cannot be interrupted.

**"Why can't hooks be called conditionally?"**
> Hooks are stored as a linked list on the Fiber node. React maps hook calls to stored state by position. Conditional calls change the position → wrong state assigned to wrong hook.

**"What does `useLayoutEffect` do vs `useEffect`?"**
> `useLayoutEffect` runs synchronously after DOM mutations but before paint (commit phase). `useEffect` runs asynchronously after paint. Use `useLayoutEffect` to measure DOM or prevent layout flash.

**"What is `startTransition`?"**
> Marks a state update as `TransitionLane` (low priority). React can interrupt this render if higher-priority work (like typing) arrives. Keeps UI responsive during expensive renders.

**"When does React.memo NOT help?"**
> When props are object/function literals created inline in the parent — new reference every render breaks the shallow comparison. Fix with `useMemo` and `useCallback`.

### Common output questions

```jsx
// Q: How many renders does this cause?
function App() {
  const [a, setA] = useState(0);
  const [b, setB] = useState(0);

  function handleClick() {
    setA(1);
    setB(2);
  }
  // A: ONE render (React 18 automatic batching)
}

// Q: What does this log?
function Component() {
  const [count, setCount] = useState(0);
  useEffect(() => {
    setCount(1);
    console.log(count); // 0 — state update is async, console.log sees old value
  }, []);
}
```

---

*Next: Module 03 — React Native Architecture (Old Bridge → New JSI/Fabric)*

---

# MODULE 03 — React Native Architecture

## Theory 03 — Concepts & Mental Models


> Read this first. No code. Understand why things are the way they are.

---

## Table of Contents

1. [What React Native Actually Does](#1-what-react-native-actually-does)
2. [App Boot — What Happens When You Open the App](#2-app-boot--what-happens-when-you-open-the-app)
3. [The Old Architecture — The Bridge](#3-the-old-architecture--the-bridge)
4. [Why the Bridge Was a Problem](#4-why-the-bridge-was-a-problem)
5. [The New Architecture — JSI](#5-the-new-architecture--jsi)
6. [Fabric — The New UI Renderer](#6-fabric--the-new-ui-renderer)
7. [TurboModules — Lazy Native Modules](#7-turbomodules--lazy-native-modules)
8. [CodeGen — Type Safety at Build Time](#8-codegen--type-safety-at-build-time)
9. [Hermes — The JavaScript Engine](#9-hermes--the-javascript-engine)
10. [Metro — The Bundler](#10-metro--the-bundler)
11. [The Threading Model](#11-the-threading-model)
12. [Navigation — Mental Model](#12-navigation--mental-model)
13. [Deep Linking — How It Works](#13-deep-linking--how-it-works)
14. [Performance — Root Causes of Problems](#14-performance--root-causes-of-problems)
15. [Storage — Why Three Options Exist](#15-storage--why-three-options-exist)
16. [OTA Updates — What They Can and Can't Do](#16-ota-updates--what-they-cant-and-cant-do)

---

## 1. What React Native Actually Does

React Native is **not a web view** in a native shell. It is not Cordova or PhoneGap. Your React components do not render to HTML — they render to actual native iOS and Android views.

When you write `<View>`, React Native creates a real `UIView` on iOS or an `android.view.View` on Android. When you write `<Text>`, it creates a `UILabel` / `TextView`. The JavaScript describes what should exist; the native layer creates the actual platform UI.

The JavaScript and native layers are two separate worlds that need to communicate. **How they communicate** is the entire architecture story of React Native.

---

## 2. App Boot — What Happens When You Open the App

Understanding the boot sequence is one of the most common interview questions. Here's the actual sequence:

**Step 1 — Native boots first.** The operating system starts the app process. iOS runs `AppDelegate.mm`, Android runs `MainApplication.kt`. These are native files. The JS engine hasn't started yet.

**Step 2 — JS engine starts.** The native layer initializes the JavaScript engine (Hermes in modern RN). The engine is ready to execute JavaScript.

**Step 3 — Bundle loads.** In development, the bundle is fetched from Metro server over HTTP. In production, the pre-built bundle is read from the app binary. The engine starts parsing and executing it.

**Step 4 — index.js executes.** This is the **first JavaScript file to run**. It calls `AppRegistry.registerComponent()`, which tells the native runtime: "here is the root React component to render."

**Step 5 — Native calls runApplication.** The native side calls `AppRegistry.runApplication()`, which triggers the first React render.

**Step 6 — React renders the component tree.** Your components run. A shadow tree (layout tree) is built.

**Step 7 — Yoga calculates layout.** The Yoga engine (a C++ Flexbox implementation) calculates all positions and dimensions.

**Step 8 — Native views are created and displayed.** Actual iOS/Android views are instantiated and rendered to screen.

---

## 3. The Old Architecture — The Bridge

In the original React Native architecture, the JavaScript world and the native world were completely separate. They could not directly access each other's memory or call each other's functions.

Communication happened through a **Bridge** — a message-passing system. When JS wanted to call a native function (like showing a toast, accessing GPS, or creating a native view), it would:

1. Serialize the call and its arguments into a JSON string
2. Send that JSON string across the bridge
3. The native side would receive it, parse the JSON
4. Execute the native function
5. Serialize the result into JSON
6. Send it back across the bridge
7. JS receives and parses it

Every. Single. Call. went through this serialization-deserialization cycle.

---

## 4. Why the Bridge Was a Problem

### Serialization overhead

Converting JavaScript objects to JSON and back is not free. For simple calls it's fast enough. But for things like:
- Gesture tracking (touch events firing 60 times per second)
- Animated values (needing to update every frame)
- Large data transfers (image buffers, audio data)

...the serialization overhead became a significant bottleneck.

### Asynchronous by design

The bridge was **always asynchronous**. You could never get a value from native synchronously. Every native call was essentially a round trip with a delay. This made it impossible to build certain types of tight native integrations.

### Single queue bottleneck

All messages from JS to native and native to JS went through the same bridge queue. Heavy native module usage would congest this queue, causing delays in unrelated operations.

### Gesture and animation jank

The most visible problem was animation and gesture performance. To run a smooth animation, you need to update a value 60 times per second. With the bridge, each update had to cross the thread boundary with serialization. On busy devices, this would cause dropped frames. This is exactly why `useNativeDriver: true` existed — it moved animations to the native side so they wouldn't need to cross the bridge on every frame.

---

## 5. The New Architecture — JSI

JSI stands for **JavaScript Interface**. It is a C++ API that fundamentally changes how JS and native communicate.

Instead of sending JSON messages back and forth, JSI lets JavaScript code **directly hold a reference to a C++ object**. Think of it like a pointer in C — the JavaScript side has a handle that points directly to something in native memory.

When JS calls a method through JSI, it calls directly into C++. No serialization. No message queue. No round trips. It's a direct function call.

### What this enables

**Synchronous calls** — JS can call a native function and get the result back immediately, in the same call, without any async ceremony. This was impossible with the bridge.

**Shared memory** — JS and native can share the same memory buffer without copying it. A camera frame captured in native can be accessed in JS without being serialized first.

**Foundation for everything else** — Fabric and TurboModules are both built on top of JSI. JSI is the layer that makes the new architecture possible.

---

## 6. Fabric — The New UI Renderer

Fabric is the complete rewrite of the React Native UI rendering system, built in C++.

### Old architecture UI rendering

In the old architecture, the shadow tree (the layout tree that Yoga works on) was managed in JavaScript. When React computed a new UI, it would serialize the shadow tree operations and send them across the bridge to the native side, which would then create/update native views.

### New architecture UI rendering

In Fabric, the shadow tree lives in **C++**, not JavaScript. Fabric uses JSI to let the JS side work with C++ shadow nodes directly. Layout is calculated in C++. Native views are created and updated with much tighter integration between JS and native.

### What this improves

- **Synchronous layout** — reading layout information (like element positions) no longer requires an async bridge call
- **Concurrent React support** — Fabric is designed to work with React's concurrent rendering features (like `startTransition`)
- **Reduced memory** — the shadow tree exists once in C++, not mirrored in both JS and native
- **Faster gesture responses** — touch events can be handled with less latency

---

## 7. TurboModules — Lazy Native Modules

In the old architecture, every native module that was registered in the app was **initialized at startup**, even if your app never used it. If you had 50 native modules (which is common once you add various dependencies), all 50 were loaded when the app started, even if only 3 were ever needed.

TurboModules changes this to **lazy initialization**. A module is loaded only when it's first accessed. If the user never accesses a feature that needs a particular native module, that module never loads.

This directly improves **Time to Interactive (TTI)** — the app starts faster because it does less work at startup.

TurboModules also use JSI for communication, so all the benefits of synchronous calls apply to native modules in the new architecture.

---

## 8. CodeGen — Type Safety at Build Time

In the old architecture, the interface between JavaScript and native was defined in two separate places — the JavaScript side and the native side. Nothing enforced that they matched. You could have a type mismatch (JS sends a string, native expects a number) that only crashed at runtime.

CodeGen solves this by making JavaScript TypeScript specs the **single source of truth**. You write a TypeScript spec describing what a native module does and what types it accepts. CodeGen reads this at build time and automatically generates:
- C++ glue code
- Objective-C/Swift headers for iOS
- Java/Kotlin code for Android

Since the native code is generated from the TypeScript spec, they're always in sync. Type mismatches become build errors, not runtime crashes.

---

## 9. Hermes — The JavaScript Engine

A JavaScript engine is the program that reads and executes your JavaScript code. Chrome uses V8. Safari uses JavaScriptCore. React Native originally used JavaScriptCore.

Meta built Hermes specifically for React Native, optimized for mobile constraints.

### The key difference: Ahead-of-Time compilation

V8 and JSC use **Just-In-Time (JIT) compilation** — they profile the running code and compile hot paths to machine code while the app is running. This makes them fast for long-running sessions (like a browser tab) but the startup time is slower because compilation happens at runtime.

Hermes uses **Ahead-of-Time (AOT) compilation** — your JavaScript bundle is compiled to bytecode at build time (when you create the app binary), not at runtime on the device. When the app starts, Hermes loads the pre-compiled bytecode directly. No parsing, no compilation at startup.

### Why this matters for mobile

Mobile devices have less CPU and memory than desktops. JIT compilation is CPU and memory intensive. On a mid-range Android device, JIT can add significant startup time and memory pressure.

Hermes's AOT approach means:
- **Faster startup** — no parse/compile step at launch
- **Lower memory** — no JIT profiling data or compiled machine code stored in memory
- **Smaller memory footprint** — bytecode can be smaller than source JavaScript
- **More predictable GC** — Hermes's garbage collector is tuned for mobile

---

## 10. Metro — The Bundler

Metro is React Native's JavaScript bundler — it's equivalent to webpack for React Native.

When you write your app, you have hundreds of files importing from each other and from `node_modules`. The device (or simulator) can't run hundreds of separate files — it needs one (or a few) bundled files.

Metro's job:
1. **Start from the entry point** (index.js)
2. **Follow every import** and find all the files the app needs
3. **Transform each file** — JSX to JS, TypeScript to JS, modern JS to compatible JS (via Babel)
4. **Bundle them** into a single file (or multiple chunks for RAM bundles)
5. **Serve the bundle** — in development over HTTP, in production included in the binary

Metro also handles hot reloading — watching your source files for changes and pushing updated modules to the running app without a full restart.

---

## 11. The Threading Model

React Native uses three main threads. Understanding which work happens on which thread explains most performance issues.

### JS Thread

This is where all your JavaScript code runs — React rendering, business logic, state management, event handlers. It's a single thread (the event loop from Module 01 applies here). If you block this thread with heavy computation, the entire JavaScript world freezes — no React renders, no event handling.

### Main / UI Thread

This is the native platform's main thread — where native views are created, updated, and drawn. Touch events originate here. UIKit on iOS and the Android View system run here. This thread must remain responsive — if it's blocked, the device becomes completely unresponsive.

In the old architecture, heavy bridge communication would cause delays here. In the new architecture (with JSI and Fabric), the integration is tighter and more efficient.

### Shadow / Layout Thread

Yoga (the C++ Flexbox engine) runs here, calculating the layout of all your views. In the old architecture, the shadow tree (the layout input) was in JavaScript and had to be serialized across. In Fabric (new architecture), the shadow tree is C++ and Yoga runs directly on it.

### Why the threading model matters for you

**Animations:** If an animation's values are computed on the JS thread and sent to the UI thread via the bridge every frame, you'll drop frames when the JS thread is busy. `useNativeDriver: true` moves animation computation to the UI thread. Reanimated's worklets go further — they run entirely on the UI thread.

**Heavy computation:** If you do intensive JavaScript work (sorting/filtering large arrays, complex calculations), you block the JS thread. React can't render, event handlers can't fire, the app feels frozen. Solutions: move work to native modules, break work into chunks with `InteractionManager`, or push to a background thread.

---

## 12. Navigation — Mental Model

Navigation in React Native is fundamentally about managing a **stack of screens**. The stack is a data structure — when you navigate to a new screen, it's pushed on the stack. When you go back, it's popped off. The screen at the top of the stack is what the user sees.

React Navigation manages this stack in JavaScript and coordinates with native navigation primitives. The `NavigationContainer` at the root holds the entire navigation state. Every screen has access to `navigation` (to navigate) and `route` (to read params).

### Why navigation state matters

The navigation state is just a JavaScript object. This means you can:
- Save it (for deep linking restoration)
- Reset it (for logout flows)
- Read it (to know where the user is)
- Manipulate it directly (for complex navigation flows)

### Nested navigators

Real apps have complex navigation — a tab bar where each tab has its own stack, or a drawer that contains a tab bar, etc. Each navigator manages its own state, nested within the parent navigator's state. Understanding nesting is key to building navigation that feels natural on each platform.

---

## 13. Deep Linking — How It Works

Deep linking is the ability for an external URL to open your app and navigate directly to a specific screen. There are two types:

### URI Schemes (custom protocols)

Your app registers a custom URL protocol (like `myapp://`) with the operating system. When any app (browser, email client, another app) opens a URL starting with `myapp://`, the OS opens your app and passes the URL to it.

This works reliably but has a limitation: if the app isn't installed, the URL goes nowhere. There's no fallback to a website.

### Universal Links (iOS) / App Links (Android)

Your app claims ownership of specific HTTPS domains (like `myapp.com`). When a user opens a link to `https://myapp.com/profile/123`, if your app is installed, the OS opens the app directly. If not, the browser opens the website — a seamless fallback.

This requires verifying domain ownership by hosting a special file (`apple-app-site-association` on iOS, `assetlinks.json` on Android) on your server.

### The app lifecycle dimension

Deep links arrive in two situations:

1. **App is already open** — the OS sends an event to your running app with the URL. Your app handles it while already running.
2. **App was closed** — the OS starts your app and passes the URL as part of the launch options. Your app needs to read this initial URL before or during navigation setup.

React Navigation's `linking` configuration handles both cases, but you need to understand both to correctly implement deep linking.

---

## 14. Performance — Root Causes of Problems

Most React Native performance problems come from a few root causes:

### JS Thread blockage

Expensive JavaScript operations (heavy computation, synchronous storage reads, complex sorting) block the JS thread. React can't render, events can't be handled. The fix is always one of: make the operation cheaper, make it async, or move it to native.

### Bridge / JSI call volume

Even with JSI, calling native code in a tight loop has overhead. The classic example: updating an animation value 60 times per second via JS → native. The fix: move the animation logic to run natively (`useNativeDriver`, Reanimated worklets).

### Excessive re-renders

Components re-rendering more than needed wastes JS thread time. Every unnecessary re-render is work that didn't need to happen. The fix: React.memo, stable references (useMemo, useCallback) — but only after profiling confirms the problem.

### FlatList rendering too many items

Rendering a list of 10,000 items means creating 10,000 native views. Even on a powerful device, this is slow. FlatList virtualizes — only the visible items (plus a buffer) exist as native views. Proper FlatList configuration (`getItemLayout`, `windowSize`, `maxToRenderPerBatch`) controls how aggressively this virtualization works.

### Image loading

Large, unoptimized images block render. Solutions: proper `resizeMode`, caching (FastImage), progressive loading, blur hash placeholders.

---

## 15. Storage — Why Three Options Exist

Three common storage solutions exist because they solve different problems:

**AsyncStorage** is the simplest — key-value storage, async API. The problem: it's slow (disk I/O on every read/write), not encrypted, and the community-maintained version has had reliability issues at scale.

**MMKV** is a C++ key-value store from WeChat/Meta. It's synchronous (reads return values immediately, no async/await), extremely fast (memory-mapped files), and widely used in production apps. Not encrypted by default, but encryption is available. Best for app preferences, cache, Zustand persistence.

**SecureStore / Keychain** uses the platform's secure storage — iOS Keychain, Android Keystore. Data is encrypted at rest, protected by the device's security subsystem, and survives app reinstalls on iOS. Slower than MMKV (it goes through the security subsystem). Right for: authentication tokens, passwords, biometric data.

The choice is always: **sensitive?** → SecureStore. **Need sync access or high performance?** → MMKV. **Simple async, non-sensitive?** → AsyncStorage.

---

## 16. OTA Updates — What They Can and Can't Do

Over-the-air updates push a new JavaScript bundle to users' devices without going through the App Store or Play Store. This is possible because the JS bundle is just a file — it's not part of the compiled native binary.

### What OTA CAN update

Everything that lives in the JavaScript bundle: your React components, business logic, styles, assets already included in the bundle. Bug fixes, UI changes, logic changes — all can be pushed instantly.

### What OTA CANNOT update

Anything that's part of the native binary: new native modules, changes to existing native code, new app permissions (camera, location, etc.), changes to `Podfile` or `build.gradle`, native dependency updates. These require a full build and store submission.

### Why this distinction matters in interviews

It shows you understand the architecture — that RN is two worlds (JS and native), and OTA only reaches the JS world. Interviewers often ask this to check architectural understanding, not just "do you know about CodePush."

---

*You've finished Theory 03. Now open `module-03-react-native-architecture.md` for all the code examples and implementation details.*

---

## Code 03 — Examples & Patterns


> **Goal:** Understand every layer of React Native — from app boot to the new JSI architecture, threading model, navigation, performance, storage, and native modules. Your strongest domain — go master-level here.

---

## Table of Contents

1. [App Boot Sequence — File by File](#1-app-boot-sequence--file-by-file)
2. [Old Architecture — The Bridge](#2-old-architecture--the-bridge)
3. [New Architecture — JSI, Fabric, TurboModules](#3-new-architecture--jsi-fabric-turbomodules)
4. [Hermes Engine](#4-hermes-engine)
5. [Metro Bundler](#5-metro-bundler)
6. [Threading Model](#6-threading-model)
7. [Navigation — React Navigation Deep Dive](#7-navigation--react-navigation-deep-dive)
8. [Deep Linking](#8-deep-linking)
9. [Performance Optimization](#9-performance-optimization)
10. [Storage](#10-storage)
11. [Native Modules](#11-native-modules)
12. [Platform-Specific APIs](#12-platform-specific-apis)
13. [Deployment & OTA](#13-deployment--ota)
14. [Interview Cheatsheet](#14-interview-cheatsheet)

---

## 1. App Boot Sequence — File by File

This is one of the most asked questions: "What is the first file that runs in a React Native app?"

### Full boot sequence

```
1. Native layer boots
   iOS:  AppDelegate.mm  →  [ReactNativeDelegate application:didFinishLaunchingWithOptions:]
   Android: MainApplication.kt → ReactNativeHost → ReactInstanceManager

2. JS engine starts
   Hermes (or JSC) loads and parses the JS bundle (or .hbc bytecode)

3. index.js runs ← FIRST JS FILE
   AppRegistry.registerComponent('AppName', () => App)

4. Native calls AppRegistry.runApplication()
   Triggers first render of your root component

5. React renders the component tree
   Each component call → Fiber nodes created → Shadow tree built

6. Layout calculated (Yoga)
   Flexbox layout computed on the Shadow/UI thread

7. Native views created & mounted
   Actual iOS/Android views rendered to screen
```

### index.js — the real entry point

```js
// index.js — this is file #1 in JS land
import { AppRegistry } from 'react-native';
import App from './App';
import { name as appName } from './app.json';

// Registers your root component with the native runtime
// Native side calls AppRegistry.runApplication(appName, ...) to trigger first render
AppRegistry.registerComponent(appName, () => App);
```

### app.json

```json
{
  "name": "MyApp",       // ← must match what AppDelegate/MainApplication uses
  "displayName": "My App"
}
```

### iOS: AppDelegate.mm

```objc
// iOS: AppDelegate.mm
@implementation AppDelegate
- (BOOL)application:(UIApplication *)application
    didFinishLaunchingWithOptions:(NSDictionary *)launchOptions {

  // Creates the React Native bridge / JSI runtime
  self.moduleName = @"MyApp"; // must match app.json "name"

  // Loads the bundle: dev → fetches from Metro (localhost:8081)
  //                  prod → reads from bundle file in app
  return [super application:application
      didFinishLaunchingWithOptions:launchOptions];
}
@end
```

### Android: MainApplication.kt

```kotlin
class MainApplication : Application(), ReactApplication {
  override val reactNativeHost: ReactNativeHost =
    DefaultReactNativeHost(this) {
      override fun getPackages(): List<ReactPackage> =
        PackageList(application).packages.apply {
          // Add your native modules here
          add(MyCustomPackage())
        }
      override fun getJSMainModuleName(): String = "index" // index.js
    }
}
```

---

## 2. Old Architecture — The Bridge

### How it worked

```
JS Thread                  Bridge                    Native/UI Thread
─────────────────          ──────────────────────    ──────────────────
JS runs your React code    Serializes to JSON        Native renders UI
      │                         │                         │
      ├─── setState called ──→  JSON.stringify ──────────→ handleMessage
      │                         │                         │
      ←─── callback/event ───  JSON.parse   ←────────────┤
      │                                                    │
      ├─── Animated.timing ──→  Bridge (async) ──────────→ iOS CAAnimation
```

### Bridge problems

1. **Serialization overhead** — everything converted to JSON. Large data (image buffers, frequent gesture events) kills performance.
2. **Async by default** — you cannot make a synchronous native call. `NativeModules.SomeModule.doThing()` returns a Promise, not a value.
3. **Single bottleneck** — all JS↔Native communication funnels through one bridge. Heavy apps hit bridge congestion.
4. **Gesture jank** — touch/scroll events cross the bridge 60 times per second. Even tiny delays cause dropped frames.

```js
// Old arch: this is ASYNCHRONOUS — you can't get the value synchronously
NativeModules.MyModule.getValue((value) => {
  // must use callback or Promise
  console.log(value);
});
```

---

## 3. New Architecture — JSI, Fabric, TurboModules

### JSI — JavaScript Interface

JSI is a **C++ API** that lets JavaScript hold direct references to C++ host objects. No serialization. No bridge.

```
Old: JS → JSON.stringify → Bridge queue → JSON.parse → Native
New: JS → C++ function call via JSI hostObject → Native (synchronous!)
```

```js
// New arch: can be synchronous via JSI
const value = NativeModules.MyTurboModule.getValue(); // returns value directly
```

```cpp
// C++ side — the host object JS gets a reference to
class MyModule : public jsi::HostObject {
public:
  jsi::Value get(jsi::Runtime& rt, const jsi::PropNameID& name) {
    if (name.utf8(rt) == "getValue") {
      return jsi::Function::createFromHostFunction(rt, name, 0,
        [this](jsi::Runtime& rt, const jsi::Value& thisVal,
               const jsi::Value* args, size_t count) {
          return jsi::Value(42); // direct return!
        });
    }
    return jsi::Value::undefined();
  }
};
```

### Fabric — New UI Renderer

Fabric is the new rendering system built entirely in C++.

```
Old: JS Thread → Shadow Tree (JS) → Bridge → UIManager → Native Views
New: JS Thread → C++ Fabric renderer → Native Views (directly)

Key improvements:
- Shadow tree lives in C++ (not JS) → no serialization for layout
- Layout calculated synchronously on the UI thread
- Concurrent rendering support
- Scroll/gesture events handled without crossing to JS thread first
```

### TurboModules — Lazy Native Modules

Old architecture loaded ALL native modules at startup, even ones you never used.

```js
// Old: all modules loaded eagerly at startup
// New: modules loaded lazily when first accessed

// TurboModule definition (JS side — generated by CodeGen)
export interface Spec extends TurboModule {
  readonly getValue: () => number;
  readonly processData: (data: string) => Promise<string>;
}
export default TurboModuleRegistry.get<Spec>('MyModule');
```

### CodeGen

New arch uses CodeGen to auto-generate the C++/Java/ObjC glue code from TypeScript specs.

```
TypeScript spec file (NativeMyModule.ts)
         ↓
    CodeGen runs at build time
         ↓
Generates: MyModule.h (iOS) + MyModuleJSI.cpp + MyModule.java (Android)
         ↓
Type-safe, no runtime type checking needed
```

### Architecture comparison

| Feature | Old Arch (Bridge) | New Arch (JSI) |
|---------|------------------|----------------|
| Communication | Async JSON | Sync C++ calls |
| Native modules | Eagerly loaded | Lazy (TurboModules) |
| UI renderer | JS Shadow Tree | C++ Fabric |
| Gesture handling | JS thread → bridge | UI thread direct |
| Type safety | Runtime | Build time (CodeGen) |
| React concurrent | Not supported | Supported |

---

## 4. Hermes Engine

### What is Hermes?

Hermes is Meta's custom JavaScript engine optimized for React Native. Available since RN 0.60, **default since RN 0.70**.

### Why Hermes over JavaScriptCore (JSC)?

| Feature | Hermes | JSC |
|---------|--------|-----|
| Compilation | **Ahead-of-time bytecode** | JIT at runtime |
| TTI (Time to Interactive) | **Faster** (no parse step on device) | Slower |
| Memory usage | **Lower** (no JIT profiling data) | Higher |
| Bundle size | **Smaller** (.hbc bytecode) | Larger (.js source) |
| GC | **Optimized, lower pauses** | Standard |

### How Hermes compiles your bundle

```
Development:
  .js source → Metro bundles → served over HTTP to device
  Device runs: Hermes parses and executes .js at runtime

Production:
  .js source → Metro bundles → Hermes compiler → .hbc (bytecode)
  Device runs: Hermes loads .hbc directly (no parse step!)
               ↑ This is the startup speedup
```

### Check if running on Hermes

```js
const isHermes = () => !!global.HermesInternal;
console.log('Running on Hermes:', isHermes());
```

### Enable Hermes (RN < 0.70)

```ruby
# iOS: Podfile
use_react_native!(
  :hermes_enabled => true
)
```

```gradle
// Android: android/app/build.gradle
project.ext.react = [
  enableHermes: true
]
```

---

## 5. Metro Bundler

Metro is RN's JavaScript bundler — equivalent to webpack for web.

### What Metro does

```
Your JS/TS files
       ↓
  Module resolution (resolves imports/requires)
       ↓
  Transformation (Babel — JSX, TypeScript, modern JS → compatible JS)
       ↓
  Bundling (combines all modules into one file)
       ↓
  Development: serves bundle over HTTP (localhost:8081)
  Production:  writes bundle to disk → included in app binary
```

### metro.config.js

```js
const { getDefaultConfig, mergeConfig } = require('@react-native/metro-config');

const config = {
  resolver: {
    // Add file extensions Metro should handle
    assetExts: [...getDefaultConfig(__dirname).resolver.assetExts, 'lottie', 'glb'],
    sourceExts: [...getDefaultConfig(__dirname).resolver.sourceExts, 'mjs'],
  },
  transformer: {
    // Custom transformer (e.g. for SVG files)
    babelTransformerPath: require.resolve('react-native-svg-transformer'),
  },
  watchFolders: [
    // Additional folders to watch (monorepo)
    path.resolve(__dirname, '../../packages'),
  ],
};

module.exports = mergeConfig(getDefaultConfig(__dirname), config);
```

### Bundle splitting / RAM bundles

```js
// RAM bundles: load modules lazily on first require
// Only bundle code needed for initial screen, load rest on demand

// Enable in Android (inline bundles):
// android/app/build.gradle
project.ext.react = [
  bundleInRelease: true,
  extraPackagerArgs: ['--indexed-ram-bundle']
]
```

---

## 6. Threading Model

React Native has 3–4 main threads:

```
┌─────────────────────────────────────────────────┐
│  JS Thread                                       │
│  - Runs your React code                          │
│  - Executes your business logic                  │
│  - Manages state, component rendering            │
│  - Single-threaded (event loop)                  │
└─────────────────────────────────────────────────┘
         ↓ (Bridge/JSI)
┌─────────────────────────────────────────────────┐
│  UI / Main Thread (Native)                       │
│  - Renders actual native views                   │
│  - Handles touch events, gestures                │
│  - iOS: Main thread (UIKit)                      │
│  - Android: Main thread (Android View system)    │
└─────────────────────────────────────────────────┘
         ↓
┌─────────────────────────────────────────────────┐
│  Shadow / Layout Thread (Yoga)                   │
│  - Calculates Flexbox layout                     │
│  - Runs Yoga (C++ Flexbox engine)                │
│  - Old arch: mirrors JS shadow tree              │
│  - New arch (Fabric): runs in C++, more direct   │
└─────────────────────────────────────────────────┘
         ↓
┌─────────────────────────────────────────────────┐
│  Native Module Thread (optional)                 │
│  - Background work for native modules            │
│  - Network requests, file I/O                    │
└─────────────────────────────────────────────────┘
```

### Why threading matters for performance

```js
// This blocks the JS thread — UI becomes unresponsive
function heavyComputation() {
  let result = 0;
  for (let i = 0; i < 100000000; i++) {
    result += i; // blocks JS thread for seconds
  }
  return result;
}

// Fix 1: Move to native module (runs on native module thread)
// Fix 2: Break into chunks with InteractionManager
InteractionManager.runAfterInteractions(() => {
  // Runs after all animations complete, on JS thread but deferred
  performHeavyWork();
});

// Fix 3: Use requestAnimationFrame for chunked work
function chunkedWork(items, chunkSize = 100) {
  let index = 0;
  function processChunk() {
    const chunk = items.slice(index, index + chunkSize);
    chunk.forEach(processItem);
    index += chunkSize;
    if (index < items.length) {
      requestAnimationFrame(processChunk); // yield between chunks
    }
  }
  processChunk();
}
```

---

## 7. Navigation — React Navigation Deep Dive

### Stack structure

```js
// App.tsx
import { NavigationContainer } from '@react-navigation/native';
import { createNativeStackNavigator } from '@react-navigation/native-stack';

// Type your params (TypeScript)
type RootStackParams = {
  Home: undefined;
  Profile: { userId: string; userName: string };
  Settings: { section?: string };
};

const Stack = createNativeStackNavigator<RootStackParams>();

export default function App() {
  return (
    <NavigationContainer>
      <Stack.Navigator
        initialRouteName="Home"
        screenOptions={{
          headerShown: false,
          animation: 'slide_from_right',
        }}
      >
        <Stack.Screen name="Home" component={HomeScreen} />
        <Stack.Screen name="Profile" component={ProfileScreen} />
        <Stack.Screen name="Settings" component={SettingsScreen} />
      </Stack.Navigator>
    </NavigationContainer>
  );
}
```

### Navigate, push, replace, goBack

```js
// Navigate: goes to screen, won't duplicate if already in stack
navigation.navigate('Profile', { userId: '123' });

// Push: always adds new screen even if already in stack
navigation.push('Profile', { userId: '123' });

// Replace: replace current screen (no back button to it)
navigation.replace('Home');

// Go back
navigation.goBack();
navigation.navigate('Home'); // go back multiple levels

// Reset stack (e.g. after login)
navigation.reset({
  index: 0,
  routes: [{ name: 'Home' }],
});
```

### Accessing params

```js
// TypeScript-typed params
function ProfileScreen({ route, navigation }: NativeStackScreenProps<RootStackParams, 'Profile'>) {
  const { userId, userName } = route.params;
  // ...
}

// useRoute hook
import { useRoute } from '@react-navigation/native';
const route = useRoute<RouteProp<RootStackParams, 'Profile'>>();
const { userId } = route.params;
```

### Nested navigators

```js
// Tab navigator containing stack navigators
const Tab = createBottomTabNavigator();
const HomeStack = createNativeStackNavigator();
const ProfileStack = createNativeStackNavigator();

function HomeStackNav() {
  return (
    <HomeStack.Navigator>
      <HomeStack.Screen name="Feed" component={FeedScreen} />
      <HomeStack.Screen name="Post" component={PostScreen} />
    </HomeStack.Navigator>
  );
}

function RootTabs() {
  return (
    <Tab.Navigator>
      <Tab.Screen name="HomeTab" component={HomeStackNav} />
      <Tab.Screen name="ProfileTab" component={ProfileStackNav} />
    </Tab.Navigator>
  );
}

// Navigate into nested: specify nested screen
navigation.navigate('HomeTab', {
  screen: 'Post',
  params: { postId: '123' },
});
```

---

## 8. Deep Linking

### Two types

| Type | Format | How it works |
|------|--------|-------------|
| URI Scheme | `myapp://profile/123` | Custom protocol registered with OS |
| Universal Links (iOS) / App Links (Android) | `https://myapp.com/profile/123` | HTTPS URL that opens app |

### Setup in NavigationContainer

```js
const linking = {
  prefixes: [
    'myapp://',              // URI scheme
    'https://myapp.com',     // Universal/App links
    'https://www.myapp.com',
  ],
  config: {
    screens: {
      Home: 'home',
      Profile: {
        path: 'profile/:userId',  // :userId extracted as route param
        parse: {
          userId: (id: string) => parseInt(id, 10), // transform param type
        },
      },
      Settings: {
        path: 'settings',
        screens: {
          Privacy: 'privacy',       // myapp://settings/privacy
          Notifications: 'notifs',
        },
      },
    },
  },
};

<NavigationContainer linking={linking} fallback={<Splash />}>
  {/* ... */}
</NavigationContainer>
```

### iOS: URI scheme setup

```xml
<!-- ios/MyApp/Info.plist -->
<key>CFBundleURLTypes</key>
<array>
  <dict>
    <key>CFBundleURLName</key>
    <string>com.myapp</string>
    <key>CFBundleURLSchemes</key>
    <array>
      <string>myapp</string>  <!-- opens myapp:// links -->
    </array>
  </dict>
</array>
```

### Android: Intent filter

```xml
<!-- android/app/src/main/AndroidManifest.xml -->
<activity android:name=".MainActivity">
  <intent-filter android:autoVerify="true">
    <action android:name="android.intent.action.VIEW" />
    <category android:name="android.intent.category.DEFAULT" />
    <category android:name="android.intent.category.BROWSABLE" />
    <!-- URI scheme -->
    <data android:scheme="myapp" />
  </intent-filter>

  <intent-filter android:autoVerify="true">
    <action android:name="android.intent.action.VIEW" />
    <category android:name="android.intent.category.DEFAULT" />
    <category android:name="android.intent.category.BROWSABLE" />
    <!-- App links (HTTPS) -->
    <data android:scheme="https" android:host="myapp.com" />
  </intent-filter>
</activity>
```

### Handle incoming links in code

```js
import { Linking } from 'react-native';

// App is OPEN — listen for new deep links
useEffect(() => {
  const subscription = Linking.addEventListener('url', ({ url }) => {
    handleDeepLink(url);
  });
  return () => subscription.remove();
}, []);

// App was CLOSED — check initial URL that opened app
useEffect(() => {
  Linking.getInitialURL().then(url => {
    if (url) handleDeepLink(url);
  });
}, []);

// Open a URL programmatically
Linking.openURL('myapp://profile/123');
Linking.openURL('https://maps.google.com/?q=Bengaluru');
```

### Testing deep links

```bash
# iOS Simulator
npx uri-scheme open myapp://profile/123 --ios

# Android Emulator
adb shell am start -W -a android.intent.action.VIEW \
  -d "myapp://profile/123" com.myapp

# Universal links (iOS) — requires AASA file on server
```

---

## 9. Performance Optimization

### FlatList — full optimization

```jsx
<FlatList
  data={items}
  keyExtractor={(item) => item.id.toString()}

  // If all items same height: enables accurate scroll-to-index
  // Skips async measurement → smoother initial render
  getItemLayout={(data, index) => ({
    length: ITEM_HEIGHT,        // item height
    offset: ITEM_HEIGHT * index, // distance from top
    index,
  })}

  // How many items to render initially
  initialNumToRender={10}

  // Max items to render per JS frame
  maxToRenderPerBatch={5}

  // Render window: screen height × windowSize rendered above/below
  windowSize={10}  // 5 viewports above, 5 below

  // Unmount items far outside render window (Android)
  removeClippedSubviews={true}

  // Perf: skip items where layout doesn't change
  initialScrollIndex={0}

  // Avoid anonymous inline functions in renderItem
  renderItem={renderItem}  // defined outside component or memoized

  // Avoid creating new array every render
  // Pass data directly from state, don't filter inline
/>

// Define outside or memoize:
const renderItem = useCallback(({ item }) => (
  <MemoizedItem item={item} />
), []);
```

### Animated API — useNativeDriver

```js
const opacity = useRef(new Animated.Value(0)).current;

// Without useNativeDriver: animation calculated on JS thread
// Every frame: JS → Bridge → UI thread = potential jank
Animated.timing(opacity, {
  toValue: 1,
  duration: 300,
  useNativeDriver: false, // BAD for opacity/transform
}).start();

// With useNativeDriver: animation runs entirely on UI thread
// JS thread is completely free
Animated.timing(opacity, {
  toValue: 1,
  duration: 300,
  useNativeDriver: true, // GOOD
}).start();

// Works with useNativeDriver: true
// ✅ opacity, transform (translateX, translateY, scale, rotate)
// ❌ width, height, left, top, margin, padding (layout properties)
```

### Reanimated 2 — worklets

```js
import Animated, {
  useSharedValue,
  useAnimatedStyle,
  withSpring,
  withTiming,
  runOnJS,   // call JS thread function from UI thread
} from 'react-native-reanimated';

function Card() {
  const scale = useSharedValue(1);
  const translateX = useSharedValue(0);

  // This function runs on the UI thread (worklet)
  const animatedStyle = useAnimatedStyle(() => {
    return {
      transform: [
        { scale: scale.value },
        { translateX: translateX.value },
      ],
    };
  }); // no dependency array needed — auto-tracks .value reads

  function handlePress() {
    // Update from JS thread — animates on UI thread
    scale.value = withSpring(0.95, { damping: 15 });
    setTimeout(() => {
      scale.value = withSpring(1);
    }, 150);
  }

  return (
    <Animated.View style={[styles.card, animatedStyle]}>
      <TouchableOpacity onPress={handlePress}>
        <Text>Press me</Text>
      </TouchableOpacity>
    </Animated.View>
  );
}
```

### Performance checklist

```
□ FlatList: getItemLayout, keyExtractor, memoized renderItem
□ Animations: useNativeDriver:true or Reanimated
□ Images: @shopify/flash-list over FlatList for complex lists
□ Images: proper resizeMode, cached with @d11/react-native-fast-image
□ Navigation: lazy load tab screens (lazy: true)
□ JS: avoid heavy sync operations on JS thread (use InteractionManager)
□ Re-renders: React.memo, useMemo, useCallback where measured
□ Bundle: enable Hermes, RAM bundles, code splitting
□ Native: move heavy computation to native modules
□ Profiling: Flipper → Performance, React DevTools Profiler
```

---

## 10. Storage

### Three main options

```js
// 1. AsyncStorage — async, plain text, community-maintained
// Use for: simple key-value, user preferences
import AsyncStorage from '@react-native-async-storage/async-storage';

await AsyncStorage.setItem('user', JSON.stringify(userData));
const raw = await AsyncStorage.getItem('user');
const user = raw ? JSON.parse(raw) : null;
await AsyncStorage.removeItem('user');
await AsyncStorage.clear(); // nuclear option

// 2. MMKV — synchronous, C++ backed, 10x faster than AsyncStorage
// Use for: frequent reads/writes, replacing AsyncStorage
import { MMKV } from 'react-native-mmkv';

const storage = new MMKV();
storage.set('counter', 42);              // sync
const count = storage.getNumber('counter'); // sync
storage.set('user', JSON.stringify(userData));
const user = JSON.parse(storage.getString('user') ?? '{}');
storage.delete('counter');

// MMKV with Zustand
const useStore = create(
  persist(
    (set) => ({ count: 0, increment: () => set(s => ({ count: s.count + 1 })) }),
    {
      name: 'app-storage',
      storage: createJSONStorage(() => ({
        getItem: (key) => storage.getString(key) ?? null,
        setItem: (key, value) => storage.set(key, value),
        removeItem: (key) => storage.delete(key),
      })),
    }
  )
);

// 3. SecureStore (Expo) / Keychain — encrypted
// Use for: JWT tokens, passwords, sensitive data
import * as SecureStore from 'expo-secure-store';

await SecureStore.setItemAsync('authToken', token);
const token = await SecureStore.getItemAsync('authToken');
await SecureStore.deleteItemAsync('authToken');

// iOS: Keychain Services
// Android: Android Keystore / EncryptedSharedPreferences
```

### Decision tree

```
Is data sensitive? (tokens, passwords)
  → YES: SecureStore / react-native-keychain

Do you need sync access? (zustand persist, fast reads)
  → YES: MMKV

Simple async key-value? (settings, cache)
  → AsyncStorage

Relational/complex queries?
  → WatermelonDB / SQLite (op-sqlite)

Large files?
  → react-native-fs (file system)
```

---

## 11. Native Modules

### TurboModule (New Arch)

```typescript
// NativeMyModule.ts — CodeGen spec
import { TurboModule, TurboModuleRegistry } from 'react-native';

export interface Spec extends TurboModule {
  getValue(): number;
  processString(input: string): Promise<string>;
  addListener(eventName: string): void;
  removeListeners(count: number): void;
}

export default TurboModuleRegistry.get<Spec>('MyModule');
```

```swift
// iOS: MyModule.swift
@objc(MyModule)
class MyModule: NSObject {
  @objc func getValue() -> NSNumber {
    return 42
  }

  @objc func processString(_ input: String,
    resolve: @escaping RCTPromiseResolveBlock,
    reject: @escaping RCTPromiseRejectBlock) {
    resolve(input.uppercased())
  }
}
```

### NativeEventEmitter — sending events to JS

```swift
// iOS: send event from native to JS
RCTEventEmitter.supportedEvents() // declare supported events

// In native code:
sendEvent(withName: "locationUpdate", body: ["lat": 37.7, "lng": -122.4])
```

```js
// JS: subscribe to native event
import { NativeEventEmitter, NativeModules } from 'react-native';

const emitter = new NativeEventEmitter(NativeModules.LocationModule);

useEffect(() => {
  const subscription = emitter.addListener('locationUpdate', (location) => {
    console.log(location); // { lat: 37.7, lng: -122.4 }
  });
  return () => subscription.remove(); // always clean up
}, []);
```

---

## 12. Platform-Specific APIs

### Platform.OS and Platform.select

```js
import { Platform } from 'react-native';

Platform.OS === 'ios'     // 'ios' | 'android' | 'web'
Platform.Version          // iOS: '16.2', Android: 31 (API level)

// Platform.select — cleaner than if/else
const styles = StyleSheet.create({
  container: {
    paddingTop: Platform.select({
      ios: 44,
      android: 24,
      default: 0,
    }),
  },
});

// Platform-specific files (Metro resolves automatically)
// MyComponent.ios.tsx   — used on iOS
// MyComponent.android.tsx — used on Android
// MyComponent.tsx       — fallback
```

### SafeAreaView — notch/dynamic island handling

```js
import { SafeAreaView, SafeAreaProvider } from 'react-native-safe-area-context';

// Wrap root in Provider
function App() {
  return (
    <SafeAreaProvider>
      <NavigationContainer>
        {/* ... */}
      </NavigationContainer>
    </SafeAreaProvider>
  );
}

// Use in screens (or use useSafeAreaInsets for custom padding)
function Screen() {
  const insets = useSafeAreaInsets();
  return (
    <View style={{ flex: 1, paddingTop: insets.top, paddingBottom: insets.bottom }}>
      {/* content */}
    </View>
  );
}
```

### Keyboard handling

```js
import { Keyboard, KeyboardAvoidingView, Platform } from 'react-native';

// Dismiss keyboard
Keyboard.dismiss();

// Listen to keyboard events
useEffect(() => {
  const show = Keyboard.addListener('keyboardDidShow', (e) => {
    setKeyboardHeight(e.endCoordinates.height);
  });
  const hide = Keyboard.addListener('keyboardDidHide', () => {
    setKeyboardHeight(0);
  });
  return () => { show.remove(); hide.remove(); };
}, []);

// KeyboardAvoidingView — adjusts scroll/layout when keyboard appears
<KeyboardAvoidingView
  behavior={Platform.OS === 'ios' ? 'padding' : 'height'}
  style={{ flex: 1 }}
>
  <ScrollView>
    {/* form fields */}
  </ScrollView>
</KeyboardAvoidingView>
```

### Dimensions & useWindowDimensions

```js
import { Dimensions, useWindowDimensions } from 'react-native';

// Static — doesn't update on rotation
const { width, height } = Dimensions.get('window'); // screen excluding nav bars
const { width: sw, height: sh } = Dimensions.get('screen'); // full screen

// Hook — updates reactively on rotation/split-screen
function ResponsiveComponent() {
  const { width, height, scale, fontScale } = useWindowDimensions();
  const isTablet = width >= 768;
  return <View style={{ flexDirection: isTablet ? 'row' : 'column' }} />;
}
```

---

## 13. Deployment & OTA

### Build types

```
Debug build
  - Metro dev server
  - Dev tools enabled (Flipper, error overlays)
  - Faster builds, larger binary

Release build
  - Bundle pre-built and included in binary
  - ProGuard/Hermes optimizations
  - What goes to App Store / Play Store

Staging/Beta
  - Release build but pointing to staging APIs
  - Distributed via TestFlight (iOS) or Firebase App Distribution
```

### EAS Build (Expo) — CI/CD

```bash
# Install EAS CLI
npm install -g eas-cli

# Configure project
eas build:configure

# Build for testing
eas build --profile preview --platform ios

# Build for store submission
eas build --profile production --platform all

# Submit to stores
eas submit --platform ios
eas submit --platform android
```

### OTA Updates — what you can and can't change

```
OTA-safe changes (JS bundle only):
  ✅ Bug fixes in JS/React code
  ✅ UI changes (layout, colors, text)
  ✅ Business logic changes
  ✅ New screens added in JS
  ✅ Asset changes (images already in bundle)

Requires full store release:
  ❌ New native modules added
  ❌ Changes to Podfile (iOS) or build.gradle (Android)
  ❌ App permissions changes (camera, location, etc.)
  ❌ New native dependencies
  ❌ iOS binary changes
```

```bash
# EAS Update (OTA for Expo)
eas update --branch production --message "Fix login crash"

# CodePush OTA (non-Expo)
appcenter codepush release-react -a MyOrg/MyApp-iOS \
  -d Production --description "Fix login crash"
```

---

## 14. Interview Cheatsheet

### Must-know answers

**"What is the first file that runs in a React Native app?"**
> On the JS side, `index.js`. It calls `AppRegistry.registerComponent()`. The native side (AppDelegate on iOS, MainApplication on Android) boots the JS engine and calls `AppRegistry.runApplication()` to start the first render.

**"Old architecture vs new architecture?"**
> Old: JS and native communicated via an async JSON bridge — slow, serialization overhead, gesture jank. New: JSI (JavaScript Interface) lets JS hold direct C++ references → synchronous native calls, no serialization. Fabric is the C++ UI renderer. TurboModules are lazily loaded. CodeGen generates type-safe glue code.

**"What is Hermes?"**
> Meta's JS engine for RN. Pre-compiles JS to bytecode at build time → faster TTI, smaller memory footprint, lower GC pauses. Default since RN 0.70.

**"What is JSI?"**
> JavaScript Interface — C++ API that lets JS hold a direct reference to a C++ host object. Enables synchronous native calls without JSON serialization. Foundation of the new architecture.

**"How does deep linking work?"**
> URI scheme or Universal/App Links configured in Info.plist/AndroidManifest. `NavigationContainer` linking config maps URL paths to screen names and extracts params. `Linking.addEventListener('url')` handles links when app is open; `Linking.getInitialURL()` handles the cold-start case.

**"Why is useNativeDriver important?"**
> Without it, animation values are computed on the JS thread and sent to the UI thread every frame via the bridge — potential 60 fps bottleneck. With it, animation runs entirely on the UI thread. JS thread is free. Works only with non-layout properties (opacity, transform).

**"AsyncStorage vs MMKV vs SecureStore?"**
> AsyncStorage: async, plain text, slow. MMKV: synchronous C++, 10x faster, not encrypted. SecureStore: encrypted, uses Keychain/Keystore, for sensitive data like tokens.

**"What is Metro?"**
> React Native's JS bundler. Resolves imports, transforms via Babel, bundles to single file. In dev: serves bundle over HTTP. In prod: bundle baked into app binary.

**"When does OTA update NOT work?"**
> Any native code change — new native module, Podfile/Gradle change, new permissions, binary changes. OTA only updates the JS bundle.

### Quick-fire comparison table

| Concept | Old Arch | New Arch |
|---------|----------|----------|
| JS ↔ Native comm | Async JSON bridge | Synchronous JSI C++ |
| UI rendering | JS Shadow Tree | C++ Fabric |
| Native modules | All loaded at startup | Lazy TurboModules |
| Type safety | Runtime checks | Build-time CodeGen |
| Concurrent React | Not supported | Fully supported |
| Gesture handling | JS thread → bridge | UI thread direct |

---

*Next: Module 04 — TypeScript Production Patterns*

---

# Module 04 — TypeScript Production Patterns

> **Goal:** Move beyond basic typing. Understand the type system internals, generics, utility types, and patterns that separate junior from senior TS devs.

---

## Table of Contents

1. [Type System Fundamentals](#1-type-system-fundamentals)
2. [Generics Deep Dive](#2-generics-deep-dive)
3. [Utility Types — All of Them](#3-utility-types--all-of-them)
4. [Advanced Types](#4-advanced-types)
5. [React Native with TypeScript](#5-react-native-with-typescript)
6. [Interview Cheatsheet](#6-interview-cheatsheet)

---

## 1. Type System Fundamentals

### Structural typing ("duck typing")

TypeScript uses **structural typing** — if it has the right shape, it's compatible.

```ts
interface Point { x: number; y: number; }

function plot(p: Point) { console.log(p.x, p.y); }

const p3D = { x: 1, y: 2, z: 3 }; // has extra property
plot(p3D); // ✅ works — it has x and y (structural match)

// Contrast with nominal typing (Java/C#): only the declared type works
```

### Type narrowing

```ts
function process(value: string | number) {
  if (typeof value === 'string') {
    // TypeScript knows: value is string here
    return value.toUpperCase();
  }
  // TypeScript knows: value is number here
  return value.toFixed(2);
}

// instanceof narrowing
function handleError(error: Error | string) {
  if (error instanceof Error) {
    return error.message; // Error type
  }
  return error; // string type
}

// in operator narrowing
interface Cat { meow(): void; }
interface Dog { bark(): void; }

function makeSound(animal: Cat | Dog) {
  if ('meow' in animal) {
    animal.meow(); // Cat
  } else {
    animal.bark(); // Dog
  }
}
```

### Discriminated unions (critical pattern)

```ts
// Each variant has a unique literal 'type' field — the discriminant
type Action =
  | { type: 'INCREMENT'; amount: number }
  | { type: 'DECREMENT'; amount: number }
  | { type: 'RESET' }
  | { type: 'SET_USER'; user: User };

function reducer(state: State, action: Action): State {
  switch (action.type) {
    case 'INCREMENT':
      return { ...state, count: state.count + action.amount }; // amount available
    case 'RESET':
      return initialState; // no amount field
    case 'SET_USER':
      return { ...state, user: action.user }; // user available
    default:
      return state;
  }
}

// API response pattern
type ApiResult<T> =
  | { status: 'success'; data: T }
  | { status: 'error'; error: string }
  | { status: 'loading' };

function render<T>(result: ApiResult<T>) {
  if (result.status === 'success') {
    return result.data; // T
  }
  if (result.status === 'error') {
    return result.error; // string
  }
  return null; // loading
}
```

### Type guards

```ts
// Custom type guard — returns type predicate
function isUser(value: unknown): value is User {
  return (
    typeof value === 'object' &&
    value !== null &&
    'id' in value &&
    typeof (value as User).id === 'string'
  );
}

function processData(data: unknown) {
  if (isUser(data)) {
    console.log(data.id); // TypeScript knows it's User
  }
}

// Assertion function
function assertDefined<T>(val: T | undefined): asserts val is T {
  if (val === undefined) throw new Error('Value is undefined');
}

const maybeUser: User | undefined = getUser();
assertDefined(maybeUser);
console.log(maybeUser.id); // TypeScript knows it's defined
```

---

## 2. Generics Deep Dive

### Basic generics

```ts
// Generic function
function identity<T>(value: T): T {
  return value;
}
identity<string>('hello'); // T = string
identity(42);              // T inferred as number

// Generic with constraint
function getProperty<T, K extends keyof T>(obj: T, key: K): T[K] {
  return obj[key];
}

const user = { id: '1', name: 'Aashi', age: 25 };
getProperty(user, 'name'); // string
getProperty(user, 'age');  // number
getProperty(user, 'foo');  // TS error: 'foo' not in user
```

### Generic constraints

```ts
interface Identifiable { id: string; }

function findById<T extends Identifiable>(items: T[], id: string): T | undefined {
  return items.find(item => item.id === id);
}

// Must have id: string — works with User, Post, Product, etc.
findById(users, '123');  // returns User | undefined
findById(posts, '456');  // returns Post | undefined
```

### Default generics

```ts
interface ApiResponse<T = unknown> {
  data: T;
  status: number;
  message: string;
}

const r1: ApiResponse = { data: 'anything', status: 200, message: 'OK' }; // T = unknown
const r2: ApiResponse<User> = { data: user, status: 200, message: 'OK' }; // T = User
```

---

## 3. Utility Types — All of Them

```ts
interface User {
  id: string;
  name: string;
  email: string;
  age: number;
  role: 'admin' | 'user';
}

// Partial — all fields optional
type PartialUser = Partial<User>;
// { id?: string; name?: string; email?: string; ... }

// Required — all fields required (removes ?)
type RequiredUser = Required<PartialUser>;
// same as User

// Pick — select specific fields
type UserPreview = Pick<User, 'id' | 'name'>;
// { id: string; name: string; }

// Omit — remove specific fields
type UserWithoutId = Omit<User, 'id'>;
// { name: string; email: string; age: number; role: ... }

// Record — key-value map with typed keys and values
type UserMap = Record<string, User>;
// { [key: string]: User }

type RolePermissions = Record<User['role'], string[]>;
// { admin: string[]; user: string[] }

// ReturnType — extract return type of function
function fetchUser(id: string): Promise<User> { /* */ }
type FetchUserReturn = ReturnType<typeof fetchUser>; // Promise<User>
type UserType = Awaited<ReturnType<typeof fetchUser>>; // User

// Parameters — extract parameter types of function
type FetchParams = Parameters<typeof fetchUser>; // [id: string]

// NonNullable — remove null and undefined
type MaybeUser = User | null | undefined;
type DefiniteUser = NonNullable<MaybeUser>; // User

// Extract — keep types assignable to U
type StringOrNumber = string | number | boolean;
type OnlyStrings = Extract<StringOrNumber, string>; // string

// Exclude — remove types assignable to U
type NotString = Exclude<StringOrNumber, string>; // number | boolean

// Readonly — make all fields readonly
type ImmutableUser = Readonly<User>;
// Can't reassign any field

// ReadonlyArray — immutable array
const ids: ReadonlyArray<string> = ['1', '2', '3'];
// ids.push('4'); // TS error

// Awaited — unwrap Promise type (TS 4.5+)
type ResolvedUser = Awaited<Promise<User>>; // User
type DeepResolved = Awaited<Promise<Promise<User>>>; // User
```

---

## 4. Advanced Types

### Conditional types

```ts
// T extends U ? X : Y
type IsArray<T> = T extends any[] ? true : false;
IsArray<string[]> // true
IsArray<string>   // false

// Distribute over union
type Unwrap<T> = T extends Promise<infer R> ? R : T;
Unwrap<Promise<User>> // User
Unwrap<string>        // string (not a Promise)

// infer — extract type from a pattern
type ReturnTypeOf<T> = T extends (...args: any[]) => infer R ? R : never;
ReturnTypeOf<() => User>           // User
ReturnTypeOf<(id: string) => void> // void
```

### Mapped types

```ts
// Transform every field in a type
type Optional<T> = { [K in keyof T]?: T[K] };  // same as Partial<T>
type Nullable<T> = { [K in keyof T]: T[K] | null };
type Stringify<T> = { [K in keyof T]: string };

// Filter fields by type
type OnlyStrings<T> = {
  [K in keyof T as T[K] extends string ? K : never]: T[K]
};

interface Mixed { id: string; count: number; name: string; active: boolean; }
type StringFields = OnlyStrings<Mixed>; // { id: string; name: string; }
```

### Template literal types

```ts
type EventName = 'click' | 'focus' | 'blur';
type Handler = `on${Capitalize<EventName>}`;
// 'onClick' | 'onFocus' | 'onBlur'

type Route = '/users' | '/posts' | '/settings';
type ApiRoute = `/api/v1${Route}`;
// '/api/v1/users' | '/api/v1/posts' | '/api/v1/settings'
```

---

## 5. React Native with TypeScript

### Typing navigation (React Navigation)

```ts
import { NativeStackScreenProps } from '@react-navigation/native-stack';

// Define all params in one place
type RootStackParamList = {
  Home: undefined;
  Profile: { userId: string; userName?: string };
  Modal: { message: string };
};

// Screen props
type ProfileScreenProps = NativeStackScreenProps<RootStackParamList, 'Profile'>;

function ProfileScreen({ route, navigation }: ProfileScreenProps) {
  const { userId, userName } = route.params; // fully typed
  navigation.navigate('Home');               // typed — can't navigate to unknown screen
}

// Hook versions
import { useNavigation, useRoute, RouteProp } from '@react-navigation/native';

function Component() {
  const nav = useNavigation<NativeStackNavigationProp<RootStackParamList>>();
  const route = useRoute<RouteProp<RootStackParamList, 'Profile'>>();
}
```

### Typing component props

```ts
// Props with children
interface CardProps {
  title: string;
  subtitle?: string;
  onPress: () => void;
  style?: ViewStyle;
  children: React.ReactNode;
}

// Generic list component
interface ListProps<T> {
  data: T[];
  keyExtractor: (item: T) => string;
  renderItem: (item: T) => React.ReactElement;
  onEndReached?: () => void;
}

function TypedList<T>({ data, renderItem, keyExtractor }: ListProps<T>) {
  return (
    <FlatList
      data={data}
      renderItem={({ item }) => renderItem(item)}
      keyExtractor={keyExtractor}
    />
  );
}
```

---

## 6. Interview Cheatsheet

**"What is structural typing?"** → Types are compatible if they have the same shape, not the same name. An object with `{x, y, z}` is assignable to `{x, y}`.

**"What is a discriminated union?"** → A union where each member has a unique literal field (`type`, `kind`). TypeScript narrows based on that field — enables exhaustive switch/if checking.

**"Difference between `type` and `interface`?"**
- `interface`: can be extended, can be merged (declaration merging), better for objects
- `type`: can represent unions, intersections, primitives, tuples — more flexible
- Prefer `interface` for object shapes, `type` for complex type algebra

**"What does `infer` do?"** → Inside conditional types, `infer R` tells TypeScript to capture a type. Like pattern matching on types.

---

# Module 05 — Testing Strategy

> **Goal:** Know how to test React Native apps at unit, integration, and E2E level.

---

## Jest Setup

```js
// jest.config.js
module.exports = {
  preset: 'react-native',
  setupFilesAfterFramework: ['@testing-library/jest-native/extend-expect'],
  moduleNameMapper: {
    '\\.svg': '<rootDir>/__mocks__/svgMock.js',
  },
  transformIgnorePatterns: [
    'node_modules/(?!(react-native|@react-native|react-navigation|...)/)',
  ],
};
```

## Component Testing (React Testing Library)

```tsx
import { render, screen, fireEvent, waitFor } from '@testing-library/react-native';

describe('LoginForm', () => {
  it('calls onSubmit with credentials when form is submitted', async () => {
    const mockSubmit = jest.fn();
    render(<LoginForm onSubmit={mockSubmit} />);

    fireEvent.changeText(screen.getByPlaceholderText('Email'), 'aashi@test.com');
    fireEvent.changeText(screen.getByPlaceholderText('Password'), 'secret123');
    fireEvent.press(screen.getByText('Login'));

    await waitFor(() => {
      expect(mockSubmit).toHaveBeenCalledWith({
        email: 'aashi@test.com',
        password: 'secret123',
      });
    });
  });

  it('shows error when email is invalid', async () => {
    render(<LoginForm onSubmit={jest.fn()} />);
    fireEvent.changeText(screen.getByPlaceholderText('Email'), 'not-an-email');
    fireEvent.press(screen.getByText('Login'));

    await waitFor(() => {
      expect(screen.getByText('Invalid email address')).toBeTruthy();
    });
  });
});
```

## Mocking

```ts
// Mock a module
jest.mock('@react-native-async-storage/async-storage', () => ({
  getItem: jest.fn(() => Promise.resolve(null)),
  setItem: jest.fn(() => Promise.resolve()),
}));

// Mock navigation
const mockNavigate = jest.fn();
jest.mock('@react-navigation/native', () => ({
  ...jest.requireActual('@react-navigation/native'),
  useNavigation: () => ({ navigate: mockNavigate }),
}));

// Mock fetch
global.fetch = jest.fn(() =>
  Promise.resolve({
    ok: true,
    json: () => Promise.resolve({ id: '1', name: 'Aashi' }),
  })
) as jest.Mock;
```

## Hook Testing

```ts
import { renderHook, act } from '@testing-library/react-native';
import { useCounter } from './useCounter';

test('increments counter', () => {
  const { result } = renderHook(() => useCounter());
  expect(result.current.count).toBe(0);

  act(() => result.current.increment());
  expect(result.current.count).toBe(1);
});
```

---

# Module 06 — System Design Cheatsheet (FE/Mobile)

> Quick reference for common mobile system design interview questions.

---

## Design a news feed (Twitter/Instagram)

```
Key decisions:
1. Pagination: cursor-based (not offset) — avoids duplicate/missing items as feed updates
2. Optimistic UI: show post immediately, revert if API fails
3. Virtualization: FlashList / FlatList with getItemLayout
4. Image loading: progressive, blur hash placeholder, cache headers
5. Real-time: WebSocket for new posts, poll for counts (likes, comments)
6. Offline: cache last N posts in MMKV, show stale + sync on reconnect

State shape:
{
  feed: { ids: string[], byId: { [id]: Post }, nextCursor: string | null },
  loading: boolean,
  error: string | null,
}
```

## Design offline-first todo app

```
Key decisions:
1. Local-first: write to local DB (WatermelonDB/SQLite) immediately
2. Sync: background sync when network available — conflict resolution (last-write-wins or CRDT)
3. Queue: failed mutations go to a retry queue (MMKV-backed)
4. Indicators: show sync status (synced ✓, pending ↑, error !)
5. Auth: token refresh handled separately — queue mutations if token expired

Sync algorithm:
1. User creates todo → write to local DB, add to sync queue
2. Network available → flush queue → PATCH/POST to API
3. API returns canonical ID → update local record
4. Pull changes from server (since last sync timestamp)
5. Merge: server wins for deletions, client wins for new items
```

## Design auth with refresh tokens

```
Storage:
- Access token: MMKV (short-lived, 15 min)
- Refresh token: SecureStore/Keychain (long-lived, 30 days)

Flow:
1. API call with access token in Authorization header
2. 401 response → try refresh (POST /auth/refresh with refresh token)
3. Refresh success → new tokens → retry original request
4. Refresh fail → clear tokens → redirect to Login

Queue: while refreshing, queue other 401 requests, replay after refresh
Interceptor pattern (Axios):
  - Request interceptor: attach access token
  - Response interceptor: handle 401 → refresh → retry
```

---

*This completes the core FE + Mobile study guide. Revisit each module and practice explaining every concept out loud — that's how you crack the interview.*
