---
description: "Keep inputs visible above the keyboard with KeyboardAvoidingView and scroll insets, and build modals and bottom sheets that feel native."
---

# 09 · The Keyboard, Modals & Bottom Sheets

The software keyboard is a system window that slides up over the bottom ~40% of the screen. It
doesn't resize your layout for you on every platform and configuration, so the input the user is
typing into — usually at the bottom of the screen, like a chat box or the "add habit" bar from
Level 1 — ends up hidden behind it. This lesson fixes that, then covers the overlay UI that often
holds those inputs: modals and bottom sheets.

## Why the keyboard behaves differently on each platform

- **iOS** never resizes your app's window for the keyboard. Your layout stays the same size and the
  keyboard covers it; you have to move content yourself.
- **Android** historically resized the window (`windowSoftInputMode="adjustResize"`), so
  `flex: 1` layouts shrank automatically. With **edge-to-edge** display, which recent Android
  versions push apps towards and newer React Native/Expo templates enable, the app draws behind
  system bars and the keyboard is reported as an **inset** instead — so on modern setups you often
  need to handle it explicitly on Android too.

Test keyboard behaviour on both platforms, on a real device if possible. It's one of the most
common sources of "works on my phone" bugs.

## KeyboardAvoidingView

The built-in component moves or shrinks its content as the keyboard appears:

```tsx
import { KeyboardAvoidingView, Platform } from 'react-native';
import { useHeaderHeight } from 'expo-router/react-navigation';

function ChatScreen() {
  const headerHeight = useHeaderHeight(); // height of the navigation header above this screen
  return (
    <KeyboardAvoidingView
      style={{ flex: 1 }}
      behavior={Platform.OS === 'ios' ? 'padding' : undefined}
      keyboardVerticalOffset={headerHeight}
    >
      <MessageList />
      <Composer />
    </KeyboardAvoidingView>
  );
}
```

- `behavior="padding"` adds bottom padding equal to the keyboard height — the most predictable
  option on iOS. `"height"` shrinks the view; `"position"` moves it.
- `keyboardVerticalOffset` compensates for anything **above** the view that the component doesn't
  know about — usually the navigation header. Get it wrong and the composer ends up slightly hidden
  or floating too high.
  (In the SDK 57 project this course was checked against, Expo Router ships its own copy of React
  Navigation and re-exports it from `expo-router/react-navigation`; with older SDKs or plain React
  Navigation, import `useHeaderHeight` from `@react-navigation/elements`.)
- On Android, whether you need a `behavior` depends on your app's soft-input mode and edge-to-edge
  setup. Start with `undefined`, test, and adjust.

!!! tip "When the built-in tools aren't enough"
    For chat-style screens, interactive keyboard dismissal, or consistent behaviour across both
    platforms, many teams use the community library
    [`react-native-keyboard-controller`](https://github.com/kirillzyusko/react-native-keyboard-controller),
    which tracks the keyboard frame by frame on the UI thread. It needs a development build (Level 3).

## Forms in a ScrollView

For long forms, the pattern is a `ScrollView` that knows about the keyboard:

```tsx
<KeyboardAvoidingView style={{ flex: 1 }} behavior={Platform.OS === 'ios' ? 'padding' : undefined}>
  <ScrollView
    contentContainerStyle={{ padding: 16, gap: 12 }}
    keyboardShouldPersistTaps="handled"
    keyboardDismissMode="interactive"
    automaticallyAdjustKeyboardInsets   // iOS: adjusts scroll insets so the focused input stays visible
  >
    {/* fields */}
  </ScrollView>
</KeyboardAvoidingView>
```

- `keyboardShouldPersistTaps="handled"` — a tap on a button while the keyboard is open triggers the
  button instead of only dismissing the keyboard.
- `keyboardDismissMode="interactive"` (iOS) lets the user drag the keyboard down with the scroll;
  `"on-drag"` dismisses as soon as scrolling starts (works on both).
- Tapping outside inputs should dismiss the keyboard: `Keyboard.dismiss()` from a `Pressable`
  background, or rely on the scroll view's dismiss mode.

## Listening to the keyboard

Sometimes you need to know whether it's open (to hide a footer, for instance):

```tsx
import { useEffect, useState } from 'react';
import { Keyboard, Platform } from 'react-native';

export function useKeyboardVisible() {
  const [visible, setVisible] = useState(false);
  useEffect(() => {
    const showEvt = Platform.OS === 'ios' ? 'keyboardWillShow' : 'keyboardDidShow';
    const hideEvt = Platform.OS === 'ios' ? 'keyboardWillHide' : 'keyboardDidHide';
    const s = Keyboard.addListener(showEvt, () => setVisible(true));
    const h = Keyboard.addListener(hideEvt, () => setVisible(false));
    return () => { s.remove(); h.remove(); };
  }, []);
  return visible;
}
```

iOS provides "will" events (before the animation), Android only "did" events (after), which is why
the event names differ.

## Modals

You have two kinds:

1. **Route modals** — screens presented modally by the navigator (`presentation: 'modal'` in Expo
   Router, lesson 2). Use these for full tasks with their own URL: "New habit", "Edit profile".
2. **The `Modal` component** — an overlay rendered from within a screen. Use it for small,
   transient UI that doesn't deserve a route: a confirmation, a picker, a quick filter.

```tsx
import { Modal, Pressable, Text, View } from 'react-native';

<Modal
  visible={open}
  animationType="slide"
  presentationStyle="pageSheet"          // iOS card-style sheet; ignored on Android
  onRequestClose={() => setOpen(false)}  // Android back button / iOS swipe-down on pageSheet
>
  <View style={{ flex: 1, padding: 24 }}>
    <Text style={{ fontSize: 20, fontWeight: '700' }}>Filter habits</Text>
    {/* … */}
    <Pressable onPress={() => setOpen(false)}><Text>Done</Text></Pressable>
  </View>
</Modal>
```

Always implement `onRequestClose`: on Android, the back button calls it, and without it users can
get stuck. For a dimmed, centred dialog instead of a sheet, use `transparent` and draw your own
backdrop.

## Bottom sheets

Bottom sheets — panels that slide up from the bottom, can be dragged between heights and dismissed
with a swipe — are a staple of mobile UI (maps, music players, share sheets). A simple sheet can be a
transparent `Modal`:

```tsx title="src/SimpleSheet.tsx"
import { ReactNode } from 'react';
import { KeyboardAvoidingView, Modal, Platform, Pressable, StyleSheet, View } from 'react-native';
import { useSafeAreaInsets } from 'react-native-safe-area-context';

type Props = { open: boolean; onClose: () => void; children: ReactNode };

export function SimpleSheet({ open, onClose, children }: Props) {
  const insets = useSafeAreaInsets();
  return (
    <Modal visible={open} transparent animationType="slide" onRequestClose={onClose} statusBarTranslucent>
      <KeyboardAvoidingView style={styles.fill} behavior={Platform.OS === 'ios' ? 'padding' : undefined}>
        <Pressable style={styles.backdrop} onPress={onClose} accessibilityRole="button" accessibilityLabel="Close sheet" />
        <View style={[styles.sheet, { paddingBottom: Math.max(insets.bottom, 16) }]} accessibilityViewIsModal>
          <View style={styles.grabber} />
          {children}
        </View>
      </KeyboardAvoidingView>
    </Modal>
  );
}

const styles = StyleSheet.create({
  fill: { flex: 1, justifyContent: 'flex-end' },
  backdrop: { ...StyleSheet.absoluteFill, backgroundColor: 'rgba(15,23,42,0.4)' },
  sheet: { backgroundColor: 'white', borderTopLeftRadius: 20, borderTopRightRadius: 20, padding: 16, gap: 12 },
  grabber: { alignSelf: 'center', width: 40, height: 5, borderRadius: 3, backgroundColor: '#cbd5e1', marginBottom: 4 },
});
```

(One flaw: `animationType="slide"` slides the dimmed backdrop up too. Fixing that properly needs an
animated sheet, which Level 3 builds with Reanimated and Gesture Handler.) For production sheets with
snap points and drag gestures, the widely used `@gorhom/bottom-sheet` library is built on those same
two libraries.

## Worked example: the habit tracker's add bar, fixed

Recall the Level 1 project's known bug: the keyboard covered the add bar. The fix:

```tsx title="app/index.tsx"
import { KeyboardAvoidingView, Platform, FlatList, View } from 'react-native';
import { useSafeAreaInsets } from 'react-native-safe-area-context';
import { AddHabitBar } from '../src/AddHabitBar';
import { useKeyboardVisible } from '../src/useKeyboardVisible';

export default function Today() {
  const insets = useSafeAreaInsets();
  const keyboardOpen = useKeyboardVisible();
  return (
    <KeyboardAvoidingView style={{ flex: 1 }} behavior={Platform.OS === 'ios' ? 'padding' : undefined}>
      <FlatList data={[]} renderItem={null} keyboardDismissMode="on-drag" keyboardShouldPersistTaps="handled" />
      <View>
        {/* When the keyboard is open it covers the home indicator, so the bottom inset isn't needed. */}
        <AddHabitBar bottomInset={keyboardOpen ? 0 : insets.bottom} onAdd={() => {}} />
      </View>
    </KeyboardAvoidingView>
  );
}
```

```tsx title="src/AddHabitBar.tsx"
import { useState } from 'react';
import { TextInput, View } from 'react-native';

export function AddHabitBar({ bottomInset, onAdd }: { bottomInset: number; onAdd: (name: string) => void }) {
  const [name, setName] = useState('');
  return (
    <View style={{ padding: 12, paddingBottom: Math.max(bottomInset, 12), backgroundColor: 'white' }}>
      <TextInput value={name} onChangeText={setName} onSubmitEditing={() => { onAdd(name); setName(''); }} placeholder="New habit" />
    </View>
  );
}
```

```ts title="src/useKeyboardVisible.ts"
import { useEffect, useState } from 'react';
import { Keyboard, Platform } from 'react-native';

export function useKeyboardVisible() {
  const [visible, setVisible] = useState(false);
  useEffect(() => {
    const s = Keyboard.addListener(Platform.OS === 'ios' ? 'keyboardWillShow' : 'keyboardDidShow', () => setVisible(true));
    const h = Keyboard.addListener(Platform.OS === 'ios' ? 'keyboardWillHide' : 'keyboardDidHide', () => setVisible(false));
    return () => { s.remove(); h.remove(); };
  }, []);
  return visible;
}
```

Dropping the safe-area padding while the keyboard is open removes the awkward gap between the input
and the keyboard on iPhones with a home indicator.

## How It Actually Works

When a `TextInput` gains focus, the OS shows the keyboard window and posts notifications with its
final frame and animation duration — `UIKeyboardWillShowNotification` on iOS; on Android, a change in
the window's IME insets (or a window resize under `adjustResize`). React Native's `Keyboard` module
forwards these to JavaScript as `keyboardWillShow`/`keyboardDidShow` events with
`endCoordinates.height` and `duration`.

`KeyboardAvoidingView` listens to those events, compares the keyboard's top edge with its own frame
on screen (measured via `onLayout`, plus `keyboardVerticalOffset`), and sets padding, height or a
position offset to the overlap. On iOS it animates the change with `LayoutAnimation` using the
keyboard's own duration and easing, so the content moves in step with the keyboard. Because this is
driven from JavaScript, a busy JS thread can make the movement lag — which is the problem
UI-thread-driven libraries solve.

`Modal` creates a separate native window (on iOS a presented view controller; on Android a `Dialog`)
above your app's root view, which is why it covers navigation headers and tab bars, and why it needs
its own safe-area handling and its own `onRequestClose` for the hardware back button.

## Common mistakes

- **Forgetting `keyboardVerticalOffset`** under a navigation header.
- **Applying the iOS `padding` behaviour on Android** without testing — double adjustment, content
  jumps.
- **No `keyboardShouldPersistTaps`** — users tap "Save" twice.
- **A `Modal` without `onRequestClose`** — the Android back button does nothing.
- **Using `Modal` for full flows** that should be routes (and deep-linkable).
- **Bottom inset padding while the keyboard is open** — a gap between input and keyboard.

## Exercise

1. Build a chat-style screen: an inverted `FlatList` of messages (`inverted` prop) and a composer
   pinned to the bottom. The composer must stay above the keyboard on iOS and Android, with the
   header offset correct.
2. Add a "Filters" button that opens `SimpleSheet` containing a `TextInput`. Confirm the sheet rises
   above the keyboard and the Android back button closes it.
3. Make the composer grow with multi-line input up to 5 lines (`multiline` + `maxHeight`), then
   scroll internally.
