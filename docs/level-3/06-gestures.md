---
description: "Recognise gestures natively with React Native Gesture Handler: pan, tap, long press, composed gestures and a Reanimated swipe-to-complete row."
---

# 06 · Gestures with React Native Gesture Handler

`Pressable` (Level 1) is fine for taps. Swipe-to-delete, drag-to-reorder, pinch-to-zoom, a bottom
sheet you can fling, a card you can throw — these need **continuous** gestures that track the finger
every frame and coordinate with scrolling. React Native's built-in responder system makes those
decisions in JavaScript, so a busy JS thread makes gestures feel sticky. **React Native Gesture
Handler** (RNGH) recognises gestures **natively**, on the UI thread, and pairs with Reanimated so
the whole gesture-to-animation loop never touches the JS thread.

```bash
npx expo install react-native-gesture-handler
```

Wrap your app once, as close to the root as possible:

```tsx title="app/_layout.tsx"
import { GestureHandlerRootView } from 'react-native-gesture-handler';
import { Stack } from 'expo-router';

export default function RootLayout() {
  return (
    <GestureHandlerRootView style={{ flex: 1 }}>
      <Stack />
    </GestureHandlerRootView>
  );
}
```

!!! note "Versions"
    When this lesson was checked, npm's latest RNGH release was a 3.x version with a reworked API,
    while `npx expo install` in an SDK 57 project installed **2.32**. This lesson uses the 2.x
    `Gesture` builder API that SDK installs — always let `npx expo install` choose the version, and
    check the RNGH docs for your installed major version.

## The Gesture builder API

You describe a gesture with a builder and attach it with `GestureDetector`:

```tsx
import { Gesture, GestureDetector } from 'react-native-gesture-handler';
import Animated, { useAnimatedStyle, useSharedValue, withSpring } from 'react-native-reanimated';

function DraggableDot() {
  const x = useSharedValue(0);
  const y = useSharedValue(0);

  const pan = Gesture.Pan()
    .onChange((e) => {        // runs on the UI thread as a worklet
      x.value += e.changeX;
      y.value += e.changeY;
    })
    .onEnd(() => {
      x.value = withSpring(0);
      y.value = withSpring(0);
    });

  const style = useAnimatedStyle(() => ({ transform: [{ translateX: x.value }, { translateY: y.value }] }));

  return (
    <GestureDetector gesture={pan}>
      <Animated.View style={[{ width: 64, height: 64, borderRadius: 32, backgroundColor: '#4f46e5' }, style]} />
    </GestureDetector>
  );
}
```

Gesture callbacks are **worklets** when Reanimated is installed: they run on the UI thread and can
write shared values directly. Drag the dot while JS is blocked and it still follows your finger.

Available gestures include `Tap` (with `numberOfTaps` for double-tap), `LongPress`, `Pan`, `Pinch`,
`Rotation`, `Fling` and `Native` (to wrap a native scroll view). Each has configuration methods such
as `.minDistance(10)`, `.activeOffsetX([-10, 10])` and `.failOffsetY([-8, 8])`.

## Composing gestures

Real UIs combine gestures:

```ts
const doubleTap = Gesture.Tap().numberOfTaps(2).onEnd(() => { scale.value = withSpring(scale.value > 1 ? 1 : 2); });
const singleTap = Gesture.Tap().onEnd(() => { /* toggle controls */ });

// A single tap must wait to see if it's actually a double tap.
const taps = Gesture.Exclusive(doubleTap, singleTap);

// Pinch and rotate at the same time.
const transform = Gesture.Simultaneous(Gesture.Pinch(), Gesture.Rotation());

// Long press, then drag (drag-to-reorder).
const reorder = Gesture.Pan().activateAfterLongPress(300);
```

- `Gesture.Exclusive(a, b)` — only one can activate; `a` has priority.
- `Gesture.Simultaneous(a, b)` — both can be active together.
- `Gesture.Race(a, b)` — the first to activate cancels the other.

## Living inside scroll views

The hardest part of gestures is coexisting with scrolling. A horizontal swipe on a row inside a
vertical list must not steal vertical scrolls, and vice versa. The tool is **activation offsets**:

```ts
const swipe = Gesture.Pan()
  .activeOffsetX([-12, 12])  // activate only after 12 pt of horizontal movement
  .failOffsetY([-8, 8]);     // give up if the finger moves 8 pt vertically first
```

Now a mostly-vertical drag fails the swipe and the list scrolls; a mostly-horizontal drag activates
the swipe and the list stays put.

## Worked example: swipe a habit to complete it

```tsx title="src/SwipeToComplete.tsx"
import { ReactNode } from 'react';
import { StyleSheet, Text, View, useWindowDimensions } from 'react-native';
import { Gesture, GestureDetector } from 'react-native-gesture-handler';
import Animated, { interpolate, Extrapolation, useAnimatedStyle, useSharedValue, withSpring, withTiming } from 'react-native-reanimated';
import { scheduleOnRN } from 'react-native-worklets';
import * as Haptics from 'expo-haptics';

type Props = { children: ReactNode; onComplete: () => void; label?: string };

export function SwipeToComplete({ children, onComplete, label = 'Done' }: Props) {
  const { width } = useWindowDimensions();
  const threshold = width * 0.35;
  const x = useSharedValue(0);
  const passed = useSharedValue(false);

  const pan = Gesture.Pan()
    .activeOffsetX(12)          // only rightward swipes activate
    .failOffsetY([-8, 8])
    .onChange((e) => {
      x.value = Math.max(0, x.value + e.changeX);
      const nowPassed = x.value > threshold;
      if (nowPassed !== passed.value) {
        passed.value = nowPassed;
        scheduleOnRN(Haptics.selectionAsync);   // tick when crossing the threshold
      }
    })
    .onEnd(() => {
      if (x.value > threshold) {
        x.value = withTiming(width, { duration: 180 }, (finished) => {
          if (finished) {
            scheduleOnRN(onComplete);
            x.value = 0;          // reset for reuse (the parent usually re-renders the row as done)
            passed.value = false;
          }
        });
      } else {
        x.value = withSpring(0, { damping: 18 });
        passed.value = false;
      }
    });

  const rowStyle = useAnimatedStyle(() => ({ transform: [{ translateX: x.value }] }));
  const bgStyle = useAnimatedStyle(() => ({
    opacity: interpolate(x.value, [0, threshold], [0.3, 1], Extrapolation.CLAMP),
  }));

  return (
    <View style={styles.container} accessibilityActions={[{ name: 'activate', label }]} onAccessibilityAction={onComplete}>
      <Animated.View style={[styles.background, bgStyle]}>
        <Text style={styles.backgroundText}>✓ {label}</Text>
      </Animated.View>
      <GestureDetector gesture={pan}>
        <Animated.View style={rowStyle}>{children}</Animated.View>
      </GestureDetector>
    </View>
  );
}

const styles = StyleSheet.create({
  container: { borderRadius: 14, overflow: 'hidden' },
  background: { ...StyleSheet.absoluteFill, backgroundColor: '#16a34a', justifyContent: 'center', paddingLeft: 20 },
  backgroundText: { color: 'white', fontWeight: '800', fontSize: 16 },
});
```

Two details make it production-quality. The haptic tick fires **once** when crossing the
threshold in either direction (tracked in a shared value so it lives on the UI thread). And the
`accessibilityActions` give screen-reader users an equivalent — a swipe gesture alone is invisible to
VoiceOver and TalkBack users (lesson 7). Use it inside a `FlatList` row: vertical scrolling still
works because of `failOffsetY`.

## How It Actually Works

RNGH installs native gesture recognizers — `UIGestureRecognizer` subclasses on iOS and its own
handler orchestrator on Android — on the native views wrapped by `GestureDetector`. Touches are
processed by those recognizers on the UI thread: each handler moves through a state machine
(`UNDETERMINED → BEGAN → ACTIVE → END`, or `FAILED`/`CANCELLED`), and activation criteria such as
`activeOffsetX` and `failOffsetY` decide transitions. Relations like `Exclusive` and `Simultaneous`
tell the orchestrator which handlers may be active together and which must wait for others to fail.

When a handler emits an event and Reanimated is present, RNGH calls your callback **as a worklet on
the UI runtime** — the same runtime that runs `useAnimatedStyle`. So the path from finger movement to
`x.value` to a new transform on the native view is: UI thread → UI thread → UI thread, every frame. The
JS thread is only involved when you explicitly `scheduleOnRN` something.

`GestureHandlerRootView` is required because the Android implementation needs a root view that
intercepts touch events and routes them through RNGH's orchestrator before React Native's own
responder system sees them.

## Common mistakes

- **Missing `GestureHandlerRootView`** — gestures silently don't work (especially on Android) or
  don't work inside modals. A `Modal` is a new native window; wrap its content in its own root view.
- **Calling JS functions directly in callbacks** (`onComplete()` instead of `scheduleOnRN(onComplete)`).
- **No activation offsets in scrollable lists** — rows hijack vertical scrolling.
- **Gesture-only features** with no accessible alternative.
- **Creating gestures conditionally** in ways that change their structure between renders; build them
  consistently (memoise with `useMemo` if they depend on props).

## Exercise

1. Wrap each habit row in `SwipeToComplete` and confirm vertical scrolling still works smoothly.
2. Add a **left** swipe that reveals a red "Archive" action. Should both directions share one `Pan`
   gesture or be two gestures? Try both and explain the trade-off.
3. Build an image viewer that supports pinch-to-zoom, pan while zoomed, and double-tap to toggle
   between 1× and 2×, using `Simultaneous` and `Exclusive`.
4. Test the swipe with VoiceOver or TalkBack on, using the custom action.
