---
description: "Make long React Native lists fast: FlatList props that matter, getItemLayout, light rows, then FlashList v2 recycling, item types and state."
---

# 02 · Big Lists: Tuning FlatList & Using FlashList

Lists are where React Native performance problems concentrate: they render the most components,
re-render the most often, and are scrolled — the interaction where users notice every dropped frame.
Level 1 introduced `FlatList`'s virtualization. This lesson makes lists fast: first by tuning
`FlatList` and the rows inside it, then by switching to **FlashList**, a list that recycles views
instead of mounting and unmounting them.

## First, fix the rows

No list setting rescues a slow row. Before touching list props:

1. **Memoise rows** and pass **primitive or stable props**:

```tsx
const HabitRow = memo(function HabitRow({ id, name, streak, done, onToggle }: RowProps) { /* … */ });

const renderItem = useCallback<ListRenderItem<Habit>>(
  ({ item }) => <HabitRow id={item.id} name={item.name} streak={item.streak} done={item.done} onToggle={toggle} />,
  [toggle],
);
```

2. **Keep rows shallow.** Each `View` is a native view. A row with eight nested wrappers is eight
   native views × every visible row.
3. **Size images** to the row (L2-08) and give them `recyclingKey` in recycling lists.
4. **No heavy work in render** — format dates and numbers when data arrives, not per row per render.
5. **Avoid inline functions that create new closures per row** if the row is memoised — pass the ID
   and a stable handler instead (`onToggle(id)`).

## FlatList props that matter

```tsx
<FlatList
  data={habits}
  keyExtractor={(h) => h.id}
  renderItem={renderItem}
  getItemLayout={(_, index) => ({ length: ROW_HEIGHT, offset: ROW_HEIGHT * index, index })}
  initialNumToRender={12}
  maxToRenderPerBatch={10}
  windowSize={11}
  removeClippedSubviews
/>
```

| Prop | What it does | Trade-off |
|------|--------------|-----------|
| `getItemLayout` | Tells the list each row's size and offset up front | Only possible with fixed (or computable) heights; skips measurement and makes `scrollToIndex` exact |
| `initialNumToRender` | Rows rendered in the first batch | Enough to fill the first screen — more slows initial render |
| `maxToRenderPerBatch` | Rows rendered per incremental batch while scrolling | Higher fills faster but blocks JS longer per batch |
| `windowSize` | How many screen-heights stay mounted (default 21) | Lower saves memory and work; too low shows blank areas on fast flings |
| `removeClippedSubviews` | Detaches off-screen native views | Saves UI work on long lists; can cause rendering glitches in some layouts — test |
| `updateCellsBatchingPeriod` | Delay between batches | Rarely needs changing |

Change one at a time and measure in a release build — the defaults are reasonable, and "optimised"
values copied from a blog post can make things worse for your rows.

## Why FlatList has a ceiling

`FlatList` virtualizes by **mounting** rows as they enter the render window and **unmounting** them as
they leave. Scrolling through 1,000 rows means creating and destroying roughly 1,000 sets of React
components and native views. Each creation is JavaScript work (render) plus native work (create views).
On fast scrolls the list can't keep up, and users see blank space where rows haven't rendered yet.

## FlashList: recycle instead of re-create

[FlashList](https://shopify.github.io/flash-list/) (by Shopify) keeps a small pool of row
components. When a row scrolls off one edge, its component is **reused** for a row entering from the
other edge — React just re-renders it with new props. The native views stay alive.

```bash
npx expo install @shopify/flash-list
```

It's designed as a near drop-in replacement:

```tsx
import { FlashList } from '@shopify/flash-list';

<FlashList
  data={habits}
  keyExtractor={(h) => h.id}
  renderItem={({ item }) => <HabitRow {...item} onToggle={toggle} />}
/>
```

!!! note "FlashList v2"
    The SDK 57 project this course was checked against installed FlashList **2.0**. Version 2 is
    built for the New Architecture and measures items itself, so the `estimatedItemSize` prop that
    v1 tutorials insist on is no longer needed. If you're reading older code or docs, that's the
    biggest difference you'll notice.

### Item types: don't recycle a header into a photo row

If your list mixes row layouts — section headers, text rows, photo rows — tell FlashList, so it only
recycles a cell into another cell of the same kind:

```tsx
type Item =
  | { kind: 'header'; id: string; title: string }
  | { kind: 'note'; id: string; text: string }
  | { kind: 'photo'; id: string; uri: string; caption: string };

<FlashList
  data={items}
  keyExtractor={(i) => i.id}
  getItemType={(item) => item.kind}
  renderItem={({ item }) => {
    switch (item.kind) {
      case 'header': return <SectionHeader title={item.title} />;
      case 'note': return <NoteRow text={item.text} />;
      case 'photo': return <PhotoRow uri={item.uri} caption={item.caption} />;
    }
  }}
/>
```

### The recycling gotcha: local state

Because a component instance is reused for different items, **local state survives recycling**. A row
with `const [expanded, setExpanded] = useState(false)` expanded for item 3 may appear expanded for
item 27 after scrolling. Options:

- Lift per-item UI state up, keyed by item ID (best for things that should persist).
- Use FlashList's `useRecyclingState`, which resets when the item changes:

```tsx
import { useRecyclingState } from '@shopify/flash-list';

function NoteRow({ id, text }: { id: string; text: string }) {
  const [expanded, setExpanded] = useRecyclingState(false, [id]); // resets when id changes
  return (
    <Pressable onPress={() => setExpanded((e) => !e)}>
      <Text numberOfLines={expanded ? undefined : 2}>{text}</Text>
    </Pressable>
  );
}
```

The same reasoning applies to images: without `recyclingKey={id}` on `expo-image`, a recycled row can
flash the previous item's picture.

## Worked example: a 5,000-row check-in log

```tsx title="app/log.tsx"
import { memo, useCallback, useMemo } from 'react';
import { StyleSheet, Text, View } from 'react-native';
import { FlashList, ListRenderItem } from '@shopify/flash-list';

type Row =
  | { kind: 'day'; id: string; label: string }
  | { kind: 'checkin'; id: string; habit: string; time: string };

const fmt = new Intl.DateTimeFormat(undefined, { weekday: 'short', month: 'short', day: 'numeric' });

function buildRows(count: number): Row[] {
  const rows: Row[] = [];
  const start = new Date(2026, 0, 1);
  for (let d = 0; d < count / 5; d++) {
    const date = new Date(start.getFullYear(), start.getMonth(), start.getDate() + d);
    rows.push({ kind: 'day', id: `d${d}`, label: fmt.format(date) });   // format once, here
    for (let i = 0; i < 4; i++) rows.push({ kind: 'checkin', id: `d${d}-${i}`, habit: `Habit ${i + 1}`, time: `${7 + i * 3}:00` });
  }
  return rows;
}

const DayHeader = memo(({ label }: { label: string }) => <Text style={styles.day}>{label}</Text>);
const CheckinRow = memo(({ habit, time }: { habit: string; time: string }) => (
  <View style={styles.row}><Text style={styles.habit}>{habit}</Text><Text style={styles.time}>{time}</Text></View>
));

export default function Log() {
  const rows = useMemo(() => buildRows(5000), []);
  const renderItem = useCallback<ListRenderItem<Row>>(({ item }) =>
    item.kind === 'day' ? <DayHeader label={item.label} /> : <CheckinRow habit={item.habit} time={item.time} />, []);

  return (
    <FlashList
      data={rows}
      keyExtractor={(r) => r.id}
      getItemType={(r) => r.kind}
      renderItem={renderItem}
    />
  );
}

const styles = StyleSheet.create({
  day: { paddingHorizontal: 16, paddingTop: 20, paddingBottom: 6, fontWeight: '700', color: '#475569', backgroundColor: '#f8fafc' },
  row: { flexDirection: 'row', justifyContent: 'space-between', paddingHorizontal: 16, paddingVertical: 12 },
  habit: { fontSize: 16 },
  time: { color: '#64748b', fontVariant: ['tabular-nums'] },
});
```

To compare, swap `FlashList` for `FlatList` (removing `getItemType`), build both in release mode,
and fling through the list on a mid-range Android phone with the perf monitor open. Record what
*you* observe — blank areas, JS frame rate — rather than relying on anyone's published benchmark.

## How It Actually Works

`FlatList` computes a window of indexes around the viewport and renders exactly those items; items
leaving the window are unmounted (their React subtree and native views destroyed) and replaced with
spacer views. Its work scales with the number of items scrolled past.

FlashList v2 runs a layout manager that tracks each item's measured size and position and decides
which indexes are visible plus a `drawDistance` buffer. It keeps a pool of cell containers per item
type. When an index becomes visible, FlashList takes a free container of the right type — usually one
that just scrolled out of view — and renders the new item into it, so React reconciles a props change
on an existing subtree instead of mounting a new one, and Fabric updates existing native views instead
of creating new ones. Its work scales with the number of *visible* items, which is why it holds up
better on long, fast scrolls — and why local state and images must be recycling-aware.

## Common mistakes

- **Tuning list props before fixing slow rows.**
- **`getItemLayout` with wrong heights** — jumpy scrolling and incorrect `scrollToIndex`.
- **Very low `windowSize`** — blank areas on fast scroll.
- **Mixed row types in FlashList without `getItemType`.**
- **Local state in recycled rows** without `useRecyclingState` or lifting it.
- **Images without `recyclingKey`** in FlashList.
- **Copying `estimatedItemSize` advice from FlashList v1 tutorials** into v2 code.

## Exercise

1. Build the 5,000-row log with `FlatList` first. Add `getItemLayout` (headers and rows will need
   different heights — compute offsets from a precomputed array) and measure the difference.
2. Switch to FlashList with `getItemType`. Measure again in release mode on the same device.
3. Add an expandable note row with local state, reproduce the recycling bug by scrolling, then fix it
   with `useRecyclingState`.
4. Write a short note for your team: which list would you use for (a) a 30-item settings screen,
   (b) a 10,000-message chat, (c) a photo grid — and why.
