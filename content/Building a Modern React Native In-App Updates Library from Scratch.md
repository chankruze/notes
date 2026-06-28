---
title: Building a Modern React Native In-App Updates Library from Scratch
tags:
  - android
  - ios
  - react-native
created: 2026-06-28
updated:
status: draft
---
## Why I Built This

While working on [PoojaPath Partner](https://play.google.com/store/apps/details?id=geekofia.poojapatha.partner&hl=en_IN), a React Native application for managing puja service bookings, I needed to implement in-app updates — the mechanism that prompts users to update the app without leaving to the Play Store or App Store manually.

The obvious choice was [`sp-react-native-in-app-updates`](https://github.com/SudoPlz/sp-react-native-in-app-updates), the most popular library for this. After a close inspection, I found critical blockers:

| Problem | Impact |
|---|---|
| Uses old `ReactContextBaseJavaModule` (bridge architecture) | Breaks with React Native New Architecture (`newArchEnabled=true`) |
| iOS relies on `react-native-siren` v0.0.7 | Unmaintained since ~2019, no new arch support |
| Pinned deps: `react-native-device-info@10.3.0`, `underscore@1.12.1` | Security and compatibility risk |
| Last meaningful update: June 2024 | Maintenance concern |

My project runs RN **0.84.0** with `newArchEnabled=true`. The library would break immediately.

The iOS side was the real insight: `sp-react-native-in-app-updates` wraps `react-native-siren`, which just calls the iTunes Search API — an HTTP fetch. There's **no native iOS code needed at all**. The iOS implementation is pure TypeScript.

This led to the decision: build a lightweight, New Architecture-native replacement.

---
## Architecture Overview

The library is split cleanly by platform:

```
src/
├── types.ts                          ← Shared enums and types
├── NativeReactNativeInAppUpdates.ts  ← TurboModule codegen spec (Android)
├── InAppUpdates.ts                   ← Web/fallback platform stub
├── InAppUpdates.android.ts           ← TurboModule wrapper + NativeEventEmitter
├── InAppUpdates.ios.ts               ← Pure TypeScript: iTunes Search API
└── index.ts                          ← Unified public API
```

**Android** — A Kotlin TurboModule wrapping the Google Play In-App Updates API (`com.google.android.play:app-update-ktx:2.1.0`). Supports FLEXIBLE (background download) and IMMEDIATE (full-screen mandatory) update flows.

**iOS** — Zero native code. A pure TypeScript implementation that queries the iTunes Search API (`https://itunes.apple.com/lookup`), compares version strings, and opens the App Store via `Linking.openURL`.

This asymmetry is intentional. The Play Store has a native SDK that manages the entire download-and-install UI. The App Store does not — iOS in-app updates are just version detection + redirect.

---
## The TurboModule Spec

The codegen spec is the contract between JavaScript and the Kotlin native module. It lives in TypeScript and tells the React Native codegen exactly what methods to generate C++ bridge code for.

```typescript
// src/NativeReactNativeInAppUpdates.ts
import { TurboModuleRegistry, type TurboModule } from 'react-native';

export interface Spec extends TurboModule {
  checkForUpdate(): Promise<Object>;
  startUpdate(updateType: number): Promise<void>;
  installUpdate(): void;
  // Required by NativeEventEmitter
  addListener(eventType: string): void;
  removeListeners(count: number): void;
}

export default TurboModuleRegistry.getEnforcing<Spec>(
  'ReactNativeInAppUpdates'
);
```

Three key decisions here:

**`checkForUpdate()` returns `Promise<Object>` instead of a typed shape.** Codegen's type system doesn't support complex return types directly — it generates `Promise<Object>` at the C++ level. We cast it to `AndroidUpdateInfo` in the TypeScript wrapper, which is a safe cast since the Kotlin side always produces the correct shape.

**`startUpdate(updateType: number)`.** Codegen maps TypeScript `number` to Kotlin `Double`. The Kotlin implementation receives it as `Double` and converts with `.toInt()`. This is a known codegen quirk — always `Double` at the Kotlin boundary.

**`addListener`/`removeListeners` are mandatory.** These are required by React Native's `NativeEventEmitter` system. Without them, the emitter cannot track subscription counts across the bridge, and events are silently dropped.

The `package.json` `codegenConfig` wires this up:

```json
{
  "codegenConfig": {
    "name": "ReactNativeInAppUpdatesSpec",
    "type": "modules",
    "jsSrcsDir": "src",
    "android": {
      "javaPackageName": "com.chankruze.reactnativeinappupdates"
    }
  }
}
```

Running `yarn prepare` triggers `react-native-builder-bob`, which builds the JS output, and the codegen runs during the Android Gradle build to generate the C++ bridge and Java spec class.

---
## The Android Kotlin TurboModule

The Kotlin module extends `NativeReactNativeInAppUpdatesSpec` — the generated Java class that defines the abstract methods from our codegen spec. It also implements `ActivityEventListener` to receive the result of `startUpdateFlowForResult`.

```kotlin
class ReactNativeInAppUpdatesModule(reactContext: ReactApplicationContext) :
  NativeReactNativeInAppUpdatesSpec(reactContext), ActivityEventListener {

  private val appUpdateManager = AppUpdateManagerFactory.create(reactContext)
  private var listenerCount = 0
```

### The `InstallStateUpdatedListener`

For FLEXIBLE updates, the Play Store downloads the update in the background. We need to track download progress and signal completion to JavaScript so the app can call `installUpdate()` at the right moment.

```kotlin
private val installStateListener = InstallStateUpdatedListener { state ->
  if (listenerCount == 0) return@InstallStateUpdatedListener
  val map: WritableMap = Arguments.createMap().apply {
    putInt("status", state.installStatus())
    putDouble("bytesDownloaded", state.bytesDownloaded().toDouble())
    putDouble("totalBytesToDownload", state.totalBytesToDownload().toDouble())
  }
  emit(EVENT_STATUS, map)
}
```

The `listenerCount` guard prevents emitting events when no JavaScript subscribers exist — an important optimisation. The count is managed by the `addListener`/`removeListeners` methods required by codegen.

### `checkForUpdate`

```kotlin
override fun checkForUpdate(promise: Promise) {
  appUpdateManager.appUpdateInfo
    .addOnFailureListener { promise.reject("CHECK_FAILED", it.message, it) }
    .addOnSuccessListener { info ->
      val map: WritableMap = Arguments.createMap().apply {
        val available = info.updateAvailability() == UpdateAvailability.UPDATE_AVAILABLE
        putBoolean("updateAvailable", available)
        putInt("availabilityStatus", info.updateAvailability())
        putBoolean("flexibleAllowed", info.isUpdateTypeAllowed(AppUpdateType.FLEXIBLE))
        putBoolean("immediateAllowed", info.isUpdateTypeAllowed(AppUpdateType.IMMEDIATE))
        putInt("updatePriority", info.updatePriority())
        putInt("versionCode", info.availableVersionCode())
        val staleness = info.clientVersionStalenessDays()
        if (staleness != null) putInt("daysSinceRelease", staleness) else putNull("daysSinceRelease")
      }
      promise.resolve(map)
    }
}
```

`AppUpdateManager.appUpdateInfo` returns a `Task<AppUpdateInfo>` — Google Play's async result type. We bridge it to a JavaScript Promise via `addOnSuccessListener`/`addOnFailureListener`. The returned map includes everything a caller needs to make update type decisions: availability, priority, staleness days, and which update types are allowed.

### `startUpdate` with Graceful Fallback

The most interesting part of the Android implementation is the update type resolution:

```kotlin
private fun launchUpdateFlow(
  info: AppUpdateInfo,
  updateType: Int,
  activity: Activity,
  promise: Promise,
) {
  val resolvedType = when {
    info.isUpdateTypeAllowed(updateType) -> updateType
    info.isUpdateTypeAllowed(AppUpdateType.FLEXIBLE) -> AppUpdateType.FLEXIBLE
    info.isUpdateTypeAllowed(AppUpdateType.IMMEDIATE) -> AppUpdateType.IMMEDIATE
    else -> {
      promise.reject("UPDATE_TYPE_UNAVAILABLE", "Neither FLEXIBLE nor IMMEDIATE is allowed")
      return
    }
  }
  appUpdateManager.startUpdateFlowForResult(
    info, activity,
    AppUpdateOptions.newBuilder(resolvedType)
      .setAllowAssetPackDeletion(true)
      .build(),
    REQUEST_CODE,
  )
  promise.resolve(null)
}
```

Rather than rejecting immediately if the requested type isn't allowed, the module gracefully degrades: if IMMEDIATE isn't available, it tries FLEXIBLE, and vice versa. Only if neither is permitted does it reject. `setAllowAssetPackDeletion(true)` allows the Play Store to free up space by deleting unused asset packs — a recommendation from Google's documentation for large apps.

### Activity Result Handling

The FLEXIBLE update flow requires the host activity to handle `onActivityResult`:

```kotlin
override fun onActivityResult(
  activity: Activity, requestCode: Int, resultCode: Int, data: Intent?
) {
  if (requestCode != REQUEST_CODE) return
  val map: WritableMap = Arguments.createMap().apply {
    putBoolean("installed", resultCode == Activity.RESULT_OK)
  }
  emit(EVENT_RESULT, map)
}
```

`REQUEST_CODE = 42139` is an arbitrary constant used to identify our update request among other activity results the app might receive. When the user accepts or cancels the IMMEDIATE update dialog, this callback fires. `RESULT_OK` means the update was installed; any other code means the user cancelled.

### Resource Cleanup

```kotlin
override fun invalidate() {
  appUpdateManager.unregisterListener(installStateListener)
  super.invalidate()
}
```

`invalidate()` is called by React Native when the module is torn down (app reload in dev, or module GC). Unregistering the listener prevents memory leaks and ghost callbacks after the module is gone.

---
## The iOS Pure TypeScript Implementation

The iOS side has no native code whatsoever. The entire implementation is a fetch call to the iTunes Search API.

```typescript
// src/InAppUpdates.ios.ts
const ITUNES_LOOKUP = 'https://itunes.apple.com/lookup';

export async function checkForUpdate(
  options: CheckOptions = {}
): Promise<IosUpdateInfo> {
  const { bundleId, country } = options;

  if (!bundleId) {
    throw new Error('bundleId is required on iOS');
  }

  // Timestamp busts CDN/proxy cache — ensures fresh version data
  const params = new URLSearchParams({ bundleId, _: String(Date.now()) });
  if (country) params.set('country', country);

  const res = await fetch(`${ITUNES_LOOKUP}?${params}`, {
    cache: 'no-store',
  });

  const json = await res.json();
  const entry = json?.results?.[0];

  const storeVersion: string = entry.version;
  const releaseDate: string | null = entry.currentVersionReleaseDate ?? null;
  const appStoreUrl: string | null = entry.trackViewUrl
    ? ((entry.trackViewUrl as string).split('?')[0] ?? null)
    : null;

  const updateAvailable =
    options.curVersion != null
      ? compareVersions(storeVersion, options.curVersion) > 0
      : false;

  return { updateAvailable, storeVersion, releaseDate, appStoreUrl };
}
```

Several details worth noting:

**Cache busting with `_=Date.now()`.** CDNs and iOS's aggressive URL caching can cause the iTunes API to return stale version data. Adding a timestamp query parameter ensures the request always hits the origin. `cache: 'no-store'` provides a second layer of cache prevention at the fetch level.

**`appStoreUrl` stripped of query params.** iTunes `trackViewUrl` includes tracking parameters (`?uo=4&mt=8&...`). We strip everything after `?` to produce a clean `https://apps.apple.com/app/id123456789` URL suitable for `Linking.openURL`.

**Version comparison without semver.** Rather than pulling in the `semver` package, we implement a lightweight numeric comparison:

```typescript
function compareVersions(a: string, b: string): number {
  const aParts = a.split('.').map(Number);
  const bParts = b.split('.').map(Number);
  const len = Math.max(aParts.length, bParts.length);
  for (let i = 0; i < len; i++) {
    const diff = (aParts[i] ?? 0) - (bParts[i] ?? 0);
    if (diff !== 0) return diff;
  }
  return 0;
}
```

This correctly handles `1.10.0 > 1.2.0` because it compares numerically, not lexicographically. Zero extra dependencies.

**Android stubs on iOS.** The iOS file exports no-op stubs for Android-only functions:

```typescript
export function installUpdate(): void {}
export function startUpdate(_updateType?: number): Promise<void> {
  return Promise.resolve();
}
export function addUpdateListener<K extends UpdateEventName>(
  _event: K,
  _listener: (payload: UpdateEventMap[K]) => void
): () => void {
  return () => {};
}
```

This keeps the public API surface symmetric — callers don't need `Platform.OS` guards for every function call. The TypeScript type system ensures the no-ops are type-safe.

---
## The Platform-Split Architecture

React Native's Metro bundler resolves platform-specific files automatically:

```
InAppUpdates.android.ts  →  loaded on Android
InAppUpdates.ios.ts      →  loaded on iOS
InAppUpdates.ts          →  loaded on web/other (throws meaningful errors)
```

The `index.ts` exports from `./InAppUpdates` — Metro resolves this to the correct platform file at bundle time. There's no runtime `Platform.OS` switch anywhere in the library internals.

The base `InAppUpdates.ts` provides TypeScript's view of the API (used by the compiler for type checking) and throws descriptive errors on unsupported platforms:

```typescript
// InAppUpdates.ts — TypeScript reference + web/unknown platform fallback
const ERR = '@chankruze/react-native-in-app-updates is not supported on this platform';

export async function checkForUpdate(_options?: CheckOptions): Promise<UpdateInfo> {
  throw new Error(ERR);
}
```

---
## The Type System

```typescript
// src/types.ts

export enum UpdateType {
  FLEXIBLE = 0,   // Background download, user continues using app
  IMMEDIATE = 1,  // Full-screen mandatory update overlay
}

export enum InstallStatus {
  UNKNOWN = 0,
  PENDING = 1,
  DOWNLOADING = 2,
  INSTALLING = 3,
  INSTALLED = 4,
  FAILED = 5,
  CANCELED = 6,
  DOWNLOADED = 11, // Ready to install — call installUpdate()
}

export type AndroidUpdateInfo = {
  updateAvailable: boolean;
  availabilityStatus: AvailabilityStatus;
  flexibleAllowed: boolean;
  immediateAllowed: boolean;
  updatePriority: number;       // 0–5, server-set via Play Console
  daysSinceRelease: number | null;
  versionCode: number;
};

export type IosUpdateInfo = {
  updateAvailable: boolean;
  storeVersion: string;
  releaseDate: string | null;
  appStoreUrl: string | null;   // Clean App Store URL for Linking.openURL
};
```

The `UpdateInfo = AndroidUpdateInfo | IosUpdateInfo` union type enforces platform-appropriate usage at compile time. A caller that destructures `versionCode` on iOS gets a TypeScript error at compile time, not a runtime crash.

---
## Usage in PoojaPath Partner

The library is consumed through a custom hook that encapsulates the platform differences:

```typescript
// src/lib/in-app-updates/use-in-app-updates.ts
import { Platform } from 'react-native';
import {
  checkForUpdate,
  startUpdate,
  UpdateType,
  type IosUpdateInfo,
  type AndroidUpdateInfo,
} from '@chankruze/react-native-in-app-updates';

export function useInAppUpdates() {
  const [iosUpdateInfo, setIosUpdateInfo] = useState<IosUpdateInfo | null>(null);
  const hasChecked = useRef(false);

  useEffect(() => {
    if (hasChecked.current) return;
    hasChecked.current = true;

    const run = async () => {
      try {
        if (Platform.OS === 'android') {
          const info = (await checkForUpdate({})) as AndroidUpdateInfo;
          if (!info.updateAvailable) return;
          // IMMEDIATE forces the Play Store full-screen mandatory overlay
          await startUpdate(UpdateType.IMMEDIATE);
        } else {
          const info = (await checkForUpdate({
            bundleId: 'geekofia.poojapatha.partner',
            country: 'in',  // India-only app
          })) as IosUpdateInfo;
          if (info.updateAvailable) {
            setIosUpdateInfo(info);
          }
        }
      } catch {
        // Updates are best-effort — never interrupt the user on failure
      }
    };

    run();
  }, []);

  return { iosUpdateInfo };
}
```

On Android, IMMEDIATE update type forces the Play Store's full-screen overlay — the user cannot use the app until they update. This is the right choice for a partner/B2B app where all partners must be on the same version for operational consistency.

On iOS, the library returns version info and an App Store URL. The app renders its own modal and calls `Linking.openURL(info.appStoreUrl)`.

---
## The `publishConfig` Gotcha

Publishing scoped packages (`@chankruze/...`) to npm requires explicit access configuration. Without it, npm defaults to private — which requires a paid account and causes a confusing `403` error.

```json
{
  "publishConfig": {
    "registry": "https://registry.npmjs.org/",
    "access": "public"
  }
}
```

Publishing with 2FA enabled requires an OTP:

```bash
yarn prepare  # Build lib/ output
npm publish --access public --otp=YOUR_6_DIGIT_CODE
```

---
## Key Technical Decisions

### Why Not Nitro Modules?

[Nitro Modules](https://nitro.margelo.com) (by Marc Rousavy) are faster than TurboModules for synchronous, performance-critical operations. In-app updates are inherently async and infrequent — one check on app launch, one update flow. The JSI synchronous performance benefit of Nitro is irrelevant here. TurboModule is the correct, well-documented choice.

### Why Not the `semver` Package?

Adding `semver` as a production dependency introduces ~20KB to the bundle for a function that's 8 lines of plain arithmetic. The custom `compareVersions` handles all real-world version strings correctly including `1.10.0 > 1.2.0`.

### Why `create-react-native-library`?

The scaffolding tool handles the most error-prone parts of TurboModule setup: codegen spec wiring, `package.json` `codegenConfig`, Android `BaseReactPackage` registration, and `react-native-builder-bob` build configuration. Manual setup of these produces subtle bugs that are hard to diagnose.

### Development Workflow

During active development, Metro can resolve the library from source without rebuilding:

```js
// metro.config.js in the consuming app
const config = {
  watchFolders: [libraryRoot],
  resolver: {
    unstable_conditionNames: [
      'chankruze-react-native-in-app-updates-source',
      'require',
      'react-native',
    ],
  },
};
```

The `chankruze-react-native-in-app-updates-source` condition is declared in the library's `package.json` exports and points to `./src/index.tsx` — the raw TypeScript source. Metro resolves this directly, so TypeScript changes are reflected immediately without running `yarn prepare`.

---
## Limitations and Future Work

**`checkForUpdate` on Android always calls the Play Store API.** On debug builds, `AppUpdateInfo.updateAvailability()` always returns `UPDATE_NOT_AVAILABLE`. The library works correctly only on release builds installed from the Play Store (internal testing track or higher).

**No dark mode for the iOS App Store modal.** The consuming app renders its own modal — this is a feature, not a limitation. The app's design system controls the modal appearance.

---
## Conclusion

The total implementation is approximately 150 lines of Kotlin and 80 lines of TypeScript. The result is:

- ✅ **New Architecture (TurboModule)** — works with `newArchEnabled=true`
- ✅ **Zero extra dependencies** — no `react-native-device-info`, no `semver`, no `underscore`
- ✅ **No native iOS code** — iTunes Search API is pure TypeScript
- ✅ **Type-safe platform split** — compiler errors instead of runtime crashes
- ✅ **Published to npm** — `@chankruze/react-native-in-app-updates`

The exercise also clarified something worth remembering: **evaluate dependencies before accepting them**. A popular library is not the same as a correct library. The old-arch flag, unmaintained transitive dependency, and trivially replaceable iOS implementation were all visible from a 15-minute code review.

---
## Resources

- [Google Play In-App Updates API](https://developer.android.com/guide/playcore/in-app-updates)
- [iTunes Search API](https://developer.apple.com/library/archive/documentation/AudioVideo/Conceptual/iTuneSearchAPI/)
- [React Native TurboModules Guide](https://reactnative.dev/docs/next/the-new-architecture/pillars-turbomodule)
- [create-react-native-library](https://github.com/callstack/react-native-builder-bob)
- [npm: @chankruze/react-native-in-app-updates](https://www.npmjs.com/package/@chankruze/react-native-in-app-updates)
- [GitHub: chankruze/react-native-in-app-updates](https://github.com/chankruze/react-native-in-app-updates)
