---
description: "Style React Native apps with StyleSheet: density-independent units, style arrays, no cascade, design tokens, dark mode and a reusable AppText."
---

# 03 · Styling with StyleSheet

React Native styling uses JavaScript objects with CSS-like property names in camelCase. There are no
stylesheets, no selectors, no cascade and no specificity wars. Every component gets exactly the
styles you pass to its `style` prop — nothing more. Once you stop looking for CSS features that
aren't there, this turns out to be a very predictable system.

## The basics

```tsx
import { StyleSheet, Text, View } from 'react-native';

export function Badge({ label }: { label: string }) {
  return (
    <View style={styles.badge}>
      <Text style={styles.text}>{label}</Text>
    </View>
  );
}

const styles = StyleSheet.create({
  badge: {
    backgroundColor: '#eef2ff',
    paddingHorizontal: 10,
    paddingVertical: 4,
    borderRadius: 999,
    alignSelf: 'flex-start',
  },
  text: { color: '#4338ca', fontSize: 12, fontWeight: '600', letterSpacing: 0.3 },
});
```

Differences from CSS to internalise:

- **Units:** numbers are **density-independent pixels** (dp on Android, points on iOS). `fontSize: 16`
  is the same physical size on a low- and a high-density screen. There is no `px`, `rem` or `em`.
  Percentages are allowed as strings for sizes and positions (`width: '50%'`).
- **Shorthands are partial:** `paddingHorizontal` / `paddingVertical` / `marginHorizontal` exist
  instead of `padding: '4px 10px'`. `border: '1px solid'` doesn't exist — use `borderWidth`,
  `borderColor`, `borderStyle`.
- **`fontWeight` is a string** (`'400'`, `'600'`, `'bold'`) — or a number in newer versions; strings
  work everywhere.
- **No inheritance**, except nested `Text` (lesson 2).
- **No pseudo-classes or media queries** — `:hover`, `:pressed`, dark mode and breakpoints are all
  done in JavaScript with state and hooks.

## Combining styles: arrays

The `style` prop accepts an array. Later entries win, and falsy entries are ignored, which makes
conditional styling clean:

```tsx
<View style={[styles.card, selected && styles.cardSelected, { marginTop: index === 0 ? 0 : 12 }]} />
```

This is how you replace CSS modifiers like `.card--selected`. Inline objects (the last entry) are
fine for values that are truly dynamic; put static styles in `StyleSheet.create`.

## What `StyleSheet.create` actually gives you

`StyleSheet.create` returns the same objects you passed in (in development it validates property
names and logs mistakes). Its value is mainly organisational and for tooling — typed autocompletion
and catching typos like `backgroundColour`. There is no hidden performance trick to rely on; inline
style objects work too. Use `create` because it keeps styles named, typed and out of your render
logic.

Other helpers:

```tsx
StyleSheet.hairlineWidth;          // thinnest visible line on this device's screen
StyleSheet.absoluteFill;           // { position: 'absolute', left: 0, right: 0, top: 0, bottom: 0 }
StyleSheet.flatten([a, b]);        // merge a style array into one object (useful in tests)
```

## Shadows: the platform split

Shadows show the platforms' different heritage. Recent React Native versions support the
cross-platform `boxShadow` property (with the New Architecture), using CSS syntax:

```tsx
const styles = StyleSheet.create({
  card: {
    backgroundColor: 'white',
    borderRadius: 16,
    boxShadow: '0 2px 8px rgba(15, 23, 42, 0.12)',
  },
});
```

Older code you'll read uses the per-platform properties: `shadowColor`, `shadowOffset`,
`shadowOpacity`, `shadowRadius` on iOS, and `elevation` on Android. A shadow needs a
**background colour** on the view to render correctly on both platforms; a transparent view with a
shadow often shows nothing (Android) or shadows every child separately (iOS).

## Design tokens instead of a stylesheet language

Without Sass variables, put your design decisions in plain TypeScript:

```ts title="theme.ts"
export const colors = {
  light: { bg: '#f8fafc', surface: '#ffffff', text: '#0f172a', muted: '#64748b', primary: '#4f46e5' },
  dark:  { bg: '#020617', surface: '#0f172a', text: '#f1f5f9', muted: '#94a3b8', primary: '#818cf8' },
} as const;

export const space = { xs: 4, sm: 8, md: 12, lg: 16, xl: 24 } as const;
export const radius = { sm: 8, md: 12, lg: 16, pill: 999 } as const;
export const font = { body: 16, small: 13, title: 22 } as const;

// Both palettes share keys; widen the literal colour values to string so either fits.
export type Palette = Record<keyof (typeof colors)['light'], string>;
```

## Worked example: dark mode and an AppText component

`useColorScheme()` returns `'light'`, `'dark'` or `null`, and re-renders when the user switches
system theme. Build one small hook and one text component, and most of your styling problems go
away:

```tsx title="useTheme.ts"
import { useColorScheme } from 'react-native';
import { colors, Palette } from './theme';

export function useTheme(): Palette {
  return useColorScheme() === 'dark' ? colors.dark : colors.light;
}
```

```tsx title="AppText.tsx"
import { Text, TextProps } from 'react-native';
import { useTheme } from './useTheme';
import { font } from './theme';

type Variant = 'body' | 'title' | 'muted';

export function AppText({ variant = 'body', style, ...rest }: TextProps & { variant?: Variant }) {
  const c = useTheme();
  const variantStyle = {
    body: { fontSize: font.body, color: c.text },
    title: { fontSize: font.title, color: c.text, fontWeight: '700' as const },
    muted: { fontSize: font.small, color: c.muted },
  }[variant];
  return <Text {...rest} style={[variantStyle, style]} />;
}
```

Note the order `[variantStyle, style]`: the caller's `style` comes last so callers can always
override. Then in `app.json`, set `"userInterfaceStyle": "automatic"` — the blank template ships
with `"light"`, which forces light mode and makes `useColorScheme()` always return `'light'`.

```tsx
function HabitRow({ name, done }: { name: string; done: boolean }) {
  const c = useTheme();
  return (
    <View style={[rowStyles.row, { backgroundColor: c.surface }]}>
      <AppText style={done && rowStyles.done}>{name}</AppText>
      <AppText variant="muted">{done ? 'Done' : 'Pending'}</AppText>
    </View>
  );
}

const rowStyles = StyleSheet.create({
  row: { flexDirection: 'row', justifyContent: 'space-between', padding: 16, borderRadius: 12 },
  done: { textDecorationLine: 'line-through', opacity: 0.6 },
});
```

## How It Actually Works

Your style objects travel with the component's props to Fabric. Fabric splits each style into two
groups: **layout props** (`width`, `flex`, `padding`, `margin`, `position`, …), which are handed to
Yoga to compute geometry, and **view props** (`backgroundColor`, `borderRadius`, `opacity`,
`transform`, …), which are applied to the native view. When you pass an array, React Native
flattens it left to right into one object before diffing, which is why later entries win and why
`false`/`null`/`undefined` entries vanish harmlessly.

Because there is no cascade, the renderer never has to work out which rules apply to which element —
each node's style is exactly its own flattened object. That is the trade: you write a little more
(passing typography down explicitly, or wrapping it in `AppText`), and in exchange there's no
action-at-a-distance. A style change in one file can only affect the components that import it.

Density-independent units are converted at the native edge: on a 3× iPhone, `width: 100` becomes
300 physical pixels. `PixelRatio.get()` exposes that scale if you ever need exact physical pixels
(for example, requesting a correctly sized image from a CDN).

## Common mistakes

- **Writing CSS strings** like `padding: '8px 16px'` or `border: '1px solid #ccc'` — not supported.
- **Expecting `color` on a `View` to style the text inside.**
- **Hard-coding colours in every component**, then discovering dark mode means touching 80 files.
  Tokens from day one.
- **Leaving `userInterfaceStyle: "light"`** in `app.json` and wondering why dark mode never
  triggers.
- **Putting the caller's `style` first** in a reusable component's array — the component's own
  styles then silently override what the caller asked for.

## Exercise

1. Create `theme.ts`, `useTheme.ts` and `AppText.tsx` as above.
2. Build a `Card` component that takes `title`, `subtitle` and an optional `highlighted` boolean,
   uses the theme surface colour, and applies a border in the primary colour when highlighted.
3. Switch your phone between light and dark mode and confirm everything updates without reloading.
4. Add a `Button` component with `variant: 'primary' | 'secondary'` and a `disabled` state that
   lowers opacity — keeping the caller-override order correct.
