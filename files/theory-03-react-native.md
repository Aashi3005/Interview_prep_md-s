# Theory 03 — React Native Architecture

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
