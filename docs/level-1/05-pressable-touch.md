---
description: "Handle touches with Pressable: pressed styles, Android ripple, hitSlop, long press, disabled states, and how a tap is told apart from a scroll."
---

# 05 · Touch & Feedback with Pressable

On the web, a `click` handler on any element is enough. On a phone, touch is richer and less
forgiving: fingers are imprecise, users expect instant visual feedback, a tap inside a scrolling list
has to be distinguished from the start of a scroll, and Android users expect a ripple. React
Native's answer is one component, **`Pressable`**, and it replaces the older `TouchableOpacity` /
`TouchableHighlight` family you'll still see in older code.

## The essentials

```tsx
import { Pressable, Text, StyleSheet } from 'react-native';

<Pressable
  onPress={() => console.log('pressed')}
  style={({ pressed }) => [styles.button, pressed && styles.pressed]}
  accessibilityRole="button"
  accessibilityLabel="Mark habit as done"
>
  {({ pressed }) => <Text style={styles.label}>{pressed ? 'Release to save' : 'Done'}</Text>}
</Pressable>
```

Both `style` and `children` can be **functions** that receive `{ pressed }`. That's how you build a
press state without managing it in `useState` yourself.

The event props, in the order they fire for a simple tap:

1. `onPressIn` — finger down (fires immediately; good for starting feedback).
2. `onPressOut` — finger up or the touch was cancelled.
3. `onPress` — fires after `onPressOut`, only if the gesture counted as a press.

For a press-and-hold, `onLongPress` fires after `delayLongPress` (500 ms by default) and then
`onPress` does **not** fire.

## Make targets big enough: `hitSlop` and padding

Platform guidelines recommend touch targets of at least about **44×44 points** (Apple) and
**48×48 dp** (Android Material). A 20-point icon is too small to tap reliably. Either pad it, or
extend the touchable area invisibly with `hitSlop`:

```tsx
<Pressable onPress={onDelete} hitSlop={12} accessibilityRole="button" accessibilityLabel="Delete">
  <TrashIcon size={20} />
</Pressable>
```

`hitSlop={12}` extends the touch area 12 points beyond each edge without changing layout.
`pressRetentionOffset` controls how far a finger can drift outside the element during a press
before the press is cancelled.

## Android ripple

Android users expect the Material ripple. `Pressable` draws it natively:

```tsx
<Pressable
  android_ripple={{ color: 'rgba(79,70,229,0.18)', borderless: false }}
  style={styles.row}
  onPress={open}
>
  <Text>Settings</Text>
</Pressable>
```

On iOS the prop is ignored, so combine it with a `pressed` style that only applies on iOS (lesson 8
covers `Platform.OS`) or simply use a subtle opacity change on both.

## Disabled state

```tsx
<Pressable
  disabled={saving}
  onPress={save}
  accessibilityRole="button"
  accessibilityState={{ disabled: saving, busy: saving }}
  style={({ pressed }) => [styles.button, (pressed || saving) && { opacity: 0.6 }]}
>
  <Text style={styles.label}>{saving ? 'Saving…' : 'Save'}</Text>
</Pressable>
```

`disabled` stops press events. `accessibilityState` tells screen readers the button is disabled —
the two are separate, so set both.

## Worked example: a habit check button with haptic-style feedback

```tsx title="HabitCheck.tsx"
import { useState } from 'react';
import { Pressable, StyleSheet, Text, View } from 'react-native';

type Props = { name: string; initiallyDone?: boolean; onToggle?: (done: boolean) => void; onEdit?: () => void };

export function HabitCheck({ name, initiallyDone = false, onToggle, onEdit }: Props) {
  const [done, setDone] = useState(initiallyDone);

  function toggle() {
    const next = !done;
    setDone(next);
    onToggle?.(next);
  }

  return (
    <Pressable
      onPress={toggle}
      onLongPress={onEdit}
      delayLongPress={400}
      android_ripple={{ color: 'rgba(15,23,42,0.08)' }}
      accessibilityRole="checkbox"
      accessibilityState={{ checked: done }}
      accessibilityHint="Long press to edit"
      style={({ pressed }) => [styles.row, pressed && styles.rowPressed]}
    >
      {({ pressed }) => (
        <>
          <Text style={[styles.name, done && styles.nameDone]} numberOfLines={1}>{name}</Text>
          <View style={[styles.circle, done && styles.circleDone, pressed && styles.circlePressed]}>
            {done && <Text style={styles.tick}>✓</Text>}
          </View>
        </>
      )}
    </Pressable>
  );
}

const styles = StyleSheet.create({
  row: { flexDirection: 'row', alignItems: 'center', gap: 12, padding: 16, backgroundColor: 'white', borderRadius: 12, overflow: 'hidden' },
  rowPressed: { backgroundColor: '#f1f5f9' },
  name: { flex: 1, fontSize: 16 },
  nameDone: { color: '#94a3b8', textDecorationLine: 'line-through' },
  circle: { width: 28, height: 28, borderRadius: 14, borderWidth: 2, borderColor: '#cbd5e1', alignItems: 'center', justifyContent: 'center' },
  circleDone: { backgroundColor: '#22c55e', borderColor: '#22c55e' },
  circlePressed: { transform: [{ scale: 0.9 }] },
  tick: { color: 'white', fontWeight: '800' },
});
```

Notice `accessibilityRole="checkbox"` with `accessibilityState={{ checked }}`: a screen reader
announces "Drink water, checkbox, checked" instead of just "Drink water". `overflow: 'hidden'` keeps
the Android ripple inside the rounded corners. (Real device vibration comes in Level 3 with
`expo-haptics`.)

## How It Actually Works

Touches start natively. The OS delivers touch events to React Native's root view on the UI thread,
which forwards them to JavaScript. JavaScript decides which component owns the touch using the
**gesture responder system**: when a touch starts, React Native asks components (deepest first, via
`onStartShouldSetResponder`, and allows ancestors to capture) whether they want to become the
**responder**. `Pressable` says yes.

`Pressable` is built on an internal state machine called **Pressability**. It tracks states like
"not responder", "responder inactive press in", "responder active press in", "responder active
long press in". A small press delay before the press is considered active, plus the press-retention
region, lets it tell a tap from the start of a scroll: if the finger moves enough for the parent
`ScrollView` to claim the gesture, the native scroll view requests **termination** of the JS
responder; `Pressable` receives `onResponderTerminate`, fires `onPressOut`, and never fires
`onPress`. That's why tapping a row that you then drag doesn't trigger it.

Because the decision is made in JavaScript, a busy JS thread delays press feedback. That's one
reason `android_ripple` is valuable: the ripple is drawn by Android's native view on the UI thread.
For gestures that must stay responsive under JS load (drags, swipes), Level 3 uses React Native
Gesture Handler, which recognises gestures natively.

## Common mistakes

- **Using `onPress` on a `View` or `Text` container to make whole rows tappable.** `View` has no
  `onPress`. (`Text` does, but it lacks press states and ripple.) Wrap in `Pressable`.
- **No visual feedback.** A button that doesn't change on touch feels broken on mobile.
- **Tiny targets** — icons without padding or `hitSlop`.
- **Nesting `Pressable` inside `Pressable`** without thinking: the innermost responder wins, so the
  outer row's `onPress` won't fire when the inner button is tapped. That's usually what you want —
  but test it.
- **Setting `disabled` but not `accessibilityState.disabled`**, so screen readers announce an
  active button that does nothing.

## Exercise

Build a "segmented control" with three options (Daily, Weekly, Monthly) using three `Pressable`s
in a row. Requirements: the selected option has a filled background; pressed options show a
lighter background; each has `accessibilityRole="tab"` and `accessibilityState={{ selected }}`;
each target is at least 44 points tall; long-pressing an option shows an `Alert` describing it
(`import { Alert } from 'react-native'`). Then put the control inside a `ScrollView`, tap and drag
on an option, and confirm it does not change selection.
