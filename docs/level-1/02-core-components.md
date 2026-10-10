---
description: "Learn View, Text, Image and ScrollView in depth — the text-nesting rule, image sizing, scroll containers — and build a profile card screen from them."
---

# 02 · Core Components: View, Text, Image & ScrollView

React Native ships a small set of **core components** that map to native views. Almost every screen
you build is a composition of four of them: `View`, `Text`, `Image` and `ScrollView`. They look like
`div`, `span`, `img` and an overflow container, but each has rules that differ from the web in ways
that cause real bugs.

## View — the box

`View` is a container that supports layout (Flexbox), styling, touch handling hooks and
accessibility. It maps to `UIView` on iOS and `android.view.ViewGroup` on Android.

```tsx
<View style={{ padding: 16, backgroundColor: '#f8fafc', borderRadius: 12 }}>
  {/* children */}
</View>
```

Things a `View` cannot do: hold raw text, scroll, or respond to `onPress` (it has no such prop —
use `Pressable`, lesson 5).

## Text — the only place text can live

The single most important rule: **every string must be inside a `<Text>`.**

```tsx
// ❌ Crashes: "Text strings must be rendered within a <Text> component."
<View>Hello</View>

// ✅
<View><Text>Hello</Text></View>
```

This also catches a classic conditional-rendering bug:

```tsx
const unread = 0;
// ❌ renders the number 0 as a bare string inside View → crash on native
<View>{unread && <Text>{unread} new</Text>}</View>

// ✅ make the condition a real boolean
<View>{unread > 0 && <Text>{unread} new</Text>}</View>
```

`Text` nests, and nested `Text` inherits style from its parent — the only place in React Native where
styles inherit at all:

```tsx
<Text style={{ fontSize: 16, color: '#334155' }}>
  Your streak is <Text style={{ fontWeight: '700', color: '#16a34a' }}>12 days</Text>. Keep going!
</Text>
```

Nested `Text` is laid out as one paragraph (inline spans), not as separate boxes. Useful props:

| Prop | Use |
|------|-----|
| `numberOfLines={2}` | Truncate with an ellipsis after two lines |
| `ellipsizeMode="middle"` | Where the `…` goes (`head`, `middle`, `tail`, `clip`) |
| `selectable` | Let users long-press to copy |
| `allowFontScaling={false}` | Opt out of system font size — almost never do this; see lesson L3-07 |

## Image — sizes are your job

```tsx
import { Image } from 'react-native';

// Local asset: bundled into the app, dimensions known at build time
<Image source={require('./assets/avatar.png')} />

// Remote image: you MUST give it a size
<Image
  source={{ uri: 'https://example.com/avatar.jpg' }}
  style={{ width: 96, height: 96, borderRadius: 48 }}
  resizeMode="cover"
/>
```

A remote image with no `width`/`height` (and no `flex` or `aspectRatio` from its parent) renders at
**0 × 0** — it downloads but you never see it. The browser figures out an `<img>`'s size after it
loads; React Native does not, because layout must be known before the image arrives.

`resizeMode` controls fitting: `cover` (fill and crop), `contain` (fit inside, letterbox),
`stretch`, `center`, `repeat`. For production apps, Level 2 introduces `expo-image`, which adds disk
caching, placeholders and transitions.

## ScrollView — scrolling is opt-in

A `View` never scrolls. If content can be taller than the screen, wrap it in `ScrollView`:

```tsx
<ScrollView contentContainerStyle={{ padding: 16, gap: 12 }}>
  {/* long content */}
</ScrollView>
```

Note the two style props: `style` applies to the scroll view's outer frame (the visible window), and
`contentContainerStyle` applies to the inner view that holds the children. Padding and `gap` almost
always belong in `contentContainerStyle`.

`ScrollView` renders **all** of its children immediately. That is fine for a settings screen; it is
wrong for a list of 1,000 items — use `FlatList` (lesson 7).

## Worked example: a profile card screen

```tsx title="ProfileScreen.tsx"
import { Image, ScrollView, StyleSheet, Text, View } from 'react-native';

type Stat = { label: string; value: number };

const stats: Stat[] = [
  { label: 'Habits', value: 6 },
  { label: 'Best streak', value: 41 },
  { label: 'Check-ins', value: 318 },
];

export default function ProfileScreen() {
  return (
    <ScrollView contentContainerStyle={styles.content}>
      <View style={styles.card}>
        <Image
          source={{ uri: 'https://i.pravatar.cc/192?img=12' }}
          style={styles.avatar}
          accessibilityIgnoresInvertColors
        />
        <Text style={styles.name} numberOfLines={1}>
          Priya Raman
        </Text>
        <Text style={styles.bio} numberOfLines={3}>
          Runner, reader, recovering snooze-button addict. Building one small habit at a time.
        </Text>
        <View style={styles.statsRow}>
          {stats.map((s) => (
            <View key={s.label} style={styles.stat}>
              <Text style={styles.statValue}>{s.value}</Text>
              <Text style={styles.statLabel}>{s.label}</Text>
            </View>
          ))}
        </View>
      </View>
      <Text style={styles.sectionTitle}>About</Text>
      <Text style={styles.paragraph}>
        Joined in <Text style={styles.bold}>March</Text>. Prefers morning reminders.
      </Text>
    </ScrollView>
  );
}

const styles = StyleSheet.create({
  content: { padding: 16, gap: 16 },
  card: { backgroundColor: 'white', borderRadius: 16, padding: 20, alignItems: 'center', gap: 8 },
  avatar: { width: 96, height: 96, borderRadius: 48, backgroundColor: '#e2e8f0' },
  name: { fontSize: 22, fontWeight: '700' },
  bio: { fontSize: 15, color: '#475569', textAlign: 'center' },
  statsRow: { flexDirection: 'row', alignSelf: 'stretch', justifyContent: 'space-around', marginTop: 8 },
  stat: { alignItems: 'center' },
  statValue: { fontSize: 20, fontWeight: '700' },
  statLabel: { fontSize: 12, color: '#64748b' },
  sectionTitle: { fontSize: 13, fontWeight: '600', color: '#64748b', textTransform: 'uppercase' },
  paragraph: { fontSize: 16, lineHeight: 24 },
  bold: { fontWeight: '700' },
});
```

The `backgroundColor` on the avatar is deliberate: while the remote image loads (or if it fails),
the user sees a neutral circle rather than a hole in the layout.

## How It Actually Works

Each core component is a thin JavaScript wrapper around a **host component** — a component type
that Fabric knows how to create natively. `View` becomes a host component named `View` (which iOS
backs with an `RCTViewComponentView` and Android with a `ReactViewGroup`). When React renders one,
Fabric creates a **shadow node** in C++ holding its props and style, Yoga lays it out, and only then is
the real platform view created or updated.

`Text` is special because platform text systems want **one** attributed string per paragraph, not a
tree of views. So Fabric flattens a `<Text>` and all its nested `<Text>` children into a single
attributed string (runs of characters with their own font, colour and weight), measures it with the
platform's text engine to get the paragraph's size for Yoga, and renders it in one native text
view. That is why nested text inherits style and flows inline, and why a `View` can't hold a
string — a `View` has no text engine.

Images are measured before they load for the same reason: Yoga computes the full layout
synchronously from styles. A remote image's pixel size isn't known until bytes arrive, so without an
explicit size its layout box is zero.

## Common mistakes

- **Bare strings or numbers in a `View`** — including `{count && ...}` when `count` is `0`.
- **Remote images without dimensions** — invisible, with no error.
- **Padding on `ScrollView`'s `style`** — the padding is applied to the frame, so the last item is
  cut off or the scroll indicator is inset strangely. Use `contentContainerStyle`.
- **`ScrollView` for long, dynamic lists** — mounts everything, wastes memory, slow to render.
- **Expecting `fontFamily` or `color` on a `View` to flow to its `Text` children.** Only nested
  `Text` inherits. Put typography on `Text`, or build a small `AppText` component (next lesson).

## Exercise

Build a "recipe card" screen: a hero image at the top (remote, 16:9 using `aspectRatio: 16 / 9` and
`width: '100%'`), a title truncated to two lines, a row of three small stats (time, servings,
difficulty), and a long ingredients paragraph inside a `ScrollView`. Make one ingredient bold using
nested `Text`. Then deliberately break it: remove the image size and render `{0 && <Text/>}` — read
both errors/behaviours on your phone so you recognise them later.
