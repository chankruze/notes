---
title: "Delivery Checklist: Server Verification + a Render-Perf Bug Hunt"
tags: []
created: 2026-08-25
updated:
status: draft
---
Delivery partners were able to mark a booking's items ("samagri") as delivered by entering a customer-shared OTP — with no check that they'd actually handed over everything. We added an **item-level verification checklist** as a real gate in front of that OTP step, enforced on both ends:

- **Backend**: `booking_items.delivery_verified_at`. `Booking#mark_items_delivered!`

now raises unless every deliverable item (package items/addons — not fees or discounts) is verified. New vendor-only endpoint: `PATCH /bookings/:uuid/booking_items/:item_uuid/verify_delivery`.

- **Partner app**: tapping "Deliver items" opens a checklist screen instead of jumping straight to OTP entry. "Continue to OTP" only enables once every item is checked, plus a "Select all" control with an explicit confirmation statement ("every item delivered in the quantity shown, nothing missing").

We chose the **partner (vendor) app**, not the customer app, for this — the partner is physically holding the items and is the one who submits the delivery OTP, so that's the natural place to gate the action. It's enforced server-side too, not just as a UI nicety.
## The bug: checkbox taps felt laggy

Once built, tapping a checklist item had a very noticeable ~300-500ms lag before the checkbox visually updated. Small detail, but on a screen a partner uses standing at someone's door, it read as broken.
### Ruling things out, one at a time

**First theory: network latency.** Each tap called the verify-delivery API directly and awaited it before updating the UI. Reasonable first guess — but switching to **pure local state** (no network call per tap at all, sync everything only when "Continue" is pressed) *still* felt laggy. That ruled out the network.

  **Second theory: dev-mode / Hermes interpreter overhead.** Metro's dev server

always ships `__DEV__ = true` JS — extra warnings, no JIT, general slowness that vanishes in a release build. Testing this rigorously without a full release build:

```bash
npm run android:no-metro
```

This builds a **production-mode bundle** (`--dev false`) but installs it via the fast debug-signed variant — isolates "dev JS vs release JS" without doing a full R8/ProGuard release build. Result: same ~300-500ms lag. Not dev-mode overhead either.
### Actually measuring it

At that point we stopped guessing and instrumented the screen with React's built-in `<Profiler>`:
```tsx
const logRender: ProfilerOnRenderCallback = (id, phase, actualDuration, baseDuration) => {
	if (!__DEV__) return;

	console.log(`[profile] ${id} ${phase} actual=${actualDuration}ms base=${baseDuration}ms`);
};

<Profiler id="delivery-items-checklist" onRender={logRender}>
<ChecklistSection ... />
</Profiler>
```

Sample output from a single tap:  

```
[profile] handleToggle -> setState scheduled in 0.39ms
[profile] delivery-items-checklist update actual=260.56ms base=246.80ms
```

Two things jumped out:

1. **The state update itself was instant** (0.39ms) — confirms local state was never the bottleneck.
2. **`actualDuration ≈ baseDuration`** on every render. `baseDuration` is

React's estimate of render cost *with zero memoization* — seeing the two numbers nearly equal means **every row in the list was being fully re-rendered on every single tap**, even though only one checkbox's state had actually changed. For a 3-row list, ~250ms of unmemoized reconciliation (image decode, icon glyph lookups, style resolution) was entirely plausible — and it explains why it was consistent across dev and production, local state and server calls: none of those experiments changed *how many rows re-rendered per tap*.
## The real fix

Extracted the row into its own component and wrapped it in `React.memo`, with the toggle handler stabilized via `useCallback` so its identity doesn't change every render (otherwise `memo` is a no-op — a new function prop every render defeats the shallow-equality check):

```tsx
const DeliveryItemChecklistRowComponent = ({ itemUuid, checked, onToggle, ... }) => {
	// ...row JSX...
};

export const DeliveryItemChecklistRow = memo(DeliveryItemChecklistRowComponent);
```

```tsx
// in the parent:
const toggleItem = useCallback((itemUuid: string, verified: boolean) => {
	mutate({ bookingUuid, itemUuid, payload: { verified } });
}, [bookingUuid, mutate]);
```

After this, tapping one row only re-renders that row. Taps became instant.
## Bringing back server persistence, safely

With local-only state we'd sidestepped the network to prove it wasn't the cause — but that meant a partner's progress lived only in memory, lost if they backgrounded the app or navigated away mid-checklist. Once the real bug (unmemoized re-renders) was fixed, we switched back to **per-tap API calls**, using an **optimistic update** so it stays instant:

```tsx
onMutate: ({ bookingUuid, itemUuid, payload }) => {
	const previousBooking = queryClient.getQueryData(queryKey);
	// Write the new state into the cache immediately — UI updates
	// before the network call even starts.
	queryClient.setQueryData(queryKey, (previous) => ({
		...previous,
		booking: { ...previous.booking, bookingItems: /* patched */
	},
}));

	return { previousBooking }; // for rollback
},

onError: (error, variables, context) => {
	// Roll back to the pre-tap snapshot and show a toast.
	if (context?.previousBooking) {
		queryClient.setQueryData(queryKey, context.previousBooking);
	}
},
```

`setQueryData` (direct cache write) rather than `invalidateQueries` (background refetch) was the deliberate choice here — for a tap that needs to *feel* instant, a refetch reintroduces the exact network round-trip we're trying to avoid, and by the time `onSuccess` fires we already have the server's authoritative response in hand, so there's nothing left to reconcile with a second fetch.
## Takeaways

- **Isolate variables one at a time.** Local-state-only and a production JS bundle test each eliminated one plausible cause cleanly, rather than guessing at a fix and hoping.
- **`React.Profiler` beats guessing.** `actualDuration ≈ baseDuration` is a very specific, readable signal: "nothing is memoized here." That one data point pointed straight at the fix.
- **Memoize before you reach for heavier tools.** We considered `FlashList` (virtualization) at one point — wrong tool for a 2-5 item list that's always fully visible; the problem was re-render count, not list size, and `React.memo` + stable callbacks fixed it directly.
- **Optimistic updates let you have both.** Instant-feeling UI *and* durable server state aren't mutually exclusive once the underlying render-cost bug is actually fixed.