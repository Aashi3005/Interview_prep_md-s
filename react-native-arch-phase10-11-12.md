# React Native Architecture Masterclass
## PHASE 10, 11 & 12: Native Communication, Performance, Advanced Topics

---

## PHASE 10: Native Communication

### 10.1 How JavaScript Calls Native Code

**The Complete Flow (Old vs New):**

**OLD ARCHITECTURE (Bridge-based):**

```
JavaScript Call:
NativeModules.Camera.takePicture((error, photo) => {
  console.log(photo);
});

STEP 1: Serialize call to JSON
{
  module: 'Camera',
  method: 'takePicture',
  callbackId: 1,
  args: []
}

STEP 2: Send JSON through bridge
JavaScript Thread → Bridge Queue → Native Thread

STEP 3: Parse JSON on native
JSON.parse() → Reconstruct arguments

STEP 4: Find and call native method
RCTCameraModule.takePicture(callback)

STEP 5: Native execution
  ├─ Access hardware (camera)
  ├─ Wait for user to take photo
  ├─ Process image
  └─ Invoke callback

STEP 6: Serialize response
{
  callbackId: 1,
  result: { uri: '/path/to/photo.jpg' },
  error: null
}

STEP 7: Send back through bridge
Native Thread → Bridge Queue → JavaScript Thread

STEP 8: Parse JSON in JavaScript
Reconstruct photo object

STEP 9: Execute callback
callback(null, { uri: '/path/to/photo.jpg' })

STEP 10: JavaScript continues
console.log(photo)

Total latency: 5-20ms (due to serialization)
```

**NEW ARCHITECTURE (JSI-based):**

```
JavaScript Call:
const photo = await Camera.takePicture();

STEP 1: Call JSI binding (C++)
CameraModule.takePicture() → Direct function call

STEP 2: C++ binding receives call
No serialization needed (direct memory access)

STEP 3: C++ calls native Objective-C/Kotlin
Directly invoke method with arguments

STEP 4: Native execution
  ├─ Access hardware (camera)
  ├─ Wait for user to take photo
  ├─ Process image
  └─ Return result

STEP 5: C++ receives result
Direct C++ object (no JSON)

STEP 6: JSI binding wraps result for JavaScript
Create JavaScript object from C++ object

STEP 7: Return to JavaScript
photo = { uri: '/path/to/photo.jpg' }

Total latency: <1ms (no serialization)

20x faster!
```

---

### 10.2 Real Examples: Camera, GPS, AsyncStorage

**Example 1: Camera (takePicture)**

**OLD Way (Bridge):**

```javascript
import { NativeModules } from 'react-native';
const { CameraModule } = NativeModules;

// Callback-based
CameraModule.takePicture((error, result) => {
  if (error) {
    console.error('Camera error:', error);
  } else {
    console.log('Photo taken:', result.uri);
    // Use result
  }
});
```

**NEW Way (JSI/TurboModule):**

```javascript
import { Camera } from '@react-native/camera';

// Promise-based
try {
  const photo = await Camera.takePicture();
  console.log('Photo taken:', photo.uri);
} catch (error) {
  console.error('Camera error:', error);
}
```

**Native Implementation:**

```swift
// iOS (Swift)
@objc(RCTCamera)
class RCTCamera: NSObject, RCTTurboModule {
  // iOS API access
  @objc
  func takePicture(_ resolve: @escaping RCTPromiseResolveBlock,
                   reject: @escaping RCTPromiseRejectBlock) {
    let picker = UIImagePickerController()
    
    // Capture image
    picker.sourceType = .camera
    
    // When done:
    let image = capturedImage
    let data = image.jpegData(compressionQuality: 0.9)
    let url = saveToTemp(data)
    
    resolve([
      "uri": url.absoluteString,
      "width": image.size.width,
      "height": image.size.height
    ])
  }
  
  static func moduleName() -> String! {
    "Camera"
  }
}
```

```kotlin
// Android (Kotlin)
class RCTCamera(reactContext: ReactApplicationContext) :
  ReactContextBaseJavaModule(reactContext), TurboModule {
  
  @ReactMethod
  fun takePicture(promise: Promise) {
    val intent = Intent(MediaStore.ACTION_IMAGE_CAPTURE)
    
    // When done:
    val bitmap = data.getParcelableExtra<Bitmap>("data")
    val file = File(context.cacheDir, "photo.jpg")
    val fos = FileOutputStream(file)
    bitmap.compress(Bitmap.CompressFormat.JPEG, 90, fos)
    
    promise.resolve(mapOf(
      "uri" to Uri.fromFile(file).toString(),
      "width" to bitmap.width,
      "height" to bitmap.height
    ))
  }
  
  override fun getName() = "Camera"
}
```

**Calling Flow:**

```
User presses "Take Photo" button
  ↓ (JS thread)
Camera.takePicture() called
  ↓ (Native thread via JSI)
iOS: UIImagePickerController opens
Android: Camera intent opens
  ↓
User takes photo
  ↓
iOS: imagePickerController delegate called
Android: onActivityResult called
  ↓
Photo processed
  ↓
Promise resolved in JS
  ↓
try/catch block handles result
  ↓
JavaScript continues (no blocking)
```

---

**Example 2: GPS (getCurrentPosition)**

**React Native Location Module:**

```javascript
// JavaScript
import { getCurrentPosition } from '@react-native-camera-roll/camera-roll';

try {
  const location = await getCurrentPosition();
  console.log('Lat:', location.latitude);
  console.log('Long:', location.longitude);
  console.log('Accuracy:', location.accuracy);
} catch (error) {
  console.error('Location error:', error);
}
```

**Native Implementation (iOS):**

```swift
import CoreLocation

class RCTGeolocation: NSObject, RCTTurboModule, CLLocationManagerDelegate {
  let locationManager = CLLocationManager()
  var resolver: RCTPromiseResolveBlock?
  var rejecter: RCTPromiseRejectBlock?
  
  @objc
  func getCurrentPosition(_ resolve: @escaping RCTPromiseResolveBlock,
                         reject: @escaping RCTPromiseRejectBlock) {
    // Request location permissions
    locationManager.requestWhenInUseAuthorization()
    
    resolver = resolve
    rejecter = reject
    
    // Start location updates
    locationManager.delegate = self
    locationManager.startUpdatingLocation()
  }
  
  // Delegate method called when location available
  func locationManager(_ manager: CLLocationManager,
                       didUpdateLocations locations: [CLLocation]) {
    guard let location = locations.last else { return }
    
    // Stop updating
    manager.stopUpdatingLocation()
    
    // Return result via JSI (no serialization!)
    resolver?([
      "latitude": location.coordinate.latitude,
      "longitude": location.coordinate.longitude,
      "accuracy": location.horizontalAccuracy,
      "altitude": location.altitude,
      "heading": location.course,
      "speed": location.speed
    ])
  }
  
  func locationManager(_ manager: CLLocationManager,
                       didFailWithError error: Error) {
    rejecter?("LOCATION_ERROR", error.localizedDescription, error)
  }
}
```

**Native Implementation (Android):**

```kotlin
import android.location.Location
import com.google.android.gms.location.FusedLocationProviderClient
import com.google.android.gms.location.LocationServices

class RCTGeolocation(private val reactContext: ReactApplicationContext) :
  ReactContextBaseJavaModule(reactContext), TurboModule {
  
  private val fusedLocationClient: FusedLocationProviderClient =
    LocationServices.getFusedLocationProviderClient(reactContext)
  
  @ReactMethod
  fun getCurrentPosition(promise: Promise) {
    // Check permissions
    if (!hasLocationPermissions()) {
      promise.reject("PERMISSION_ERROR", "Location permission not granted")
      return
    }
    
    // Get location asynchronously
    fusedLocationClient.lastLocation
      .addOnSuccessListener { location ->
        if (location != null) {
          // Return via JSI
          promise.resolve(mapOf(
            "latitude" to location.latitude,
            "longitude" to location.longitude,
            "accuracy" to location.accuracy,
            "altitude" to location.altitude,
            "heading" to location.bearing,
            "speed" to location.speed
          ))
        } else {
          promise.reject("NO_LOCATION", "Unable to get location")
        }
      }
      .addOnFailureListener { e ->
        promise.reject("LOCATION_ERROR", e.message, e)
      }
  }
  
  override fun getName() = "Geolocation"
}
```

**Behind the Scenes:**

```
JS: await getCurrentPosition()
  ↓
JSI: Direct C++ call (native binding)
  ↓
iOS: CLLocationManager.startUpdatingLocation()
Android: FusedLocationProviderClient.lastLocation()
  ↓
OS Hardware: Request GPS hardware
  ↓
GPS Hardware: Acquire satellite signal
  ↓
(May take 5-30 seconds)
  ↓
Hardware: Location obtained
  ↓
Delegate/Callback triggered
  ↓
Native code constructs result
  ↓
JSI returns to JavaScript (instant)
  ↓
Promise resolves with location
```

---

**Example 3: AsyncStorage (Persistent Storage)**

**JavaScript:**

```javascript
import AsyncStorage from '@react-native-async-storage/async-storage';

// Write
await AsyncStorage.setItem('user_name', 'John');

// Read
const name = await AsyncStorage.getItem('user_name');
console.log(name); // 'John'

// Remove
await AsyncStorage.removeItem('user_name');
```

**How AsyncStorage Works (Backend):**

**iOS (RocksDB database):**

```
Data stored in:
  ~/Library/Application Support/[AppName]/RocksDB/

RocksDB:
  ├─ Key: "user_name"
  └─ Value: "John"

Write:
  1. Serialize JavaScript object to string
  2. Write to RocksDB
  3. Flush to disk
  4. Promise resolves

Read:
  1. Look up key in RocksDB
  2. Deserialize to JavaScript object
  3. Return via JSI
```

**Android (SharedPreferences or Database):**

```
Data stored in:
  /data/data/[PackageName]/shared_prefs/RocksDB.xml

File content:
  <map>
    <string name="user_name">John</string>
  </map>

Write:
  1. Serialize to string
  2. Write to file
  3. Commit
  4. Promise resolves

Read:
  1. Look up value
  2. Return via JSI
```

**Performance Characteristics:**

```
Write Operation:
  ├─ Serialize (JS): <1ms
  ├─ JSI call (C++): <1ms
  ├─ Disk I/O: 5-50ms
  ├─ Flush: 5-50ms
  └─ Total: 10-102ms (blocked)

Read Operation:
  ├─ Disk read: 1-10ms
  ├─ JSI return (C++): <1ms
  ├─ Deserialize (JS): <1ms
  └─ Total: 1-11ms (blocked)

Tip: Use transactions for multiple writes
```

---

### 10.3 Thread Safety & Synchronization

**Challenge: Two Threads, One App**

```
┌─────────────────────────┐
│ JavaScript Thread       │
├─────────────────────────┤
│ ├─ React render        │
│ ├─ Event listeners     │
│ └─ State management    │
└─────────────────────────┘
           ↕ (communication)
┌─────────────────────────┐
│ Native Thread           │
├─────────────────────────┤
│ ├─ Hardware access     │
│ ├─ Rendering          │
│ └─ Native views       │
└─────────────────────────┘

Both threads access shared data.
Race conditions possible!
```

**Example Race Condition:**

```javascript
// JavaScript thread
let data = { name: 'John', age: 30 };
sendToNative(data);  // JSI call (async)

// Meanwhile... (still JS thread)
data.name = 'Jane';

// Native thread (unpredictable timing)
receiveFromJS(data);  // data could be { name: 'John' } or { name: 'Jane' }!
```

**Solution: Copy Data**

```javascript
// Make a copy before sending
const dataCopy = { ...data };
sendToNative(dataCopy);

// Modify original without affecting native
data.name = 'Jane';

// Native receives original snapshot
```

**Synchronization Mechanisms:**

**iOS (GCD - Grand Central Dispatch):**

```swift
// Dispatch to main thread (UI thread)
DispatchQueue.main.async {
  // Update native views here
}

// Dispatch to background
DispatchQueue.global().async {
  // Heavy computation here
}

// Dispatch synchronously (blocking)
DispatchQueue.main.sync {
  // Must finish before continuing
}
```

**Android (Handler & Looper):**

```kotlin
// Dispatch to main thread
Handler(Looper.getMainLooper()).post {
  // Update native views here
}

// Dispatch to background
Handler(HandlerThread("background").looper).post {
  // Heavy computation
}
```

---

## PHASE 11: Performance Engineering

### 11.1 Re-renders: The Root of Performance Issues

**What Causes Unnecessary Re-renders:**

```javascript
function Parent() {
  const [count, setCount] = useState(0);
  
  return (
    <View>
      {/* This re-renders every time count changes */}
      <Child /> {/* UNNECESSARY RE-RENDER */}
      <AnotherChild /> {/* UNNECESSARY RE-RENDER */}
    </View>
  );
}
```

**Why This Happens:**

```
count changes
  ↓
Parent component re-renders
  ↓
Parent function called again
  ↓
Child and AnotherChild are in Parent's JSX
  ↓
Even though their props didn't change
  ↓
React's default behavior: re-render anyway
```

**Impact on Performance:**

```
1 parent re-render = 10 child re-renders
  └─ 10 component functions called
  └─ 10 render functions executed
  └─ 10 fiber trees traversed
  └─ Potentially 10 native view updates

With poor structure: 1000 re-renders per state change!
```

---

### 11.2 Memoization Techniques

**React.memo (Prevent Child Re-renders)**

```javascript
// WITHOUT memo
function Child({ name }) {
  console.log('Child rendering with:', name);
  return <Text>{name}</Text>;
}

function Parent() {
  const [count, setCount] = useState(0);
  
  return (
    <View>
      <Button onPress={() => setCount(count + 1)} />
      <Child name="John" /> {/* Re-renders every time count changes */}
    </View>
  );
}

// OUTPUT:
// "Child rendering with: John" (on every count change)

// WITH memo
const Child = React.memo(function Child({ name }) {
  console.log('Child rendering with:', name);
  return <Text>{name}</Text>;
});

function Parent() {
  const [count, setCount] = useState(0);
  
  return (
    <View>
      <Button onPress={() => setCount(count + 1)} />
      <Child name="John" /> {/* Doesn't re-render! */}
    </View>
  );
}

// OUTPUT:
// "Child rendering with: John" (only first time)
// (no logs on subsequent count changes)
```

**How React.memo Works:**

```
React.memo does shallow comparison of props:

OLD PROPS: { name: 'John' }
NEW PROPS: { name: 'John' }

Are they equal? (shallow comparison)
├─ name: 'John' === 'John' → TRUE
└─ All props same → Don't re-render
```

**useMemo (Cache Expensive Calculations)**

```javascript
// WITHOUT useMemo
function List({ items }) {
  // This object is recreated every render
  const filteredItems = items.filter(item => item.active);
  
  return (
    <FlatList
      data={filteredItems} {/* Different object every render */}
      renderItem={({ item }) => <Item item={item} />}
      keyExtractor={item => item.id}
    />
  );
}

// WITH useMemo
function List({ items }) {
  // This object is cached based on dependencies
  const filteredItems = useMemo(() => {
    return items.filter(item => item.active);
  }, [items]); {/* Recalculate only when items change */}
  
  return (
    <FlatList
      data={filteredItems} {/* Same object (if items same) */}
      renderItem={({ item }) => <Item item={item} />}
      keyExtractor={item => item.id}
    />
  );
}
```

**useCallback (Cache Functions)**

```javascript
// WITHOUT useCallback
function Parent() {
  const [count, setCount] = useState(0);
  
  // New function created every render
  const handlePress = () => {
    console.log('Pressed');
  };
  
  return (
    <View>
      <Child onPress={handlePress} /> {/* handlePress is different every render */}
    </View>
  );
}

const Child = React.memo(({ onPress }) => {
  return <Button onPress={onPress} />;
});

// Child re-renders every time (different function reference)


// WITH useCallback
function Parent() {
  const [count, setCount] = useState(0);
  
  // Function cached based on dependencies
  const handlePress = useCallback(() => {
    console.log('Pressed');
  }, []); {/* No dependencies, function cached forever */}
  
  return (
    <View>
      <Child onPress={handlePress} /> {/* Same function reference */}
    </View>
  );
}

// Child doesn't re-render (same function)
```

---

### 11.3 FlatList Internals & Virtualization

**What is Virtualization?**

Only render visible items. Remove items off-screen from memory.

```
Screen shows 10 items at a time.
Data has 1000 items.

Without Virtualization:
  ├─ Create 1000 view objects
  ├─ Render all 1000
  ├─ Keep in memory: 1000 * 100KB = 100MB
  ├─ Scroll performance: TERRIBLE

With Virtualization:
  ├─ Create 10 visible items + buffer (20 items total)
  ├─ Render 20 items
  ├─ Keep in memory: 20 * 100KB = 2MB
  ├─ Scroll performance: GREAT
```

**FlatList Implementation:**

```javascript
import { FlatList } from 'react-native';

const data = [
  { id: '1', name: 'Item 1' },
  { id: '2', name: 'Item 2' },
  // ... 1000 items
];

function MyList() {
  return (
    <FlatList
      data={data}
      renderItem={({ item }) => <Item item={item} />}
      keyExtractor={item => item.id}
      initialNumToRender={10}      // Items to render upfront
      maxToRenderPerBatch={10}     // Max to render per scroll
      updateCellsBatchingPeriod={10} // Batching interval (ms)
      removeClippedSubviews={true}  // Remove off-screen views
    />
  );
}

function Item({ item }) {
  return <Text>{item.name}</Text>;
}
```

**How FlatList Works Internally:**

```
INITIAL LOAD:
  1. Calculate visible rect (window size)
  2. Render initialNumToRender (10 items)
  3. Add buffer above/below
  4. Display on screen

USER SCROLLS DOWN:
  1. Calculate new visible rect
  2. Detect off-screen items above
  3. Remove off-screen items from DOM
  4. Calculate new on-screen items below
  5. Create new item views
  6. Add to DOM
  7. Re-measure and layout
  8. Display updated list

RESULT:
  Only visible items in memory
  Smooth scrolling even with 10,000 items
```

**Performance Tips:**

```javascript
// ❌ SLOW - renderItem creates new objects
<FlatList
  renderItem={({ item }) => {
    return <Item name={item.name} onClick={() => console.log(item.id)} />;
  }}
/>

// ✅ FAST - stable function
const renderItem = useCallback(({ item }) => {
  return <Item name={item.name} onClick={() => console.log(item.id)} />;
}, []);

<FlatList renderItem={renderItem} />
```

---

### 11.4 Thread Blocking Issues

**JavaScript Thread Blocking:**

```
Heavy JS computation blocks render:

USER TAPS BUTTON
  ↓
setCount(1) called
  ↓
Schedule render
  ↓
Start render phase
  ↓
Component function called
  ↓
Heavy computation (1000ms loop):
  for (let i = 0; i < 1000000000; i++) {
    // Math calculations
  }
  ↓ [JS thread blocked for 1000ms]
  
During blocking time:
  ├─ User can't interact (touches ignored)
  ├─ Native can't reach JS
  ├─ Animations frozen
  └─ App looks frozen

After 1000ms:
  ├─ Computation done
  ├─ Render completes
  ├─ Native receives update
  ├─ View finally updates
  └─ App responds to touches
```

**Native Thread Blocking:**

```
Heavy native computation blocks rendering:

NATIVE CODE UPDATES NATIVE VIEW
  ↓
Very complex layout calculation (500ms)
  ├─ Main thread blocked
  ├─ Can't respond to touches
  ├─ Can't render
  └─ Animations stutter

Result:
  ├─ User taps button: No response
  ├─ Animation plays: Drops frames
  └─ App looks janky
```

**Solutions:**

**Move Work Off Thread:**

```javascript
// WRONG - Blocks JS thread
function expensiveCalculation() {
  for (let i = 0; i < 1000000000; i++) {
    // Calculation
  }
}

// RIGHT - Move to background (if possible)
import { runOnJS } from 'react-native-reanimated';

const handlePress = () => {
  runOnJS(expensiveCalculation)();
  // Returns immediately, calculation happens elsewhere
};
```

**Use Suspense / Lazy Loading:**

```javascript
// Split work across frames
function* generateData() {
  for (let i = 0; i < 10000; i++) {
    yield processItem(i);
    
    // Yield to allow other work
    if (i % 100 === 0) {
      await new Promise(resolve => setTimeout(resolve, 0));
    }
  }
}
```

---

## PHASE 12: Advanced React Native

### 12.1 TurboModules Deep Dive

**Creating a TurboModule:**

```typescript
// TypeScript specification (source of truth)
export interface Spec extends TurboModule {
  add(a: number, b: number): number;
  greet(name: string): Promise<string>;
  addEventListener(event: string): void;
}

export default TurboModuleRegistry.getEnforcing<Spec>('MathModule');
```

**Codegen Auto-generates Native Code:**

```swift
// iOS (auto-generated)
@objc(RCTMathModule)
class RCTMathModule: NSObject, RCTTurboModule {
  @objc
  func add(_ a: Double, _ b: Double) -> Double {
    return a + b
  }
  
  @objc
  func greet(_ name: String,
             resolve: @escaping RCTPromiseResolveBlock,
             reject: @escaping RCTPromiseRejectBlock) {
    resolve("Hello, \(name)")
  }
}
```

```kotlin
// Android (auto-generated)
class RCTMathModule(reactContext: ReactApplicationContext) :
  ReactContextBaseJavaModule(reactContext), TurboModule {
  
  @ReactMethod
  fun add(a: Double, b: Double, promise: Promise) {
    promise.resolve(a + b)
  }
  
  @ReactMethod
  fun greet(name: String, promise: Promise) {
    promise.resolve("Hello, $name")
  }
}
```

**Benefits:**

```
├─ Codegen generates bridge code (less manual work)
├─ Type-safe (TypeScript spec validates)
├─ Consistent across platforms
├─ JSI under the hood (fast)
└─ Less boilerplate
```

---

### 12.2 Fabric Internals

**Fabric Commit Hooks:**

```cpp
// Pseudo-C++ code (simplified)
class FabricUIManager {
  void commit(ShadowTree& shadowTree) {
    // 1. Before mutation
    beforeMutation(shadowTree);
    
    // 2. Perform mutations
    for (auto& mutation : shadowTree.mutations) {
      switch (mutation.type) {
        case Mutation::Create:
          createView(mutation);
          break;
        case Mutation::Update:
          updateView(mutation);
          break;
        case Mutation::Delete:
          deleteView(mutation);
          break;
      }
    }
    
    // 3. After mutation
    afterMutation(shadowTree);
  }
  
  void createView(const Mutation& m) {
    auto view = ViewFactory::create(m.type);
    view->setProps(m.props);
    mountView(view, m.parent);
  }
};
```

---

### 12.3 Bridgeless Mode

**What is Bridgeless Mode?**

React Native without the Bridge!

```
OLD ARCHITECTURE:
  JS ↔ [BRIDGE] ↔ Native
  
BRIDGELESS:
  JS → [JSI Directly] → Native
```

**Challenges Bridgeless Solves:**

```
Bridge problems:
  ├─ Serialization overhead
  ├─ Message queuing
  ├─ Thread context switching
  ├─ Limited throughput
  └─ Debugging complexity

Bridgeless advantages:
  ├─ Direct C++ access
  ├─ No serialization
  ├─ Faster communication
  ├─ Simpler architecture
  └─ Easier debugging
```

**What Changes:**

```
Bridge:
  NativeModules.Camera.takePicture(callback)

Bridgeless:
  Camera.takePicture() (direct JSI)
```

**Status (as of 2024):**

```
Bridgeless mode is:
  ├─ Partially implemented
  ├─ Being rolled out gradually
  ├─ May require native code changes
  └─ Currently experimental for some modules
```

---

### 12.4 Concurrent React Native

**What is Concurrent Rendering?**

React can pause render, yield to browser/OS, then resume.

**Old (Non-concurrent):**

```
Render entire tree
  ├─ Start render phase
  ├─ Can't stop
  ├─ Must finish all work
  └─ Blocks main thread
```

**New (Concurrent):**

```
Render incrementally
  ├─ Render component 1
  ├─ Yield to OS (let it paint)
  ├─ Render component 2
  ├─ Yield to OS
  ├─ Continue...
  └─ App stays responsive
```

**Benefits:**

```
1. Faster apparent response time
2. Smoother animations
3. Better prioritization
4. Suspense support
```

**React Transitions:**

```javascript
import { useTransition } from 'react';

function MyComponent() {
  const [isPending, startTransition] = useTransition();
  const [name, setName] = useState('');
  
  const handleChange = (e) => {
    startTransition(() => {
      // This update is low priority
      // Can be interrupted for more urgent updates
      setName(e.target.value);
    });
  };
  
  return (
    <input
      value={name}
      onChange={handleChange}
      disabled={isPending}
    />
  );
}
```

---

### 12.5 React Compiler

**What is React Compiler?**

Automatically optimizes React code at build time.

**Without Compiler:**

```javascript
function Component() {
  const [count, setCount] = useState(0);
  
  // Developer must manually memoize
  const memoizedValue = useMemo(() => {
    return expensiveCalculation();
  }, [count]);
  
  return <Text>{memoizedValue}</Text>;
}
```

**With Compiler (auto-optimized):**

```javascript
function Component() {
  const [count, setCount] = useState(0);
  
  // Compiler automatically adds memoization
  // No useMemo needed
  const value = expensiveCalculation();
  
  return <Text>{value}</Text>;
}

// Compiler transforms to:
// const value = useMemo(() => expensiveCalculation(), [count]);
```

**Benefits:**

```
├─ Automatic memoization
├─ Fewer developer mistakes
├─ Better performance
├─ Less code to write
└─ Easier to maintain
```

---

## Summary of Phase 10, 11 & 12

**Native Communication (Phase 10):**
- Old: Bridge serializes to JSON (5-20ms latency)
- New: JSI direct C++ access (<1ms latency)
- Camera, GPS, AsyncStorage examples
- Thread safety requires data copying
- Race conditions possible without care

**Performance Engineering (Phase 11):**
- Unnecessary re-renders cause performance issues
- React.memo prevents child re-renders
- useMemo caches expensive calculations
- useCallback caches function references
- FlatList virtualizes (only renders visible items)
- Thread blocking freezes app

**Advanced Topics (Phase 12):**
- TurboModules use JSI for speed
- Codegen auto-generates native code
- Fabric is new rendering engine
- Bridgeless mode removes bridge entirely
- Concurrent rendering allows pausing work
- React Compiler auto-optimizes code

**Next Phase:** Debugging Architecture - understand typical bugs and how to find them.
