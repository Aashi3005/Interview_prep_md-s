# Theory 02 — React Core & Fiber Architecture

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

**Fiber React ka internal engine hai jo decide karta hai — kab, kya, aur kis order mein update karna hai. Basically fiber is an engine which efficiently updates the Virtual DOM**

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
