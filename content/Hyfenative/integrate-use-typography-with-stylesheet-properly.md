---
title: How to integrate useTypography with StyleSheet.create properly
tags:
  - react-native
created: 2026-03-02
updated:
status: draft
---
If you use `useTypography()` naively with `StyleSheet.create`, you’ll either:

- ❌ recreate styles on every render
- ❌ lose memoization
- ❌ defeat your caching layer
- ❌ cause unnecessary re-renders

Let’s integrate it correctly.
## The Core Problem

This is wrong:

```tsx
const typography = useTypography();

const styles = StyleSheet.create({
  title: {
    ...typography.h1,
  },
});
```

Because:

- `useTypography()` runs during render
- `StyleSheet.create()` runs during render
- Styles are recreated every time

Even if cached internally, the object reference changes. We need controlled memoization.
## The Correct Pattern

## Pattern 1 — Memoized Dynamic Styles (Recommended)

```tsx
import React, { useMemo } from 'react';
import { StyleSheet, Text, View } from 'react-native';
import { useTypography } from '@/theme/typography/useTypography';

export const ExampleScreen = () => {
  const typography = useTypography();

  const styles = useMemo(
    () =>
      StyleSheet.create({
        container: {
          padding: 16,
        },
        title: {
          ...typography.h1,
        },
        body: {
          ...typography.body,
        },
      }),
    [typography],
  );

  return (
    <View style={styles.container}>
      <Text style={styles.title}>Heading</Text>
      <Text style={styles.body}>Body text</Text>
    </View>
  );
};
```
### Why This Is Correct

- `typography` is memoized    
- Styles only recreate when typography changes
- Runtime theme switching works
- Orientation changes work
- Accessibility changes work
- No unnecessary re-renders

This is production-safe.

## Pattern 2 — Split Static and Typography Styles

Separate static layout styles from dynamic text styles.

```tsx
const staticStyles = StyleSheet.create({
  container: {
    padding: 16,
  },
});
```

Then inside component:

```tsx
const typography = useTypography();

const textStyles = useMemo(
  () =>
    StyleSheet.create({
      title: typography.h1,
      body: typography.body,
    }),
  [typography],
);
```

Usage:

```tsx
<View style={staticStyles.container}>
  <Text style={textStyles.title}>Heading</Text>
  <Text style={textStyles.body}>Body</Text>
</View>
```

Now:

- Layout never recreates
- Only text styles update when needed

Very clean separation.

## Pattern 3 — `useThemedStyles` Hook 🏆

We can abstract this pattern completely.
### Create a Hook Wrapper

```ts
import { useMemo } from 'react';
import { StyleSheet } from 'react-native';
import { useTypography } from './useTypography';

export const useThemedStyles = <
  T extends StyleSheet.NamedStyles<T>
>(
  factory: (typography: ReturnType<typeof useTypography>) => T,
) => {
  const typography = useTypography();

  return useMemo(
    () => StyleSheet.create(factory(typography)),
    [typography],
  );
};
```
### Use It Like This

```tsx
const styles = useThemedStyles(typography => ({
  container: {
    padding: 16,
  },
  title: {
    ...typography.h1,
  },
  body: {
    ...typography.body,
  },
}));
```

Now your component is ultra clean:

```tsx
<Text style={styles.title}>Hello</Text>
```

## Why This Is the Cleanest Solution

- Typography logic stays centralized
- Components never import scale logic
- No duplication
- Fully reactive
- StyleSheet still used
- Zero performance issues
- Enterprise-safe
## ⚠️ What NOT To Do

❌ Don’t call `StyleSheet.create()` outside component if typography is dynamic  
❌ Don’t pass typography as props everywhere  
❌ Don’t compute scale inside components  
❌ Don’t inline large style objects
