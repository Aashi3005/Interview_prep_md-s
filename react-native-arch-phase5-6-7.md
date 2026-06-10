# React Native Architecture Masterclass
## PHASE 5, 6 & 7: React Native Fundamentals, Old & New Architecture

---

## PHASE 5: React Native Fundamentals

### 5.1 Why React Native Exists

**The Problem Before React Native:**

```
Mobile App Development:
  ├─ iPhone: Write in Swift/Objective-C
  ├─ Android: Write in Kotlin/Java
  └─ Windows: Write in C#
  
Each platform:
  ├─ Different language
  ├─ Different APIs
  ├─ Different UI
  ├─ Different testing
  └─ 3-4x development cost + time
```

**The Vision:**

"Learn Once, Write Anywhere" (not "Write Once, Run Anywhere")

Use JavaScript + React (skills devs already have) to build native apps.

```
BEFORE REACT NATIVE:
JavaScript Dev
  ├─ Wants to build mobile app
  ├─ Has to learn Swift + iOS APIs (6 months)
  ├─ Has to learn Kotlin + Android APIs (6 months)
  └─ Maintains two codebases

AFTER REACT NATIVE:
JavaScript Dev
  ├─ Uses React (already knows it)
  ├─ Writes JavaScript
  ├─ Runs on both iOS and Android
  └─ Maintains one codebase (mostly)
```

---

### 5.2 React Native Architecture Overview

**Three Pillars:**

```
┌─────────────────────────────────┐
│ JAVASCRIPT LAYER                │
│ ├─ React Components             │
│ ├─ Business Logic               │
│ ├─ State Management             │
│ └─ Runs on JS Thread            │
└─────────────────────────────────┘
           ↓↑
      [BRIDGE]
           ↓↑
┌─────────────────────────────────┐
│ NATIVE LAYER                    │
│ ├─ iOS: Swift/Objective-C       │
│ ├─ Android: Kotlin/Java         │
│ ├─ Native Views                 │
│ └─ Runs on Native Thread        │
└─────────────────────────────────┘
           ↓
┌─────────────────────────────────┐
│ OS / HARDWARE                   │
│ ├─ Camera                       │
│ ├─ GPS                          │
│ ├─ Storage                      │
│ └─ Rendering                    │
└─────────────────────────────────┘
```

**Why Three Separate Layers?**

JavaScript can't directly access hardware (camera, GPS, etc.). Needs native code to bridge the gap.

---

### 5.3 JavaScript Thread

**What is the JavaScript Thread?**

A separate thread (not the main UI thread) that runs your JavaScript code.

```
ANDROID:
  ┌─────────────────────┐
  │ Main UI Thread      │
  ├─────────────────────┤
  │ ├─ Handle touches   │
  │ ├─ Render views     │
  │ └─ Native code      │
  └─────────────────────┘
  
  ┌─────────────────────┐
  │ JavaScript Thread   │
  ├─────────────────────┤
  │ ├─ Run JS code      │
  │ ├─ Event loop       │
  │ └─ React rendering  │
  └─────────────────────┘

iOS:
  Same concept with Grand Central Dispatch threads
```

**Why Separate Thread?**

If JavaScript ran on main thread:
- Heavy JS computation blocks UI rendering
- App becomes unresponsive
- Animations drop frames

**How Communication Happens:**

```
JS Thread                    Native Thread
    │                            │
    ├─ Run JS code              │
    │                            │
    ├─ Call native function     │
    ├──────────────────────────→ │
    │ (Sends message across)    │
    │                          ├─ Process native call
    │                            │
    │                            ├─ Update native view
    │                            │
    │ (Receives response)       │
    │←──────────────────────────┤
    │                            │
    └─ Continue JS execution    │
```

**Critical: They run on different threads → Must be synchronized**

---

### 5.4 UI Thread (Native Thread)

**The Main/UI Thread:**

Only thread that can update the UI (on both iOS and Android).

```javascript
// WRONG - Trying to update UI from JS thread
const view = getNativeView();
view.backgroundColor = 'red'; // Blocks native thread
```

This doesn't work. JavaScript can't directly modify native views.

**Instead: Send Message**

```javascript
// RIGHT - Send message to native
NativeModules.UIManager.setViewColor(viewTag, 'red');
// Message goes through bridge to native thread
// Native thread updates the UI
```

**UI Thread Responsibilities:**

```
┌─────────────────────────────────┐
│ UI THREAD (Main Thread)          │
├─────────────────────────────────┤
│ 1. Receive touch events          │
│ 2. Update native views           │
│ 3. Run animations                │
│ 4. Handle gestures               │
│ 5. Render to screen              │
│ 6. Respond to bridge messages    │
└─────────────────────────────────┘
```

**Frame Rate:**

```
60 FPS (normal) = 16.67ms per frame
120 FPS (high refresh) = 8.33ms per frame

Main thread MUST finish all work in this time!

If work takes 50ms:
  ├─ Frame 1 (16.67ms): Not done yet, skip frame
  ├─ Frame 2 (16.67ms): Still not done, skip frame
  ├─ Frame 3 (16.67ms): Finally done!
  └─ Result: Dropped 2 frames, janky animation
```

---

### 5.5 Native Modules

**What is a Native Module?**

Code that bridges JavaScript to native functionality (camera, GPS, etc.).

**Without Native Module:**

```javascript
// IMPOSSIBLE - JavaScript can't access hardware directly
const camera = navigator.mediaDevices.getUserMedia();
// This works on Web, but React Native doesn't have this
```

**With Native Module:**

```javascript
// iOS (Swift)
@objc(CameraModule)
class CameraModule: NSObject {
  @objc
  func takePicture(_ callback: @escaping RCTResponseSenderBlock) {
    // Native code to take picture
    let image = UIImagePickerController.takePicture()
    callback([NSNull(), image])
  }
}

// Android (Kotlin)
class CameraModule(reactContext: ReactApplicationContext) : ReactContextBaseJavaModule(reactContext) {
  @ReactMethod
  fun takePicture(promise: Promise) {
    // Native code to take picture
    val image = Camera.takePicture()
    promise.resolve(image)
  }
}

// JavaScript (both platforms)
import { NativeModules } from 'react-native';
const { CameraModule } = NativeModules;

CameraModule.takePicture((error, image) => {
  if (error) console.error(error);
  else console.log('Image:', image);
});
```

**How Native Module Works:**

```
JS Code Calls NativeModules.Camera.takePicture()
  ↓
Bridge serializes arguments
  ↓
Sends message to native
  ↓
Native thread receives message
  ↓
Native code finds CameraModule
  ↓
Calls takePicture() on native side
  ↓
Native camera hardware opens
  ↓
User takes picture
  ↓
Native code captures image
  ↓
Bridge serializes image
  ↓
Sends back to JS
  ↓
Callback executed with image
```

---

### 5.6 Native Views

**What is a Native View?**

A UI element rendered by the native OS, not HTML.

```javascript
// Web (React)
function Button() {
  return <button onClick={handlePress}>Click</button>;
}

// Mobile (React Native)
function Button() {
  return <TouchableOpacity onPress={handlePress}>
    <Text>Click</Text>
  </TouchableOpacity>;
}
```

**On Web:** `<button>` → HTML → Browser renders HTML button

**On Mobile:** `<TouchableOpacity>` → Native View → OS renders native button

```
iOS:
  TouchableOpacity → UIButton (native iOS component)

Android:
  TouchableOpacity → Button (native Android component)
```

**How Native View is Created:**

```
React Native Code:
  <View style={{ backgroundColor: 'blue' }}>
    <Text>Hello</Text>
  </View>
  ↓
React creates fiber tree with View and Text components
  ↓
During commit phase, React calls:
  UIManager.createView(viewTag, 'RCTView', { backgroundColor: 'blue' })
  UIManager.createView(textTag, 'RCTText', { })
  ↓
Message sent through bridge
  ↓
Native side receives message
  ↓
Native creates actual native views:
  iOS: UIView with blue background
  Android: View with blue background
  ↓
Native views are displayed on screen
```

**Native View Tree:**

```
React Fiber Tree          Native View Tree (iOS)   Display
View Fiber                UIView 1                 ┌─────┐
  ├─ Text Fiber      →      └─ UILabel            │ "Hi" │
  └─ Button Fiber               └─ UIButton       └─────┘
                      Android:
                      ViewGroup 1
                        ├─ TextView
                        └─ Button
```

---

### 5.7 Yoga Layout Engine

**The Problem:**

Different platforms have different layout systems:
- iOS: Auto Layout + Frame-based
- Android: LayoutParams + LinearLayout
- Web: CSS Box Model

React Native needs one unified layout system.

**Yoga = CSS Flexbox for All Platforms**

```javascript
const styles = StyleSheet.create({
  container: {
    flex: 1,
    flexDirection: 'row',
    justifyContent: 'space-between',
    alignItems: 'center',
    paddingTop: 20,
  }
});

// This flexbox layout works on:
// ├─ iOS
// ├─ Android
// └─ Web (React)
```

**How Yoga Works:**

```
┌──────────────────────────────────┐
│ 1. PARSE STYLES                  │
│    StyleSheet → CSS values       │
└──────────────────────────────────┘
           ↓
┌──────────────────────────────────┐
│ 2. CREATE LAYOUT TREE            │
│    Fiber tree → Yoga nodes       │
└──────────────────────────────────┘
           ↓
┌──────────────────────────────────┐
│ 3. CALCULATE LAYOUT              │
│    Flexbox algorithm             │
│    ├─ Calculate width            │
│    ├─ Calculate height           │
│    ├─ Calculate X position       │
│    └─ Calculate Y position       │
└──────────────────────────────────┘
           ↓
┌──────────────────────────────────┐
│ 4. APPLY TO NATIVE VIEWS         │
│    iOS: frame = (x, y, w, h)     │
│    Android: layout(x, y, x+w, y+h)│
└──────────────────────────────────┘
```

**Example:**

```javascript
<View style={{ flex: 1, flexDirection: 'row' }}>
  <View style={{ flex: 1 }} /> {/* Takes 1/3 width */}
  <View style={{ flex: 2 }} /> {/* Takes 2/3 width */}
</View>

// Yoga calculations:
// Total flex: 1 + 2 = 3
// First child: (1/3) × parentWidth
// Second child: (2/3) × parentWidth
```

**Performance Impact:**

Layout is expensive. React measures component dimensions multiple times.

```
Layout Calculation:
  1. Parse styles (fast)
  2. Create layout tree (fast)
  3. Run flexbox algorithm (SLOW - happens twice sometimes)
     ├─ First pass: estimate
     └─ Second pass: adjust
  4. Apply to native views (fast)
```

---

## PHASE 6: Old React Native Architecture (The Bridge)

### 6.1 The Bridge: Overview

**What is the Bridge?**

The communication mechanism between JavaScript and Native code.

```
JAVASCRIPT THREAD                NATIVE THREAD
         │                            │
         │                            │
    JSON.stringify()                  │
         │                            │
         ├─ Serializes data ──────────→ JSON.parse()
         │                            │
         │                    Process native code
         │                            │
         │                            │
    JSON.parse()   ←────────────────── JSON.stringify()
         │                            │
         └─ Result                    │
```

**The Bridge is a Serialization Layer**

Data must be converted to JSON, sent across, then reconstructed.

---

### 6.2 Message Passing (Detailed)

**Complete Flow:**

```
JS Side:
  NativeModules.Camera.takePicture((err, result) => {
    console.log(result);
  });
  ↓
  1. Create callback ID: 1
  2. Store callback in memory: callbacks[1] = function(err, result) { ... }
  3. Serialize arguments to JSON:
     {
       module: 'Camera',
       method: 'takePicture',
       args: [],
       callbackId: 1
     }
  4. Send JSON through bridge
  
Native Side (iOS):
  ├─ Bridge receives JSON message
  ├─ Parse JSON
  ├─ Look up module: Camera
  ├─ Look up method: takePicture
  ├─ Call: [camera takePicture: callback]
  ├─ Native code executes (access hardware)
  ├─ Camera opens, user takes picture
  ├─ Callback invoked with image data
  ├─ Serialize response to JSON:
     {
       callbackId: 1,
       result: { uri: '/path/to/image.jpg' }
     }
  └─ Send JSON back through bridge

JS Side:
  ├─ Bridge receives JSON response
  ├─ Parse JSON
  ├─ Look up callback: callbacks[1]
  ├─ Call callback with result:
     callback(null, { uri: '/path/to/image.jpg' })
  ├─ Function executes
  ├─ console.log prints image URI
  └─ Clean up: delete callbacks[1]
```

**Timing:**

```
Time →
0ms:   takePicture() called
1ms:   Message queued
5ms:   Message sent to native
10ms:  Native receives
15ms:  Camera app opens
500ms: User takes picture
505ms: Native processes image
510ms: Response serialized
515ms: Response sent back
520ms: JS callback executed
```

---

### 6.3 Serialization

**What Gets Serialized?**

Only JSON-serializable data:
- Numbers, strings, booleans
- Objects, arrays
- null

**What Can't Be Serialized?**

- Functions
- Symbols
- undefined (becomes null)
- Circular references

```javascript
// ✅ WORKS - Serializable
NativeModules.Storage.save({ name: 'John', age: 30 });

// ❌ DOESN'T WORK - Functions can't be serialized
NativeModules.Storage.save({ 
  name: 'John', 
  process: () => {} // Error!
});

// ❌ DOESN'T WORK - Circular reference
const obj = { name: 'John' };
obj.self = obj;
NativeModules.Storage.save(obj); // Error!
```

**Size Matters:**

Each message is serialized and sent. Large objects = slow.

```javascript
// ✅ FAST - Small payload
NativeModules.API.send({ userId: 123 }); // ~20 bytes

// ❌ SLOW - Large payload
NativeModules.API.send({ 
  userData: hugeObjectWith1000Fields // ~100KB
});
// Takes time to serialize, send, deserialize
```

---

### 6.4 Bridge Bottlenecks

**Bottleneck 1: Serialization Overhead**

```
Large JS Object
  ↓ JSON.stringify() (CPU work)
  ↓ Send through bridge (time)
  ↓ JSON.parse() (CPU work)
  ↓ Large Native Object

If object has 1000 properties, this is slow.
```

**Bottleneck 2: Batching**

```
Synchronous calls:
  Call 1: Send → Wait → Receive (5ms)
  Call 2: Send → Wait → Receive (5ms)
  Call 3: Send → Wait → Receive (5ms)
  Total: 15ms

Batched calls:
  Call 1, 2, 3: Send together → Wait → Receive (7ms)
  Total: 7ms (2x faster!)

React batches updates to optimize this.
```

**Bottleneck 3: Thread Context Switching**

```
JS Thread
  ├─ Running code...
  └─ Message sent to native
    ↓
  (Context switch overhead)
    ↓
Native Thread
  ├─ Running code...
  └─ Message sent to JS
    ↓
  (Context switch overhead)
    ↓
JS Thread
  └─ Resume...

Each context switch has overhead.
Multiple round-trips are slow.
```

**Bottleneck 4: JS Thread Blocking**

```
If JS thread is doing heavy computation:
  ├─ Calculation 1 (50ms)
  ├─ Calculation 2 (50ms)
  └─ NativeModules.API.call() (stuck waiting)

Native side can't reach JS. Deadlock potential.

Example: State reconciliation blocks bridge calls
```

**Performance Impact:**

```
With Bridge (Old Architecture):
  Simple native call: 5-10ms latency
  Bridge can handle 10-20 calls/second max
  
Without Bridge (New Architecture):
  Direct C++ access: <1ms latency
  No serialization overhead
```

---

### 6.5 Thread Communication Issues

**Race Conditions:**

```javascript
// JS Side
let state = 0;

NativeModules.API.getValue((error, value) => {
  state = value; // Callback from native
});

// Meanwhile...
setTimeout(() => {
  console.log(state); // What is state?
  // 0? or the value from native?
  // RACE CONDITION!
}, 0);
```

**Synchronous Calls (Limited):**

React Native allows synchronous calls, but it's dangerous:

```javascript
// Synchronous call - blocks everything
const value = NativeModules.API.getValue(); // BLOCKS!

// During this block:
// - JS thread can't do anything
// - If native thread tries to call JS, deadlock
```

**Batching Complexity:**

```javascript
// These calls get batched:
NativeModules.API.call1();
NativeModules.API.call2();
NativeModules.API.call3();

// Sent in one message
// But if call1 depends on response from native,
// this breaks! They all execute simultaneously.

// Must be very careful about dependencies.
```

---

## PHASE 7: New React Native Architecture

### 7.1 Why Meta Redesigned React Native

**Problems with Old Architecture:**

```
1. SERIALIZATION OVERHEAD
   Every call → JSON → Send → Parse
   Slow for large data or frequent calls
   
2. ASYNC-ONLY COMMUNICATION
   Can't access native immediately
   Limited to callbacks/promises
   
3. THREAD SYNCHRONIZATION
   JS and Native threads out of sync
   Race conditions possible
   
4. PERFORMANCE CEILING
   Limited throughput
   Can't handle fast-paced interactions
   
5. MEMORY DUPLICATION
   JS objects and Native objects separate
   Same data in two places
   
6. LIMITED DEBUGGING
   Hard to trace cross-platform issues
```

**Meta's Solution: JSI (JavaScript Interface)**

Instead of:
```
JS →JSON→ Bridge → Parse → Native
```

Use:
```
JS → Direct C++ → Native
```

---

### 7.2 JSI (JavaScript Interface)

**What is JSI?**

A C++ layer that allows JavaScript to directly call C++ functions, and vice versa.

```
┌─────────────────────────────────┐
│ JAVASCRIPT                      │
├─────────────────────────────────┤
│ JS Code                         │
│ │                               │
│ └─ JSI Bindings (C++)           │
│    │                            │
│    └─ Direct function calls     │
│       (no serialization)        │
└─────────────────────────────────┘
         ↓
┌─────────────────────────────────┐
│ C++ LAYER                       │
├─────────────────────────────────┤
│ ├─ Native modules               │
│ ├─ Logic                        │
│ └─ Can call Objective-C/Kotlin  │
└─────────────────────────────────┘
         ↓
┌─────────────────────────────────┐
│ NATIVE LAYER                    │
├─────────────────────────────────┤
│ iOS (Swift/Obj-C)              │
│ Android (Kotlin/Java)          │
└─────────────────────────────────┘
```

**JSI vs Bridge:**

```
Bridge (Old):
  JS → JSON.stringify() → Send → Parse → Native
  No direct access

JSI (New):
  JS → Direct C++ pointer → Native
  Direct access to C++ objects
```

**JSI Example:**

```javascript
// Old way (Bridge)
import { NativeModules } from 'react-native';
NativeModules.Camera.takePicture((err, result) => {
  console.log(result);
});

// New way (JSI)
import CameraModule from './CameraModule';
const result = await CameraModule.takePicture();
console.log(result);

// Or even:
const camera = new NativeCamera();
camera.open();
const photo = camera.capture();
camera.close();
```

**Performance:**

```
Bridge:
  takePicture() call
  ├─ Serialize arguments: 1ms
  ├─ Send message: 1ms
  ├─ Parse on native: 1ms
  ├─ Actual work: 500ms (hardware)
  ├─ Serialize result: 1ms
  ├─ Send back: 1ms
  ├─ Parse in JS: 1ms
  └─ Total: ~507ms (8ms overhead)

JSI:
  takePicture() call
  ├─ Direct C++ call: 0.1ms
  ├─ Actual work: 500ms (hardware)
  ├─ Return result: 0.1ms
  └─ Total: ~500.2ms (0.2ms overhead)
```

40x faster for serialization!

---

### 7.3 Fabric (New Rendering Engine)

**What is Fabric?**

New rendering system that replaces UIManager.

**Old (UIManager/Bridge):**

```
React            Bridge           Native
├─ Render        ├─ Serialize     ├─ Create View
├─ Diffing       ├─ Send JSON     ├─ Update Props
├─ Props         └─ Wait...       └─ Layout

Problem: React waits for native to finish
        Then native waits for React
        Many round-trips
```

**New (Fabric):**

```
React                  Fabric (C++)              Native
├─ Render             ├─ Create ShadowTree     ├─ Layout
├─ Diffing            ├─ Calculate Layout      ├─ Create View
├─ Commit             ├─ Hold reference        └─ Render
└─ Pass to Fabric     └─ Sync views
   (pass tree)
```

React passes complete tree once, Fabric handles the rest.

**Fabric Architecture:**

```
┌──────────────────────────────────────────┐
│ FABRIC (C++)                             │
├──────────────────────────────────────────┤
│                                          │
│ ┌─ ShadowTree                           │
│ │  (React representation in C++)        │
│ │  ├─ Props                             │
│ │  ├─ Layout info                       │
│ │  └─ Children                          │
│ │                                        │
│ ├─ Yoga Layout Engine                   │
│ │  (Calculate positions/sizes)          │
│ │                                        │
│ ├─ MountingManager                      │
│ │  (Create/update native views)         │
│ │                                        │
│ └─ Platform-specific code               │
│    ├─ iOS: UIView creation              │
│    └─ Android: View creation            │
│                                          │
└──────────────────────────────────────────┘
```

**Why This is Better:**

```
Old Flow:
  React renders
  ├─ Creates tree
  ├─ Serializes to JSON
  ├─ Sends through bridge
  ├─ Native waits, receives
  ├─ Parses JSON
  ├─ Creates views
  └─ Total latency: Network + Parsing

New Flow:
  React renders
  ├─ Creates tree
  ├─ Passes C++ reference to Fabric
  ├─ Fabric has instant access (same memory)
  ├─ Calculates layout in C++ (fast)
  ├─ Creates views directly
  └─ Total latency: C++ execution (much faster)
```

---

### 7.4 TurboModules

**What are TurboModules?**

Native modules that use JSI instead of Bridge.

```javascript
// Old NativeModule (uses Bridge)
NativeModules.Camera.takePicture((err, photo) => {
  // Slow, serialization overhead
});

// New TurboModule (uses JSI)
import { Camera } from '@react-native/camera';
const photo = await Camera.takePicture();
// Fast, no serialization
```

**Creating a TurboModule:**

```javascript
// TypeScript specification
export interface CameraModule extends TurboModule {
  takePicture(): Promise<string>;
  openCamera(): void;
}

export default TurboModuleRegistry.getEnforcing<CameraModule>('Camera');

// iOS (Swift)
@objc(RCTCamera)
class RCTCamera: NSObject, RCTTurboModule {
  @objc
  func takePicture(_ resolve: @escaping RCTPromiseResolveBlock,
                   reject: @escaping RCTPromiseRejectBlock) {
    // Native implementation
  }
}

// Android (Kotlin)
class RCTCamera(reactContext: ReactApplicationContext) : ReactContextBaseJavaModule(reactContext), TurboModule {
  @ReactMethod
  fun takePicture(promise: Promise) {
    // Native implementation
  }
}
```

---

### 7.5 Codegen

**What is Codegen?**

Automatically generates bridge code from TypeScript specs.

**Without Codegen (Manual):**

```
TypeScript Spec
  ↓ (manually implement)
iOS Native Code
  ↓ (manually implement)
Android Native Code
  ↓ (manually write bridge code)
Bridge Code

Error-prone, duplicated work.
```

**With Codegen:**

```
TypeScript Spec
  ↓ (codegen automatically generates)
  ├─ iOS Native Code
  ├─ Android Native Code
  └─ Bridge Code

One source of truth.
Less manual work.
```

**Example TypeSpec:**

```typescript
// CameraModule.ts
import type { TurboModule } from 'react-native';
import { TurboModuleRegistry } from 'react-native';

export interface Spec extends TurboModule {
  takePicture(): Promise<string>;
  openCamera(): void;
  closeCamera(): void;
}

export default TurboModuleRegistry.getEnforcing<Spec>('Camera');
```

Codegen automatically generates all the native binding code!

---

### 7.6 Shadow Tree

**What is Shadow Tree?**

A C++ representation of the component tree before creating native views.

```
React Fiber Tree (JavaScript)
        ↓
Converted to ShadowNode Tree (C++)
        ↓
Layout calculations (Yoga)
        ↓
Create Native Views (if needed)
        ↓
Mount to screen
```

**Why Two Trees?**

```
React Tree (JS):
  ├─ Component logic
  ├─ State management
  └─ Props handling

Shadow Tree (C++):
  ├─ Optimized for layout
  ├─ Cached calculations
  ├─ Prepared for mounting
  └─ Shared between React & Native
```

**Memory:**

```
React Fiber         Shadow Node         Native View
├─ Render logic     ├─ Layout cache     ├─ Actual UI
├─ Props            ├─ Style            └─ Pixels
├─ State            └─ Children refs
└─ Hooks
```

Less memory duplication than old architecture.

---

### 7.7 C++ Core

**Why C++?**

```
JavaScript:
  ├─ Easy to write
  ├─ But slower at runtime
  └─ Garbage collected

C++:
  ├─ Harder to write
  ├─ But much faster
  ├─ Direct memory control
  └─ Closer to hardware
```

**New Architecture Uses C++ For:**

```
1. Yoga Layout Engine
   └─ Flexbox calculations (fastest)

2. Fabric Renderer
   └─ Creating views (no bridge delay)

3. ShadowTree
   └─ Component representation (shared)

4. JSI Bindings
   └─ JS ↔ Native communication (fast)
```

**Performance:**

```
Old: JS → JSON → Bridge → Parse → Native (slow)
New: JS → JSI → C++ → Native (fast)

C++ is 100-1000x faster for heavy computation.
```

---

### 7.8 Old vs New Architecture (Complete Comparison)

**Rendering Flow:**

```
OLD ARCHITECTURE:

React                 Bridge              UIManager (Native)
├─ Render             ├─ Serialize        ├─ Parse JSON
├─ Diffing             │ props to JSON     ├─ Create/update views
├─ Create tree        ├─ Send             ├─ Run layout
├─ Call UIManager     ├─ Receive          ├─ Render
│  with props         │                    └─ Display
├─ (Waits)            │                    
│                     └─ (Overhead)


NEW ARCHITECTURE:

React                 Fabric (C++)           Native Layer
├─ Render             ├─ Receive tree        ├─ Layout
├─ Diffing            ├─ Run Yoga            ├─ Create views
├─ Create tree        ├─ Commit              └─ Display
├─ Pass to Fabric     ├─ Mount views
│  (instant)          └─ (No JSON!)
└─ (Continues)
```

**Communication:**

| Aspect | Old | New |
|--------|-----|-----|
| Method | Bridge + JSON | JSI + C++ |
| Serialization | Yes (slow) | No (fast) |
| Latency | 5-10ms | <1ms |
| Throughput | Limited | High |
| Data Duplication | Yes | No |
| Async | Required | Optional |

**Memory:**

```
Old: Objects in 3 places
  ├─ JavaScript Heap (React tree)
  ├─ Bridge cache
  └─ Native Heap (native views)

New: Objects shared
  ├─ JavaScript Heap (React tree)
  └─ C++ ShadowTree (shared reference)
  └─ Native Heap (created from ShadowTree)
```

**Performance:**

```
Old:  setState → Serialize → Send → Parse → Create View: 50ms
New:  setState → Direct C++ → Create View: 5ms
      10x faster!
```

---

## Summary of Phase 5, 6 & 7

**React Native Fundamentals (Phase 5):**
- JavaScript thread runs React code
- Native thread renders native views
- Bridge communicates between them
- Native modules bridge hardware access
- Yoga engine calculates layout
- Three layers: JS, Bridge, Native

**Old Architecture (Phase 6):**
- Uses Bridge for communication
- Data serialized to JSON
- Async-only communication
- Bottlenecks: serialization, thread sync, message passing
- Performance limited by bridge throughput

**New Architecture (Phase 7):**
- JSI for direct C++ access (no JSON)
- Fabric renderer (new React Native engine)
- Shadow Tree (C++ representation)
- TurboModules use JSI
- Codegen auto-generates bridge code
- 10-100x faster communication
- Shared memory (less duplication)

**Next Phase:** Hermes JavaScript Engine - understand how JS source becomes bytecode and executes.
