---
title: End-to-end integration of Firebase Cloud Messaging (FCM)
tags:
  - firebase
  - react-native
  - push-notification
  - android
  - ios
created: 2026-03-01
updated:
status: draft
---
## Step 1 — Create Firebase Project

1. Go to Firebase Console
2. Create new project
3. Add Android app
4. Add iOS app (optional for now)
5. Download:
    - `google-services.json`
    - `GoogleService-Info.plist`

## Step 2 — Install Packages

```bash
npm install @react-native-firebase/app @react-native-firebase/messaging
npx pod-install ios
```

## Step 3 — Android Setup

Put:

```
android/app/google-services.json
```

Add to `android/build.gradle`

```gradle
buildscript {
  dependencies {
    classpath("com.google.gms:google-services:4.4.4")
  }
}
```

Add to `android/app/build.gradle`

```gradle
apply plugin: 'com.google.gms.google-services'
```

Enable Internet + Service. Make sure `AndroidManifest.xml` contains:

```xml
<uses-permission android:name="android.permission.INTERNET" />
<uses-permission android:name="android.permission.POST_NOTIFICATIONS" />
```

Firebase usually auto-adds messaging service.

## Step 4 — iOS Setup

Place:

```
ios/GoogleService-Info.plist
```

Then in Xcode:

Enable:
- Push Notifications
- Background Modes → Remote notifications

## Step 5 — Minimal Test Code

Add this temporarily inside `App.tsx`:

```ts
import { useEffect } from 'react';
import { Alert } from 'react-native';
import {
  getMessaging,
  requestPermission,
  getToken,
  onMessage,
} from '@react-native-firebase/messaging';

export const App = () => {
  useEffect(() => {
    const messaging = getMessaging();

    async function init() {
      const authStatus = await requestPermission(messaging);
      const token = await getToken(messaging);

	  console.log('Auth Status:', authStatus);
      console.log('FCM TOKEN:', token);
    }

    init();

    const unsubscribe = onMessage(messaging, async remoteMessage => {
      Alert.alert(
        'Foreground Message',
        JSON.stringify(remoteMessage.data)
      );
    });

    return unsubscribe; // cleanup
  }, []);

  return <RootApp />;
};
```

## Step 6 — Send Real Push

Go to https://firebase.google.com/docs/cloud-messaging/send/v1-api#complete-http-request-json-sample and click **Run** to open API explorer.

1. **Token**: Test device token (paste token from console)
2. Use following **Data Message**  and Press send. (If the app is open, you'll see the Alert.)

```json
{
  "message": {
    "token": "DEVICE_TOKEN",
    "android": {
      "priority": "HIGH"
    },
    "data": {
      "type": "incoming_order",
      "order_id": "12345"
    }
  }
}
```

To receive FCM messages in **background/quit state**, you have to register `setBackgroundMessageHandler`. With RN Firebase v22+ (modular API), this **must be defined at the root level**, outside React components.

Add this to `index.js`:

```tsx
import { AppRegistry } from 'react-native';
import App from './App';
import { name as appName } from './app.json';

import {
  getMessaging,
  setBackgroundMessageHandler,
} from '@react-native-firebase/messaging';

const messaging = getMessaging();

// REQUIRED for background / quit state messages
setBackgroundMessageHandler(messaging, async remoteMessage => {
  console.log('Message handled in background!', remoteMessage);
});

AppRegistry.registerComponent(appName, () => App);
```

We can keep our foreground handler inside the app:

```tsx
import { useEffect } from 'react';
import { Alert } from 'react-native';
import {
  getMessaging,
  onMessage,
} from '@react-native-firebase/messaging';

export const App = () => {
  useEffect(() => {
    const messaging = getMessaging();

    const unsubscribe = onMessage(messaging, async remoteMessage => {
      Alert.alert(
        'Foreground Message',
        JSON.stringify(remoteMessage.data)
      );
    });

    return unsubscribe;
  }, []);

  return <RootApp />;
};
```

To handle notification tap (to open screen etc.):

```tsx
import { getInitialNotification } from '@react-native-firebase/messaging';

useEffect(() => {
  const messaging = getMessaging();

  getInitialNotification(messaging).then(remoteMessage => {
    if (remoteMessage) {
      console.log('Opened from quit state:', remoteMessage);
    }
  });
}, []);
```

|App State|Handler|
|---|---|
|Foreground|`onMessage()`|
|Background|`setBackgroundMessageHandler()`|
|Quit / Terminated|`setBackgroundMessageHandler()`|
|Notification Tap|`getInitialNotification()`

## ⚠️ Data Messages

Since we're sending: `"data": { ... }`. This is a **data-only message**, which means:

- Foreground → handled by `onMessage`
- Background → handled by `setBackgroundMessageHandler`
- No system tray notification unless you create one manually

If we want Android to automatically show a notification, we must add:

```json
"notification": {  
  "title": "New Order",  
  "body": "You have a new order"  
}
```

## ⚠️ Important Notes

- Physical device only (push doesn’t work reliably on emulator)
- App must be installed from debug build
- Android needs Google Play Services
