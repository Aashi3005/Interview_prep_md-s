# Theory 01 — JavaScript Internals

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
