# React Native Architecture Masterclass
## PHASE 8 & 9: Hermes JavaScript Engine & Rendering Pipeline

---

## PHASE 8: Hermes JavaScript Engine

### 8.1 What is a JavaScript Engine?

**The Problem:**

JavaScript is text. Computers only understand machine code (0s and 1s).

```
JavaScript Source Code (Text)
  ↓ ??? (Magic happens)
  ↓
Machine Code (CPU instructions)
```

**JavaScript Engine = The translator**

Converts JavaScript text → Machine code → Execution

**Major JavaScript Engines:**

| Engine | Used In | Language |
|--------|---------|----------|
| V8 | Chrome, Node.js | C++ |
| JavaScriptCore (JSC) | Safari, iOS | C++ |
| SpiderMonkey | Firefox | C++ |
| Hermes | React Native | C++ |

---

### 8.2 V8 (Chrome's Engine)

**V8 Overview:**

```
SOURCE CODE: "var x = 5 + 3;"
       ↓
PARSING:
  ├─ Tokenization: ["var", "x", "=", "5", "+", "3", ";"]
  ├─ Syntax Analysis: Valid JavaScript? ✓
  └─ Build AST (Abstract Syntax Tree)

AST:
  VariableDeclaration
    └─ BinaryExpression
        ├─ left: 5
        ├─ operator: +
        └─ right: 3

COMPILATION:
  ├─ Baseline Compiler (Ignition): Convert to bytecode
  ├─ Optimizing Compiler (TurboFan): Convert to machine code
  └─ Inline Caches: Remember what types are used

EXECUTION:
  ├─ Run bytecode/machine code
  ├─ Monitor: What code runs hot?
  ├─ Optimize: Compile hot code further
  └─ Deoptimize: If types change, revert to bytecode

RESULT: x = 8
```

---

### 8.3 JavaScriptCore (JSC)

**JSC (Apple's Engine):**

Used in:
- Safari
- iOS apps (all apps use JSC for JS execution)
- React Native on iOS

**Similar to V8:**
```
Source → Lexer → Parser → AST → Compilation → Execution
```

**Differences from V8:**

```
V8: Focuses on web (optimize for website performance)
JSC: Focuses on mobile (optimize for battery & memory)

V8: JIT (compile during execution)
JSC: Tiered compilation (baseline → optimized)

V8: Larger binary size
JSC: Smaller, built-in to iOS
```

---

### 8.4 Hermes: Optimized for React Native

**Why Meta Created Hermes:**

V8 and JSC were designed for:
- Websites (V8)
- Safari (JSC)

Not optimized for:
- Mobile startup
- Limited memory
- Battery consumption

**Hermes Optimizations:**

```
V8:             Hermes:
├─ JIT             ├─ AOT (Ahead-of-Time)
├─ Large binary    ├─ Smaller binary
├─ High memory     ├─ Low memory
└─ Slow startup    └─ Fast startup
```

---

### 8.5 JavaScript Execution Flow (Deep Dive)

**Step 1: Parsing**

```
SOURCE CODE:
const add = (a, b) => a + b;
const result = add(5, 3);

TOKENIZATION:
[const, add, =, (, a, ,, b, ), =>, a, +, b, ;, const, result, =, add, (, 5, ,, 3, ), ;]

SYNTAX ANALYSIS:
Is this valid JS? ✓

BUILD AST:
Program
├─ VariableDeclaration (const add)
│  └─ ArrowFunction
│     ├─ params: [a, b]
│     └─ body: BinaryExpression (a + b)
└─ VariableDeclaration (const result)
   └─ CallExpression
      ├─ callee: Identifier (add)
      └─ arguments: [5, 3]
```

**Memory Allocation (Parsing):**

```
┌──────────────────────────────┐
│ HEAP                         │
├──────────────────────────────┤
│ AST nodes (objects)          │
│ String values ("add", "a")   │
│ Scope information            │
└──────────────────────────────┘

Each AST node = Object in memory
```

**Step 2: Compilation to Bytecode**

**What is Bytecode?**

Intermediate code between source and machine code.

```
SOURCE: x = 5 + 3
  ↓
BYTECODE:
  LOAD_CONSTANT 5
  LOAD_CONSTANT 3
  ADD
  STORE_VARIABLE x

MACHINE CODE:
  mov eax, 5          ; Load 5 into register
  mov ebx, 3          ; Load 3 into register
  add eax, ebx        ; Add them
  mov [esp], eax      ; Store in variable x
```

**Bytecode Benefits:**

```
Not as fast as machine code (extra layer)
But:
  ├─ Smaller file size
  ├─ Faster to generate
  ├─ Platform independent
  └─ Can be optimized later
```

**Hermes Bytecode Compilation:**

```
SOURCE CODE
  ↓
AST
  ↓
BYTECODE (Hermes-specific format)
  ├─ Optimized for mobile
  ├─ No JIT compilation needed
  ├─ Execute immediately
  └─ No startup delay

RESULT: Fast app startup
```

**Comparison:**

```
V8:
  Source → AST → Bytecode → Execution → Monitor Hot Code → Compile to Machine Code
  (Slow startup, fast runtime)

Hermes:
  Source → AST → Bytecode → Execution
  (Fast startup, decent runtime)
```

---

### 8.6 JIT vs AOT

**JIT (Just-In-Time) - V8 Approach:**

```
TIME 0:    Application starts
           └─ Load JavaScript

TIME 1:    Run code (as bytecode)
           └─ Monitor what's hot

TIME 100:  Heavy computation detected
           └─ Compile to machine code
           └─ Execute fast

Trade-off: Startup slow, runtime fast
```

**AOT (Ahead-Of-Time) - Hermes Approach:**

```
TIME 0:    Precompile to bytecode
           └─ When building app

TIME 1:    Application starts
           └─ Load precompiled bytecode
           └─ Execute immediately

TIME 100:  Heavy computation
           └─ Already compiled to bytecode
           └─ Execute bytecode (not fastest, but OK)

Trade-off: Startup fast, runtime decent
```

**Which is Better?**

```
For Web (V8):
  ├─ Users don't care about 1s startup
  ├─ Care about smooth experience for 10+ minutes
  └─ JIT optimization worth it

For Mobile (Hermes):
  ├─ Users care about startup
  ├─ Want app to open instantly
  ├─ Runtime performance is less critical
  └─ AOT fast startup worth it
```

---

### 8.7 Bytecode Generation in Hermes

**Bytecode Instruction Set (Simplified):**

```
LOAD n              Load constant n onto stack
STORE x             Store top of stack into variable x
ADD                 Pop two values, add them, push result
CALL f, n           Call function f with n arguments
RETURN              Return from function
```

**Example:**

```javascript
function add(a, b) {
  return a + b;
}

HERMES BYTECODE:
  0: CreateClosure add_func
  2: DefVar add
  3: Mov a, arg[0]
  4: Mov b, arg[1]
  5: Load a
  6: Load b
  7: Add
  8: Return
```

**Execution:**

```
STACK:
  1. Load a → [5]
  2. Load b → [5, 3]
  3. Add → [8]
  4. Return → 8 returned to caller
```

---

### 8.8 Startup Time Improvements (Hermes vs V8)

**Hermes Startup:**

```javascript
// App.js (1000 lines of JavaScript)

// HERMES:
// 1. Load bundle.hbc (precompiled bytecode)
// 2. Initialize runtime
// 3. Execute top-level code
// Total: 100-300ms

// V8 (via React Native):
// 1. Load bundle.js (source code)
// 2. Parse all 1000 lines
// 3. Compile to bytecode
// 4. Execute
// Total: 300-500ms
```

**Real Metrics:**

```
React Native App (Medium Complexity):

With V8:
  ├─ App startup: 400ms
  ├─ Time to interactive: 600ms
  └─ User sees blank screen for 0.6s

With Hermes:
  ├─ App startup: 200ms (2x faster!)
  ├─ Time to interactive: 300ms
  └─ User sees app in 0.3s
```

**Memory Improvements:**

```
V8 (with JIT):
  ├─ Initial: 20MB
  ├─ Hot code compiled: +30MB
  ├─ Optimization data: +10MB
  └─ Total: ~60MB

Hermes (AOT):
  ├─ Initial: 15MB
  ├─ Bytecode: ~15MB
  └─ Total: ~30MB

Hermes uses 50% less memory!
```

---

### 8.9 Common Issues with JavaScript Engines

**Issue 1: Typeof Confusion**

```javascript
typeof undefined  // "undefined"
typeof null       // "object" (BUG in all engines!)
typeof function() {} // "function"
```

**Why is null → "object"?**

Legacy bug. Never fixed because it would break websites.

**Issue 2: Type Coercion**

```javascript
"5" + 3         // "53" (string concatenation)
"5" - 3         // 2 (numeric subtraction)
"5" > 3         // true (numeric comparison)

// Confusing! Different behavior for +, -, >
```

**How Engines Handle This:**

```
V8/Hermes:
  1. Check types
  2. If mixed (string + number):
     ├─ For +: Convert to string
     ├─ For -: Convert to number
     └─ Follow language rules
```

**Issue 3: Garbage Collection Pauses**

```javascript
// Old V8:
// Running code
const hugeArray = new Array(1000000).fill(0);
// Running code
// GC runs (blocks everything for 100ms!)
// User sees stutter
```

**How Hermes Handles It:**

```
Incremental GC:
  ├─ Don't collect everything at once
  ├─ Collect in small chunks
  ├─ Between frames (doesn't cause stutter)
  └─ Smoother experience
```

---

## PHASE 9: Rendering Pipeline

### 9.1 Complete setState() Flow to Pixels

**The Grand Tour: From setState() to Pixels on Screen**

```
┌─────────────────────────────────────────────────────────────┐
│ STEP 1: USER INTERACTION                                    │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│ User touches button on screen                              │
│ OS (iOS/Android) detects touch                             │
│ → Sends event to app                                       │
│ → Native code receives onPress                             │
│ → Calls JS callback through JSI                            │
│                                                             │
└─────────────────────────────────────────────────────────────┘
           ↓
┌─────────────────────────────────────────────────────────────┐
│ STEP 2: JAVASCRIPT EXECUTION                                │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│ onPress callback executed (JS thread)                      │
│   ├─ setCount(count + 1) called                            │
│   ├─ useState reducer updates count                        │
│   ├─ Component marked as needing update                    │
│   ├─ scheduleWork() called                                 │
│   └─ Work added to scheduler queue                         │
│                                                             │
│ JS thread returns to event loop                            │
│ → Check microtask queue                                    │
│ → Check macrotask queue                                    │
│ → requestIdleCallback window open?                         │
│                                                             │
└─────────────────────────────────────────────────────────────┘
           ↓
┌─────────────────────────────────────────────────────────────┐
│ STEP 3: REACT FIBER RENDER PHASE                            │
├─────────────────────────────────────────────────────────────┤
│ (JS Thread, can be paused)                                  │
│                                                             │
│ requestIdleCallback fires (browser is idle)                │
│   ├─ workLoop() starts                                     │
│   ├─ beginWork(rootFiber)                                  │
│   │   └─ Fiber's render function called                    │
│   │       └─ React component function executed             │
│   │           ├─ useState hook executes                    │
│   │           ├─ Returns new JSX                           │
│   │           └─ count is now 1                            │
│   │                                                         │
│   ├─ completeWork(rootFiber)                               │
│   │   ├─ Compare old fiber (count: 0) vs new fiber (count: 1)│
│   │   ├─ Determine changes needed                          │
│   │   └─ Mark fiber with flags (Update, Placement, etc)    │
│   │                                                         │
│   ├─ beginWork(childFiber)                                 │
│   │   └─ Text component re-renders                         │
│   │       └─ Returns new JSX: <Text>1</Text>              │
│   │                                                         │
│   ├─ completeWork(childFiber)                              │
│   │   ├─ Text node unchanged? No                           │
│   │   ├─ Content changed (0 → 1)                           │
│   │   ├─ Mark with "Update" flag                           │
│   │   └─ Create effect list                                │
│   │                                                         │
│   ├─ [Pause for browser animation frame?]                  │
│   └─ Continue if more work                                 │
│                                                             │
│ Render phase complete!                                     │
│ └─ New fiber tree built                                    │
│ └─ Changes identified                                      │
│ └─ Nothing actually changed yet                            │
│                                                             │
└─────────────────────────────────────────────────────────────┘
           ↓
┌─────────────────────────────────────────────────────────────┐
│ STEP 4: FABRIC SHADOW TREE (C++)                            │
├─────────────────────────────────────────────────────────────┤
│ (Happens in C++ layer)                                      │
│                                                             │
│ React passes committed fiber tree to Fabric                │
│   ├─ Create ShadowNode tree (C++ representation)           │
│   ├─ ShadowNode for View container                         │
│   └─ ShadowNode for Text                                   │
│                                                             │
│ ShadowTree Structure:                                       │
│   RootShadowNode                                           │
│     └─ ViewShadowNode                                      │
│         └─ TextShadowNode                                  │
│                                                             │
│ Each ShadowNode contains:                                  │
│   ├─ Props (background color, etc.)                        │
│   ├─ Layout info (width, height)                           │
│   ├─ Style information                                     │
│   └─ Children references                                   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
           ↓
┌─────────────────────────────────────────────────────────────┐
│ STEP 5: YOGA LAYOUT ENGINE (C++)                            │
├─────────────────────────────────────────────────────────────┤
│ (Calculates positions/sizes)                                │
│                                                             │
│ Input: ShadowTree with flex properties                     │
│ Processing:                                                │
│   ├─ Root: { flex: 1, width: 100, height: 100 }           │
│   │   └─ Calculate root layout: (0, 0, 100, 100)          │
│   │                                                         │
│   ├─ Text child: { flex: 1 }                               │
│   │   └─ Calculate text layout: (0, 0, 100, 100)          │
│   │                                                         │
│   ├─ Measure text string "1"                               │
│   │   ├─ String length: 1 character                        │
│   │   ├─ Font size: 16px                                  │
│   │   ├─ Measured width: 10px                              │
│   │   └─ Measured height: 20px                             │
│   │                                                         │
│   └─ Final layout:                                         │
│       ├─ Root: x=0, y=0, w=100, h=100                     │
│       └─ Text: x=0, y=0, w=100, h=100                     │
│                                                             │
│ Output: ShadowTree with layout information                 │
│                                                             │
└─────────────────────────────────────────────────────────────┘
           ↓
┌─────────────────────────────────────────────────────────────┐
│ STEP 6: REACT FIBER COMMIT PHASE                            │
├─────────────────────────────────────────────────────────────┤
│ (JS Thread, cannot pause)                                   │
│                                                             │
│ All work from render phase is committed (applied)          │
│                                                             │
│ Phase 1 - Before Mutation:                                 │
│   ├─ Check what changed                                    │
│   ├─ Run useLayoutEffect cleanup functions                 │
│   └─ (synchronous)                                          │
│                                                             │
│ Phase 2 - Mutation (Actually update native views):         │
│   ├─ Loop through effects list                             │
│   ├─ For each changed fiber:                               │
│   │   ├─ If Placement flag: Create new native view        │
│   │   ├─ If Update flag: Update native view properties    │
│   │   ├─ If Deletion flag: Remove native view             │
│   │   └─ Call UIManager methods                            │
│   │                                                         │
│   ├─ For Text node (count: 0 → 1):                         │
│   │   ├─ Call NativeModules.UIManager.updateView()        │
│   │   ├─ Pass: viewTag, 'text', { text: '1' }             │
│   │   ├─ Through JSI (not Bridge!)                         │
│   │   └─ Instant call (no serialization)                   │
│   │                                                         │
│   └─ (synchronous)                                          │
│                                                             │
│ Phase 3 - After Mutation (Layout Effects & Effects):       │
│   ├─ Run useLayoutEffect effects (synchronous)             │
│   ├─ Schedule useEffect effects (asynchronous in microtask)│
│   └─ Passive effects                                       │
│                                                             │
│ Commit phase complete!                                     │
│ └─ All changes applied to native code                      │
│ └─ Next frame cycle can pick up changes                    │
│                                                             │
└─────────────────────────────────────────────────────────────┘
           ↓
┌─────────────────────────────────────────────────────────────┐
│ STEP 7: MOUNTING (NATIVE LAYER)                             │
├─────────────────────────────────────────────────────────────┤
│ (Runs on Native Thread, OS level)                           │
│                                                             │
│ iOS Mounting:                                              │
│   ├─ UIManager receives updateView() call (via JSI)       │
│   ├─ Finds UILabel view with viewTag                       │
│   ├─ Sets text property:                                   │
│   │   └─ uiLabel.text = "1"                               │
│   ├─ Marks view as needing display                         │
│   └─ (does NOT render yet)                                 │
│                                                             │
│ Android Mounting:                                          │
│   ├─ ReactViewManager receives updateView() call           │
│   ├─ Finds TextView with viewTag                           │
│   ├─ Sets text property:                                   │
│   │   └─ textView.text = "1"                              │
│   ├─ Invalidates view (needs redraw)                       │
│   └─ (does NOT render yet)                                 │
│                                                             │
│ Native layer ready for next draw call                      │
│                                                             │
└─────────────────────────────────────────────────────────────┘
           ↓
┌─────────────────────────────────────────────────────────────┐
│ STEP 8: VSYNC & FRAME CALLBACK                              │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│ VSyncProvider detects VSYNC signal                         │
│   ├─ 60 FPS: Every 16.67ms                                 │
│   ├─ 120 FPS: Every 8.33ms                                 │
│   └─ (Hardware signal from display)                         │
│                                                             │
│ Frame callback triggered:                                  │
│   ├─ Measure views (if needed)                             │
│   ├─ Update layouts                                        │
│   ├─ Prepare for rendering                                 │
│   └─ Hand off to graphics system                           │
│                                                             │
│ Native thread is now ready to render                       │
│                                                             │
└─────────────────────────────────────────────────────────────┘
           ↓
┌─────────────────────────────────────────────────────────────┐
│ STEP 9: RENDERING (GPU/GRAPHICS)                            │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│ OS calls app's render method                               │
│   ├─ iOS: drawRect() or Metal/OpenGL commands              │
│   ├─ Android: onDraw() or OpenGL commands                  │
│   └─ Execute drawing commands                              │
│                                                             │
│ For Text view with "1":                                    │
│   ├─ Allocate texture/bitmap                               │
│   ├─ Rasterize text to bitmap                              │
│   │   ├─ Font: System font, 16pt                           │
│   │   ├─ Character: "1"                                    │
│   │   ├─ Layout: x=0, y=0, w=100, h=100                   │
│   │   └─ Anti-alias: smooth edges                          │
│   ├─ Position bitmap at (0, 0)                             │
│   ├─ Composite with background views                       │
│   └─ Write to frame buffer                                 │
│                                                             │
│ GPU receives rendering commands                            │
│   ├─ Vertex shader: Position vertices                      │
│   ├─ Fragment shader: Color pixels                         │
│   ├─ Texture sampler: Apply textures                       │
│   └─ Output: Final pixel data                              │
│                                                             │
└─────────────────────────────────────────────────────────────┘
           ↓
┌─────────────────────────────────────────────────────────────┐
│ STEP 10: SCAN-OUT & DISPLAY                                 │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│ Frame buffer contains pixels                               │
│ Display controller scans out pixels                         │
│   ├─ Row by row                                            │
│   ├─ Left to right                                         │
│   ├─ Top to bottom                                         │
│   └─ 60 times per second (60 FPS)                          │
│                                                             │
│ Pixels sent to display panel                               │
│   ├─ LCD panel receives signal                             │
│   ├─ Liquid crystals rotate                                │
│   ├─ Light passes through filters                          │
│   ├─ Colors blend (R, G, B subpixels)                      │
│   └─ Light reaches user's eyes                             │
│                                                             │
│ USER SEES: "1" displayed on screen!                        │
│                                                             │
│ Total time: ~16-20ms (happens in 1 frame at 60 FPS)       │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

### 9.2 Threading Model During Render

**Detailed Thread Timeline:**

```
TIME 0ms:
  User taps button
  ├─ Native thread: Touch event detected
  ├─ JS thread: Idle (event loop running)

TIME 1ms:
  ├─ Native thread: onPress callback scheduled
  ├─ JS thread: Receives callback, executes
  │   └─ setCount(count + 1)

TIME 2ms:
  ├─ JS thread: scheduleWork() called
  ├─ Native thread: Idle

TIME 3-15ms:
  ├─ JS thread: Render phase
  │   ├─ Process fiber tree
  │   ├─ Create new virtual representation
  │   ├─ [Pause for browser frame]
  │   └─ Complete render
  │
  ├─ Native thread: Running other code
  │   ├─ Responding to OS events
  │   ├─ Managing other native modules
  │   └─ Doesn't know about JS changes yet

TIME 16ms:
  ├─ JS thread: Commit phase starts
  │   ├─ Send updates to native
  │   └─ Via JSI (instant, no queue)
  │
  ├─ Native thread: BLOCKED momentarily
  │   └─ Receives commit from JS
  │   └─ Updates native views
  │   └─ Resume

TIME 17ms:
  ├─ JS thread: Commit done, returns to event loop
  ├─ Native thread: Drawing on screen

TIME 18ms:
  ├─ Native thread: VSYNC signal fires
  ├─ Drawing commands sent to GPU

TIME 19ms:
  ├─ GPU: Rendering (off-screen buffer)
  └─ Display panel: Still showing old frame

TIME 20ms:
  ├─ GPU: Finished rendering
  ├─ Display panel: Shows new frame with "1"
  ├─ User sees the change!
  └─ Total: 20ms (2 frames at 60 FPS)
```

---

### 9.3 Fiber + Fabric + Yoga Interaction

**Complete Diagram:**

```
REACT LAYER (JavaScript)
┌─────────────────────────────────────────┐
│ Fiber Tree                              │
│                                         │
│  App Fiber { count: 1 }                │
│    └─ Text Fiber { children: "1" }    │
│                                         │
│  Props & state in JavaScript heap      │
│  Render functions execute here          │
└─────────────────────────────────────────┘
           ↓ (commit)
┌─────────────────────────────────────────┐
│ FABRIC LAYER (C++)                      │
├─────────────────────────────────────────┤
│                                         │
│ ShadowNode Tree                         │
│   └─ ViewShadowNode                    │
│       └─ TextShadowNode                │
│                                         │
│ Has shared references to React props   │
└─────────────────────────────────────────┘
           ↓
┌─────────────────────────────────────────┐
│ YOGA LAYOUT ENGINE (C++)                │
├─────────────────────────────────────────┤
│                                         │
│ Input: Props (flex, padding, etc.)     │
│ Output: Layout info (x, y, w, h)       │
│                                         │
│ Layout Tree                             │
│   └─ ViewLayout { x: 0, y: 0 ... }    │
│       └─ TextLayout { x: 0, y: 0 ... }│
│                                         │
└─────────────────────────────────────────┘
           ↓
┌─────────────────────────────────────────┐
│ NATIVE LAYER                            │
├─────────────────────────────────────────┤
│                                         │
│ iOS:                                    │
│   UIView frame (0, 0, 100, 100)        │
│   UILabel text "1"                      │
│   UILabel frame (0, 0, 100, 100)       │
│                                         │
│ Android:                                │
│   ViewGroup layout params               │
│   TextView text "1"                     │
│   TextView layout params                │
│                                         │
└─────────────────────────────────────────┘
           ↓
┌─────────────────────────────────────────┐
│ OS / GPU                                │
├─────────────────────────────────────────┤
│ Draw text "1" at position (0, 0)       │
│ Render to frame buffer                  │
│ Display on screen                       │
└─────────────────────────────────────────┘
```

---

### 9.4 Memory Allocation During Render Cycle

**Detailed Memory Usage:**

```
BEFORE RENDER (count = 0):

┌──────────────────────────────────────────┐
│ JAVASCRIPT HEAP                          │
├──────────────────────────────────────────┤
│                                          │
│ Current Fiber Tree:                      │
│   App Fiber { count: 0 }                │
│     └─ Text Fiber                       │
│                                          │
│ state object: { count: 0 }              │
│ JSX object: { type: Text, children: 0} │
│                                          │
│ Total: ~5KB                             │
│                                          │
└──────────────────────────────────────────┘

┌──────────────────────────────────────────┐
│ NATIVE HEAP (C++ / Swift / Kotlin)      │
├──────────────────────────────────────────┤
│                                          │
│ ShadowNode Tree (C++)                   │
│ UILabel object (iOS) or TextView (Android)│
│ Layout cache                             │
│                                          │
│ Total: ~10KB                            │
│                                          │
└──────────────────────────────────────────┘

TOTAL MEMORY: ~15KB


DURING RENDER PHASE (count = 0 → 1):

┌──────────────────────────────────────────┐
│ JAVASCRIPT HEAP                          │
├──────────────────────────────────────────┤
│                                          │
│ OLD Fiber Tree (still here):            │
│   App Fiber { count: 0 }                │
│     └─ Text Fiber { children: 0 }      │
│                                          │
│ NEW Fiber Tree (created):               │
│   App Fiber { count: 1 }                │
│     └─ Text Fiber { children: 1 }      │
│                                          │
│ NEW state object: { count: 1 }          │
│ NEW JSX object: { type: Text, children: 1 }│
│                                          │
│ Fiber.alternate pointers (linking old/new)│
│                                          │
│ Total: ~10KB (doubled during render)    │
│                                          │
└──────────────────────────────────────────┘

TOTAL MEMORY DURING: ~20KB


AFTER COMMIT PHASE (old fiber discarded):

┌──────────────────────────────────────────┐
│ JAVASCRIPT HEAP                          │
├──────────────────────────────────────────┤
│                                          │
│ Current Fiber Tree (now count: 1):      │
│   App Fiber { count: 1 }                │
│     └─ Text Fiber { children: 1 }      │
│                                          │
│ Old Fiber Tree (eligible for GC):       │
│   App Fiber { count: 0 }                │
│     └─ Text Fiber { children: 0 }      │
│   (No longer referenced)                │
│                                          │
│ Total: ~8KB (old fiber will be GC'd)   │
│                                          │
└──────────────────────────────────────────┘

┌──────────────────────────────────────────┐
│ NATIVE HEAP (C++ / Swift / Kotlin)      │
├──────────────────────────────────────────┤
│                                          │
│ NEW ShadowNode Tree (C++)               │
│ UILabel/TextView updated (same objects) │
│ Layout cache cleared & recalculated     │
│                                          │
│ Total: ~10KB                            │
│                                          │
└──────────────────────────────────────────┘

TOTAL MEMORY: ~18KB

Then garbage collection runs:
  → Old fiber tree removed
  → Memory back to ~15KB
```

---

### 9.5 Common Performance Issues in Rendering

**Issue 1: Dropped Frames**

```
VSYNC arrives (16.67ms for 60 FPS)

0ms:   Frame cycle starts
3ms:   JS still rendering (heavy component tree)
8ms:   JS still rendering
12ms:  JS FINALLY finishes commit
13ms:  Native needs to update views
14ms:  VSYNC signal arrives (frame deadline!)
15ms:  Rendering starts too late
16.67ms: VSYNC! Frame buffer not ready
       → Display shows OLD frame
       → User sees dropped frame (stutter)

What should have happened:
0ms:   Frame cycle starts
5ms:   JS finishes render + commit
6-15ms: Native rendering
16.67ms: New frame ready
```

**Solution:**
- Faster JS execution
- Smaller component trees
- Memoization to skip unnecessary renders
- Move heavy work off main thread

**Issue 2: Layout Thrashing**

```javascript
// WRONG - Triggers many layout calculations
for (let i = 0; i < 100; i++) {
  const height = view.offsetHeight; // Read (triggers layout)
  view.style.height = height + 10;  // Write (triggers relayout)
  // Yoga re-calculates 100 times!
}

// RIGHT - Batch reads and writes
const heights = [];
for (let i = 0; i < 100; i++) {
  heights.push(view.offsetHeight);  // Read all
}
for (let i = 0; i < 100; i++) {
  views[i].style.height = heights[i] + 10;  // Write all
}
// Yoga re-calculates once
```

**Issue 3: Unoptimized Re-renders**

```javascript
// Component re-renders unnecessarily
function Parent() {
  const [count, setCount] = useState(0);
  
  return (
    <View>
      <Button onPress={() => setCount(count + 1)} />
      <Child /> {/* Re-renders every time count changes!*/}
      <AnotherChild /> {/* Also re-renders!*/}
    </View>
  );
}

// Solution: Memoize children
function Parent() {
  const [count, setCount] = useState(0);
  
  const child = useMemo(() => <Child />, []);
  
  return (
    <View>
      <Button onPress={() => setCount(count + 1)} />
      {child} {/* Doesn't re-render*/}
    </View>
  );
}
```

---

## Summary of Phase 8 & 9

**Hermes JavaScript Engine (Phase 8):**
- JavaScript engines translate source code to machine code
- V8 uses JIT (slow startup, fast runtime)
- Hermes uses AOT (fast startup, decent runtime)
- Parsing creates AST
- Compilation to bytecode (or machine code)
- Execution with optimization

**Rendering Pipeline (Phase 9):**
- User touch → JavaScript execution → setCount()
- Fiber render phase (can pause)
- Fabric shadow tree created
- Yoga layout engine calculates positions
- Fiber commit phase (cannot pause)
- Native views updated via JSI
- Native thread handles mounting
- VSYNC signal triggers rendering
- GPU renders to frame buffer
- Display shows pixels

**Memory Management:**
- Fiber tree doubled during render (old + new)
- ShadowTree shared between React and Native
- Old tree GC'd after commit
- Layout calculations cached

**Performance Considerations:**
- 16.67ms per frame (60 FPS)
- JS execution must finish before commit
- Native rendering happens after commit
- Dropped frames = janky animation
- Memoization prevents unnecessary re-renders

**Next Phase:** Native Communication - deep dive into how JS calls camera, GPS, storage.
