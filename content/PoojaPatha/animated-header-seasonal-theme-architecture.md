---
title: 📐 Animated Collapsible Header + Seasonal Theme System
tags: []
created: 2026-03-01
updated:
status: draft
---
## Core Mental Model

This is not an `AppBar`. It's a production header system that is:

- Scroll-aware
- Theme-aware
- Layered
- Backend-configurable
- Reusable

## Collapsible Header (Reusable Primitive)

### 🎯 Responsibilities

- Expands
- Collapses on scroll
- Supports gradient
- Supports sticky sub-header
- Controls status bar blending
- Accepts content slots

### 🏗 Architecture

```
CollapsibleHeader
 ├── Animated Background Layer
 ├── Expanded Content
 ├── Sticky Content
 └── ScrollView Content
```
###  `CollapsibleHeader.tsx`

```tsx
import React from 'react';
import { View, StyleSheet, StatusBar } from 'react-native';
import Animated, {
  useSharedValue,
  useAnimatedScrollHandler,
  useAnimatedStyle,
  interpolate,
  interpolateColor,
  Extrapolate,
} from 'react-native-reanimated';
import LinearGradient from 'react-native-linear-gradient';

type Props = {
  expandedHeight: number;
  collapsedHeight: number;
  gradient: string[];
  renderExpanded: () => React.ReactNode;
  renderSticky: () => React.ReactNode;
  children: React.ReactNode;
};

export const CollapsibleHeader = ({
  expandedHeight,
  collapsedHeight,
  gradient,
  renderExpanded,
  renderSticky,
  children,
}: Props) => {
  const scrollY = useSharedValue(0);

  const onScroll = useAnimatedScrollHandler(event => {
    scrollY.value = event.contentOffset.y;
  });

  const headerStyle = useAnimatedStyle(() => {
    const height = interpolate(
      scrollY.value,
      [0, expandedHeight - collapsedHeight],
      [expandedHeight, collapsedHeight],
      Extrapolate.CLAMP
    );

    return { height };
  });

  const stickyOpacity = useAnimatedStyle(() => ({
    opacity: interpolate(
      scrollY.value,
      [expandedHeight - collapsedHeight - 20, expandedHeight - collapsedHeight],
      [0, 1],
      Extrapolate.CLAMP
    ),
  }));

  return (
    <View style={{ flex: 1 }}>
      <StatusBar translucent backgroundColor="transparent" />

      <Animated.View style={[styles.header, headerStyle]}>
        <LinearGradient
          colors={gradient}
          style={StyleSheet.absoluteFill}
        />

        <View style={styles.expanded}>
          {renderExpanded()}
        </View>

        <Animated.View style={[styles.sticky, stickyOpacity]}>
          {renderSticky()}
        </Animated.View>
      </Animated.View>

      <Animated.ScrollView
        onScroll={onScroll}
        scrollEventThrottle={16}
        contentContainerStyle={{
          paddingTop: expandedHeight,
        }}
      >
        {children}
      </Animated.ScrollView>
    </View>
  );
};

const styles = StyleSheet.create({
  header: {
    position: 'absolute',
    left: 0,
    right: 0,
    top: 0,
    zIndex: 10,
    overflow: 'hidden',
  },
  expanded: {
    flex: 1,
    justifyContent: 'flex-end',
    padding: 16,
  },
  sticky: {
    position: 'absolute',
    bottom: 0,
    left: 0,
    right: 0,
  },
});
```
## Production Home Header Composition

###  `HomeHeader.tsx`

```tsx
export const HomeHeader = () => {
  const theme = useThemeValue();

  return (
    <CollapsibleHeader
      expandedHeight={220}
      collapsedHeight={90}
      gradient={theme.dynamic.headerGradient}
      renderExpanded={() => <ExpandedHomeHeader />}
      renderSticky={() => <StickyHomeHeader />}
    >
      <HomeContent />
    </CollapsibleHeader>
  );
};
```

## Section-Based Header Color Transition

To animate background color per section:

```tsx
const backgroundStyle = useAnimatedStyle(() => ({
  backgroundColor: interpolateColor(
    scrollY.value,
    [0, 400, 800],
    ['#E3F6F5', '#FCE4EC', '#FFF3E0']
  ),
}));
```

Apply to header background layer.
## Top Banner Overlay System

### `TopBanner.tsx`

```tsx
import Animated, {
  useSharedValue,
  useAnimatedStyle,
  withTiming,
} from 'react-native-reanimated';

export const TopBanner = ({ visible, children }) => {
  const translateY = useSharedValue(-100);

  useEffect(() => {
    translateY.value = withTiming(visible ? 0 : -100, {
      duration: 300,
    });
  }, [visible]);

  const style = useAnimatedStyle(() => ({
    transform: [{ translateY: translateY.value }],
  }));

  return (
    <Animated.View
      style={[
        {
          position: 'absolute',
          top: 0,
          left: 0,
          right: 0,
          zIndex: 50,
        },
        style,
      ]}
    >
      {children}
    </Animated.View>
  );
};
```

Auto-dismiss using timeout.
## Seasonal Theme Architecture

### Principle

Separate:

```ts
BaseTheme
DynamicTheme
```

### Theme Types

```ts
export type DynamicTheme = {
  headerGradient: string[];
  sectionColors: string[];
  accent: string;
};
```

### Seasonal Theme Map (App-Controlled)

```ts
const seasonalThemes: Record<string, DynamicTheme> = {
  default: {
    headerGradient: ['#E3F2FD', '#BBDEFB'],
    sectionColors: ['#E3F2FD'],
    accent: '#1976D2',
  },
  holi: {
    headerGradient: ['#FFD1DC', '#C8E6C9'],
    sectionColors: ['#FFD1DC', '#FFF59D', '#C8E6C9'],
    accent: '#FF4081',
  },
};
```

## Backend Remote Config Flow

### Backend Response

```json
{
  "season": "holi",
  "assets": {
    "heroBanner": "https://cdn.example.com/holi.png"
  }
}
```
### Fetch On App Start

```tsx
const { data } = useQuery({
  queryKey: ['season-config'],
  queryFn: fetchSeasonConfig,
});
```
### Update Theme Store

```tsx
useEffect(() => {
  if (data?.season) {
    setSeason(data.season);
  }
}, [data]);
```
## AppInitializer Pattern

```tsx
export const AppInitializer = ({ children }) => {
  const { isLoading } = useSeasonConfig();

  if (isLoading) return null;

  return children;
};
```

Usage:

```tsx
<AppInitializer>
  <Navigation />
</AppInitializer>
```

## Remote Config vs OTA

**Do NOT use OTA for:**
- Color swaps
- Gradients
- Seasonal banners
- Accent changes

**Use OTA for:**
- Logic updates    
- Layout updates
- New features

**Use Backend for:**
- Season key
- Promotional banners
- Feature flags

References:
1. https://medium.com/timeless/building-the-animated-sticky-spotify-collapsible-header-with-react-native-and-reanimated-part-i-e47222dfcb85
2. https://medium.com/timeless/building-the-animated-sticky-spotify-collapsible-header-with-react-native-and-reanimated-part-ii-ec3bff94a58a