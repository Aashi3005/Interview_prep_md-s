# React Native — Complete Concepts Guide
### From Beginner to Jordan Walke Level

> Jordan Walke created React at Facebook and later React Native evolved from his work. This guide covers everything from the absolute basics to the deep internals that power React Native.

---

## Table of Contents

1. [What is React Native?](#1-what-is-react-native)
2. [Environment Setup](#2-environment-setup)
3. [Core Components](#3-core-components)
4. [JSX](#4-jsx)
5. [Props](#5-props)
6. [State](#6-state)
7. [Styling](#7-styling)
8. [Flexbox Layout](#8-flexbox-layout)
9. [Lists & ScrollView](#9-lists--scrollview)
10. [Hooks](#10-hooks)
11. [Navigation](#11-navigation)
12. [Networking & APIs](#12-networking--apis)
13. [Platform-Specific Code](#13-platform-specific-code)
14. [Images & Media](#14-images--media)
15. [Forms & User Input](#15-forms--user-input)
16. [Animations](#16-animations)
17. [Gestures](#17-gestures)
18. [Native Modules & Bridging](#18-native-modules--bridging)
19. [State Management](#19-state-management)
20. [Performance Optimization](#20-performance-optimization)
21. [Testing](#21-testing)
22. [Debugging](#22-debugging)
23. [Security](#23-security)
24. [Deployment & Publishing](#24-deployment--publishing)
25. [The New Architecture (JSI, Fabric, TurboModules)](#25-the-new-architecture)
26. [React Native Internals](#26-react-native-internals)

---

## 1. What is React Native?

React Native is an open-source framework created by **Facebook (Meta)**, initially developed under Jordan Walke's React paradigm, that lets you build **native mobile apps** using **JavaScript and React**.

### Key Differences from React (Web)

| React (Web) | React Native |
|---|---|
| Renders HTML elements | Renders Native UI components |
| Uses CSS | Uses StyleSheet API |
| Runs in browser | Runs on iOS / Android |
| `<div>`, `<span>`, `<p>` | `<View>`, `<Text>`, `<Image>` |

### How it works (simplified)

```
JavaScript Code
      ↓
   Metro Bundler (JS bundle)
      ↓
   JavaScript Thread
      ↓
   Bridge (JSON serialization) / JSI (new arch)
      ↓
   Native Thread → Native UI Components
```

---

## 2. Environment Setup

### Using Expo (Beginner-friendly)
```bash
npx create-expo-app MyApp
cd MyApp
npx expo start
```

### Using React Native CLI (Full control)
```bash
npx @react-native-community/cli init MyApp
cd MyApp
npx react-native run-android   # Android
npx react-native run-ios       # iOS (Mac only)
```

### Project Structure
```
MyApp/
├── android/          ← Android native code
├── ios/              ← iOS native code
├── src/
│   ├── screens/
│   ├── components/
│   ├── navigation/
│   ├── hooks/
│   └── utils/
├── App.tsx           ← Entry point
├── index.js          ← Register component
├── package.json
└── metro.config.js   ← JS bundler config
```

---

## 3. Core Components

React Native maps JavaScript components to native platform widgets.

### View
The fundamental container — equivalent to `<div>`.
```jsx
import { View } from 'react-native';

<View style={{ flex: 1, backgroundColor: '#fff' }}>
  {/* children */}
</View>
```

### Text
Every string must be inside `<Text>`.
```jsx
import { Text } from 'react-native';

<Text style={{ fontSize: 18, fontWeight: 'bold' }}>Hello World</Text>
```

### Image
```jsx
import { Image } from 'react-native';

// Local
<Image source={require('./assets/logo.png')} style={{ width: 100, height: 100 }} />

// Remote
<Image source={{ uri: 'https://example.com/img.png' }} style={{ width: 100, height: 100 }} />
```

### TextInput
```jsx
import { TextInput } from 'react-native';

<TextInput
  placeholder="Enter name"
  onChangeText={(text) => setName(text)}
  value={name}
  secureTextEntry={false}
  keyboardType="email-address"
/>
```

### TouchableOpacity / Pressable
```jsx
import { TouchableOpacity, Pressable, Text } from 'react-native';

// TouchableOpacity (classic)
<TouchableOpacity onPress={() => alert('Pressed!')} activeOpacity={0.7}>
  <Text>Press Me</Text>
</TouchableOpacity>

// Pressable (modern, more control)
<Pressable
  onPress={() => {}}
  style={({ pressed }) => [{ opacity: pressed ? 0.5 : 1 }]}
>
  <Text>Press Me</Text>
</Pressable>
```

### Button
```jsx
import { Button } from 'react-native';

<Button title="Submit" onPress={() => {}} color="#6200ee" />
```

### SafeAreaView
Handles notch / status bar spacing on iOS.
```jsx
import { SafeAreaView } from 'react-native-safe-area-context';

<SafeAreaView style={{ flex: 1 }}>
  {/* content */}
</SafeAreaView>
```

### StatusBar
```jsx
import { StatusBar } from 'react-native';

<StatusBar barStyle="dark-content" backgroundColor="#ffffff" />
```

---

## 4. JSX

JSX is a syntax extension that looks like HTML but compiles to `React.createElement()` calls.

```jsx
// JSX
const element = <Text style={{ color: 'red' }}>Hello</Text>;

// What it compiles to
const element = React.createElement(Text, { style: { color: 'red' } }, 'Hello');
```

### Rules
- Return a single root element (or use `<>...</>` Fragment)
- All tags must be closed (`<View />`)
- Use `{}` for JavaScript expressions
- `className` → no, use `style`
- Conditionals with `&&` or ternary `? :`

```jsx
const App = () => {
  const isLoggedIn = true;

  return (
    <>
      {isLoggedIn ? <Text>Welcome!</Text> : <Text>Please log in</Text>}
      {isLoggedIn && <Text>Dashboard</Text>}
    </>
  );
};
```

---

## 5. Props

Props (properties) are **read-only** inputs passed from parent to child.

```jsx
// Child component
const Greeting = ({ name, age }) => (
  <Text>Hello {name}, you are {age} years old!</Text>
);

// Parent
const App = () => (
  <Greeting name="Aashik" age={22} />
);
```

### Default Props
```jsx
const Button = ({ title = 'Click me', color = '#000' }) => (
  <TouchableOpacity style={{ backgroundColor: color }}>
    <Text>{title}</Text>
  </TouchableOpacity>
);
```

### Children Prop
```jsx
const Card = ({ children }) => (
  <View style={styles.card}>{children}</View>
);

// Usage
<Card>
  <Text>Card content here</Text>
</Card>
```

### TypeScript Props
```tsx
interface ButtonProps {
  title: string;
  onPress: () => void;
  disabled?: boolean;
}

const MyButton: React.FC<ButtonProps> = ({ title, onPress, disabled = false }) => (
  <TouchableOpacity onPress={onPress} disabled={disabled}>
    <Text>{title}</Text>
  </TouchableOpacity>
);
```

---

## 6. State

State is **mutable data** that belongs to a component. When state changes, the component re-renders.

### useState
```jsx
import React, { useState } from 'react';
import { View, Text, Button } from 'react-native';

const Counter = () => {
  const [count, setCount] = useState(0);

  return (
    <View>
      <Text>Count: {count}</Text>
      <Button title="Increment" onPress={() => setCount(prev => prev + 1)} />
      <Button title="Reset" onPress={() => setCount(0)} />
    </View>
  );
};
```

### State with Objects
```jsx
const [user, setUser] = useState({ name: '', email: '' });

// Always spread to avoid losing other fields
setUser(prev => ({ ...prev, name: 'Aashik' }));
```

### Derived State
Don't store in state what can be computed from existing state.
```jsx
const [items, setItems] = useState([1, 2, 3, 4, 5]);
const evenItems = items.filter(i => i % 2 === 0); // derived, not state
```

---

## 7. Styling

React Native uses a **JavaScript object** styling system — no CSS files.

### StyleSheet API
```jsx
import { StyleSheet, View, Text } from 'react-native';

const styles = StyleSheet.create({
  container: {
    flex: 1,
    backgroundColor: '#f5f5f5',
    alignItems: 'center',
    justifyContent: 'center',
  },
  title: {
    fontSize: 24,
    fontWeight: '700',
    color: '#1a1a1a',
    letterSpacing: 0.5,
  },
});

const App = () => (
  <View style={styles.container}>
    <Text style={styles.title}>Hello</Text>
  </View>
);
```

### Inline Styles
```jsx
<View style={{ padding: 16, margin: 8, borderRadius: 12 }} />
```

### Combining Styles (Array Syntax)
```jsx
<Text style={[styles.base, styles.bold, { color: 'red' }]}>Text</Text>
```

### Key Style Properties

| Property | Values | Notes |
|---|---|---|
| `flex` | number | `flex: 1` fills available space |
| `flexDirection` | `row`, `column` | Default is `column` |
| `alignItems` | `center`, `flex-start`, `flex-end`, `stretch` | Cross axis |
| `justifyContent` | `center`, `space-between`, `space-around` | Main axis |
| `padding` / `margin` | number | All sides |
| `paddingHorizontal` | number | Left + Right |
| `paddingVertical` | number | Top + Bottom |
| `borderRadius` | number | Rounded corners |
| `elevation` | number | Android shadow |
| `shadowColor` | string | iOS shadow |
| `position` | `absolute`, `relative` | Positioning |
| `zIndex` | number | Stacking order |

---

## 8. Flexbox Layout

React Native uses Flexbox by default for **all** components. Key difference: default `flexDirection` is `column` (not `row` like web CSS).

```jsx
// Row layout
<View style={{ flexDirection: 'row', justifyContent: 'space-between' }}>
  <Text>Left</Text>
  <Text>Right</Text>
</View>

// Centered content
<View style={{ flex: 1, alignItems: 'center', justifyContent: 'center' }}>
  <Text>Centered</Text>
</View>

// Flexible children
<View style={{ flexDirection: 'row' }}>
  <View style={{ flex: 1, backgroundColor: 'red' }} />   {/* takes 1/3 */}
  <View style={{ flex: 2, backgroundColor: 'blue' }} />  {/* takes 2/3 */}
</View>
```

### flexWrap
```jsx
<View style={{ flexDirection: 'row', flexWrap: 'wrap' }}>
  {items.map(item => <Chip key={item.id} label={item.name} />)}
</View>
```

### alignSelf
Override parent's `alignItems` for one child:
```jsx
<View style={{ alignItems: 'center' }}>
  <Text style={{ alignSelf: 'flex-start' }}>Left aligned</Text>
</View>
```

---

## 9. Lists & ScrollView

### ScrollView
Good for **small**, finite content.
```jsx
import { ScrollView } from 'react-native';

<ScrollView
  horizontal={false}
  showsVerticalScrollIndicator={false}
  contentContainerStyle={{ padding: 16 }}
>
  {/* content */}
</ScrollView>
```

### FlatList
Optimal for **long lists** — renders only visible items (virtualization).
```jsx
import { FlatList } from 'react-native';

const DATA = [
  { id: '1', title: 'Item 1' },
  { id: '2', title: 'Item 2' },
];

<FlatList
  data={DATA}
  keyExtractor={(item) => item.id}
  renderItem={({ item }) => <Text>{item.title}</Text>}
  ItemSeparatorComponent={() => <View style={{ height: 1, backgroundColor: '#eee' }} />}
  ListEmptyComponent={<Text>No items</Text>}
  ListHeaderComponent={<Text style={styles.header}>My List</Text>}
  onEndReached={loadMore}
  onEndReachedThreshold={0.5}
  refreshing={isRefreshing}
  onRefresh={handleRefresh}
/>
```

### SectionList
For **grouped/sectioned** lists.
```jsx
import { SectionList } from 'react-native';

const SECTIONS = [
  { title: 'Fruits', data: ['Apple', 'Banana'] },
  { title: 'Veggies', data: ['Carrot', 'Broccoli'] },
];

<SectionList
  sections={SECTIONS}
  keyExtractor={(item, index) => item + index}
  renderItem={({ item }) => <Text>{item}</Text>}
  renderSectionHeader={({ section: { title } }) => (
    <Text style={styles.sectionHeader}>{title}</Text>
  )}
/>
```

---

## 10. Hooks

### useState
```jsx
const [value, setValue] = useState(initialValue);
```

### useEffect
```jsx
import { useEffect } from 'react';

// Runs on mount and every render
useEffect(() => { /* ... */ });

// Runs only on mount
useEffect(() => { /* ... */ }, []);

// Runs when `userId` changes
useEffect(() => {
  fetchUser(userId);
}, [userId]);

// Cleanup (runs on unmount or before re-run)
useEffect(() => {
  const subscription = subscribe();
  return () => subscription.unsubscribe(); // cleanup
}, []);
```

### useCallback
Memoize a function — prevents re-creation on every render.
```jsx
const handlePress = useCallback(() => {
  navigate('Details', { id: item.id });
}, [item.id]);
```

### useMemo
Memoize an expensive computed value.
```jsx
const sortedList = useMemo(
  () => data.sort((a, b) => a.name.localeCompare(b.name)),
  [data]
);
```

### useRef
Persistent mutable reference, doesn't trigger re-render.
```jsx
const inputRef = useRef(null);

// Focus input programmatically
inputRef.current?.focus();

// Timer ref
const timerRef = useRef(null);
useEffect(() => {
  timerRef.current = setTimeout(() => {}, 3000);
  return () => clearTimeout(timerRef.current);
}, []);
```

### useContext
```jsx
const ThemeContext = React.createContext('light');

// Provider
<ThemeContext.Provider value="dark">
  <App />
</ThemeContext.Provider>

// Consumer
const theme = useContext(ThemeContext);
```

### useReducer
For complex state logic.
```jsx
const reducer = (state, action) => {
  switch (action.type) {
    case 'INCREMENT': return { count: state.count + 1 };
    case 'DECREMENT': return { count: state.count - 1 };
    default: return state;
  }
};

const [state, dispatch] = useReducer(reducer, { count: 0 });
dispatch({ type: 'INCREMENT' });
```

### Custom Hooks
Extract reusable stateful logic:
```jsx
const useFetch = (url) => {
  const [data, setData] = useState(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);

  useEffect(() => {
    fetch(url)
      .then(res => res.json())
      .then(setData)
      .catch(setError)
      .finally(() => setLoading(false));
  }, [url]);

  return { data, loading, error };
};

// Usage
const { data, loading } = useFetch('https://api.example.com/users');
```

---

## 11. Navigation

### React Navigation Setup
```bash
npm install @react-navigation/native
npm install @react-navigation/native-stack
npm install react-native-screens react-native-safe-area-context
```

### Stack Navigator
```jsx
import { NavigationContainer } from '@react-navigation/native';
import { createNativeStackNavigator } from '@react-navigation/native-stack';

const Stack = createNativeStackNavigator();

const App = () => (
  <NavigationContainer>
    <Stack.Navigator initialRouteName="Home">
      <Stack.Screen name="Home" component={HomeScreen} />
      <Stack.Screen
        name="Details"
        component={DetailsScreen}
        options={{ title: 'Detail View', headerShown: true }}
      />
    </Stack.Navigator>
  </NavigationContainer>
);
```

### Navigating
```jsx
// Navigate to screen
navigation.navigate('Details', { id: 42, name: 'Aashik' });

// Go back
navigation.goBack();

// Push (adds to stack even if already exists)
navigation.push('Details');

// Replace current screen
navigation.replace('Login');

// Reset entire stack
navigation.reset({ index: 0, routes: [{ name: 'Home' }] });
```

### Receiving Params
```jsx
const DetailsScreen = ({ route, navigation }) => {
  const { id, name } = route.params;
  return <Text>{name}</Text>;
};
```

### Bottom Tab Navigator
```bash
npm install @react-navigation/bottom-tabs
```
```jsx
import { createBottomTabNavigator } from '@react-navigation/bottom-tabs';

const Tab = createBottomTabNavigator();

<Tab.Navigator>
  <Tab.Screen name="Home" component={HomeScreen} />
  <Tab.Screen name="Profile" component={ProfileScreen} />
</Tab.Navigator>
```

### Drawer Navigator
```bash
npm install @react-navigation/drawer
```
```jsx
import { createDrawerNavigator } from '@react-navigation/drawer';
const Drawer = createDrawerNavigator();

<Drawer.Navigator>
  <Drawer.Screen name="Home" component={HomeScreen} />
</Drawer.Navigator>
```

---

## 12. Networking & APIs

### Fetch API
```jsx
const fetchData = async () => {
  try {
    const response = await fetch('https://jsonplaceholder.typicode.com/posts', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ title: 'Hello', body: 'World' }),
    });

    if (!response.ok) throw new Error(`HTTP error: ${response.status}`);

    const data = await response.json();
    setData(data);
  } catch (err) {
    setError(err.message);
  }
};
```

### Axios
```bash
npm install axios
```
```jsx
import axios from 'axios';

const api = axios.create({
  baseURL: 'https://api.example.com',
  timeout: 10000,
  headers: { 'Authorization': `Bearer ${token}` },
});

// GET
const { data } = await api.get('/users');

// POST
const { data } = await api.post('/users', { name: 'Aashik' });

// Interceptors
api.interceptors.response.use(
  res => res,
  err => {
    if (err.response?.status === 401) logout();
    return Promise.reject(err);
  }
);
```

### React Query (TanStack Query)
```bash
npm install @tanstack/react-query
```
```jsx
import { useQuery, useMutation, QueryClient, QueryClientProvider } from '@tanstack/react-query';

const queryClient = new QueryClient();

// Provider
<QueryClientProvider client={queryClient}>
  <App />
</QueryClientProvider>

// Query
const { data, isLoading, error } = useQuery({
  queryKey: ['users'],
  queryFn: () => fetch('/api/users').then(r => r.json()),
});

// Mutation
const mutation = useMutation({
  mutationFn: (newUser) => axios.post('/users', newUser),
  onSuccess: () => queryClient.invalidateQueries({ queryKey: ['users'] }),
});
```

---

## 13. Platform-Specific Code

### Platform Module
```jsx
import { Platform } from 'react-native';

const styles = StyleSheet.create({
  container: {
    paddingTop: Platform.OS === 'ios' ? 20 : 0,
    ...Platform.select({
      ios: { shadowColor: '#000', shadowOffset: { width: 0, height: 2 } },
      android: { elevation: 4 },
    }),
  },
});
```

### Platform-Specific Files
React Native automatically picks the correct file:
```
Button.ios.tsx      ← used on iOS
Button.android.tsx  ← used on Android
Button.tsx          ← fallback for both
```

### Version Detection
```jsx
const isIOS = Platform.OS === 'ios';
const isAndroid = Platform.OS === 'android';
const isOldAndroid = Platform.Version < 21; // API level
```

---

## 14. Images & Media

### Local Images
```jsx
<Image
  source={require('./assets/photo.png')}
  style={{ width: 200, height: 200 }}
  resizeMode="cover"  // cover | contain | stretch | repeat | center
/>
```

### Remote Images
Always provide explicit dimensions for remote images.
```jsx
<Image
  source={{ uri: 'https://picsum.photos/200' }}
  style={{ width: 200, height: 200 }}
  defaultSource={require('./placeholder.png')}
  onError={(e) => console.log(e.nativeEvent.error)}
/>
```

### Fast Image (Performance)
```bash
npm install react-native-fast-image
```
```jsx
import FastImage from 'react-native-fast-image';

<FastImage
  source={{ uri: url, priority: FastImage.priority.high }}
  style={{ width: 100, height: 100 }}
  resizeMode={FastImage.resizeMode.cover}
/>
```

### Video
```bash
npm install react-native-video
```
```jsx
import Video from 'react-native-video';

<Video
  source={{ uri: 'https://example.com/video.mp4' }}
  style={{ width: 300, height: 200 }}
  controls
  resizeMode="cover"
/>
```

---

## 15. Forms & User Input

### Controlled Inputs
```jsx
const LoginForm = () => {
  const [email, setEmail] = useState('');
  const [password, setPassword] = useState('');

  return (
    <View>
      <TextInput
        value={email}
        onChangeText={setEmail}
        placeholder="Email"
        autoCapitalize="none"
        keyboardType="email-address"
        returnKeyType="next"
      />
      <TextInput
        value={password}
        onChangeText={setPassword}
        placeholder="Password"
        secureTextEntry
        returnKeyType="done"
        onSubmitEditing={handleLogin}
      />
    </View>
  );
};
```

### React Hook Form
```bash
npm install react-hook-form
```
```jsx
import { useForm, Controller } from 'react-hook-form';

const { control, handleSubmit, formState: { errors } } = useForm();

<Controller
  control={control}
  name="email"
  rules={{ required: 'Email is required' }}
  render={({ field: { onChange, value } }) => (
    <TextInput onChangeText={onChange} value={value} />
  )}
/>

{errors.email && <Text style={{ color: 'red' }}>{errors.email.message}</Text>}
```

### KeyboardAvoidingView
Prevents keyboard from covering inputs.
```jsx
import { KeyboardAvoidingView, Platform } from 'react-native';

<KeyboardAvoidingView
  behavior={Platform.OS === 'ios' ? 'padding' : 'height'}
  style={{ flex: 1 }}
>
  {/* form content */}
</KeyboardAvoidingView>
```

---

## 16. Animations

### Animated API (Built-in)
```jsx
import { Animated } from 'react-native';

const fadeAnim = useRef(new Animated.Value(0)).current;

// Fade in
const fadeIn = () => {
  Animated.timing(fadeAnim, {
    toValue: 1,
    duration: 500,
    useNativeDriver: true,  // runs on UI thread, much smoother
  }).start();
};

<Animated.View style={{ opacity: fadeAnim }}>
  <Text>Fading in!</Text>
</Animated.View>
```

### Animation Types

```jsx
// Spring (bouncy, physics-based)
Animated.spring(value, {
  toValue: 1,
  friction: 5,
  tension: 40,
  useNativeDriver: true,
}).start();

// Decay (inertia)
Animated.decay(value, {
  velocity: 0.5,
  deceleration: 0.998,
  useNativeDriver: true,
}).start();

// Sequence (one after another)
Animated.sequence([anim1, anim2, anim3]).start();

// Parallel (at the same time)
Animated.parallel([anim1, anim2]).start();

// Stagger (parallel with delay between each)
Animated.stagger(100, [anim1, anim2, anim3]).start();
```

### Interpolation
Map animation values to other ranges:
```jsx
const rotation = fadeAnim.interpolate({
  inputRange: [0, 1],
  outputRange: ['0deg', '360deg'],
});

const scale = fadeAnim.interpolate({
  inputRange: [0, 0.5, 1],
  outputRange: [1, 1.5, 1],
  extrapolate: 'clamp',
});

<Animated.View style={{ transform: [{ rotate: rotation }, { scale }] }} />
```

### Reanimated 2 (Recommended for complex animations)
```bash
npm install react-native-reanimated
```
```jsx
import Animated, {
  useSharedValue,
  useAnimatedStyle,
  withSpring,
  withTiming,
} from 'react-native-reanimated';

const offset = useSharedValue(0);

const animatedStyle = useAnimatedStyle(() => ({
  transform: [{ translateX: offset.value }],
}));

// Trigger animation
offset.value = withSpring(100);

<Animated.View style={[styles.box, animatedStyle]} />
```

---

## 17. Gestures

### PanResponder (Built-in)
```jsx
import { PanResponder, Animated } from 'react-native';

const pan = useRef(new Animated.ValueXY()).current;

const panResponder = useRef(
  PanResponder.create({
    onStartShouldSetPanResponder: () => true,
    onPanResponderMove: Animated.event(
      [null, { dx: pan.x, dy: pan.y }],
      { useNativeDriver: false }
    ),
    onPanResponderRelease: () => {
      Animated.spring(pan, { toValue: { x: 0, y: 0 }, useNativeDriver: false }).start();
    },
  })
).current;

<Animated.View
  {...panResponder.panHandlers}
  style={[styles.box, { transform: pan.getTranslateTransform() }]}
/>
```

### React Native Gesture Handler
```bash
npm install react-native-gesture-handler
```
```jsx
import { GestureDetector, Gesture } from 'react-native-gesture-handler';
import Animated, { useSharedValue, useAnimatedStyle, withSpring } from 'react-native-reanimated';

const offset = useSharedValue({ x: 0, y: 0 });

const panGesture = Gesture.Pan()
  .onUpdate((e) => {
    offset.value = { x: e.translationX, y: e.translationY };
  })
  .onEnd(() => {
    offset.value = withSpring({ x: 0, y: 0 });
  });

const animatedStyle = useAnimatedStyle(() => ({
  transform: [{ translateX: offset.value.x }, { translateY: offset.value.y }],
}));

<GestureDetector gesture={panGesture}>
  <Animated.View style={[styles.box, animatedStyle]} />
</GestureDetector>
```

---

## 18. Native Modules & Bridging

### What is the Bridge?
The original React Native architecture uses a **Bridge** to communicate between JavaScript and native code via **JSON-serialized messages**.

```
JS Thread  ──(serialized JSON)──▶  Bridge  ──▶  Native Thread (UI, Shadow)
```

### Creating a Native Module (Android — Java/Kotlin)

**MyModule.kt**
```kotlin
class MyModule(reactContext: ReactApplicationContext) :
    ReactContextBaseJavaModule(reactContext) {

  override fun getName() = "MyModule"

  @ReactMethod
  fun showToast(message: String) {
    Toast.makeText(reactApplicationContext, message, Toast.LENGTH_SHORT).show()
  }

  @ReactMethod
  fun getDeviceName(promise: Promise) {
    promise.resolve(Build.MODEL)
  }
}
```

**JS Usage:**
```jsx
import { NativeModules } from 'react-native';
const { MyModule } = NativeModules;

MyModule.showToast('Hello from JS!');
const name = await MyModule.getDeviceName();
```

### Creating a Native Module (iOS — Objective-C/Swift)

**MyModule.m**
```objc
#import <React/RCTBridgeModule.h>

@interface RCT_EXTERN_MODULE(MyModule, NSObject)
RCT_EXTERN_METHOD(showAlert:(NSString *)title)
@end
```

**MyModule.swift**
```swift
@objc(MyModule)
class MyModule: NSObject {
  @objc func showAlert(_ title: String) {
    DispatchQueue.main.async {
      // show alert
    }
  }
}
```

### Native Events (NativeEventEmitter)
```jsx
import { NativeEventEmitter, NativeModules } from 'react-native';

const emitter = new NativeEventEmitter(NativeModules.MyModule);
const subscription = emitter.addListener('onDataReceived', (data) => {
  console.log(data);
});

// Cleanup
return () => subscription.remove();
```

---

## 19. State Management

### Context + useReducer (Built-in)
Good for small-medium apps.
```jsx
const AppContext = React.createContext();

const reducer = (state, action) => {
  switch (action.type) {
    case 'SET_USER': return { ...state, user: action.payload };
    default: return state;
  }
};

export const AppProvider = ({ children }) => {
  const [state, dispatch] = useReducer(reducer, { user: null });
  return (
    <AppContext.Provider value={{ state, dispatch }}>
      {children}
    </AppContext.Provider>
  );
};

export const useApp = () => useContext(AppContext);
```

### Zustand (Lightweight, modern)
```bash
npm install zustand
```
```jsx
import { create } from 'zustand';

const useStore = create((set) => ({
  count: 0,
  user: null,
  increment: () => set((state) => ({ count: state.count + 1 })),
  setUser: (user) => set({ user }),
}));

// Component
const { count, increment } = useStore();
```

### Redux Toolkit
```bash
npm install @reduxjs/toolkit react-redux
```
```jsx
// slice
import { createSlice } from '@reduxjs/toolkit';

const counterSlice = createSlice({
  name: 'counter',
  initialState: { value: 0 },
  reducers: {
    increment: (state) => { state.value += 1; },
    decrement: (state) => { state.value -= 1; },
  },
});

export const { increment, decrement } = counterSlice.actions;

// store
import { configureStore } from '@reduxjs/toolkit';
const store = configureStore({ reducer: { counter: counterSlice.reducer } });

// Component
import { useSelector, useDispatch } from 'react-redux';
const count = useSelector((state) => state.counter.value);
const dispatch = useDispatch();
dispatch(increment());
```

### Async Storage (Persistence)
```bash
npm install @react-native-async-storage/async-storage
```
```jsx
import AsyncStorage from '@react-native-async-storage/async-storage';

// Save
await AsyncStorage.setItem('user', JSON.stringify(user));

// Read
const raw = await AsyncStorage.getItem('user');
const user = raw ? JSON.parse(raw) : null;

// Delete
await AsyncStorage.removeItem('user');
```

---

## 20. Performance Optimization

### Key Principles

1. **Use `useNativeDriver: true`** for animations whenever possible — runs on UI thread, no bridge crossing.

2. **Memoize components** with `React.memo`
```jsx
const ListItem = React.memo(({ item, onPress }) => (
  <TouchableOpacity onPress={() => onPress(item.id)}>
    <Text>{item.title}</Text>
  </TouchableOpacity>
));
```

3. **Memoize callbacks** passed to children
```jsx
const handlePress = useCallback((id) => {
  navigation.navigate('Detail', { id });
}, [navigation]);
```

4. **Avoid anonymous functions** in JSX
```jsx
// Bad — creates new function every render
<Button onPress={() => doSomething(item.id)} />

// Good
<Button onPress={handlePress} />
```

5. **Use FlatList instead of ScrollView** for long lists

6. **Avoid setState in render** — causes infinite loops

7. **Batch setState calls** (React 18 auto-batches)

8. **Use getItemLayout** in FlatList for fixed-height items
```jsx
<FlatList
  getItemLayout={(data, index) => ({
    length: ITEM_HEIGHT,
    offset: ITEM_HEIGHT * index,
    index,
  })}
  // ...
/>
```

9. **Avoid deep component trees** — each extra layer adds reconciliation cost

10. **Hermes Engine** — enable it for better performance on Android (default in RN 0.70+)

### Profiling
- Use **React DevTools Profiler**
- Use **Systrace** for native performance
- Use **Flipper** for debugging and performance monitoring

---

## 21. Testing

### Unit Testing with Jest
```bash
npm install --save-dev jest @testing-library/react-native
```

```jsx
// Button.test.tsx
import { render, fireEvent } from '@testing-library/react-native';
import MyButton from './MyButton';

describe('MyButton', () => {
  it('renders correctly', () => {
    const { getByText } = render(<MyButton title="Submit" onPress={() => {}} />);
    expect(getByText('Submit')).toBeTruthy();
  });

  it('calls onPress when pressed', () => {
    const mockPress = jest.fn();
    const { getByText } = render(<MyButton title="Submit" onPress={mockPress} />);
    fireEvent.press(getByText('Submit'));
    expect(mockPress).toHaveBeenCalledTimes(1);
  });
});
```

### Mocking
```jsx
// Mock AsyncStorage
jest.mock('@react-native-async-storage/async-storage', () =>
  require('@react-native-async-storage/async-storage/jest/async-storage-mock')
);

// Mock navigation
const mockNavigate = jest.fn();
jest.mock('@react-navigation/native', () => ({
  useNavigation: () => ({ navigate: mockNavigate }),
}));
```

### E2E Testing with Detox
```bash
npm install --save-dev detox
```
```js
describe('Login flow', () => {
  it('logs in successfully', async () => {
    await element(by.id('emailInput')).typeText('user@test.com');
    await element(by.id('passwordInput')).typeText('password123');
    await element(by.id('loginButton')).tap();
    await expect(element(by.text('Welcome!'))).toBeVisible();
  });
});
```

---

## 22. Debugging

### Flipper
The official React Native debugger. Inspect network requests, layout, logs, and Redux store.

### Shake Device / Dev Menu
Shake your device (or `Cmd+D` on iOS simulator, `Cmd+M` on Android emulator) to access:
- Reload
- Enable Fast Refresh
- Toggle Inspector
- Open Debugger

### Console Logging
```jsx
console.log('Debug:', data);
console.warn('Warning:', msg);
console.error('Error:', err);
```

### React DevTools
```bash
npx react-devtools
```

### Network Debugging
Use Flipper's Network plugin, or intercept with:
```jsx
// Log all fetch requests
const originalFetch = global.fetch;
global.fetch = (url, options) => {
  console.log('Fetch:', url, options);
  return originalFetch(url, options);
};
```

### Error Boundaries
```jsx
class ErrorBoundary extends React.Component {
  state = { hasError: false };

  static getDerivedStateFromError() {
    return { hasError: true };
  }

  componentDidCatch(error, info) {
    // Log to crash reporting service
    Sentry.captureException(error);
  }

  render() {
    if (this.state.hasError) return <Text>Something went wrong.</Text>;
    return this.props.children;
  }
}
```

---

## 23. Security

### Secure Storage
Never store sensitive data (tokens, passwords) in AsyncStorage (unencrypted).
```bash
npm install react-native-keychain
```
```jsx
import * as Keychain from 'react-native-keychain';

// Save
await Keychain.setGenericPassword('user', authToken);

// Read
const credentials = await Keychain.getGenericPassword();
if (credentials) {
  const { username, password } = credentials;
}
```

### SSL Pinning
Prevent MITM attacks by pinning your server's certificate.
```bash
npm install react-native-ssl-pinning
```

### Obfuscation
Use ProGuard on Android to obfuscate native code.

### Avoid Storing Sensitive Data in JS Bundle
The JS bundle is readable. Never hardcode:
- API keys
- Private keys
- Passwords

Use environment variables via `.env` files and `react-native-config`.

### Input Validation
Always sanitize user input before sending to APIs.
```jsx
const sanitize = (input) => input.replace(/<[^>]*>/g, '').trim();
```

---

## 24. Deployment & Publishing

### Android — Release Build
```bash
# Generate keystore
keytool -genkey -v -keystore release.keystore -alias mykey -keyalg RSA -keysize 2048 -validity 10000

# Build
cd android && ./gradlew assembleRelease     # APK
cd android && ./gradlew bundleRelease       # AAB (for Play Store)
```

### iOS — Release Build
1. Open Xcode → select your team in Signing
2. Product → Archive
3. Distribute App → App Store Connect

### CodePush (OTA Updates)
Update JS code without going through app stores.
```bash
npm install react-native-code-push
appcenter codepush release-react -a <owner/app> -d Production
```

### Environment Variables
```bash
npm install react-native-config
```
```
# .env
API_URL=https://api.example.com
APP_NAME=MyApp
```
```jsx
import Config from 'react-native-config';
console.log(Config.API_URL);
```

---

## 25. The New Architecture

React Native's New Architecture replaces the old Bridge with faster, synchronous communication.

### Three Pillars

#### 1. JSI (JavaScript Interface)
Replaces the async Bridge. Allows JS to call native functions **synchronously** using C++ host objects — no JSON serialization needed.

```
Old: JS → serialize to JSON → Bridge → deserialize → Native (async)
New: JS → JSI → C++ → Native (synchronous, direct reference)
```

#### 2. Fabric (New Renderer)
The new UI layer. Renders components using C++ and supports **concurrent features** from React 18.
- **Concurrent Mode**: React can pause, abort, or prioritize rendering work
- **Synchronous layout**: Native can synchronously read layout info

#### 3. TurboModules
Lazy-loaded native modules. Only load when first called, not at startup.
```
Old NativeModules: All modules loaded at startup (slow boot)
TurboModules: Load on demand (fast boot)
```

### Codegen
Automatically generates type-safe native interfaces from TypeScript specs.
```typescript
// NativeModuleSpec.ts
import type { TurboModule } from 'react-native';
import { TurboModuleRegistry } from 'react-native';

export interface Spec extends TurboModule {
  getDeviceName(): Promise<string>;
}

export default TurboModuleRegistry.getEnforcing<Spec>('MyModule');
```

### Enabling New Architecture
```gradle
// android/gradle.properties
newArchEnabled=true
```
```ruby
# ios/Podfile
ENV['RCT_NEW_ARCH_ENABLED'] = '1'
```

---

## 26. React Native Internals

### The Threading Model

React Native uses **3 threads**:

| Thread | Responsibility |
|---|---|
| **JS Thread** | Runs your JavaScript/React code, reconciliation, business logic |
| **UI Thread (Main)** | Renders native views, handles gestures and touch events |
| **Shadow Thread** | Calculates layout using Yoga (Facebook's Flexbox engine) |

### The Reconciliation Process

1. You call `setState()` or a hook triggers re-render
2. React builds a new **Virtual DOM** (in-memory tree)
3. **Diffing algorithm** compares old vs. new tree (O(n) heuristic)
4. Only **changed** nodes are serialized and sent across the bridge
5. Native side applies the minimal set of UI mutations

### Yoga — The Layout Engine

React Native uses **Yoga**, a cross-platform C++ library that implements the Flexbox specification. It runs on the Shadow Thread and produces layout values (x, y, width, height) for each component.

### Metro Bundler

Metro is React Native's JavaScript bundler.
- Watches for file changes (Fast Refresh)
- Transforms JSX/TypeScript → plain JS
- Resolves `require()` and `import` paths
- Creates a single `bundle.js` for the device

### Fast Refresh vs Hot Reload

| | Hot Reload (old) | Fast Refresh (new) |
|---|---|---|
| Preserves state? | Sometimes | Yes (hooks-based) |
| Handles errors? | Breaks | Recovers gracefully |
| Re-runs full module? | Yes | Minimal re-run |

### The Virtual DOM & Fiber

React uses **Fiber** — an internal reconciliation engine introduced in React 16 — to break rendering work into **units** that can be paused, prioritized, and resumed.

```
Work Loop:
  ┌─────────────────────────────────────┐
  │  performUnitOfWork                  │
  │    → beginWork (process fiber)      │
  │    → completeWork (finalize fiber)  │
  │    → commitWork (apply to native)   │
  └─────────────────────────────────────┘
```

### Jordan Walke's Vision

Jordan Walke created React at Facebook, drawing inspiration from **XHP** (PHP components) and **FaxJS**. His core insight:

> **"UI is a pure function of state"** — `UI = f(state)`

React Native extended this to mobile: instead of rendering HTML, the same pure function renders **native components**. The brilliance is that the programming model stays identical across web, iOS, and Android — only the renderer changes.

Key contributions of this model:
- **Unidirectional data flow** — predictable, debuggable state
- **Component composition** — build complex UIs from simple pieces
- **Declarative rendering** — describe what you want, not how to do it
- **Learn once, write anywhere** — same mental model across platforms

---

## Quick Reference Cheatsheet

```jsx
// Component skeleton
import React, { useState, useEffect } from 'react';
import { View, Text, StyleSheet } from 'react-native';

interface Props {
  title: string;
}

const MyComponent: React.FC<Props> = ({ title }) => {
  const [data, setData] = useState(null);

  useEffect(() => {
    // side effect here
    return () => { /* cleanup */ };
  }, []);

  return (
    <View style={styles.container}>
      <Text style={styles.title}>{title}</Text>
    </View>
  );
};

const styles = StyleSheet.create({
  container: { flex: 1, padding: 16 },
  title: { fontSize: 24, fontWeight: 'bold' },
});

export default MyComponent;
```

---

*From beginner components to Jordan Walke's Fiber internals — this guide covers the complete React Native journey.*
