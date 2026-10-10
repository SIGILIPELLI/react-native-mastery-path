---
description: "Handle notches and home indicators with safe-area insets, branch on Platform.OS, use .ios/.android files, and size layouts with useWindowDimensions."
---

# 08 · Safe Areas, Platform Differences & Screen Size

"Write once, run anywhere" is almost true in React Native, and the gap lives in three places: the
**physical shape** of each phone (notches, Dynamic Island, rounded corners, home indicators, Android
navigation bars), the **conventions** of each platform (back buttons, ripples, date pickers), and
the **size** of the screen (small phones, big phones, tablets, foldables, landscape). This lesson
handles all three.

## Safe areas with react-native-safe-area-context

The **safe area** is the part of the screen not covered by system UI. React Native's core
`SafeAreaView` only ever worked on iOS and is deprecated; the standard is the
`react-native-safe-area-context` library, which Expo projects include through Expo Router or can
install directly:

```bash
npx expo install react-native-safe-area-context
```

Wrap your app once in the provider:

```tsx title="App.tsx"
import { SafeAreaProvider } from 'react-native-safe-area-context';
import TodayScreen from './TodayScreen';

export default function App() {
  return (
    <SafeAreaProvider>
      <TodayScreen />
    </SafeAreaProvider>
  );
}
```

Then either use the drop-in `SafeAreaView` with chosen edges:

```tsx
import { SafeAreaView } from 'react-native-safe-area-context';

<SafeAreaView style={{ flex: 1 }} edges={['top', 'left', 'right']}>
  {/* content */}
</SafeAreaView>
```

…or, more flexibly, read the insets yourself:

```tsx
import { useSafeAreaInsets } from 'react-native-safe-area-context';

function Footer() {
  const insets = useSafeAreaInsets();
  return (
    <View style={{ paddingTop: 12, paddingBottom: Math.max(insets.bottom, 12), backgroundColor: 'white' }}>
      <AddButton />
    </View>
  );
}
```

The hook is better for headers and footers that should paint their background *under* the system
UI (so the colour reaches the edge) while keeping their content inside the safe area. `edges`
matters: a screen with its own bottom tab bar usually doesn't want bottom padding twice.

!!! note "Edge-to-edge on Android"
    Recent Android versions push apps to draw **edge-to-edge** — behind the status bar and the
    navigation bar — and newer React Native/Expo templates enable it by default. That makes safe
    area insets just as important on Android as on iOS. If content hides under the Android
    navigation bar, you're missing a bottom inset.

## Platform-specific code

For small differences, `Platform.OS` and `Platform.select`:

```tsx
import { Platform, StyleSheet } from 'react-native';

const styles = StyleSheet.create({
  title: {
    fontSize: 17,
    fontWeight: Platform.OS === 'ios' ? '600' : '500',
  },
  card: Platform.select({
    ios: { boxShadow: '0 1px 4px rgba(0,0,0,0.12)' },
    android: { elevation: 2 },
    default: {},
  }),
});

if (Platform.OS === 'android' && Platform.Version < 33) {
  // Android API level below 33 (Android 13) — e.g. notification permission is implicit
}
```

`Platform.Version` is the Android **API level** (a number) on Android, and the iOS version
**string** (like `"18.2"`) on iOS — compare them differently.

For larger differences, use **platform-specific files**. Metro picks the right one at bundle time:

```
DatePickerField.ios.tsx
DatePickerField.android.tsx
```

```tsx
import { DatePickerField } from './DatePickerField'; // no extension: Metro resolves per platform
```

Both files must export the same interface. With TypeScript, add a `DatePickerField.tsx` (or a
`.d.ts`) holding the shared types so imports type-check. There's also `.native.tsx`, which is chosen
for iOS and Android but not for web — handy if you also target React Native for Web.

## Screen size: useWindowDimensions

```tsx
import { useWindowDimensions } from 'react-native';

function Grid({ items }: { items: string[] }) {
  const { width, fontScale } = useWindowDimensions();
  const columns = width >= 768 ? 4 : width >= 500 ? 3 : 2;
  const gap = 12;
  const itemSize = (width - 32 - gap * (columns - 1)) / columns;
  // ...render a wrapped row of itemSize squares
}
```

`useWindowDimensions` re-renders when the window size changes — rotation, split-screen on iPad,
resizing on a foldable. Prefer it to `Dimensions.get('window')`, which returns a snapshot and won't
update your component. `fontScale` is the user's text size setting; if it's large (say above 1.3),
switch dense horizontal layouts to vertical stacks.

## Worked example: a screen that respects every edge

```tsx title="TodayScreen.tsx"
import { Platform, Pressable, StyleSheet, Text, useWindowDimensions, View, FlatList } from 'react-native';
import { useSafeAreaInsets } from 'react-native-safe-area-context';

const habits = ['Drink water', 'Stretch', 'Read', 'Walk', 'Journal', 'Sleep by 11'];

export default function TodayScreen() {
  const insets = useSafeAreaInsets();
  const { width } = useWindowDimensions();
  const wide = width >= 600;

  return (
    <View style={styles.screen}>
      <View style={[styles.header, { paddingTop: insets.top + 8, paddingLeft: insets.left + 16, paddingRight: insets.right + 16 }]}>
        <Text style={styles.title}>Today</Text>
      </View>
      <FlatList
        key={wide ? 'grid' : 'list'}        // numColumns can't change on the fly; remount instead
        data={habits}
        numColumns={wide ? 2 : 1}
        keyExtractor={(h) => h}
        columnWrapperStyle={wide ? { gap: 12 } : undefined}
        contentContainerStyle={{ padding: 16, gap: 12, paddingLeft: insets.left + 16, paddingRight: insets.right + 16 }}
        renderItem={({ item }) => (
          <View style={[styles.card, wide && { flex: 1 }]}>
            <Text style={styles.cardText}>{item}</Text>
          </View>
        )}
      />
      <View style={[styles.footer, { paddingBottom: Math.max(insets.bottom, 16) }]}>
        <Pressable android_ripple={{ color: 'rgba(255,255,255,0.25)' }} style={styles.button}>
          <Text style={styles.buttonText}>Add habit</Text>
        </Pressable>
      </View>
    </View>
  );
}

const styles = StyleSheet.create({
  screen: { flex: 1, backgroundColor: '#f8fafc' },
  header: { backgroundColor: '#4f46e5', paddingBottom: 16 },
  title: { color: 'white', fontSize: 28, fontWeight: '800' },
  card: {
    backgroundColor: 'white', borderRadius: 12, padding: 16,
    ...Platform.select({ ios: { boxShadow: '0 1px 3px rgba(0,0,0,0.1)' }, android: { elevation: 1 }, default: {} }),
  },
  cardText: { fontSize: 16 },
  footer: { paddingHorizontal: 16, paddingTop: 12, backgroundColor: 'white' },
  button: { backgroundColor: '#4f46e5', borderRadius: 12, paddingVertical: 14, alignItems: 'center', overflow: 'hidden' },
  buttonText: { color: 'white', fontWeight: '700', fontSize: 16 },
});
```

The purple header paints all the way to the top of the screen while its title sits below the
notch; the footer's white background reaches the bottom edge while the button stays above the home
indicator; and in landscape the left/right insets keep content away from the notch on the side.

## How It Actually Works

Each OS reports insets natively: iOS through `safeAreaInsets` on the root view, Android through
`WindowInsets` (status bar, navigation bar, display cutout). `SafeAreaProvider` renders a native view
that listens for those inset changes and publishes them through React context. The first value
comes from a synchronous native measurement where possible (`initialWindowMetrics`), so the first
frame can already be laid out correctly instead of jumping. When the device rotates, the native
side posts new insets, the context updates, and only components that read them re-render.

`Platform.OS` is a constant baked into each platform's JavaScript bundle. Metro builds a **separate
bundle per platform**, so `.ios.tsx` / `.android.tsx` resolution happens at bundle time and the
unused file is never shipped to the other platform. Dead branches like
`if (Platform.OS === 'ios')` in an Android bundle can be removed by the minifier in production.

## Common mistakes

- **Hard-coded top padding** (44, 47, 59…) — wrong on most devices. Use insets.
- **Core `SafeAreaView`** from `react-native` — iOS-only and deprecated.
- **Applying insets twice** — e.g. a `SafeAreaView` inside a navigator that already pads the
  header. Choose `edges` deliberately.
- **`Dimensions.get` at module top level** — captured once, wrong after rotation.
- **Comparing `Platform.Version` as a number on iOS.**
- **Testing only on the device you own.** Insets differ between a small Android phone with
  gesture navigation and an iPhone with a Dynamic Island.

## Exercise

Take the lesson 4 calculator or the Today screen and make it safe-area correct: header colour to
the top edge, footer colour to the bottom edge, and no content under any cutout in portrait or
landscape. Then add a `Banner.ios.tsx` and `Banner.android.tsx` that render different copy ("Add
to Home Screen widget" vs "Add a home screen shortcut"), share a typed interface, and import
cleanly from `./Banner`. Run on both platforms (or one device plus a simulator/emulator) and
screenshot each.
