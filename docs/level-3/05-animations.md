---
description: "Animate in React Native: Animated with the native driver, then Reanimated shared values, UI-thread worklets, springs and layout animations."
---

# 05 · Animations with Animated & Reanimated

Animation on mobile isn't decoration. It explains what just happened (a card flies into the
"done" list), confirms input (a button squashes under your thumb) and hides latency (a skeleton
shimmers while data loads). It also has to run at the display's frame rate, which — as lesson 1
showed — is hard if every frame waits for the JS thread. React Native gives you two tools: the
built-in **Animated** API, and **Reanimated**, the library most production apps use.

## The built-in Animated API, briefly

```tsx
import { useRef } from 'react';
import { Animated, Pressable, Text } from 'react-native';

export function PulseButton({ label, onPress }: { label: string; onPress: () => void }) {
  const scale = useRef(new Animated.Value(1)).current;
  const to = (v: number) => Animated.spring(scale, { toValue: v, useNativeDriver: true, speed: 40, bounciness: 6 }).start();

  return (
    <Pressable onPressIn={() => to(0.94)} onPressOut={() => to(1)} onPress={onPress}>
      <Animated.View style={{ transform: [{ scale }], backgroundColor: '#4f46e5', padding: 16, borderRadius: 12 }}>
        <Text style={{ color: 'white', fontWeight: '700' }}>{label}</Text>
      </Animated.View>
    </Pressable>
  );
}
```

`useNativeDriver: true` sends the whole animation description to native code once, and the UI
thread runs it without JavaScript. The limitation: the native driver only supports
**non-layout properties** — `transform` and `opacity` (plus a few others). Animating `width`,
`height` or `top` requires the JS driver, which runs every frame on the JS thread. And any logic
that must run *during* the animation (follow a finger, react to a scroll position with arbitrary
maths) can't be expressed declaratively.

## Reanimated: animations as code on the UI thread

Reanimated lets you write animation logic in JavaScript that **runs on the UI thread**. The core
ideas:

- **Shared values** (`useSharedValue`) hold animatable state that both threads can read and write.
- **Worklets** are small JavaScript functions that Reanimated copies to a separate JavaScript runtime
  on the UI thread. `useAnimatedStyle`'s callback is a worklet.
- **Animation functions** (`withTiming`, `withSpring`, `withDecay`, `withSequence`, `withRepeat`,
  `withDelay`) describe how a shared value moves to a new target.

```bash
npx expo install react-native-reanimated react-native-worklets
```

In Expo projects, `babel-preset-expo` configures the worklets Babel plugin for you. (Outside Expo you
add `react-native-worklets/plugin` to `babel.config.js`.) Reanimated 4 — the version paired with
recent Expo SDKs (SDK 57 installed Reanimated 4.5 in this course's check project) — requires the New
Architecture, which is the default anyway.

### A first Reanimated animation

```tsx title="src/ProgressBar.tsx"
import { useEffect } from 'react';
import { StyleSheet, View } from 'react-native';
import Animated, { useAnimatedStyle, useSharedValue, withSpring } from 'react-native-reanimated';

export function ProgressBar({ progress }: { progress: number }) {  // 0..1
  const p = useSharedValue(0);

  useEffect(() => {
    p.value = withSpring(Math.min(Math.max(progress, 0), 1), { damping: 18, stiffness: 140 });
  }, [progress, p]);

  const fill = useAnimatedStyle(() => ({
    transform: [{ scaleX: p.value }],
  }));

  return (
    <View style={styles.track} accessibilityRole="progressbar" accessibilityValue={{ min: 0, max: 100, now: Math.round(progress * 100) }}>
      <Animated.View style={[styles.fill, fill]} />
    </View>
  );
}

const styles = StyleSheet.create({
  track: { height: 10, borderRadius: 5, backgroundColor: '#e2e8f0', overflow: 'hidden' },
  fill: { ...StyleSheet.absoluteFill, backgroundColor: '#22c55e', transformOrigin: 'left' },
});
```

Setting `p.value = withSpring(…)` starts a spring **on the UI thread**. Every frame, the
`useAnimatedStyle` worklet re-runs there and updates the native view directly — React doesn't
re-render at all. Animating `scaleX` with `transformOrigin: 'left'` instead of `width` keeps it to a
transform, which is cheaper than relayout (Reanimated *can* animate `width`, but transforms avoid
running Yoga every frame).

### Deriving values with interpolate

```tsx
import { interpolate, interpolateColor, Extrapolation, useAnimatedStyle } from 'react-native-reanimated';

const style = useAnimatedStyle(() => ({
  opacity: interpolate(p.value, [0, 0.2, 1], [0.4, 1, 1], Extrapolation.CLAMP),
  backgroundColor: interpolateColor(p.value, [0, 1], ['#f59e0b', '#22c55e']),
}));
```

`interpolate` maps an input range to an output range — the workhorse of every scroll-driven header,
parallax image and gesture-driven card.

## Layout animations: entering, exiting and moving

The fastest win in Reanimated: animate items as they mount, unmount, or move because of a layout
change — with one prop each.

```tsx
import Animated, { FadeInDown, FadeOut, LinearTransition } from 'react-native-reanimated';

{habits.map((h) => (
  <Animated.View
    key={h.id}
    entering={FadeInDown.springify().damping(16)}
    exiting={FadeOut.duration(180)}
    layout={LinearTransition}
  >
    <HabitRow habit={h} />
  </Animated.View>
))}
```

Delete a habit and it fades out while the rows below slide up into the gap. For `FlatList`, use
`Animated.FlatList` with `itemLayoutAnimation={LinearTransition}`.

## Respecting reduced motion

Some users turn on **Reduce Motion** because large movements make them unwell. Respect it:

```tsx
import { ReduceMotion, useReducedMotion, withTiming } from 'react-native-reanimated';

const reduce = useReducedMotion();
p.value = reduce ? target : withSpring(target);

// Or per animation: follow the system setting automatically
p.value = withTiming(target, { duration: 300, reduceMotion: ReduceMotion.System });
```

Prefer cross-fades over big slides when reduced motion is on, rather than removing feedback entirely.

## Worked example: a "habit completed" celebration

```tsx title="src/CompleteButton.tsx"
import { Pressable, StyleSheet, Text } from 'react-native';
import Animated, {
  useAnimatedStyle, useSharedValue, withSequence, withSpring, withTiming, interpolateColor, useReducedMotion,
} from 'react-native-reanimated';
import { scheduleOnRN } from 'react-native-worklets';
import * as Haptics from 'expo-haptics';

export function CompleteButton({ done, onToggle }: { done: boolean; onToggle: () => void }) {
  const scale = useSharedValue(1);
  const progress = useSharedValue(done ? 1 : 0);
  const reduce = useReducedMotion();

  function press() {
    const next = !done;
    progress.value = withTiming(next ? 1 : 0, { duration: 220 });
    if (next && !reduce) {
      scale.value = withSequence(
        withTiming(0.85, { duration: 90 }),
        withSpring(1.12, { damping: 6, stiffness: 300 }),
        withSpring(1, { damping: 12 }, (finished) => {
          // This callback runs on the UI thread; hop back to JS for non-worklet APIs.
          if (finished) scheduleOnRN(Haptics.notificationAsync, Haptics.NotificationFeedbackType.Success);
        }),
      );
    }
    onToggle();
  }

  const circle = useAnimatedStyle(() => ({
    transform: [{ scale: scale.value }],
    backgroundColor: interpolateColor(progress.value, [0, 1], ['#ffffff', '#22c55e']),
    borderColor: interpolateColor(progress.value, [0, 1], ['#cbd5e1', '#22c55e']),
  }));
  const tick = useAnimatedStyle(() => ({ opacity: progress.value, transform: [{ scale: 0.6 + progress.value * 0.4 }] }));

  return (
    <Pressable onPress={press} hitSlop={8} accessibilityRole="checkbox" accessibilityState={{ checked: done }}>
      <Animated.View style={[styles.circle, circle]}>
        <Animated.Text style={[styles.tick, tick]}>✓</Animated.Text>
      </Animated.View>
    </Pressable>
  );
}

const styles = StyleSheet.create({
  circle: { width: 36, height: 36, borderRadius: 18, borderWidth: 2, alignItems: 'center', justifyContent: 'center' },
  tick: { color: 'white', fontSize: 18, fontWeight: '800' },
});
```

`scheduleOnRN` (from `react-native-worklets`) schedules a call on the JS thread from a worklet. Older
Reanimated code uses `runOnJS(fn)(args)` for the same purpose; in the Reanimated 4 / worklets
versions checked for this lesson, `scheduleOnRN` is the API exported for it.

## How It Actually Works

At build time, the worklets Babel plugin finds functions that must run on the UI thread — callbacks
of `useAnimatedStyle`, functions marked with a `'worklet'` directive, gesture callbacks — and
records their source and the variables they capture (their **closure**). At runtime, Reanimated
creates a second JavaScript runtime that lives on the UI thread, and sends it a serialised copy of
each worklet and its captured values.

A **shared value** is a small object living in C++ that both runtimes can access via JSI. When you set
`p.value = withSpring(1)` on the JS thread, the animation object is handed to the UI runtime, which
advances it on every display frame (driven by the platform's frame callback, `CADisplayLink` on iOS
or `Choreographer` on Android). After each step, Reanimated re-runs the affected `useAnimatedStyle`
worklets on the UI thread and applies the resulting style directly to the native view, bypassing
React's render and Fabric's commit for those props. That's why the animation stays smooth while your
JS thread is busy — try it with the "block JS" button from lesson 1.

## Common mistakes

- **Reading `.value` during React render** to display text — React won't re-render when it changes.
  Use an animated style/prop, or `useDerivedValue` + a component that reads it on the UI thread.
- **Calling ordinary JS functions inside worklets** (navigation, state setters, haptics) — use
  `scheduleOnRN` to hop to the JS thread.
- **Animating layout properties when a transform would do.**
- **Capturing large objects in a worklet's closure** — they're copied to the UI runtime.
- **Ignoring Reduce Motion.**
- **Using the JS-driven `Animated` API for continuous, gesture-linked motion.**

## Exercise

1. Replace the habit tracker's check circle with `CompleteButton`.
2. Wrap the habit list rows in `Animated.View` with `entering`, `exiting` and `layout` so adding and
   deleting habits animates. Turn on Reduce Motion in your phone's accessibility settings and adjust
   so the animations become simple fades.
3. Build a collapsing header: an `Animated.ScrollView` with `useAnimatedScrollHandler` that shrinks a
   large title from 34 to 17 points over the first 80 points of scroll using `interpolate`.
4. Press "block JS for 3 seconds" (lesson 1) while your header animates — does it keep moving? Why?
