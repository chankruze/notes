---
title: Real-time order dispatch flow
tags:
  - system-design
  - firebase
  - rails
  - react-native
  - push-notification
created: 2026-03-01
updated:
status: draft
---
FCM can:
- Wake the app (even in background / killed state)
- Deliver order payload
- Open a specific screen
- Trigger a full-screen “Incoming Order” UI

But:
> FCM should only trigger the flow.  
> Your backend + WebSocket should manage the actual order state.

## 🚀 How Uber / Zomato Do It (Conceptually)

When customer books:
1. Backend creates active job
2. Backend selects nearby driver/partner
3. Backend sends high-priority FCM push
4. Partner app wakes up
5. App opens Incoming Order screen
6. Partner swipes to accept
7. App calls backend API to confirm
8. Backend locks job to that partner
## Implementation Strategy

### Step 1 — Backend Sends High Priority Data Message

Our Rails backend should send:

```json
{
  "message": {
    "token": "<DEVICE_TOKEN>",
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

Important:
- Use **data-only message**
	- It works in background
	- We control navigation manually
	- No default OS popup interference
- Use high priority
- Do NOT rely only on notification payload

### Step 2 — Handle Message In App

Inside messaging service:
```ts
messaging().onMessage(async remoteMessage => {
  handleIncomingOrder(remoteMessage.data);
});

// also
messaging().setBackgroundMessageHandler(async remoteMessage => {
  handleIncomingOrder(remoteMessage.data);
});
```

### Step 3 — Open Full-Screen Order UI

We have 2 options:
#### Option A — Navigate To Screen (Normal App State)

```ts
navigation.navigate('IncomingOrder', { orderId });
```

Works if app is foreground/background.
#### Option B — Full-Screen Intent (Android)

For Uber-style lock-screen takeover:

Use:
- Android fullScreenIntent
- High priority notification
- Custom native config

This requires Android native config:

```xml
android:showWhenLocked="true"
android:turnScreenOn="true"
```

And special notification channel.

**⚠️ Critical Warning**

FCM is unreliable for critical dispatch alone.

We MUST:

- Fetch order details from backend after receiving push
- Confirm order availability
- Use timeout logic server-side

Never trust push payload as source of truth.

###  Step 4 — Swipe To Accept UI

Inside IncomingOrder screen:
- Animated swipe button
- Countdown timer
- Auto-expire

When user swipes:

```ts
await api.post('/orders/accept', { order_id });
```

Backend must:
- Lock job atomically
- Return success/failure
# Notes

1. iOS does NOT allow full takeover unless:
	1. We use VoIP push
	2. Or CallKit style incoming call
	3. Uber uses:
		- PushKit (VoIP)
		- CallKit (That is more advanced.)
2. Two partners may receive same order.
	1. Backend must:
		- Use DB locking
		- Use status = "available"
		- First accept wins
		- Others get rejected

## Architecture Diagram

```text
Customer App
     ↓
Rails Backend
     ↓
FCM (high priority data)
     ↓
Partner App
     ↓
IncomingOrder Screen
     ↓
Accept API
     ↓
Backend locks job
```

##  Advanced Enhancement (Recommended)

After receiving push, Immediately open WebSocket connection to:
- Subscribe to order status updates
- Cancel UI if order taken by someone else
- Push = wakeup  
- WebSocket = real-time state
