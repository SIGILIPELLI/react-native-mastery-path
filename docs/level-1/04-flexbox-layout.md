---
description: "Master React Native Flexbox: column-first defaults, flex: 1, flexGrow vs flexShrink, gap, absolute positioning and building a header-body-footer screen."
---

# 04 · Flexbox Layout the React Native Way

Every `View` in React Native is a flex container. There is no `display: block`, no floats, no
inline flow (outside `Text`) and no CSS Grid. That sounds limiting, but Flexbox plus absolute
positioning is enough for every mobile layout you will build — once you know where React Native's
defaults differ from the web's.

## The defaults that differ from CSS

| Property | Web default | React Native default |
|----------|-------------|----------------------|
| `display` | `block` / `inline` | always `flex` |
| `flexDirection` | `row` | **`column`** |
| `alignContent` | `normal` (`stretch`) | `flex-start` |
| `flexShrink` | `1` | **`0`** |
| `position` | `static` | `relative` |
| box sizing | `content-box` | border-box behaviour (borders and padding are inside `width`) |

The column default exists because phones are tall: most screens stack things vertically. The
`flexShrink: 0` default bites people who come from the web: a long row of items will **overflow**
rather than shrink unless you opt in.

## `flex: 1` — the most important line you'll write

In React Native, `flex` takes a single number. `flex: 1` means "grow to fill the available space
along the main axis, and allow shrinking."

```tsx
<View style={{ flex: 1 }}>          {/* fills the screen */}
  <View style={{ height: 56 }} />   {/* fixed header */}
  <View style={{ flex: 1 }} />      {/* body takes everything left */}
  <View style={{ height: 64 }} />   {/* fixed footer */}
</View>
```

If your root view doesn't have `flex: 1`, it is only as tall as its content — the common cause of
"my background colour only covers half the screen" and "my list doesn't scroll".

Multiple siblings with `flex` share space proportionally: `flex: 2` and `flex: 1` split remaining
space 2:1.

`flex` in React Native is not the CSS shorthand. A positive number `n` behaves like
`flexGrow: n, flexShrink: 1, flexBasis: 0`. If you need the CSS-style independent control, use
`flexGrow`, `flexShrink` and `flexBasis` directly.

## Axes, justify and align

- `flexDirection` sets the **main axis** (`column` default, or `row`).
- `justifyContent` distributes children along the main axis: `flex-start`, `center`, `flex-end`,
  `space-between`, `space-around`, `space-evenly`.
- `alignItems` positions children on the **cross axis**: `stretch` (default), `flex-start`,
  `center`, `flex-end`, `baseline`.
- `alignSelf` overrides `alignItems` for one child.
- `gap`, `rowGap`, `columnGap` put space between children — much cleaner than margins on every
  child.
- `flexWrap: 'wrap'` lets a row wrap onto new lines (chips, tags, a photo grid).

Because the default `alignItems` is `stretch`, children in a column are full width. Set
`alignItems: 'center'` or `alignSelf: 'flex-start'` when you want a child to hug its content —
otherwise a "badge" stretches across the whole screen.

## A row that doesn't overflow

The classic list-row layout — avatar, title that can be long, and a trailing value:

```tsx
<View style={{ flexDirection: 'row', alignItems: 'center', gap: 12, padding: 16 }}>
  <View style={{ width: 40, height: 40, borderRadius: 20, backgroundColor: '#c7d2fe' }} />
  <View style={{ flex: 1 }}>
    <Text numberOfLines={1} style={{ fontSize: 16, fontWeight: '600' }}>
      Read 20 pages of a book that has a remarkably long title
    </Text>
    <Text style={{ color: '#64748b' }}>Daily · 7:00 PM</Text>
  </View>
  <Text style={{ fontWeight: '700' }}>12🔥</Text>
</View>
```

The middle column's `flex: 1` is what makes this work: it takes whatever width remains after the
fixed avatar and the trailing value, and `numberOfLines={1}` truncates inside that width. Remove
`flex: 1` and the text pushes the streak off the screen, because nothing is allowed to shrink.

## Absolute positioning

`position: 'absolute'` takes a view out of the flex flow and positions it relative to its parent
(every view is `relative` by default, so the parent is always the containing block):

```tsx
<View>
  <Image source={{ uri }} style={{ width: 64, height: 64, borderRadius: 32 }} />
  <View style={{
    position: 'absolute', right: -2, bottom: -2,
    width: 18, height: 18, borderRadius: 9,
    backgroundColor: '#22c55e', borderWidth: 2, borderColor: 'white',
  }} />
</View>
```

Use it for badges, overlays and floating action buttons. Don't use it for general layout; it doesn't
adapt to different screen sizes or font sizes.

## Worked example: a full screen layout

```tsx title="TodayScreen.tsx"
import { Pressable, ScrollView, StyleSheet, Text, View } from 'react-native';

const habits = ['Drink water', 'Stretch 10 min', 'Read 20 pages', 'No phone after 10pm'];

export default function TodayScreen() {
  return (
    <View style={styles.screen}>
      <View style={styles.header}>
        <Text style={styles.title}>Today</Text>
        <View style={styles.progressPill}>
          <Text style={styles.progressText}>2 / 4</Text>
        </View>
      </View>

      <ScrollView style={styles.body} contentContainerStyle={styles.bodyContent}>
        <View style={styles.chips}>
          {['Morning', 'Health', 'Focus', 'Evening', 'Weekend'].map((c) => (
            <Text key={c} style={styles.chip}>{c}</Text>
          ))}
        </View>
        {habits.map((h) => (
          <View key={h} style={styles.row}>
            <Text style={styles.rowText} numberOfLines={1}>{h}</Text>
            <View style={styles.check} />
          </View>
        ))}
      </ScrollView>

      <View style={styles.footer}>
        <Pressable style={styles.addButton} accessibilityRole="button">
          <Text style={styles.addText}>Add habit</Text>
        </Pressable>
      </View>
    </View>
  );
}

const styles = StyleSheet.create({
  screen: { flex: 1, backgroundColor: '#f8fafc' },
  header: { flexDirection: 'row', alignItems: 'center', justifyContent: 'space-between', paddingHorizontal: 16, paddingTop: 56, paddingBottom: 12 },
  title: { fontSize: 28, fontWeight: '800' },
  progressPill: { backgroundColor: '#e0e7ff', borderRadius: 999, paddingHorizontal: 12, paddingVertical: 4 },
  progressText: { color: '#3730a3', fontWeight: '700' },
  body: { flex: 1 },
  bodyContent: { padding: 16, gap: 10 },
  chips: { flexDirection: 'row', flexWrap: 'wrap', gap: 8, marginBottom: 8 },
  chip: { backgroundColor: 'white', borderRadius: 999, paddingHorizontal: 12, paddingVertical: 6, overflow: 'hidden' },
  row: { flexDirection: 'row', alignItems: 'center', backgroundColor: 'white', borderRadius: 12, padding: 16, gap: 12 },
  rowText: { flex: 1, fontSize: 16 },
  check: { width: 24, height: 24, borderRadius: 12, borderWidth: 2, borderColor: '#cbd5e1' },
  footer: { padding: 16, paddingBottom: 32, backgroundColor: 'white' },
  addButton: { backgroundColor: '#4f46e5', borderRadius: 12, paddingVertical: 14, alignItems: 'center' },
  addText: { color: 'white', fontSize: 16, fontWeight: '700' },
});
```

The `paddingTop: 56` and `paddingBottom: 32` are stand-ins for the notch and the home indicator. They
are wrong on half the devices out there; lesson 8 replaces them with real safe-area insets.

## How It Actually Works

React Native does not use a browser's layout engine. It uses **Yoga**, a cross-platform C++
implementation of a Flexbox subset, maintained by Meta. In the New Architecture, every host
component has a C++ shadow node, and each shadow node owns a Yoga node with its layout styles.

When props change, Fabric clones the affected shadow nodes, Yoga recomputes layout for the dirty
part of the tree, and the result — an absolute `x`, `y`, `width`, `height` for every node — is
compared with the previous frame. Only views whose frames or props actually changed receive native
mutations. Layout runs off the UI thread, and measuring text calls into the platform's text system
so Yoga knows how tall a paragraph is at a given width.

This is why there's no CSS Grid or floats: Yoga implements what React Native needs, consistently on
every platform. It's also why the defaults differ — Yoga's defaults were chosen for mobile
(`column`, no shrinking) rather than for web compatibility. (React Native for Web, by contrast, maps
these styles back onto browser CSS.)

## Common mistakes

- **Missing `flex: 1` on the root** or on a `ScrollView`'s parent — content collapses or won't scroll.
- **Text pushing siblings off-screen in a row** — give the text's container `flex: 1` (or
  `flexShrink: 1`).
- **A child stretching full-width unexpectedly** — default `alignItems: 'stretch'`; use `alignSelf`.
- **`justifyContent: 'center'` on a `ScrollView`'s `style`** — it must go on
  `contentContainerStyle`, and centring scrollable content usually also needs
  `flexGrow: 1` there.
- **Hard-coded paddings for notches** (as in this lesson's example) left in production.

## Exercise

1. Build a calculator keypad: a display area that takes the top third (`flex: 1`) and a 4×5 grid of
   buttons in the bottom two thirds (`flex: 2`), using rows with `flex: 1` buttons and `gap`.
2. Make the "0" button span two columns using `flex: 2` within its row.
3. Add a small red notification badge, absolutely positioned on the top-right corner of one button.
4. Rotate your phone to landscape (enable rotation in `app.json` with `"orientation": "default"`)
   and check the grid still fits — no fixed heights allowed.
