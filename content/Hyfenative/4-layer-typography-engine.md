---
title: A Clean, Scalable, Production-grade 4-layer Typography Architecture
tags:
  - react-native
created: 2026-03-02
updated:
status: draft
---
## The 4-Layer Typogrpahy Architecture

```
Layer 1 → Base Tokens (Design System)
Layer 2 → Scale Resolver (Device + Preference Logic)
Layer 3 → Typography Builder (Applies multiplier)
Layer 4 → Cached Accessor (Performance)
```

Each layer has a single responsibility.
## Layer 1 — Base Tokens (Pure Design Data)

- No device logic.
- No scaling.
- Just design intent.

```ts
// theme/typography/base.ts

import type { ThemeTypography } from '../types';

export type BaseTypographyToken = {
  fontSize: number;
  lineHeight: number;
  letterSpacing?: number;
  fontWeight: ThemeTypography[keyof ThemeTypography]['fontWeight'];
  fontRole: 'regular' | 'semibold' | 'bold' | 'extrabold';
};

export const BASE_TYPOGRAPHY: Record<
  keyof ThemeTypography,
  BaseTypographyToken
> = {
  kicker: {
    fontSize: 12,
    lineHeight: 16,
    letterSpacing: 1,
    fontWeight: '600',
    fontRole: 'semibold',
  },

  label: {
    fontSize: 13,
    lineHeight: 18,
    fontWeight: '600',
    fontRole: 'semibold',
  },

  bodySm: {
    fontSize: 14,
    lineHeight: 20,
    fontWeight: '400',
    fontRole: 'regular',
  },

  body: {
    fontSize: 16,
    lineHeight: 24,
    fontWeight: '400',
    fontRole: 'regular',
  },

  h6: {
    fontSize: 18,
    lineHeight: 24,
    fontWeight: '600',
    fontRole: 'semibold',
  },

  h5: {
    fontSize: 20,
    lineHeight: 26,
    fontWeight: '600',
    fontRole: 'semibold',
  },

  h4: {
    fontSize: 24,
    lineHeight: 30,
    fontWeight: '700',
    fontRole: 'bold',
  },

  h3: {
    fontSize: 28,
    lineHeight: 34,
    fontWeight: '700',
    fontRole: 'bold',
  },

  h2: {
    fontSize: 32,
    lineHeight: 38,
    fontWeight: '800',
    fontRole: 'extrabold',
  },

  h1: {
    fontSize: 36,
    lineHeight: 42,
    fontWeight: '800',
    fontRole: 'extrabold',
  },

  button: {
    fontSize: 16,
    lineHeight: 20,
    fontWeight: '700',
    fontRole: 'bold',
  },
};
```

## Layer 2 — Scale Resolver (Device + User Logic)

This is where all intelligence lives.

```ts
// theme/typography/scaleResolver.ts

import { Dimensions, PixelRatio } from 'react-native';
import type { ThemeFontScalePreference } from '../types';

const { width } = Dimensions.get('window');
const GUIDELINE_BASE_WIDTH = 375;

const moderateWidthScale = (size = 1, factor = 0.5) => {
  const scale = width / GUIDELINE_BASE_WIDTH;
  return 1 + (scale - 1) * factor;
};

const APP_SCALE_MAP = {
  small: 0.9,
  medium: 1,
  large: 1.15,
} as const;

export const resolveFinalFontScale = (
  mode: ThemeFontScalePreference,
): number => {
  const widthScale = moderateWidthScale();

  const systemScale = PixelRatio.getFontScale();

  const appScale =
    mode === 'system' ? systemScale : APP_SCALE_MAP[mode];

  return widthScale * appScale;
};
```

Notice:

- Width scaling handled once
- Accessibility handled once
- App preference handled once
- Returns a pure multiplier

## Layer 3 — Typography Builder (Pure Transformer)

No device logic.
No PixelRatio.getFontScale().
Only applies multiplier.

```ts
// theme/typography/builder.ts

import { PixelRatio } from 'react-native';
import { BASE_TYPOGRAPHY } from './base';
import type { ThemeFontFamilies, ThemeTypography } from '../types';

const round = (value: number) =>
  PixelRatio.roundToNearestPixel(value);

export const buildTypography = (
  scale: number,
  fontFamilies: ThemeFontFamilies,
): ThemeTypography => {
  const result = {} as ThemeTypography;

  for (const key in BASE_TYPOGRAPHY) {
    const token = BASE_TYPOGRAPHY[key as keyof ThemeTypography];

    result[key as keyof ThemeTypography] = {
      fontSize: round(token.fontSize * scale),
      lineHeight: round(token.lineHeight * scale),
      fontWeight: token.fontWeight,
      fontFamily: fontFamilies[token.fontRole],
      ...(token.letterSpacing !== undefined && {
        letterSpacing: round(
          token.letterSpacing * Math.min(scale, 1.2),
        ),
      }),
    };
  }

  return result;
};
```

Builder is now:
- Deterministic
- Pure
- Easy to test
- Reusable

## Layer 4 — Cached Accessor (Performance Layer)

```ts
// theme/typography/cache.ts

import type { ThemeFontFamilies, ThemeTypography } from '../types';
import { buildTypography } from './builder';

const typographyCache = new Map<string, ThemeTypography>();

const getCacheKey = (
  scale: number,
  fontFamilies: ThemeFontFamilies,
) =>
  `${scale.toFixed(4)}::${Object.values(fontFamilies).join('|')}`;

export const getCachedTypography = (
  scale: number,
  fontFamilies: ThemeFontFamilies,
): ThemeTypography => {
  const key = getCacheKey(scale, fontFamilies);

  const cached = typographyCache.get(key);
  if (cached) return cached;

  const typography = buildTypography(scale, fontFamilies);

  typographyCache.set(key, typography);

  return typography;
};
```

## Final Usage

```ts
import { resolveFinalFontScale } from './scaleResolver';
import { getCachedTypography } from './cache';

const scale = resolveFinalFontScale(fontPreference);

const typography = getCachedTypography(
  scale,
  fontFamilies,
);
```

That’s it.

|Layer|Responsibility|Pure?|Testable?|
|---|---|---|---|
|Base|Design tokens|✅|✅|
|Resolver|Device logic|⚠ (RN dep)|Mostly|
|Builder|Apply multiplier|✅|✅|
|Cache|Performance|✅|✅|

Next I'll add the real example of how I'm going to use this engine!