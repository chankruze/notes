---
title: End-to-end integration of Firebase Cloud Messaging (FCM)
tags:
  - firebase
  - react-native
  - android
  - ios
  - push-notification
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
```

Then:

```bash
cd ios && pod install && cd ..
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
    classpath 'com.google.gms:google-services:4.3.15'
  }
}
```

Add to `android/app/build.gradle`

At bottom:

```gradle
apply plugin: 'com.google.gms.google-services'
```

Enable Internet + Service. Make sure `AndroidManifest.xml` contains:

```xml
<uses-permission android:name="android.permission.INTERNET" />
```

Firebase usually auto-adds messaging service.

## Step 4 — iOS Setup (Quick)

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
import messaging from '@react-native-firebase/messaging';
import { useEffect } from 'react';
import { Alert } from 'react-native';

export const App = () => {
  useEffect(() => {
    async function init() {
      const authStatus = await messaging().requestPermission();
      const token = await messaging().getToken();
      console.log('FCM TOKEN:', token);

      messaging().onMessage(async remoteMessage => {
        Alert.alert(
          'Foreground Message',
          JSON.stringify(remoteMessage.data)
        );
      });
    }

    init();
  }, []);

  return <RootApp />;
};
```

## Step 6 — Send Real Push

Go to Firebase Console > Cloud Messaging > Send Message

1. Send to: Test device token (paste token from console)
2. Use following **Data Message**  and Press send. If you see alert → setup works

```json
{
  "type": "test",
  "hello": "world"
}
```
## ⚠️ Important Notes

- Physical device only (push doesn’t work reliably on emulator)
- App must be installed from debug build
- Android needs Google Play Services