---
description: "Project: build a complete habit tracker in Expo — add habits, check them off daily, compute streaks with a tested pure function, safe areas and dark mode."
---

# 10 · Project — Habit Tracker

Time to combine Level 1 into one app. You'll build a **habit tracker**: a list of habits, a form to
add one, a tap to mark today done, a long press to delete, and a streak counter for each habit. It
uses every component from this level, respects safe areas and dark mode, and keeps its logic in
pure functions so it can be tested later.

State lives in memory for now; Level 2 adds navigation and persistence and turns this into a
multi-screen app.

## Requirements

- Show habits in a `FlatList`, each with its name, current streak and a check button for today.
- Add a habit with a validated `TextInput` (2–40 characters, no duplicates, case-insensitive).
- Tap to toggle today; long-press to confirm and delete (`Alert`).
- Streak = number of consecutive days ending today (or yesterday, if today isn't done yet).
- A header shows "X of Y done today".
- Safe-area correct, works in light and dark mode, sensible empty state.

## Project structure

```
habit-app/
├── App.tsx
├── src/
│   ├── habits.ts        # types + pure logic (streaks, toggling, validation)
│   ├── theme.ts         # colours and spacing
│   ├── HabitRow.tsx
│   └── AddHabitBar.tsx
```

## Step 1 — the pure logic

Dates are stored as local `YYYY-MM-DD` strings. That avoids time-zone surprises from comparing
`Date` objects created at different times of day.

```ts title="src/habits.ts"
export type Habit = {
  id: string;
  name: string;
  createdAt: string;      // YYYY-MM-DD
  doneDates: string[];    // YYYY-MM-DD, unsorted, unique
};

export function dayKey(d: Date): string {
  const y = d.getFullYear();
  const m = String(d.getMonth() + 1).padStart(2, '0');
  const day = String(d.getDate()).padStart(2, '0');
  return `${y}-${m}-${day}`;
}

export function addDays(key: string, delta: number): string {
  const [y, m, d] = key.split('-').map(Number);
  return dayKey(new Date(y, m - 1, d + delta));
}

/** Consecutive done days ending today, or ending yesterday if today isn't done yet. */
export function currentStreak(doneDates: string[], today: string): number {
  const done = new Set(doneDates);
  let cursor = done.has(today) ? today : addDays(today, -1);
  let streak = 0;
  while (done.has(cursor)) {
    streak += 1;
    cursor = addDays(cursor, -1);
  }
  return streak;
}

export function toggleDay(habit: Habit, day: string): Habit {
  const has = habit.doneDates.includes(day);
  return {
    ...habit,
    doneDates: has ? habit.doneDates.filter((d) => d !== day) : [...habit.doneDates, day],
  };
}

export function validateHabitName(name: string, existing: Habit[]): string | null {
  const trimmed = name.trim();
  if (trimmed.length < 2) return 'Use at least 2 characters.';
  if (trimmed.length > 40) return 'Keep it under 40 characters.';
  if (existing.some((h) => h.name.toLowerCase() === trimmed.toLowerCase())) return 'You already track that habit.';
  return null;
}

let counter = 0;
export function newId(): string {
  counter += 1;
  return `${Date.now().toString(36)}-${counter}`;
}
```

`addDays` builds a new `Date` from year, month and day parts. JavaScript's `Date` constructor
normalises overflow (day 0 becomes the last day of the previous month), so month and year boundaries
work without special cases. Building it from local parts — instead of adding 86,400,000 ms — also
avoids the daylight-saving bug where one "day" is 23 or 25 hours long.

These functions were run under Jest in an Expo SDK 57 project; the checks below passed:

```ts title="src/habits.test.ts"
import { addDays, currentStreak, validateHabitName } from './habits';

test('addDays crosses month and year boundaries', () => {
  expect(addDays('2026-03-01', -1)).toBe('2026-02-28');
  expect(addDays('2024-03-01', -1)).toBe('2024-02-29'); // leap year
  expect(addDays('2026-12-31', 1)).toBe('2027-01-01');
});

test('streak counts back from today, or from yesterday if today is not done', () => {
  const today = '2026-10-10';
  expect(currentStreak(['2026-10-10', '2026-10-09', '2026-10-08'], today)).toBe(3);
  expect(currentStreak(['2026-10-09', '2026-10-08'], today)).toBe(2);
  expect(currentStreak(['2026-10-10', '2026-10-08'], today)).toBe(1);
  expect(currentStreak([], today)).toBe(0);
});

test('validation rejects short names and case-insensitive duplicates', () => {
  const existing = [{ id: '1', name: 'Read', createdAt: '2026-10-01', doneDates: [] }];
  expect(validateHabitName(' a ', existing)).toMatch(/at least 2/);
  expect(validateHabitName('read', existing)).toMatch(/already/);
  expect(validateHabitName('Walk', existing)).toBeNull();
});
```

## Step 2 — theme

```ts title="src/theme.ts"
import { useColorScheme } from 'react-native';

const palettes = {
  light: { bg: '#f8fafc', surface: '#ffffff', text: '#0f172a', muted: '#64748b', border: '#e2e8f0', primary: '#4f46e5', success: '#16a34a' },
  dark:  { bg: '#020617', surface: '#0f172a', text: '#f1f5f9', muted: '#94a3b8', border: '#1e293b', primary: '#818cf8', success: '#4ade80' },
};
export type Palette = typeof palettes.light;
export const useTheme = (): Palette => (useColorScheme() === 'dark' ? palettes.dark : palettes.light);
```

## Step 3 — a row

```tsx title="src/HabitRow.tsx"
import { memo } from 'react';
import { Pressable, StyleSheet, Text, View } from 'react-native';
import { Palette } from './theme';

type Props = {
  id: string;
  name: string;
  streak: number;
  doneToday: boolean;
  colors: Palette;
  onToggle: (id: string) => void;
  onDelete: (id: string) => void;
};

export const HabitRow = memo(function HabitRow({ id, name, streak, doneToday, colors, onToggle, onDelete }: Props) {
  return (
    <Pressable
      onPress={() => onToggle(id)}
      onLongPress={() => onDelete(id)}
      android_ripple={{ color: colors.border }}
      accessibilityRole="checkbox"
      accessibilityState={{ checked: doneToday }}
      accessibilityLabel={`${name}, ${streak} day streak`}
      accessibilityHint="Double tap to toggle today. Long press to delete."
      style={({ pressed }) => [styles.row, { backgroundColor: colors.surface, opacity: pressed ? 0.85 : 1 }]}
    >
      <View style={styles.texts}>
        <Text style={[styles.name, { color: colors.text }]} numberOfLines={1}>{name}</Text>
        <Text style={{ color: colors.muted }}>{streak > 0 ? `${streak}-day streak` : 'Start today'}</Text>
      </View>
      <View style={[styles.check, { borderColor: doneToday ? colors.success : colors.border, backgroundColor: doneToday ? colors.success : 'transparent' }]}>
        {doneToday && <Text style={styles.tick}>✓</Text>}
      </View>
    </Pressable>
  );
});

const styles = StyleSheet.create({
  row: { flexDirection: 'row', alignItems: 'center', gap: 12, padding: 16, borderRadius: 14, overflow: 'hidden' },
  texts: { flex: 1, gap: 2 },
  name: { fontSize: 17, fontWeight: '600' },
  check: { width: 30, height: 30, borderRadius: 15, borderWidth: 2, alignItems: 'center', justifyContent: 'center' },
  tick: { color: 'white', fontWeight: '800', fontSize: 16 },
});
```

The row receives primitives and stable callbacks so `memo` can skip re-rendering rows whose data
didn't change.

## Step 4 — the add bar

```tsx title="src/AddHabitBar.tsx"
import { useState } from 'react';
import { Pressable, StyleSheet, Text, TextInput, View } from 'react-native';
import { Palette } from './theme';

type Props = { colors: Palette; bottomInset: number; validate: (name: string) => string | null; onAdd: (name: string) => void };

export function AddHabitBar({ colors, bottomInset, validate, onAdd }: Props) {
  const [name, setName] = useState('');
  const [error, setError] = useState<string | null>(null);

  function submit() {
    const problem = validate(name);
    setError(problem);
    if (problem) return;
    onAdd(name.trim());
    setName('');
  }

  return (
    <View style={[styles.bar, { backgroundColor: colors.surface, borderTopColor: colors.border, paddingBottom: Math.max(bottomInset, 12) }]}>
      {error && <Text style={styles.error}>{error}</Text>}
      <View style={styles.inputRow}>
        <TextInput
          value={name}
          onChangeText={(t) => { setName(t); if (error) setError(null); }}
          placeholder="New habit"
          placeholderTextColor={colors.muted}
          returnKeyType="done"
          onSubmitEditing={submit}
          maxLength={40}
          accessibilityLabel="New habit name"
          style={[styles.input, { color: colors.text, borderColor: error ? '#dc2626' : colors.border }]}
        />
        <Pressable onPress={submit} accessibilityRole="button" style={({ pressed }) => [styles.add, { backgroundColor: colors.primary, opacity: pressed ? 0.8 : 1 }]}>
          <Text style={styles.addText}>Add</Text>
        </Pressable>
      </View>
    </View>
  );
}

const styles = StyleSheet.create({
  bar: { paddingHorizontal: 16, paddingTop: 12, borderTopWidth: StyleSheet.hairlineWidth, gap: 6 },
  inputRow: { flexDirection: 'row', gap: 8 },
  input: { flex: 1, borderWidth: 1, borderRadius: 10, paddingHorizontal: 12, paddingVertical: 10, fontSize: 16 },
  add: { paddingHorizontal: 18, borderRadius: 10, justifyContent: 'center', minHeight: 44 },
  addText: { color: 'white', fontWeight: '700', fontSize: 16 },
  error: { color: '#dc2626', fontSize: 13 },
});
```

## Step 5 — the screen

```tsx title="App.tsx"
import { useCallback, useMemo, useState } from 'react';
import { Alert, FlatList, StyleSheet, Text, View } from 'react-native';
import { StatusBar } from 'expo-status-bar';
import { SafeAreaProvider, useSafeAreaInsets } from 'react-native-safe-area-context';
import { currentStreak, dayKey, Habit, newId, toggleDay, validateHabitName } from './src/habits';
import { HabitRow } from './src/HabitRow';
import { AddHabitBar } from './src/AddHabitBar';
import { useTheme } from './src/theme';

function HabitScreen() {
  const colors = useTheme();
  const insets = useSafeAreaInsets();
  const today = dayKey(new Date());
  const [habits, setHabits] = useState<Habit[]>([
    { id: 'seed-1', name: 'Drink 2L of water', createdAt: today, doneDates: [] },
  ]);

  const doneCount = useMemo(() => habits.filter((h) => h.doneDates.includes(today)).length, [habits, today]);

  const onToggle = useCallback((id: string) => {
    setHabits((prev) => prev.map((h) => (h.id === id ? toggleDay(h, today) : h)));
  }, [today]);

  const onDelete = useCallback((id: string) => {
    Alert.alert('Delete habit?', 'Its history will be lost.', [
      { text: 'Cancel', style: 'cancel' },
      { text: 'Delete', style: 'destructive', onPress: () => setHabits((prev) => prev.filter((h) => h.id !== id)) },
    ]);
  }, []);

  const onAdd = useCallback((name: string) => {
    setHabits((prev) => [...prev, { id: newId(), name, createdAt: today, doneDates: [] }]);
  }, [today]);

  return (
    <View style={[styles.screen, { backgroundColor: colors.bg }]}>
      <View style={{ paddingTop: insets.top + 12, paddingHorizontal: 16, paddingBottom: 8 }}>
        <Text style={[styles.title, { color: colors.text }]}>Today</Text>
        <Text style={{ color: colors.muted }} accessibilityLiveRegion="polite">
          {doneCount} of {habits.length} done
        </Text>
      </View>
      <FlatList
        data={habits}
        keyExtractor={(h) => h.id}
        contentContainerStyle={styles.list}
        keyboardShouldPersistTaps="handled"
        renderItem={({ item }) => (
          <HabitRow
            id={item.id}
            name={item.name}
            streak={currentStreak(item.doneDates, today)}
            doneToday={item.doneDates.includes(today)}
            colors={colors}
            onToggle={onToggle}
            onDelete={onDelete}
          />
        )}
        ListEmptyComponent={<Text style={[styles.empty, { color: colors.muted }]}>No habits yet. Add your first one below.</Text>}
      />
      <AddHabitBar colors={colors} bottomInset={insets.bottom} validate={(n) => validateHabitName(n, habits)} onAdd={onAdd} />
      <StatusBar style="auto" />
    </View>
  );
}

export default function App() {
  return (
    <SafeAreaProvider>
      <HabitScreen />
    </SafeAreaProvider>
  );
}

const styles = StyleSheet.create({
  screen: { flex: 1 },
  title: { fontSize: 32, fontWeight: '800' },
  list: { padding: 16, gap: 10, flexGrow: 1 },
  empty: { textAlign: 'center', marginTop: 48, fontSize: 16 },
});
```

Run it with `npx expo start`. You'll notice one rough edge on purpose: when the keyboard opens, it
covers the add bar on some devices. That's the subject of Level 2, lesson 9 — note it, don't hack
around it with a fixed margin.

## How It Actually Works

Follow one tap through the app. The touch reaches `Pressable` through the responder system; on
release, `onPress` calls `onToggle(id)`. The functional `setHabits` update produces a **new array**
in which only the toggled habit is a new object (the others keep their identity). React re-renders
`HabitScreen`; `FlatList` receives new `data` and re-runs `renderItem` for visible rows. Each
`HabitRow` is memoised: rows whose `streak` and `doneToday` primitives didn't change, and whose
callbacks are the same `useCallback` references, skip rendering entirely. Only the toggled row
produces new host-component props, so Fabric sends a handful of mutations (a background colour, a
border colour, a new tick `Text`) to the native views.

The streak is computed during render from `doneDates` rather than stored. Derived data that's
cheap to compute shouldn't be duplicated in state — duplicating it is how "the streak says 4 but
the history says 3" bugs happen.

## Common mistakes

- **Storing `Date` objects and comparing with `===`** — two `Date`s for the same day are different
  objects. Use day keys.
- **Adding 24 hours to move a day** — breaks on daylight-saving transitions.
- **Inline callbacks into memoised rows** — defeats `memo`.
- **Storing the streak in state** and forgetting to update it on un-toggle.
- **Hard-coding today at module load** — the app left open past midnight shows yesterday. (A
  production version recomputes `today` when the app returns to the foreground; Level 3 uses
  `AppState` for this.)

## Exercise

Extend the tracker:

1. Add a **7-day strip** under each habit name: seven small circles for the last seven days, filled
   if done. Compute the keys with `addDays`.
2. Add a **"best streak"** function — the longest run anywhere in the history — and unit-test it
   with at least four cases including an empty history and a gap.
3. Let users **reorder** habits by long-pressing → "Move up" / "Move down" in the `Alert` instead of
   delete-only.
4. Make sure everything still works with your phone's largest text size and in dark mode.
