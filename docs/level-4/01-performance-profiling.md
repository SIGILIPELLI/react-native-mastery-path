---
description: "Find and fix React Native jank: measure in release mode, read the performance monitor, profile renders and Hermes CPU samples, and fix the usual culprits."
---

# 01 · Performance: Finding & Fixing Jank

"The app feels slow" is not a bug report you can fix. Performance work in React Native is a loop:
**measure** on a realistic device, **find** which thread is overloaded and why, **fix** the cause,
**measure again**. Most wasted effort comes from skipping the first step — memoising everything,
rewriting lists, or adding native code based on a guess.

## Rule zero: measure a release build on a slow device

Development builds are much slower than release builds: JavaScript runs with extra checks, warnings
and development-only code paths, and Metro serves an un-minified bundle. A screen that stutters in
development may be smooth in release — and a problem you "fixed" in development may never have
existed. So:

- Test performance with a **release** build: `npx expo run:android --variant release`,
  `npx expo run:ios --configuration Release`, or an EAS preview build.
- Test on a **low-to-mid-range Android phone**, not only on the latest iPhone. That's where most real
  performance problems live.

## Step 1 — which thread?

Open the dev menu → **Perf Monitor** (in a development build), or use platform tools in release:

- **JS frame rate dropping, UI fine** → JavaScript is doing too much: heavy renders, expensive
  computation in render, big JSON parsing, too many re-renders. Taps feel delayed; JS-driven
  animations stutter; scrolling usually stays smooth.
- **UI frame rate dropping** → the native side is overloaded: too many views, expensive shadows or
  transparency, huge images being decoded, layout thrashing. Scrolling itself stutters.

That single observation tells you which tools to reach for next.

## Step 2a — JavaScript: what's rendering, and why?

**React DevTools Profiler** (in React Native DevTools, press `j` in the Expo terminal → Profiler tab):
record an interaction, then look at the flame graph for components that render often or take long.
Enable "Record why each component rendered" to see whether it was props, state, context or a parent.

Common findings and fixes:

| Finding | Fix |
|---------|-----|
| A whole list re-renders when one item changes | `memo` the row, stable `renderItem`/callbacks (`useCallback`), keep item identity stable |
| Every screen re-renders on any store change | Selectors (Zustand) or split contexts (L2-04) |
| Expensive derived data recomputed every render | `useMemo`, or compute once when data arrives |
| A component re-renders on every keystroke elsewhere | Move the input's state down into the input component |
| Inline objects/arrays passed to memoised children | Hoist constants or memoise |

**React Compiler.** Expo projects can enable the React Compiler, which inserts memoisation
automatically at build time. It reduces the need for hand-written `useMemo`/`useCallback`, but it's
not a substitute for measuring — check your SDK's docs for its status and how to enable it.

## Step 2b — JavaScript: where is the CPU going?

If renders aren't the problem but the JS thread is still busy, take a **CPU profile** of Hermes. In
React Native DevTools, the **Performance** panel records a sampling profile of JavaScript execution:
start recording, perform the slow interaction, stop, and read the flame chart from the top down,
looking for wide bars — functions that took a long time, including their callees.

Typical CPU hogs:

- Sorting/filtering thousands of items in render — do it once, memoise, or move it to SQLite
  (`ORDER BY`, `WHERE`) where it belongs.
- `JSON.parse` of a multi-megabyte response — request less data, paginate.
- Date/number formatting in tight loops — create `Intl` formatters once, reuse them.
- `console.log` in hot paths during development.

## Step 2c — the UI thread

For native-side problems, use the platform profilers:

- **Android Studio Profiler** (CPU and memory) and the developer option **Profile HWUI rendering**,
  which draws per-frame bars on screen.
- **Xcode Instruments** — *Time Profiler* and *Animation Hitches* — on iOS.

Typical UI-thread culprits:

- **Deep or wide view trees** — many nested `View`s per list row. Flatten wrappers that exist only for
  styling.
- **Large images** decoded at full resolution (L2-08) — request sized images.
- **Shadows and opacity on many views**, especially animated — rasterise or simplify.
- **Animating layout properties** (width/height/top) instead of transforms (L3-05).

## Worked example: a slow habit history screen

Symptom: on a mid-range Android phone, typing in the search box on the History screen (1,500
check-ins) lags behind the keyboard.

1. **Perf monitor:** JS frame rate falls sharply while typing; UI frame rate is normal. → JavaScript.
2. **Profiler:** each keystroke re-renders `HistoryScreen` and all ~30 visible `DayRow`s; each render of
   `HistoryScreen` spends most of its time in a function called `groupByWeek`.
3. **Read the code:**

```tsx
// Before
function HistoryScreen({ checkins }: { checkins: Checkin[] }) {
  const [query, setQuery] = useState('');
  const weeks = groupByWeek(checkins.filter((c) => c.habitName.toLowerCase().includes(query.toLowerCase())));
  return (
    <>
      <TextInput value={query} onChangeText={setQuery} />
      <FlatList data={weeks} renderItem={({ item }) => <WeekRow week={item} onPress={() => open(item.id)} />} />
    </>
  );
}
```

Three problems: grouping 1,500 items on every keystroke; a new `renderItem` and a new `onPress`
closure each render, defeating any memoisation of `WeekRow`; and filtering on the raw query
immediately, with no debounce.

```tsx
// After
function HistoryScreen({ checkins }: { checkins: Checkin[] }) {
  const [query, setQuery] = useState('');
  const deferredQuery = useDeferredValue(query);            // keep typing responsive

  const weeks = useMemo(() => {
    const q = deferredQuery.trim().toLowerCase();
    const filtered = q ? checkins.filter((c) => c.habitName.toLowerCase().includes(q)) : checkins;
    return groupByWeek(filtered);
  }, [checkins, deferredQuery]);

  const renderItem = useCallback<ListRenderItem<Week>>(({ item }) => <WeekRow week={item} onOpen={open} />, []);

  return (
    <>
      <TextInput value={query} onChangeText={setQuery} />
      <FlatList data={weeks} keyExtractor={(w) => w.id} renderItem={renderItem} />
    </>
  );
}

const WeekRow = memo(function WeekRow({ week, onOpen }: { week: Week; onOpen: (id: string) => void }) {
  return <Pressable onPress={() => onOpen(week.id)}>{/* … */}</Pressable>;
});
```

`useDeferredValue` lets React render the input's new value immediately and recompute the expensive
list in a lower-priority render that can be interrupted by the next keystroke. `useMemo` avoids
regrouping when nothing relevant changed, and the stable `renderItem` plus `memo` stops unchanged
rows re-rendering.

4. **Measure again** on the same phone, in a release build. Only call it fixed if the measurement
   says so — and note the numbers *you* observed, on *your* device.

## How It Actually Works

Each frame, the UI thread has a fixed budget (about 16.7 ms at 60 Hz, less at 90 or 120 Hz) to process
input, run native animations, apply any mutations Fabric has committed, lay out and draw. JavaScript
work doesn't directly drop UI frames — the threads are separate — but it delays everything that
*needs* JavaScript: responding to touches, committing new React trees, JS-driven animations. React
renders are synchronous chunks of JS work; concurrent features (`useDeferredValue`,
`startTransition`) let React split low-priority renders into interruptible pieces so urgent updates,
like an input's text, aren't stuck behind them.

Hermes's sampling profiler works by interrupting the JS thread at a regular interval and recording
the current call stack; functions that appear in many samples are where time is spent. That's why
wide bars in the flame chart, not tall ones, are what matter.

## Common mistakes

- **Profiling development builds** and optimising things that are fast in release.
- **Testing only on a flagship phone.**
- **Memoising everything blindly** — it adds complexity and memory, and can hide the real cause.
- **Ignoring which thread is slow** — JS fixes don't help a UI-thread problem and vice versa.
- **Not re-measuring** after a change.
- **Quoting numbers from someone else's device** in a bug report or PR — measure your own.

## Exercise

1. Create a screen that renders 2,000 fake check-ins grouped by week with a search box (copy the
   "before" version). Make a release build and try it on the slowest phone you have access to.
2. Use the perf monitor to decide which thread is struggling, then the React profiler to find the
   expensive component. Write down what you saw.
3. Apply the "after" changes one at a time, re-measuring after each, and note which change made the
   biggest difference on your device.
4. Record a Hermes CPU profile of the slow version and identify `groupByWeek` in the flame chart.
