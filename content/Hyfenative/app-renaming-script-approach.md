---
title: App renaming script approach
tags:
  - react-native
created: 2026-03-01
updated:
status: draft
---
## Goal

Make the template behave like Expo in terms of renaming, so changing app name and package identifier can be done via a single command:

```bash
npm run configure -- --name="ClientApp" --id="com.client.app"
```

The objective is to avoid manual editing across Android and iOS native files.
## Single Source of Truth

Use a TypeScript config file instead of scattered native edits: `hyfenative.config.ts`

```ts
export type HyfenativeConfig = {
  app: {
    name: string;
    slug: string;
    scheme: string;
  };
  android: {
    package: string;
  };
  ios: {
    bundleId: string;
  };
};

const config: HyfenativeConfig = {
  app: {
    name: 'Hyfenative',
    slug: 'hyfenative',
    scheme: 'hyfenative',
  },
  android: {
    package: 'com.hyfenative',
  },
  ios: {
    bundleId: 'com.hyfenative',
  },
};

export default config;
```

This file becomes the single source of truth for:
- App display name
- Slug
- Deep link scheme
- Android applicationId
- Android namespace
- iOS bundle identifier
## Android Strategy

Avoid fragile global search/replace. Use variables.
#### 1. gradle.properties

```properties
APP_ID=com.hyfenative
APP_NAME=Hyfenative
```
#### 2. android/app/build.gradle

```gradle
applicationId project.APP_ID
namespace project.APP_ID
```

Only update `APP_ID` and `APP_NAME` inside `gradle.properties` via the configure script.
#### 3. Rename Java/Kotlin Package Folder

Convert package id to folder path:

```
com.hyfenative → com/hyfenative
```

Move:

```
android/app/src/main/java/com/hyfenative
```

to:

```
android/app/src/main/java/com/client/app
```

Update only the first-line package declaration in:
- MainApplication
- MainActivity
### 4. AndroidManifest.xml

Update the root package attribute:

```xml
package="com.hyfenative"
```
## iOS Strategy

Do not aggressively modify `project.pbxproj`. Use xcconfig-based configuration.

#### 1. ios/AppConfig.xcconfig

```text
PRODUCT_BUNDLE_IDENTIFIER = com.hyfenative
APP_DISPLAY_NAME = Hyfenative
```
#### 2. Ensure Xcode Uses Variable

```
$(PRODUCT_BUNDLE_IDENTIFIER)
```

The configure script only updates values inside `AppConfig.xcconfig`.
#### 3. Update Info.plist Safely

Modify using proper XML parsing (not regex):
- CFBundleDisplayName
- CFBundleIdentifier
## package.json

Update the `name` field:

```json
"name": "clientapp"
```

Slugify the provided name.
## Deep Link Scheme

Replace scheme in:
- Android intent filters
- iOS URL types

Use the old value from `hyfenative.config.ts` to update safely.

## CLI Script

Create:

```
scripts/configure-app.ts
```

Responsibilities:

1. Read current `hyfenative.config.ts`
2. Parse CLI arguments (`--name`, `--id`)
3. Validate package id format
4. Compute slug and scheme
5. Update:
    - hyfenative.config.ts
    - gradle.properties
    - AndroidManifest.xml
    - Java/Kotlin package folder
    - xcconfig
    - Info.plist
    - package.json
6. Log success summary

Add to `package.json`:

```json
"scripts": {
  "configure": "ts-node scripts/configure-app.ts"
}
```
## Safety Rules

- No global search-replace across entire project
- Only modify targeted files
- Preserve formatting
- Make script idempotent
- Validate inputs before applying changes
- Abort safely on missing files
## Why This Approach

- Expo-like experience without Expo
- Centralized configuration
- Clean native separation
- Easy multi-client setup
- Future-proof for white-label and flavors
- Avoids manual Xcode and Gradle editing
## Core Principle

Native files should read from variables.  
The template should read from `hyfenative.config.ts`.  
The CLI should update configuration, not rewrite the entire project.