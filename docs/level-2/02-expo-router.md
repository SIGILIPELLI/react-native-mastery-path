---
description: "Build navigation from your folder structure with Expo Router: _layout files, route groups, a tab bar, nested stacks, modals, Link and the router API."
---

# 02 · File-Based Routing with Expo Router

**Expo Router** turns your `app/` directory into a navigation tree. Every file is a route; every
`_layout.tsx` file defines the navigator for its folder. Under the hood it's built on React
Navigation, so everything from the previous lesson — stacks, tabs, modals, focus — applies
directly. What you gain is a single source of truth: the folder structure *is* the navigation tree,
and every screen automatically has a URL (which makes deep links, lesson 3, nearly free).

## Setup

New projects from the default template (`npx create-expo-app@latest` without `--template`) come with
Expo Router already configured. To add it to the blank template from Level 1:

```bash
npx expo install expo-router react-native-safe-area-context react-native-screens expo-linking expo-constants expo-status-bar
```

Then in `package.json` set the entry point, and in `app.json` add a URL scheme:

```json title="package.json (excerpt)"
{ "main": "expo-router/entry" }
```

```json title="app.json (excerpt)"
{ "expo": { "scheme": "habitapp", "plugins": ["expo-router"] } }
```

Delete `App.tsx` and `index.ts`; the `app/` folder replaces them.

## Files become routes

```
app/
├── _layout.tsx            → root layout (a Stack)
├── index.tsx              → "/"
├── about.tsx              → "/about"
├── settings/
│   ├── _layout.tsx        → a Stack for the settings section
│   ├── index.tsx          → "/settings"
│   └── notifications.tsx  → "/settings/notifications"
└── +not-found.tsx         → shown for unmatched URLs
```

A route file default-exports a component:

```tsx title="app/about.tsx"
import { Text, View } from 'react-native';

export default function About() {
  return (
    <View style={{ flex: 1, padding: 16 }}>
      <Text>Habit app v1</Text>
    </View>
  );
}
```

And a layout default-exports a navigator:

```tsx title="app/_layout.tsx"
import { Stack } from 'expo-router';

export default function RootLayout() {
  return (
    <Stack>
      <Stack.Screen name="index" options={{ title: 'Today' }} />
      <Stack.Screen name="about" options={{ title: 'About' }} />
    </Stack>
  );
}
```

Listing `Stack.Screen`s is optional — any file in the folder is a route either way. You list them
to set options (titles, presentation, header buttons) or to control order.

## Moving around: Link and router

Declarative navigation with `Link`:

```tsx
import { Link } from 'expo-router';

<Link href="/about">About</Link>

// Wrap your own pressable instead of rendering a text link:
<Link href="/settings/notifications" asChild>
  <Pressable style={styles.row}><Text>Notifications</Text></Pressable>
</Link>
```

Imperative navigation with the `router` object (or the `useRouter()` hook):

```tsx
import { router } from 'expo-router';

router.push('/settings');        // add to the stack
router.navigate('/settings');    // go there; reuse an existing route in the stack if present
router.replace('/');             // swap the current screen (no back to it)
router.back();                   // pop
router.dismissAll();             // close all screens in the current stack back to its first
```

## Groups: folders that don't appear in the URL

A folder name in parentheses is a **group**. It adds a navigator without adding a URL segment. This
is how tabs are built:

```
app/
├── _layout.tsx               → root Stack
├── (tabs)/
│   ├── _layout.tsx           → Tabs
│   ├── index.tsx             → "/"         (Today tab)
│   ├── stats.tsx             → "/stats"    (Stats tab)
│   └── settings.tsx          → "/settings" (Settings tab)
├── habit/
│   └── [id].tsx              → "/habit/42" (pushed over the tabs)
└── add.tsx                   → "/add"      (modal)
```

## Worked example: tabs, a detail screen and a modal

```tsx title="app/_layout.tsx"
import { Stack } from 'expo-router';

export default function RootLayout() {
  return (
    <Stack>
      <Stack.Screen name="(tabs)" options={{ headerShown: false }} />
      <Stack.Screen name="habit/[id]" options={{ title: 'Habit' }} />
      <Stack.Screen name="add" options={{ presentation: 'modal', title: 'New habit' }} />
    </Stack>
  );
}
```

```tsx title="app/(tabs)/_layout.tsx"
import { Tabs } from 'expo-router/js-tabs';
import { ColorValue, Text } from 'react-native';

function TabIcon({ glyph, color }: { glyph: string; color: ColorValue }) {
  return <Text style={{ fontSize: 20, color }}>{glyph}</Text>;
}

export default function TabsLayout() {
  return (
    <Tabs screenOptions={{ tabBarActiveTintColor: '#4f46e5' }}>
      <Tabs.Screen name="index" options={{ title: 'Today', tabBarIcon: ({ color }) => <TabIcon glyph="✓" color={color} /> }} />
      <Tabs.Screen name="stats" options={{ title: 'Stats', tabBarIcon: ({ color }) => <TabIcon glyph="▤" color={color} /> }} />
      <Tabs.Screen name="settings" options={{ title: 'Settings', tabBarIcon: ({ color }) => <TabIcon glyph="⚙" color={color} /> }} />
    </Tabs>
  );
}
```

!!! note "Where `Tabs` is imported from"
    In the Expo SDK 57 type definitions this course was checked against, importing `Tabs` from
    `expo-router` is marked deprecated in favour of `expo-router/js-tabs`, and a separate
    native-tabs layout (system tab bars) is exported as unstable. Older tutorials import `Tabs`
    from `expo-router` — that still works, but check the Expo Router docs for your SDK before
    starting a new project. Real apps use an icon set such as `@expo/vector-icons` rather than
    text glyphs.

```tsx title="app/(tabs)/index.tsx"
import { Link, router } from 'expo-router';
import { FlatList, Pressable, Text, View } from 'react-native';

const habits = [
  { id: '1', name: 'Drink water' },
  { id: '2', name: 'Read 20 pages' },
];

export default function Today() {
  return (
    <View style={{ flex: 1 }}>
      <FlatList
        data={habits}
        keyExtractor={(h) => h.id}
        contentContainerStyle={{ padding: 16, gap: 8 }}
        renderItem={({ item }) => (
          <Link href={{ pathname: '/habit/[id]', params: { id: item.id } }} asChild>
            <Pressable style={{ padding: 16, backgroundColor: 'white', borderRadius: 12 }}>
              <Text style={{ fontSize: 16 }}>{item.name}</Text>
            </Pressable>
          </Link>
        )}
      />
      <Pressable
        onPress={() => router.push('/add')}
        accessibilityRole="button"
        accessibilityLabel="Add habit"
        style={{ position: 'absolute', right: 20, bottom: 20, width: 56, height: 56, borderRadius: 28, backgroundColor: '#4f46e5', alignItems: 'center', justifyContent: 'center' }}
      >
        <Text style={{ color: 'white', fontSize: 28 }}>+</Text>
      </Pressable>
    </View>
  );
}
```

```tsx title="app/habit/[id].tsx"
import { Stack, useLocalSearchParams } from 'expo-router';
import { Text, View } from 'react-native';

export default function HabitDetail() {
  const { id } = useLocalSearchParams<{ id: string }>();
  return (
    <View style={{ flex: 1, padding: 16 }}>
      <Stack.Screen options={{ title: `Habit ${id}` }} />
      <Text>Details for habit {id}</Text>
    </View>
  );
}
```

```tsx title="app/add.tsx"
import { router } from 'expo-router';
import { Button, View } from 'react-native';

export default function AddHabit() {
  return (
    <View style={{ flex: 1, padding: 16 }}>
      <Button title="Save" onPress={() => router.back()} />
    </View>
  );
}
```

Rendering `<Stack.Screen options={...} />` *inside* a screen component sets that screen's options
from within — useful when the title depends on data the screen loads.

The detail route sits in the **root** stack, so it slides over the tab bar; move it into
`(tabs)/` with its own stack layout if you'd rather keep the tab bar visible.

## How It Actually Works

At build time, Metro is configured (by `expo-router/babel` support in `babel-preset-expo`) so that
`expo-router/entry` can call `require.context('./app')`, a Metro feature that bundles every file in
the `app/` directory and hands Expo Router a map of file paths to modules. At startup Expo Router
walks that map and builds a **route tree**: each `_layout` becomes a navigator node, each other file
a screen node, `[param]` segments become dynamic matchers and `(group)` folders become navigators
with no path segment.

From the route tree it generates two things: the nested React Navigation-style navigators you'd
otherwise write by hand, and a **linking configuration** that maps URL paths to navigation state.
`router.push('/habit/42')` is therefore "turn this URL into a navigation state, then dispatch the
action that gets there" — the same mechanism that handles a deep link from outside the app. In
development, adding a file triggers a Metro rebuild of the context map and the new route appears
without restarting.

## Common mistakes

- **Leaving `App.tsx` and the old `main` entry** — the app ignores `app/` entirely.
- **Putting non-route files (components, utils) in `app/`** — every file there becomes a route.
  Keep components in `src/` or `components/`.
- **A modal declared inside the tabs layout** — it can't cover the tab bar.
- **`router.push` from a "save" button on a modal** — pushes a new copy of the list instead of
  closing the modal. Use `router.back()` or `router.dismiss()`.
- **Forgetting `asChild` on `Link` around a `Pressable`** — you get a text link wrapping your
  component, with broken styling.

## Exercise

Convert the Level 1 habit tracker into an Expo Router app with three tabs (Today, Stats, Settings),
a habit detail screen pushed over the tabs, and an "Add habit" modal. Add `app/+not-found.tsx` with
a link home. Then move `habit/[id].tsx` into the Today tab's own stack (`(tabs)/(today)/…` or a
`today/` folder with a `_layout.tsx`) and observe how the tab bar behaviour changes.
