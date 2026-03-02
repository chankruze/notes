---
title: Orchestrate 4-tier typography engine usage using useTypography() hook
tags:
  - react-native
created: 2026-03-02
updated:
status: draft
---
If you’ve built the 4-layer system correctly, a `useTypography()` hook is **not for scaling** — that’s already solved.

The hook’s job is orchestration.

It connects:
- 🎨 Theme state (font preference, font families)
- 📱 Device changes (orientation / width changes if supported)
- ⚡ Cached typography engine
- ♻️ React reactivity
## What `useTypography()` Actually Does

1. Reads current theme settings (font scale preference)
2. Resolves final font multiplier
3. Retrieves cached typography
4. Memoizes result
5. Returns stable reference
6. Recomputes only when needed

So components don’t care about any typography logic. They just do:

```tsx
const typography = useTypography();
```

That’s it.

### Without the Hook

We have to do this everywhere:

```ts
const scale = resolveFinalFontScale(fontPreference);
const typography = getCachedTypography(scale, fontFamilies);
```
### With the Hook

```tsx
const typography = useTypography();
```

## Full Example Implementation

Assume you have:
- ThemeContext providing:
    - `fontPreference`
    - `fontFamilies`
### Example Theme Context

```ts
// theme/ThemeContext.ts

import React, { createContext, useContext } from 'react';
import type { ThemeFontFamilies, ThemeFontScalePreference } from './types';

type ThemeState = {
  fontPreference: ThemeFontScalePreference;
  fontFamilies: ThemeFontFamilies;
};

export const ThemeContext = createContext<ThemeState | null>(null);

export const useTheme = () => {
  const ctx = useContext(ThemeContext);
  if (!ctx) throw new Error('ThemeProvider missing');
  return ctx;
};
```
### The `useTypography()` Hook

```ts
// theme/typography/useTypography.ts

import { useMemo } from 'react';
import { useTheme } from '../ThemeContext';
import { resolveFinalFontScale } from './scaleResolver';
import { getCachedTypography } from './cache';

export const useTypography = () => {
  const { fontPreference, fontFamilies } = useTheme();

  return useMemo(() => {
    const scale = resolveFinalFontScale(fontPreference);

    return getCachedTypography(scale, fontFamilies);
  }, [fontPreference, fontFamilies]);
};
```

### Why useMemo?

- Typography objects are large
- StyleSheet comparisons depend on reference stability
- Prevents unnecessary re-renders

## Width Reactive Version

If we want dynamic scaling on orientation change:

```ts
import { useWindowDimensions } from 'react-native';

export const useTypography = () => {
  const { width } = useWindowDimensions();
  const { fontPreference, fontFamilies } = useTheme();

  return useMemo(() => {
    const scale = resolveFinalFontScale(fontPreference, width);

    return getCachedTypography(scale, fontFamilies);
  }, [fontPreference, fontFamilies, width]);
};
```

Now typography adapts live on rotation.

## What The Hook Is NOT

- A scaling function
- A font calculator
- A replacement for BASE_TYPOGRAPHY
- A style generator

> It's a bridge between theme state and typography engine.
### What Components Look Like Now

```tsx
const typography = useTypography();

<Text style={typography.h1}>Hello</Text>
<Text style={typography.body}>World</Text>
```
## Summary

| Responsibility       | Why                 |
| -------------------- | ------------------- |
| Read theme state     | Centralization      |
| Resolve multiplier   | Device + preference |
| Fetch cached tokens  | Performance         |
| Memoize result       | Prevent re-renders  |
| Return stable object | Style stability     |
|                      |                     |
It keeps components dumb and typography smart 🤪

Next, we w'll integrate this with `StyleSheet.create` properly.