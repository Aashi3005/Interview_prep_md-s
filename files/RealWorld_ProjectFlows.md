# Real-World Project Flows & Push Notifications
### Scenario-Based Deep Dive — Every Flow You'll Build in Production

---

## Table of Contents

1. [Push Notifications — Complete System](#1-push-notifications--complete-system)
2. [Authentication Flow](#2-authentication-flow)
3. [OTP / Phone Auth Flow](#3-otp--phone-auth-flow)
4. [Social Login Flow](#4-social-login-flow)
5. [Onboarding Flow](#5-onboarding-flow)
6. [Deep Linking Flow](#6-deep-linking-flow)
7. [Payment & Checkout Flow](#7-payment--checkout-flow)
8. [Chat & Messaging Flow](#8-chat--messaging-flow)
9. [File Upload / Media Flow](#9-file-upload--media-flow)
10. [E-Commerce Flow (Cart → Order → Tracking)](#10-e-commerce-flow)
11. [Offline Support Flow](#11-offline-support-flow)
12. [Location & Maps Flow](#12-location--maps-flow)
13. [App Update Flow](#13-app-update-flow)
14. [Permission Handling Flow](#14-permission-handling-flow)
15. [Session Management & Token Refresh Flow](#15-session-management--token-refresh-flow)
16. [Crash Reporting & Analytics Flow](#16-crash-reporting--analytics-flow)
17. [Background Tasks Flow](#17-background-tasks-flow)
18. [Biometric Auth Flow](#18-biometric-auth-flow)

---

## 1. Push Notifications — Complete System

### How Push Notifications Work (The Full Picture)

```
Your App (Device)
      ↓  (registers for push)
APNs (iOS) / FCM (Android)
      ↓  (returns device token)
Your App sends token → Your Backend
      ↓
Backend stores token in DB (linked to user)
      ↓
Trigger event (new message, payment, promo)
      ↓
Backend calls FCM/APNs API with token + payload
      ↓
FCM/APNs delivers to device
      ↓
Device shows notification / wakes app (silent push)
```

---

### Scenario 1.1 — App is Foregrounded (User is actively using the app)

**What happens:** The OS does NOT show a system notification banner. The notification arrives as a JS event only.

**Your job:** Handle it yourself — show an in-app banner, update a badge, play a sound.

```jsx
import messaging from '@react-native-firebase/messaging';
import { useEffect } from 'react';

useEffect(() => {
  // Foreground message handler
  const unsubscribe = messaging().onMessage(async (remoteMessage) => {
    // remoteMessage = { notification: { title, body }, data: {...} }
    showInAppBanner({
      title: remoteMessage.notification?.title,
      body: remoteMessage.notification?.body,
    });

    // Update badge count, refresh feed, etc.
    dispatch(incrementUnreadCount());
  });

  return unsubscribe; // cleanup on unmount
}, []);
```

**Why this matters:** If you don't handle foreground messages, the user sees nothing even though a notification arrived. Always build your own in-app toast/banner for this case.

---

### Scenario 1.2 — App is Backgrounded (Running but not visible)

**What happens:** The OS shows a system notification banner automatically using the `notification` payload. The `data` payload is also delivered silently.

**Your job:** Handle what happens when the user **taps** the notification.

```jsx
useEffect(() => {
  // User tapped the notification while app was in background
  messaging().onNotificationOpenedApp((remoteMessage) => {
    const { screen, id } = remoteMessage.data;
    // Navigate to the relevant screen
    navigation.navigate(screen, { id });
  });
}, []);
```

**Common mistake:** Developers forget to handle the background tap. User taps a "New Order" notification and the app opens to the home screen instead of the order screen — bad UX.

---

### Scenario 1.3 — App is Killed / Quit (Cold start)

**What happens:** User taps notification, OS launches the app from scratch. The notification that triggered the launch is available via `getInitialNotification()`.

```jsx
useEffect(() => {
  // Check if app was opened by a notification (cold start)
  messaging()
    .getInitialNotification()
    .then((remoteMessage) => {
      if (remoteMessage) {
        const { screen, id } = remoteMessage.data;
        // Set initial route before rendering navigator
        setInitialRoute({ screen, id });
      }
    });
}, []);
```

**Why it's tricky:** `getInitialNotification()` must be called BEFORE the navigator renders. If you call it after, the navigation ref isn't ready and the redirect fails.

---

### Scenario 1.4 — Silent Push Notification (Background Data Sync)

**What happens:** No notification is shown. The app is woken up silently to do work — sync data, refresh cache, download content.

**Payload structure:**
```json
{
  "data": { "type": "sync", "resource": "messages" },
  "apns": { "headers": { "apns-push-type": "background" } },
  "android": { "priority": "normal" }
}
```

```jsx
// Register background handler (outside component, at module level)
messaging().setBackgroundMessageHandler(async (remoteMessage) => {
  if (remoteMessage.data?.type === 'sync') {
    await syncMessagesFromServer(); // fetch new data, update local DB
  }
});
```

**Real use case:** WhatsApp uses silent pushes to sync messages when the app is killed. iMessage, Gmail — all use this pattern.

---

### Scenario 1.5 — FCM Token Management

Every device gets a **unique FCM token**. You must:
1. Get the token after permissions are granted
2. Send it to your backend
3. Refresh it when it changes (FCM rotates tokens)
4. Delete it on logout

```jsx
const setupPushToken = async () => {
  // 1. Request permission (iOS only — Android auto-grants)
  const authStatus = await messaging().requestPermission();
  const enabled =
    authStatus === messaging.AuthorizationStatus.AUTHORIZED ||
    authStatus === messaging.AuthorizationStatus.PROVISIONAL;

  if (!enabled) return;

  // 2. Get token
  const token = await messaging().getToken();

  // 3. Send to backend (only if changed)
  const storedToken = await AsyncStorage.getItem('fcmToken');
  if (token !== storedToken) {
    await api.post('/users/push-token', { token });
    await AsyncStorage.setItem('fcmToken', token);
  }

  // 4. Listen for token refresh
  messaging().onTokenRefresh(async (newToken) => {
    await api.post('/users/push-token', { token: newToken });
    await AsyncStorage.setItem('fcmToken', newToken);
  });
};

// On logout — delete token from backend
const handleLogout = async () => {
  const token = await messaging().getToken();
  await api.delete('/users/push-token', { data: { token } });
  await messaging().deleteToken();
  await AsyncStorage.removeItem('fcmToken');
};
```

---

### Scenario 1.6 — Notification Channels (Android)

Android 8+ requires **channels** to be created before sending notifications. Each channel has its own sound, vibration, and importance settings.

```jsx
import notifee, { AndroidImportance } from '@notifee/react-native';

const createChannels = async () => {
  await notifee.createChannel({
    id: 'orders',
    name: 'Order Updates',
    importance: AndroidImportance.HIGH,
    sound: 'order_sound',
    vibration: true,
  });

  await notifee.createChannel({
    id: 'promotions',
    name: 'Promotions',
    importance: AndroidImportance.LOW, // won't pop up, just in drawer
  });

  await notifee.createChannel({
    id: 'messages',
    name: 'Messages',
    importance: AndroidImportance.HIGH,
  });
};
```

**Real impact:** A food delivery app uses HIGH importance for "Your order is ready" but LOW for promotional offers. Users can disable promo channel without missing order updates.

---

### Scenario 1.7 — Local Notifications (No server needed)

Triggered from within the app itself — reminders, alarms, scheduled alerts.

```jsx
import notifee, { TriggerType } from '@notifee/react-native';

// Immediate local notification
const showLocalNotification = async () => {
  await notifee.displayNotification({
    title: 'Reminder',
    body: 'Your meeting starts in 5 minutes',
    android: { channelId: 'reminders', pressAction: { id: 'default' } },
  });
};

// Scheduled notification (e.g., tomorrow 9am)
const scheduleNotification = async (date) => {
  const trigger = {
    type: TriggerType.TIMESTAMP,
    timestamp: date.getTime(),
  };

  await notifee.createTriggerNotification(
    { title: 'Good morning!', body: 'Time to check your tasks', android: { channelId: 'reminders' } },
    trigger
  );
};
```

---

### Scenario 1.8 — Notification with Action Buttons

```jsx
await notifee.displayNotification({
  title: 'New message from Aashik',
  body: 'Hey, are you free tonight?',
  android: {
    channelId: 'messages',
    actions: [
      { title: 'Reply', pressAction: { id: 'reply' } },
      { title: 'Mark Read', pressAction: { id: 'mark_read' } },
    ],
  },
});

// Handle action press
notifee.onBackgroundEvent(async ({ type, detail }) => {
  if (type === EventType.ACTION_PRESS && detail.pressAction.id === 'reply') {
    navigation.navigate('Chat', { userId: detail.notification.data.userId });
  }
  if (detail.pressAction.id === 'mark_read') {
    await markMessageAsRead(detail.notification.data.messageId);
    await notifee.cancelNotification(detail.notification.id);
  }
});
```

---

## 2. Authentication Flow

### Scenario 2.1 — Standard Email/Password Login

```
User enters email + password
        ↓
Frontend validates (non-empty, valid email format)
        ↓
POST /auth/login  →  Backend
        ↓
Backend: check email exists → verify password hash (bcrypt)
        ↓
  Success: return { accessToken, refreshToken, user }
  Failure: return 401 { message: "Invalid credentials" }
        ↓
App stores tokens securely (Keychain/Keystore)
        ↓
Set default auth header: axios.defaults.headers['Authorization'] = `Bearer ${token}`
        ↓
Navigate to Home
```

```jsx
const login = async (email, password) => {
  setLoading(true);
  try {
    const { data } = await api.post('/auth/login', { email, password });

    // Store tokens securely
    await Keychain.setGenericPassword('tokens', JSON.stringify({
      accessToken: data.accessToken,
      refreshToken: data.refreshToken,
    }));

    // Set auth header for all future requests
    api.defaults.headers.common['Authorization'] = `Bearer ${data.accessToken}`;

    // Store user in state
    dispatch(setUser(data.user));

    navigation.reset({ index: 0, routes: [{ name: 'Main' }] });
  } catch (err) {
    setError(err.response?.data?.message || 'Login failed');
  } finally {
    setLoading(false);
  }
};
```

**Why `navigation.reset` instead of `navigation.navigate`?** Reset clears the navigation stack so the user can't press back to get to the Login screen after logging in.

---

### Scenario 2.2 — App Launch: Check Auth State

**Every time the app opens**, you must check if the user is already logged in.

```jsx
const checkAuthState = async () => {
  try {
    const stored = await Keychain.getGenericPassword();
    if (!stored) {
      // No token — send to auth screen
      setInitialRoute('Auth');
      return;
    }

    const { accessToken, refreshToken } = JSON.parse(stored.password);

    // Verify token is still valid with backend
    api.defaults.headers.common['Authorization'] = `Bearer ${accessToken}`;
    const { data } = await api.get('/auth/me');

    dispatch(setUser(data.user));
    setInitialRoute('Main');
  } catch (err) {
    // Token expired or invalid
    await attemptTokenRefresh(); // see Scenario 15
  } finally {
    setAppReady(true); // hide splash screen
  }
};
```

---

### Scenario 2.3 — Logout Flow

```jsx
const logout = async () => {
  try {
    // 1. Tell backend to invalidate refresh token
    await api.post('/auth/logout');
  } catch (_) {
    // Proceed even if backend call fails
  }

  // 2. Clear FCM token (stop receiving push notifications)
  await messaging().deleteToken();

  // 3. Clear stored credentials
  await Keychain.resetGenericPassword();

  // 4. Clear auth header
  delete api.defaults.headers.common['Authorization'];

  // 5. Clear app state
  dispatch(clearUser());
  queryClient.clear(); // clear all cached queries

  // 6. Reset navigation to Auth screen
  navigation.reset({ index: 0, routes: [{ name: 'Auth' }] });
};
```

**Why clear query cache on logout?** If User A logs out and User B logs in on the same device, User B would see User A's cached data without this step.

---

## 3. OTP / Phone Auth Flow

### Scenario 3.1 — Phone Number Verification

```
User enters phone number (+91 9876543210)
        ↓
POST /auth/send-otp  { phone: "+919876543210" }
        ↓
Backend generates 6-digit OTP → stores in Redis with 5-min TTL
        ↓
Backend sends SMS via Twilio/MSG91/Firebase
        ↓
App shows OTP input screen with 60-second resend timer
        ↓
User enters OTP
        ↓
POST /auth/verify-otp  { phone, otp }
        ↓
Backend: check OTP in Redis → match? → delete from Redis
        ↓
Return tokens (new user) or login (existing user)
```

```jsx
// OTP Input with auto-focus
const OTPInput = ({ length = 6, onComplete }) => {
  const [otp, setOtp] = useState(Array(length).fill(''));
  const inputs = useRef([]);

  const handleChange = (value, index) => {
    const newOtp = [...otp];
    newOtp[index] = value;
    setOtp(newOtp);

    // Auto-advance to next input
    if (value && index < length - 1) {
      inputs.current[index + 1]?.focus();
    }

    // Auto-submit when all filled
    if (newOtp.every(Boolean)) {
      onComplete(newOtp.join(''));
    }
  };

  const handleKeyPress = (e, index) => {
    // Auto-back on delete
    if (e.nativeEvent.key === 'Backspace' && !otp[index] && index > 0) {
      inputs.current[index - 1]?.focus();
    }
  };

  return (
    <View style={{ flexDirection: 'row', gap: 12 }}>
      {otp.map((digit, i) => (
        <TextInput
          key={i}
          ref={(ref) => (inputs.current[i] = ref)}
          value={digit}
          onChangeText={(v) => handleChange(v.slice(-1), i)}
          onKeyPress={(e) => handleKeyPress(e, i)}
          keyboardType="number-pad"
          maxLength={1}
          style={styles.otpBox}
        />
      ))}
    </View>
  );
};

// Resend timer
const ResendTimer = ({ onResend }) => {
  const [seconds, setSeconds] = useState(60);

  useEffect(() => {
    if (seconds === 0) return;
    const timer = setTimeout(() => setSeconds(s => s - 1), 1000);
    return () => clearTimeout(timer);
  }, [seconds]);

  return seconds > 0
    ? <Text>Resend in {seconds}s</Text>
    : <TouchableOpacity onPress={() => { setSeconds(60); onResend(); }}>
        <Text>Resend OTP</Text>
      </TouchableOpacity>;
};
```

---

### Scenario 3.2 — Auto-Read SMS OTP (Android)

Android can read the SMS automatically using the SMS Retriever API.

```jsx
import SmsRetriever from 'react-native-sms-retriever';

const startSmsRetriever = async () => {
  try {
    const registered = await SmsRetriever.startSmsRetriever();
    if (registered) {
      SmsRetriever.addSmsListener((event) => {
        // event.message = "<#> Your OTP is 482910 abc123def"
        const otp = event.message?.match(/\d{6}/)?.[0];
        if (otp) {
          setOtpValue(otp);
          verifyOtp(otp); // auto-submit
        }
        SmsRetriever.removeSmsListener();
      });
    }
  } catch (err) {
    console.log('SMS retriever not available');
  }
};
```

**The SMS must contain your app's hash** (generated from your signing key). The backend appends it: `"Your OTP is 482910 <app_hash>"`.

---

## 4. Social Login Flow

### Scenario 4.1 — Google Sign-In

```
User taps "Sign in with Google"
        ↓
Google Sign-In SDK opens Google account picker (native sheet)
        ↓
User selects account → Google authenticates
        ↓
App receives: { idToken, user: { name, email, photo } }
        ↓
App sends idToken to YOUR backend
        ↓
Backend verifies idToken with Google's public keys
        ↓
Backend: find or create user in DB
        ↓
Return your own JWT tokens
        ↓
App stores tokens, navigates to home
```

```jsx
import { GoogleSignin } from '@react-native-google-signin/google-signin';

GoogleSignin.configure({
  webClientId: 'YOUR_WEB_CLIENT_ID.apps.googleusercontent.com',
});

const signInWithGoogle = async () => {
  try {
    await GoogleSignin.hasPlayServices();
    const { idToken } = await GoogleSignin.signIn();

    // Send idToken to YOUR backend (never trust client-side data alone)
    const { data } = await api.post('/auth/google', { idToken });

    await Keychain.setGenericPassword('tokens', JSON.stringify(data.tokens));
    dispatch(setUser(data.user));
    navigation.reset({ index: 0, routes: [{ name: 'Main' }] });
  } catch (err) {
    if (err.code === statusCodes.SIGN_IN_CANCELLED) return; // user cancelled
    setError('Google sign-in failed');
  }
};
```

**Why send idToken to backend instead of using client data?** Anyone can fake a name/email on the client. The `idToken` is a signed JWT from Google — only Google's public key can verify it. Your backend must verify it server-side.

---

### Scenario 4.2 — Apple Sign-In (Required for iOS apps with social login)

Apple mandates that if you offer Google/Facebook login, you must also offer Sign in with Apple on iOS.

```jsx
import appleAuth from '@invertase/react-native-apple-authentication';

const signInWithApple = async () => {
  const appleAuthRequestResponse = await appleAuth.performRequest({
    requestedOperation: appleAuth.Operation.LOGIN,
    requestedScopes: [appleAuth.Scope.FULL_NAME, appleAuth.Scope.EMAIL],
  });

  // Apple only gives email on FIRST sign-in. Store it immediately.
  const { identityToken, fullName, email } = appleAuthRequestResponse;

  const { data } = await api.post('/auth/apple', {
    identityToken,
    name: `${fullName?.givenName} ${fullName?.familyName}`,
    email,
  });

  await Keychain.setGenericPassword('tokens', JSON.stringify(data.tokens));
  dispatch(setUser(data.user));
};
```

**Critical gotcha:** Apple only sends the user's email and name on the **very first** sign-in. On subsequent sign-ins, they're null. Your backend must save these on first sign-in and never expect them again.

---

## 5. Onboarding Flow

### Scenario 5.1 — First-Time User Onboarding

```
App launches
        ↓
Check AsyncStorage: 'hasSeenOnboarding'
        ↓
  false → Show onboarding slides
  true  → Go to Auth / Home
        ↓
User swipes through slides
        ↓
User taps "Get Started"
        ↓
Set 'hasSeenOnboarding' = true in AsyncStorage
        ↓
Navigate to Auth (signup/login)
```

```jsx
const OnboardingScreen = ({ navigation }) => {
  const slides = [
    { id: 1, title: 'Discover', subtitle: 'Find amazing products', image: require('./slide1.png') },
    { id: 2, title: 'Order', subtitle: 'Quick & easy checkout', image: require('./slide2.png') },
    { id: 3, title: 'Deliver', subtitle: 'Fast delivery to your door', image: require('./slide3.png') },
  ];

  const [activeIndex, setActiveIndex] = useState(0);
  const flatListRef = useRef(null);

  const handleNext = () => {
    if (activeIndex < slides.length - 1) {
      flatListRef.current?.scrollToIndex({ index: activeIndex + 1 });
    } else {
      handleFinish();
    }
  };

  const handleFinish = async () => {
    await AsyncStorage.setItem('hasSeenOnboarding', 'true');
    navigation.reset({ index: 0, routes: [{ name: 'Auth' }] });
  };

  return (
    <View style={{ flex: 1 }}>
      <FlatList
        ref={flatListRef}
        data={slides}
        horizontal
        pagingEnabled
        showsHorizontalScrollIndicator={false}
        onViewableItemsChanged={({ viewableItems }) =>
          setActiveIndex(viewableItems[0]?.index ?? 0)
        }
        renderItem={({ item }) => <OnboardingSlide {...item} />}
      />

      {/* Dot indicators */}
      <View style={styles.dotsContainer}>
        {slides.map((_, i) => (
          <View key={i} style={[styles.dot, i === activeIndex && styles.activeDot]} />
        ))}
      </View>

      <TouchableOpacity onPress={handleNext} style={styles.button}>
        <Text>{activeIndex === slides.length - 1 ? 'Get Started' : 'Next'}</Text>
      </TouchableOpacity>

      {activeIndex < slides.length - 1 && (
        <TouchableOpacity onPress={handleFinish}>
          <Text>Skip</Text>
        </TouchableOpacity>
      )}
    </View>
  );
};
```

---

### Scenario 5.2 — Permission Onboarding (Ask at right moment)

Best practice: ask for permissions **in context**, not on first launch.

```
Day 1: User sees onboarding → NO permission prompts yet
        ↓
User opens chat feature → ask for Microphone permission (with explanation why)
        ↓
User browses store → ask for Location (to show nearby stores)
        ↓
User receives first message → ask for Push Notification permission
```

This pattern is called **Just-in-Time Permission Requesting** and dramatically improves grant rates vs. asking all permissions on app open.

---

## 6. Deep Linking Flow

### Scenario 6.1 — Universal Links / App Links

A URL like `https://myapp.com/product/42` opens your app directly to that product screen.

```
User taps link in browser / WhatsApp / email
        ↓
OS checks apple-app-site-association (iOS) or assetlinks.json (Android)
        ↓
OS opens app (or App Store if not installed)
        ↓
React Navigation receives the URL
        ↓
Linking config maps URL → screen + params
        ↓
App navigates to ProductDetails screen with id=42
```

```jsx
// App.tsx — Linking configuration
const linking = {
  prefixes: ['https://myapp.com', 'myapp://'],
  config: {
    screens: {
      Main: {
        screens: {
          Home: 'home',
          ProductDetails: 'product/:id',    // myapp.com/product/42
          OrderTracking: 'order/:orderId',  // myapp.com/order/xyz123
          UserProfile: 'u/:username',       // myapp.com/u/aashik
        },
      },
      Auth: {
        screens: {
          ResetPassword: 'reset-password/:token',
        },
      },
    },
  },
};

<NavigationContainer linking={linking}>
  {/* ... */}
</NavigationContainer>
```

---

### Scenario 6.2 — Push Notification Deep Link

When user taps a notification, navigate them directly to the relevant screen.

```jsx
// Notification payload from backend:
// { data: { screen: 'OrderTracking', orderId: 'xyz123' } }

const handleNotificationPress = (remoteMessage) => {
  const { screen, ...params } = remoteMessage.data;
  if (navigationRef.isReady()) {
    navigationRef.navigate(screen, params);
  } else {
    // Store and navigate once nav is ready
    setPendingNavigation({ screen, params });
  }
};
```

**The navigationRef pattern** — needed because notifications can arrive before the navigator mounts:
```jsx
export const navigationRef = createNavigationContainerRef();

<NavigationContainer ref={navigationRef}>
```

---

## 7. Payment & Checkout Flow

### Scenario 7.1 — Standard Payment Flow (Razorpay / Stripe)

```
User reviews cart → taps "Pay ₹499"
        ↓
App calls backend: POST /orders/create
        ↓
Backend creates order in DB (status: PENDING)
Backend creates payment order with Razorpay (server-side API call)
Backend returns: { orderId, razorpayOrderId, amount, currency }
        ↓
App opens Razorpay payment sheet (SDK)
        ↓
User enters card / UPI / Netbanking
        ↓
Payment succeeds → Razorpay returns: { paymentId, orderId, signature }
        ↓
App sends these to backend: POST /orders/verify-payment
        ↓
Backend verifies Razorpay signature (HMAC-SHA256)
  Valid   → update order status to PAID → send confirmation notification
  Invalid → return 400 (tampered response)
        ↓
App shows success screen
```

```jsx
import RazorpayCheckout from 'react-native-razorpay';

const initiatePayment = async (cartTotal) => {
  setLoading(true);
  try {
    // Step 1: Create order on backend
    const { data: order } = await api.post('/orders/create', {
      items: cart,
      amount: cartTotal,
    });

    // Step 2: Open Razorpay SDK
    const options = {
      description: 'Order Payment',
      currency: 'INR',
      key: 'rzp_live_XXXXX',         // Razorpay key_id (public, safe on client)
      amount: order.amount * 100,    // in paise
      order_id: order.razorpayOrderId,
      name: 'My App',
      prefill: { email: user.email, contact: user.phone, name: user.name },
      theme: { color: '#6200EE' },
    };

    const paymentData = await RazorpayCheckout.open(options);

    // Step 3: Verify payment on backend (NEVER skip this)
    await api.post('/orders/verify-payment', {
      orderId: order.id,
      razorpayPaymentId: paymentData.razorpay_payment_id,
      razorpayOrderId: paymentData.razorpay_order_id,
      razorpaySignature: paymentData.razorpay_signature,
    });

    navigation.replace('OrderSuccess', { orderId: order.id });
  } catch (err) {
    if (err.code === 'PAYMENT_CANCELLED') return; // user cancelled
    navigation.navigate('OrderFailed', { reason: err.message });
  } finally {
    setLoading(false);
  }
};
```

**Why server-side verification is non-negotiable:** A malicious user can intercept the Razorpay callback and fake a success response. The HMAC signature verification on the backend ensures the payment actually happened and the amount is correct.

---

### Scenario 7.2 — Payment Failure Handling

```
Payment fails / network drops mid-payment
        ↓
App: show "Payment Failed" screen with reason
        ↓
Options: Retry (reuse same order ID) | Change method | Cancel
        ↓
If retrying: reuse existing Razorpay order_id (don't create new order)
        ↓
If network dropped: check payment status from backend before showing failure
(Razorpay may have charged the user even if the app didn't get the callback)
```

```jsx
const checkPaymentStatus = async (orderId) => {
  const { data } = await api.get(`/orders/${orderId}/payment-status`);
  // Backend checks with Razorpay API

  if (data.status === 'paid') {
    // Payment actually succeeded even though app didn't receive callback
    navigation.replace('OrderSuccess', { orderId });
  } else {
    navigation.replace('OrderFailed', { orderId });
  }
};
```

---

## 8. Chat & Messaging Flow

### Scenario 8.1 — Real-Time Chat with WebSockets

```
User opens chat screen
        ↓
App fetches last N messages from REST API (history)
        ↓
App connects to WebSocket server: ws://api.myapp.com/chat
        ↓
App sends auth token via WebSocket handshake or first message
        ↓
App joins room: { type: 'join', roomId: 'chat_xyz' }
        ↓
[Bidirectional real-time channel established]
        ↓
User types message → send: { type: 'message', text, roomId }
        ↓
Server broadcasts to all room members
        ↓
Other user's app receives message via WebSocket listener
        ↓
UI updates instantly (optimistic update)
```

```jsx
import { useRef, useEffect, useState } from 'react';

const useChat = (roomId) => {
  const [messages, setMessages] = useState([]);
  const [connected, setConnected] = useState(false);
  const ws = useRef(null);

  useEffect(() => {
    // Load history
    fetchMessages(roomId).then(setMessages);

    // Connect WebSocket
    ws.current = new WebSocket(`wss://api.myapp.com/chat?token=${token}`);

    ws.current.onopen = () => {
      setConnected(true);
      ws.current.send(JSON.stringify({ type: 'join', roomId }));
    };

    ws.current.onmessage = (event) => {
      const msg = JSON.parse(event.data);
      if (msg.type === 'message') {
        setMessages(prev => [...prev, msg]);
        // If chat is open, mark as read
        if (isChatOpen) markAsRead(roomId);
      }
      if (msg.type === 'typing') setTypingUser(msg.userId);
    };

    ws.current.onclose = () => {
      setConnected(false);
      // Auto-reconnect after 3 seconds
      setTimeout(() => reconnect(), 3000);
    };

    return () => ws.current?.close();
  }, [roomId]);

  const sendMessage = (text) => {
    // Optimistic update — show message immediately
    const tempMessage = { id: Date.now(), text, senderId: myId, status: 'sending' };
    setMessages(prev => [...prev, tempMessage]);

    ws.current.send(JSON.stringify({ type: 'message', text, roomId }));
  };

  return { messages, sendMessage, connected };
};
```

---

### Scenario 8.2 — Typing Indicator

```jsx
const handleTyping = () => {
  ws.current.send(JSON.stringify({ type: 'typing', roomId, userId: myId }));

  // Debounce — stop typing event after user pauses
  clearTimeout(typingTimeout.current);
  typingTimeout.current = setTimeout(() => {
    ws.current.send(JSON.stringify({ type: 'stop_typing', roomId }));
  }, 1500);
};

// Receiver side
ws.current.onmessage = (event) => {
  const msg = JSON.parse(event.data);
  if (msg.type === 'typing' && msg.userId !== myId) {
    setIsOtherUserTyping(true);
    clearTimeout(typingClearTimeout.current);
    typingClearTimeout.current = setTimeout(() => setIsOtherUserTyping(false), 2000);
  }
};
```

---

### Scenario 8.3 — Message Status (Sent → Delivered → Read)

```
Sender sends message → status: 'sent' (single tick)
        ↓
Server receives it → ACK sent back → status: 'delivered' (double tick)
        ↓
Recipient opens chat → sends read_receipt → status: 'read' (blue double tick)
```

```jsx
// Recipient marks messages as read
useEffect(() => {
  if (isFocused && messages.length > 0) {
    const unread = messages.filter(m => m.senderId !== myId && m.status !== 'read');
    if (unread.length > 0) {
      ws.current.send(JSON.stringify({
        type: 'read_receipt',
        roomId,
        messageIds: unread.map(m => m.id),
      }));
    }
  }
}, [isFocused, messages]);
```

---

## 9. File Upload / Media Flow

### Scenario 9.1 — Image Upload with Progress

```
User selects image from gallery / camera
        ↓
App optionally compresses image (reduce size)
        ↓
App creates FormData with image file
        ↓
POST /upload with multipart/form-data
        ↓
Show upload progress bar
        ↓
Backend returns URL of uploaded file
        ↓
App uses URL to display image / saves to profile
```

```jsx
import ImagePicker from 'react-native-image-crop-picker';
import ImageResizer from '@bam.tech/react-native-image-resizer';

const uploadProfilePhoto = async () => {
  // Step 1: Pick image
  const image = await ImagePicker.openPicker({
    width: 800,
    height: 800,
    cropping: true,
    cropperCircleOverlay: true,
    mediaType: 'photo',
  });

  // Step 2: Compress (optional but recommended)
  const compressed = await ImageResizer.createResizedImage(
    image.path, 800, 800, 'JPEG', 80
  );

  // Step 3: Upload with progress
  const formData = new FormData();
  formData.append('photo', {
    uri: compressed.uri,
    type: 'image/jpeg',
    name: 'profile.jpg',
  });

  const { data } = await api.post('/users/photo', formData, {
    headers: { 'Content-Type': 'multipart/form-data' },
    onUploadProgress: (progressEvent) => {
      const percent = Math.round(
        (progressEvent.loaded * 100) / progressEvent.total
      );
      setUploadProgress(percent);
    },
  });

  dispatch(updateUserPhoto(data.photoUrl));
};
```

---

### Scenario 9.2 — Resumable Upload (Large Files)

For large files (video), use chunked/resumable upload so network interruptions don't restart from scratch.

```jsx
const CHUNK_SIZE = 5 * 1024 * 1024; // 5MB

const uploadInChunks = async (fileUri, fileSize) => {
  // Step 1: Initialize upload session on backend
  const { data: session } = await api.post('/uploads/init', {
    fileSize,
    fileName: 'video.mp4',
    mimeType: 'video/mp4',
  });

  let offset = 0;
  while (offset < fileSize) {
    const chunk = await readFileChunk(fileUri, offset, CHUNK_SIZE);

    await api.put(`/uploads/${session.uploadId}/chunk`, chunk, {
      headers: {
        'Content-Range': `bytes ${offset}-${offset + chunk.size - 1}/${fileSize}`,
      },
    });

    offset += CHUNK_SIZE;
    setProgress(Math.min(offset / fileSize, 1));
  }

  // Step 2: Finalize upload
  const { data } = await api.post(`/uploads/${session.uploadId}/complete`);
  return data.url;
};
```

---

## 10. E-Commerce Flow

### Scenario 10.1 — Cart → Checkout → Order → Tracking

```
Browse Products
        ↓
Add to Cart → stored in local state + synced to backend (for cross-device cart)
        ↓
Cart Screen: review items, update quantities, remove items
        ↓
Checkout: enter address, select delivery slot
        ↓
Payment (see Scenario 7.1)
        ↓
Order Created → status: PENDING
        ↓
Backend triggers:
  - Confirmation email/SMS
  - Push notification: "Order #1234 confirmed!"
  - Vendor notification
        ↓
Order status updates (Webhook from logistics → Backend → Push to user):
  PENDING → CONFIRMED → PACKED → SHIPPED → OUT_FOR_DELIVERY → DELIVERED
        ↓
Live tracking: WebSocket or polling every 30s for GPS coordinates
```

```jsx
// Order status polling (alternative to WebSocket for tracking)
const useOrderTracking = (orderId) => {
  const [order, setOrder] = useState(null);

  useEffect(() => {
    let interval;
    const fetchStatus = async () => {
      const { data } = await api.get(`/orders/${orderId}`);
      setOrder(data);
      // Stop polling once delivered
      if (['DELIVERED', 'CANCELLED'].includes(data.status)) {
        clearInterval(interval);
      }
    };

    fetchStatus();
    interval = setInterval(fetchStatus, 30000); // poll every 30s

    return () => clearInterval(interval);
  }, [orderId]);

  return order;
};
```

---

### Scenario 10.2 — Optimistic Cart Update

```jsx
const addToCart = (product) => {
  // Optimistic: update UI immediately
  dispatch(addItemToCart(product));

  // Then sync with backend
  api.post('/cart/add', { productId: product.id, quantity: 1 })
    .catch(() => {
      // Rollback on failure
      dispatch(removeItemFromCart(product.id));
      showToast('Failed to add item. Please try again.');
    });
};
```

---

## 11. Offline Support Flow

### Scenario 11.1 — Detect Network State

```jsx
import NetInfo from '@react-native-community/netinfo';

const useNetworkStatus = () => {
  const [isConnected, setIsConnected] = useState(true);
  const [connectionType, setConnectionType] = useState(null);

  useEffect(() => {
    const unsubscribe = NetInfo.addEventListener(state => {
      setIsConnected(state.isConnected);
      setConnectionType(state.type); // wifi | cellular | none
    });
    return unsubscribe;
  }, []);

  return { isConnected, connectionType };
};
```

---

### Scenario 11.2 — Offline Queue (Actions taken while offline get synced when back online)

```
User is offline → taps "Like" on a post
        ↓
App saves action to offline queue in AsyncStorage
        ↓
App updates UI optimistically (shows liked state)
        ↓
Network comes back
        ↓
App processes queue: sends each queued action to backend
        ↓
Clear processed actions from queue
```

```jsx
const QUEUE_KEY = 'offline_action_queue';

const queueAction = async (action) => {
  const existing = await AsyncStorage.getItem(QUEUE_KEY);
  const queue = existing ? JSON.parse(existing) : [];
  queue.push({ ...action, timestamp: Date.now() });
  await AsyncStorage.setItem(QUEUE_KEY, JSON.stringify(queue));
};

const processOfflineQueue = async () => {
  const existing = await AsyncStorage.getItem(QUEUE_KEY);
  if (!existing) return;

  const queue = JSON.parse(existing);
  const failed = [];

  for (const action of queue) {
    try {
      await api.post('/actions', action);
    } catch {
      failed.push(action); // re-queue failed actions
    }
  }

  await AsyncStorage.setItem(QUEUE_KEY, JSON.stringify(failed));
};

// Process queue when network comes back
useEffect(() => {
  const unsubscribe = NetInfo.addEventListener(state => {
    if (state.isConnected) processOfflineQueue();
  });
  return unsubscribe;
}, []);
```

---

## 12. Location & Maps Flow

### Scenario 12.1 — Get User Location

```jsx
import Geolocation from '@react-native-community/geolocation';

const getUserLocation = () => {
  return new Promise((resolve, reject) => {
    Geolocation.getCurrentPosition(
      (position) => {
        resolve({
          latitude: position.coords.latitude,
          longitude: position.coords.longitude,
          accuracy: position.coords.accuracy,
        });
      },
      (error) => {
        // PERMISSION_DENIED | POSITION_UNAVAILABLE | TIMEOUT
        reject(error);
      },
      { enableHighAccuracy: true, timeout: 15000, maximumAge: 10000 }
    );
  });
};
```

---

### Scenario 12.2 — Continuous Location Tracking (Delivery Driver / Live Tracking)

```jsx
const startTracking = () => {
  const watchId = Geolocation.watchPosition(
    (position) => {
      const { latitude, longitude } = position.coords;

      // Send to backend every 10 seconds (throttle to avoid flooding)
      throttledUpdate({ latitude, longitude });

      // Update map marker
      setDriverLocation({ latitude, longitude });
    },
    (error) => console.log(error),
    { enableHighAccuracy: true, distanceFilter: 10 } // only update if moved 10m
  );

  return () => Geolocation.clearWatch(watchId);
};
```

---

### Scenario 12.3 — Background Location (iOS & Android)

For apps that track location even when backgrounded (running apps, delivery tracking):

```jsx
import BackgroundGeolocation from 'react-native-background-geolocation';

BackgroundGeolocation.ready({
  desiredAccuracy: BackgroundGeolocation.DESIRED_ACCURACY_HIGH,
  distanceFilter: 10,
  stopTimeout: 5,
  startOnBoot: true, // Android: start after device reboot
  url: 'https://api.myapp.com/location',  // auto-POST location to server
  autoSync: true,
  headers: { Authorization: `Bearer ${token}` },
}, (state) => {
  if (!state.enabled) BackgroundGeolocation.start();
});
```

---

## 13. App Update Flow

### Scenario 13.1 — Force Update (Breaking API changes)

```
App launches → API call: GET /app/version
        ↓
Backend returns: { minVersion: "2.0.0", currentVersion: "2.3.1", forceUpdate: true }
        ↓
Compare app version (from DeviceInfo) with minVersion
        ↓
If app version < minVersion:
  Show modal: "Update Required" (no dismiss button)
  Button → opens App Store / Play Store
        ↓
If app version < currentVersion (optional update):
  Show dismissible banner: "New version available"
```

```jsx
import DeviceInfo from 'react-native-device-info';

const checkForUpdate = async () => {
  const appVersion = DeviceInfo.getVersion(); // e.g., "1.9.0"
  const { data } = await api.get('/app/version');

  const isForceUpdate = compareVersions(appVersion, data.minVersion) < 0;
  const isOptionalUpdate = compareVersions(appVersion, data.currentVersion) < 0;

  if (isForceUpdate) {
    setUpdateModal({ visible: true, forced: true });
  } else if (isOptionalUpdate) {
    setUpdateBanner(true);
  }
};

const openStore = () => {
  const url = Platform.OS === 'ios'
    ? 'https://apps.apple.com/app/id123456789'
    : 'https://play.google.com/store/apps/details?id=com.myapp';
  Linking.openURL(url);
};
```

---

### Scenario 13.2 — OTA Update with CodePush

For JS-only changes (no native code), push updates without App Store review.

```jsx
import CodePush from 'react-native-code-push';

const codePushOptions = {
  checkFrequency: CodePush.CheckFrequency.ON_APP_RESUME,
  installMode: CodePush.InstallMode.ON_NEXT_RESUME,
  rollbackRetryOptions: { delayInHours: 8, maxRetryAttempts: 2 },
};

export default CodePush(codePushOptions)(App);
```

```
App resumes from background
        ↓
CodePush checks for update on AppCenter
        ↓
New bundle found → download in background
        ↓
Next time app goes to background + comes back → apply update
        ↓
User sees new version without going to App Store
```

---

## 14. Permission Handling Flow

### Scenario 14.1 — Graceful Permission Request

Never just call the permission API directly. Always:
1. Explain WHY you need it
2. Show a pre-permission rationale screen
3. Handle all states (granted, denied, blocked)

```jsx
import { check, request, PERMISSIONS, RESULTS } from 'react-native-permissions';

const requestCameraPermission = async () => {
  const permission = Platform.OS === 'ios'
    ? PERMISSIONS.IOS.CAMERA
    : PERMISSIONS.ANDROID.CAMERA;

  const status = await check(permission);

  switch (status) {
    case RESULTS.GRANTED:
      openCamera(); // already have it
      break;

    case RESULTS.DENIED:
      // Show rationale modal first
      setShowCameraRationale(true);
      break;

    case RESULTS.BLOCKED:
      // User previously denied + checked "Don't ask again"
      // Can only fix in Settings
      Alert.alert(
        'Camera Permission Required',
        'Please enable camera access in Settings to use this feature.',
        [
          { text: 'Cancel', style: 'cancel' },
          { text: 'Open Settings', onPress: () => Linking.openSettings() },
        ]
      );
      break;
  }
};

// After showing rationale and user agrees
const onRationaleAccept = async () => {
  setShowCameraRationale(false);
  const result = await request(PERMISSIONS.IOS.CAMERA);
  if (result === RESULTS.GRANTED) openCamera();
};
```

---

## 15. Session Management & Token Refresh Flow

### Scenario 15.1 — Automatic Access Token Refresh

Access tokens expire (usually 15min - 1hr). The app must transparently refresh them.

```
Any API call returns 401 Unauthorized
        ↓
Interceptor catches it
        ↓
Check: is this already a retry? (prevent infinite loop)
        ↓
Call POST /auth/refresh with refreshToken
        ↓
Backend validates refreshToken → returns new accessToken
        ↓
Update stored accessToken
        ↓
Retry the original failed request with new token
        ↓
If refresh also fails (refreshToken expired) → logout user
```

```jsx
let isRefreshing = false;
let failedQueue = [];

const processQueue = (error, token = null) => {
  failedQueue.forEach(prom => {
    if (error) prom.reject(error);
    else prom.resolve(token);
  });
  failedQueue = [];
};

api.interceptors.response.use(
  (response) => response,
  async (error) => {
    const originalRequest = error.config;

    if (error.response?.status === 401 && !originalRequest._retry) {
      if (isRefreshing) {
        // Queue this request until refresh completes
        return new Promise((resolve, reject) => {
          failedQueue.push({ resolve, reject });
        }).then(token => {
          originalRequest.headers['Authorization'] = `Bearer ${token}`;
          return api(originalRequest);
        });
      }

      originalRequest._retry = true;
      isRefreshing = true;

      try {
        const stored = await Keychain.getGenericPassword();
        const { refreshToken } = JSON.parse(stored.password);

        const { data } = await axios.post('/auth/refresh', { refreshToken });

        // Save new access token
        await Keychain.setGenericPassword('tokens', JSON.stringify({
          accessToken: data.accessToken,
          refreshToken: data.refreshToken ?? refreshToken,
        }));

        api.defaults.headers.common['Authorization'] = `Bearer ${data.accessToken}`;
        processQueue(null, data.accessToken);

        originalRequest.headers['Authorization'] = `Bearer ${data.accessToken}`;
        return api(originalRequest);
      } catch (refreshError) {
        processQueue(refreshError, null);
        // Refresh failed — force logout
        await logout();
        return Promise.reject(refreshError);
      } finally {
        isRefreshing = false;
      }
    }

    return Promise.reject(error);
  }
);
```

**Why the queue?** If 3 API calls fire simultaneously and all get 401, without the queue all 3 would try to refresh the token at once — causing race conditions. The queue makes one refresh happen and re-sends all 3 requests with the new token.

---

## 16. Crash Reporting & Analytics Flow

### Scenario 16.1 — Sentry Crash Reporting

```jsx
import * as Sentry from '@sentry/react-native';

// Initialize once at app start
Sentry.init({
  dsn: 'https://xxxxx@sentry.io/xxxxx',
  environment: __DEV__ ? 'development' : 'production',
  tracesSampleRate: 0.2, // sample 20% of transactions for performance
  attachStacktrace: true,
});

// Tag user for better error grouping
Sentry.setUser({ id: user.id, email: user.email });

// Manual error capture
try {
  await riskyOperation();
} catch (err) {
  Sentry.captureException(err, {
    tags: { screen: 'Checkout', action: 'payment_init' },
    extra: { cartTotal, userId },
  });
}

// Wrap App for automatic error boundary
export default Sentry.wrap(App);
```

---

### Scenario 16.2 — Analytics Events Flow

```jsx
import analytics from '@react-native-firebase/analytics';

// Track screens
const useAnalyticsScreen = (screenName) => {
  useFocusEffect(useCallback(() => {
    analytics().logScreenView({
      screen_name: screenName,
      screen_class: screenName,
    });
  }, [screenName]));
};

// Track events
const trackEvent = async (eventName, params = {}) => {
  await analytics().logEvent(eventName, {
    ...params,
    user_id: user?.id,
    timestamp: Date.now(),
  });
};

// Key events to track in every app:
trackEvent('signup_completed', { method: 'email' });
trackEvent('product_viewed', { productId, category });
trackEvent('add_to_cart', { productId, price });
trackEvent('checkout_started', { cartValue });
trackEvent('purchase_completed', { orderId, revenue: total });
trackEvent('notification_opened', { type: 'promo', campaignId });
```

---

## 17. Background Tasks Flow

### Scenario 17.1 — Background Fetch (Periodic data sync)

```jsx
import BackgroundFetch from 'react-native-background-fetch';

// Configure background fetch
BackgroundFetch.configure({
  minimumFetchInterval: 15, // minutes (iOS enforces minimum 15min)
  enableHeadless: true,     // Android: run even if app is killed
}, async (taskId) => {
  // This runs in background
  await syncNotifications();
  await syncMessages();
  await downloadPendingContent();

  BackgroundFetch.finish(taskId); // MUST call finish or iOS penalizes your app
}, (taskId) => {
  // Task timed out
  BackgroundFetch.finish(taskId);
});
```

---

### Scenario 17.2 — Headless Task (Android — run JS when app is killed)

```jsx
// index.js
import { AppRegistry } from 'react-native';

const HeadlessTask = async (taskData) => {
  // Runs when a background event fires even when app is killed
  if (taskData.type === 'sync') {
    await syncOfflineQueue();
  }
};

AppRegistry.registerHeadlessTask('SyncTask', () => HeadlessTask);
```

---

## 18. Biometric Auth Flow

### Scenario 18.1 — Fingerprint / Face ID Lock

```
App comes to foreground
        ↓
Check if biometric lock is enabled (AsyncStorage preference)
        ↓
Show blur overlay on app (screenshots also blurred for security)
        ↓
Prompt biometric auth: Face ID / Fingerprint
        ↓
Success → remove blur, restore app
Failure (3 attempts) → force PIN entry
        ↓
PIN correct → restore app
PIN wrong 5 times → force re-login
```

```jsx
import ReactNativeBiometrics from 'react-native-biometrics';

const rnBiometrics = new ReactNativeBiometrics();

const authenticateWithBiometrics = async () => {
  // Check what's available
  const { biometryType } = await rnBiometrics.isSensorAvailable();
  // biometryType: 'TouchID' | 'FaceID' | 'Biometrics' | null

  if (!biometryType) {
    // Device doesn't support biometrics — use PIN
    return promptPIN();
  }

  const { success, error } = await rnBiometrics.simplePrompt({
    promptMessage: 'Confirm your identity',
    cancelButtonText: 'Use PIN instead',
    fallbackPromptMessage: 'Use PIN',
  });

  if (success) {
    setAppLocked(false);
  } else if (error === 'UserCancel') {
    promptPIN();
  } else {
    setFailedAttempts(prev => prev + 1);
  }
};

// Trigger when app comes to foreground
const { appState } = useAppState();
useEffect(() => {
  if (appState === 'active' && biometricLockEnabled) {
    authenticateWithBiometrics();
  }
}, [appState]);
```

---

## Quick Reference: Which Flow Goes Where

| Scenario | When to implement |
|---|---|
| Push notification token setup | App launch, after login |
| Foreground notification handler | Root component (App.tsx) |
| Background notification handler | `index.js` (outside component tree) |
| Token refresh interceptor | Axios instance setup file |
| Deep link config | NavigationContainer |
| Biometric lock | AppState listener in root |
| Offline queue processing | NetInfo listener in root |
| Auth state check | Splash screen |
| OTA CodePush wrap | App entry point |
| Analytics screen tracking | Each screen component |

---

## The Golden Rule for Every Flow

> **Never trust the client.** Validate everything on the backend.
> - Payment success? Verify the signature server-side.
> - Social login? Verify the idToken server-side.
> - OTP? Verify and consume it server-side.
> - File upload? Validate type and size server-side.

The app is just the interface. The backend is the source of truth.
