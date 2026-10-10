---
description: "Render long lists with FlatList and SectionList: keyExtractor, separators, empty states, pull-to-refresh, infinite scroll and how virtualization works."
---

# 07 · Lists with FlatList & SectionList

Most mobile apps are lists: messages, transactions, contacts, search results, a feed. A phone has
limited memory, and mounting 2,000 native views for a list the user will mostly never scroll to is
how apps get slow and get killed by the OS. `FlatList` and `SectionList` render only what's near
the screen and recycle the rest — **virtualization** — and they come with pull-to-refresh, infinite
scroll and empty states built in.

## FlatList basics

```tsx
import { FlatList, Text, View } from 'react-native';

type Habit = { id: string; name: string; streak: number };

<FlatList
  data={habits}
  keyExtractor={(item) => item.id}
  renderItem={({ item }) => (
    <View style={{ padding: 16 }}>
      <Text>{item.name} — {item.streak} days</Text>
    </View>
  )}
/>
```

- `data` — an array (it must be a new array when contents change; mutating in place won't
  re-render).
- `renderItem` — receives `{ item, index, separators }`.
- `keyExtractor` — a stable, unique string per item. Don't use the index if items can be inserted,
  removed or reordered; React will reuse the wrong row's state.

Useful extras:

```tsx
<FlatList
  data={habits}
  keyExtractor={(h) => h.id}
  renderItem={renderHabit}
  ItemSeparatorComponent={() => <View style={{ height: 1, backgroundColor: '#e2e8f0' }} />}
  ListHeaderComponent={<Text style={{ fontSize: 28, fontWeight: '800', padding: 16 }}>Habits</Text>}
  ListEmptyComponent={<Text style={{ padding: 32, textAlign: 'center' }}>No habits yet — add one!</Text>}
  ListFooterComponent={loadingMore ? <ActivityIndicator style={{ margin: 16 }} /> : null}
  contentContainerStyle={{ paddingBottom: 32 }}
/>
```

Like `ScrollView`, `FlatList` needs a bounded height to scroll — give its parent `flex: 1`. And
never nest a vertical `FlatList` inside a vertical `ScrollView`: the outer scroll view gives it
unlimited height, so it renders everything and virtualization is lost (React Native logs a warning
about this). Put other content in `ListHeaderComponent` / `ListFooterComponent` instead.

## Pull-to-refresh and infinite scroll

```tsx
const [refreshing, setRefreshing] = useState(false);

async function onRefresh() {
  setRefreshing(true);
  try {
    await reload();
  } finally {
    setRefreshing(false);
  }
}

<FlatList
  data={items}
  renderItem={renderItem}
  keyExtractor={(i) => i.id}
  refreshing={refreshing}
  onRefresh={onRefresh}
  onEndReached={loadMore}
  onEndReachedThreshold={0.5}
/>
```

`onEndReachedThreshold={0.5}` means "call `onEndReached` when the end is within half a screen
height." `onEndReached` can fire more than once (for example, during a fast fling or when the list is
short), so guard it with a loading flag and a "has more" flag.

## SectionList for grouped data

```tsx
import { SectionList } from 'react-native';

type Section = { title: string; data: Habit[] };

const sections: Section[] = [
  { title: 'Morning', data: [{ id: '1', name: 'Stretch', streak: 4 }] },
  { title: 'Evening', data: [{ id: '2', name: 'Read', streak: 12 }, { id: '3', name: 'Journal', streak: 2 }] },
];

<SectionList
  sections={sections}
  keyExtractor={(item) => item.id}
  renderItem={({ item }) => <Text style={{ padding: 16 }}>{item.name}</Text>}
  renderSectionHeader={({ section }) => (
    <Text style={{ padding: 8, paddingHorizontal: 16, backgroundColor: '#f1f5f9', fontWeight: '700' }}>
      {section.title}
    </Text>
  )}
  stickySectionHeadersEnabled
/>
```

Sticky headers default to on for iOS and off for Android; set the prop explicitly for consistent
behaviour.

## Worked example: a searchable, refreshable list

```tsx title="HabitList.tsx"
import { memo, useCallback, useMemo, useState } from 'react';
import { FlatList, ListRenderItem, StyleSheet, Text, TextInput, View } from 'react-native';

type Habit = { id: string; name: string; streak: number };

const seed: Habit[] = Array.from({ length: 300 }, (_, i) => ({
  id: String(i + 1),
  name: `Habit #${i + 1}`,
  streak: (i * 7) % 31,
}));

const HabitRow = memo(function HabitRow({ habit }: { habit: Habit }) {
  return (
    <View style={styles.row}>
      <Text style={styles.name}>{habit.name}</Text>
      <Text style={styles.streak}>{habit.streak}🔥</Text>
    </View>
  );
});

export default function HabitList() {
  const [query, setQuery] = useState('');
  const [refreshing, setRefreshing] = useState(false);
  const [habits, setHabits] = useState(seed);

  const visible = useMemo(() => {
    const q = query.trim().toLowerCase();
    return q ? habits.filter((h) => h.name.toLowerCase().includes(q)) : habits;
  }, [habits, query]);

  const renderItem = useCallback<ListRenderItem<Habit>>(({ item }) => <HabitRow habit={item} />, []);

  const onRefresh = useCallback(() => {
    setRefreshing(true);
    // Simulate a reload: shuffle streaks so you can see the list change.
    setTimeout(() => {
      setHabits((prev) => prev.map((h) => ({ ...h, streak: (h.streak + 1) % 31 })));
      setRefreshing(false);
    }, 600);
  }, []);

  return (
    <View style={styles.screen}>
      <TextInput
        value={query}
        onChangeText={setQuery}
        placeholder="Search habits"
        autoCorrect={false}
        clearButtonMode="while-editing"
        style={styles.search}
      />
      <FlatList
        data={visible}
        keyExtractor={(h) => h.id}
        renderItem={renderItem}
        ItemSeparatorComponent={Separator}
        ListEmptyComponent={<Text style={styles.empty}>No habits match “{query}”.</Text>}
        refreshing={refreshing}
        onRefresh={onRefresh}
        keyboardDismissMode="on-drag"
        keyboardShouldPersistTaps="handled"
      />
    </View>
  );
}

function Separator() {
  return <View style={styles.separator} />;
}

const styles = StyleSheet.create({
  screen: { flex: 1, paddingTop: 56, backgroundColor: 'white' },
  search: { marginHorizontal: 16, marginBottom: 8, padding: 12, borderRadius: 10, backgroundColor: '#f1f5f9', fontSize: 16 },
  row: { flexDirection: 'row', justifyContent: 'space-between', paddingHorizontal: 16, paddingVertical: 14 },
  name: { fontSize: 16 },
  streak: { fontSize: 16, fontWeight: '700' },
  separator: { height: StyleSheet.hairlineWidth, backgroundColor: '#cbd5e1', marginLeft: 16 },
  empty: { padding: 32, textAlign: 'center', color: '#64748b' },
});
```

`memo` on the row plus a stable `renderItem` (via `useCallback`) means a row only re-renders when its
`habit` object changes. The refresh creates new objects for every habit, so every row updates —
that's correct. `keyboardDismissMode="on-drag"` hides the keyboard when the user starts scrolling,
and `keyboardShouldPersistTaps="handled"` lets taps on rows work while the keyboard is open.

## How It Actually Works

`FlatList` is built on `VirtualizedList`, which is built on `ScrollView`. As you scroll,
`VirtualizedList` tracks the scroll offset and the measured height of each rendered cell, and keeps a
**render window** — the range of item indexes to mount. By default the window covers about
`windowSize = 21` "screen heights" worth (10 above, 10 below, plus the visible one). Items outside
the window are replaced by empty spacer views of the right height, so the scroll bar and position
stay correct while the real rows are unmounted.

New items are rendered in batches (`maxToRenderPerBatch`, `initialNumToRender`) during idle time, so
a fast fling can briefly show blank space at the edge — the list is filling the window
asynchronously on the JS thread while the native scroll view keeps moving on the UI thread. Because
rows are measured after rendering, `FlatList` doesn't know heights in advance; if every row has the
same fixed height, `getItemLayout` lets you tell it, which makes `scrollToIndex` exact and saves
measurement work. Level 4 tunes all of this and compares Shopify's **FlashList**, which recycles row
views instead of unmounting them.

## Common mistakes

- **Index as key** in a list that changes — state (like an expanded row) jumps to the wrong item.
- **Mutating `data` in place** (`habits.push(x)`) — `FlatList` is a pure component and won't update.
- **`FlatList` inside a `ScrollView`** — virtualization disabled; use header/footer components.
- **Inline arrow `renderItem` with heavy rows** — every parent render re-renders every visible row.
- **Unguarded `onEndReached`** — double-fires and loads the same page twice.
- **No `flex: 1` on the parent** — the list doesn't scroll or is cut off.

## Exercise

Build a contacts screen with a `SectionList` grouped by first letter (generate 200 fake names).
Add: sticky headers on both platforms, a search box that filters while preserving sections (and
drops empty ones), pull-to-refresh, a `ListEmptyComponent`, and a tap that toggles a "favourite"
star on a row. Verify the favourite stays on the correct person after you search and clear the
search — if it doesn't, check your keys.
