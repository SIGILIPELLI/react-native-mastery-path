---
description: "Make React Native apps work with VoiceOver and TalkBack: roles, labels, states, grouping, custom actions, large text, contrast and motion."
---

# 07 · Accessibility in React Native

A meaningful share of your users rely on a screen reader, larger text, higher contrast, reduced
motion, switch control or voice control. On mobile, accessibility is built into the OS:
**VoiceOver** on iOS and **TalkBack** on Android read the screen aloud and let users navigate by
swiping between elements. React Native maps its accessibility props to those native systems, so
doing it right is mostly about giving each element the right information.

General principles of accessible design are covered in the
[HTML & CSS Mastery Path](https://sigilipelli.github.io/html-css-mastery-path/) and the React course;
this lesson is about the React Native specifics.

## How a screen reader sees your app

A screen reader moves focus element by element. For each focused element it announces, roughly:
**label**, **role**, **state**, then optional **hint**. For example: "Drink water, checkbox,
checked. Double tap to toggle." Your job is to make sure every interactive element has all four,
and that decorative or redundant elements don't add noise.

## The essential props

```tsx
<Pressable
  accessible                                   // treat as one focusable element (default for Pressable)
  accessibilityRole="checkbox"                 // what it is
  accessibilityLabel="Drink water"             // what it's called
  accessibilityState={{ checked: done }}       // its current state
  accessibilityHint="Toggles today's check-in" // what happens on activation (optional)
  onPress={toggle}
>
```

| Prop | Purpose | Notes |
|------|---------|-------|
| `accessibilityRole` | `button`, `link`, `checkbox`, `switch`, `header`, `image`, `tab`, `search`, `adjustable`, `progressbar`… | Changes how it's announced and which gestures apply |
| `accessibilityLabel` | Spoken name | Defaults to the text content; set it for icon-only controls |
| `accessibilityState` | `{ disabled, selected, checked, busy, expanded }` | Keep in sync with visual state |
| `accessibilityValue` | `{ min, max, now }` or `{ text }` | For sliders, progress, steppers |
| `accessibilityHint` | Outcome of the action | Short; users can turn hints off |
| `accessibilityLiveRegion` (Android) | `polite` / `assertive` | Announce changes to this element |
| `accessibilityElementsHidden` (iOS) / `importantForAccessibility="no-hide-descendants"` (Android) | Hide from screen readers | For decorative content |

Newer React Native versions also accept web-style ARIA props (`role`, `aria-label`,
`aria-checked`, `aria-hidden`, …) as aliases. Pick one style per codebase.

## Grouping: one card, one announcement

By default, a card with an image, a title, a subtitle and a badge can be four separate stops. Group
it so it's announced once:

```tsx
<View accessible accessibilityLabel={`${name}, ${streak} day streak, ${doneToday ? 'done today' : 'not done today'}`}>
  <Image source={icon} accessibilityIgnoresInvertColors />
  <Text>{name}</Text>
  <Text>{streak}-day streak</Text>
  {doneToday && <Text>✓</Text>}
</View>
```

`accessible` on a container makes it a single element; its children are no longer separately
focusable. Write the combined label in a natural spoken order — not "check mark".

## Custom actions instead of gesture-only features

Swipe-to-delete, long-press menus and drag-to-reorder are invisible to screen-reader users unless
you expose them as **accessibility actions**:

```tsx
<View
  accessible
  accessibilityLabel={habit.name}
  accessibilityActions={[
    { name: 'activate', label: 'Toggle done' },
    { name: 'delete', label: 'Delete habit' },
    { name: 'increment', label: 'Move down' },
    { name: 'decrement', label: 'Move up' },
  ]}
  onAccessibilityAction={(e) => {
    switch (e.nativeEvent.actionName) {
      case 'activate': toggle(habit.id); break;
      case 'delete': remove(habit.id); break;
      case 'increment': move(habit.id, +1); break;
      case 'decrement': move(habit.id, -1); break;
    }
  }}
>
```

VoiceOver users reach custom actions with a vertical swipe (the "Actions" rotor); TalkBack users
through its actions menu. `activate` maps to double-tap, and `increment`/`decrement` map to the
adjustable gestures.

## Announcements and focus

```ts
import { AccessibilityInfo, findNodeHandle } from 'react-native';

AccessibilityInfo.announceForAccessibility('Habit saved'); // speak a message without moving focus

// Move screen-reader focus, e.g. to an error after a failed submit:
const node = findNodeHandle(errorRef.current);
if (node) AccessibilityInfo.setAccessibilityFocus(node);
```

Announce results of actions that don't otherwise change what's focused ("Saved", "3 results").
Don't over-announce.

## Text size, contrast and motion

- **Dynamic type:** users can enlarge text a lot. `Text` scales with the system setting by default.
  Don't disable it (`allowFontScaling={false}`) — instead, cap extreme growth where layout truly
  breaks with `maxFontSizeMultiplier={1.6}`, and design layouts that wrap (lesson L1-08's
  `fontScale` check).
- **Touch targets:** at least ~44×44 pt (iOS) / 48×48 dp (Android), via padding or `hitSlop`.
- **Contrast:** body text needs a contrast ratio of at least 4.5:1 against its background (WCAG AA).
  Light grey on white is the most common failure.
- **Don't use colour alone** to convey state: add an icon or text ("✓ Done").
- **Reduced motion and other settings** are readable at runtime:

```ts
const reduceMotion = await AccessibilityInfo.isReduceMotionEnabled();
const screenReader = await AccessibilityInfo.isScreenReaderEnabled();
const sub = AccessibilityInfo.addEventListener('screenReaderChanged', (on) => setScreenReader(on));
```

## Worked example: an accessible stepper

```tsx title="src/Stepper.tsx"
import { Pressable, StyleSheet, Text, View } from 'react-native';

type Props = { label: string; value: number; min: number; max: number; onChange: (v: number) => void };

export function Stepper({ label, value, min, max, onChange }: Props) {
  const dec = () => value > min && onChange(value - 1);
  const inc = () => value < max && onChange(value + 1);

  return (
    <View
      accessible
      accessibilityRole="adjustable"
      accessibilityLabel={label}
      accessibilityValue={{ min, max, now: value, text: `${value} per week` }}
      accessibilityActions={[{ name: 'increment' }, { name: 'decrement' }]}
      onAccessibilityAction={(e) => (e.nativeEvent.actionName === 'increment' ? inc() : dec())}
      style={styles.row}
    >
      <Text style={styles.label}>{label}</Text>
      <View style={styles.controls}>
        <Pressable onPress={dec} disabled={value <= min} hitSlop={8} style={styles.button} importantForAccessibility="no">
          <Text style={styles.buttonText}>−</Text>
        </Pressable>
        <Text style={styles.value}>{value}</Text>
        <Pressable onPress={inc} disabled={value >= max} hitSlop={8} style={styles.button} importantForAccessibility="no">
          <Text style={styles.buttonText}>+</Text>
        </Pressable>
      </View>
    </View>
  );
}

const styles = StyleSheet.create({
  row: { flexDirection: 'row', alignItems: 'center', justifyContent: 'space-between', paddingVertical: 8, minHeight: 48 },
  label: { fontSize: 16, flexShrink: 1 },
  controls: { flexDirection: 'row', alignItems: 'center', gap: 12 },
  button: { width: 44, height: 44, borderRadius: 22, backgroundColor: '#eef2ff', alignItems: 'center', justifyContent: 'center' },
  buttonText: { fontSize: 22, color: '#3730a3', fontWeight: '700' },
  value: { minWidth: 24, textAlign: 'center', fontSize: 18, fontVariant: ['tabular-nums'] },
});
```

To a sighted user it's two buttons and a number. To a screen-reader user it's **one** adjustable
element — "Times per week, 3 per week, adjustable" — that they change by swiping up or down, which is
exactly how native steppers behave.

## Testing it

- **Turn on VoiceOver** (Settings → Accessibility → VoiceOver, or ask Siri) and **TalkBack**
  (Settings → Accessibility → TalkBack) and navigate your app with your eyes closed. It's the single
  most effective test.
- **Accessibility Inspector** (Xcode → Open Developer Tool) shows labels, roles and traits on the iOS
  Simulator; Android's **Accessibility Scanner** app flags small targets and low contrast.
- In tests, React Native Testing Library queries by role and label (`getByRole('checkbox', { name:
  'Drink water' })`) — Level 4 uses this, and tests written that way double as accessibility checks.

## How It Actually Works

React Native's accessibility props are passed to the native views Fabric creates. On iOS they set
`UIAccessibility` properties — `isAccessibilityElement`, `accessibilityLabel`,
`accessibilityTraits` (from role and state), `accessibilityValue` and custom actions — which VoiceOver
reads from the native view hierarchy. On Android they populate an `AccessibilityNodeInfo` for the
view (class name from the role, content description from the label, checkable/checked from state,
custom actions), which TalkBack consumes. `accessible={true}` on a container makes the native view an
accessibility element and hides its children from the accessibility tree, which is why grouping
works. Because all of this is native, screen readers behave exactly as they do in fully native apps —
provided you supply the information.

## Common mistakes

- **Icon-only buttons with no label** — announced as "button" or a file name.
- **Roles missing on custom controls** — a `Pressable` that looks like a switch but isn't announced as one.
- **`accessibilityState` not updated** with visual state.
- **Every piece of a card focusable separately.**
- **Gesture-only actions** with no accessibility action.
- **`allowFontScaling={false}`** to "fix" layouts.
- **Never actually using VoiceOver or TalkBack.**

## Exercise

1. Turn on VoiceOver or TalkBack and complete three tasks in your habit app: add a habit, mark it
   done, delete it. Write down every point where you got stuck or heard something confusing.
2. Fix each issue: group habit rows, add roles and states, add accessibility actions for delete and
   reorder, and announce "Habit added".
3. Set your system text size to the largest setting and fix any layout that clips or overlaps.
4. Replace the "times per week" text field from Level 2 with the `Stepper` and verify it adjusts with
   swipe up/down under a screen reader.
