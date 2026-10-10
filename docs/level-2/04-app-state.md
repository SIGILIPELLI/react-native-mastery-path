---
description: "Decide where state lives in React Native, share it with Context and useReducer, then use Zustand selectors to stop mounted screens re-rendering."
---

# 04 · App State: Context, useReducer & Zustand

Once you have several screens, state has to be shared: the habit you toggle on the Today tab should
update the Stats tab and the detail screen. On mobile this matters more than on the web because of
something you learned in lesson 1 — **screens stay mounted**. A state change that re-renders "every
consumer" re-renders screens the user can't even see. This lesson is about putting each piece of
state in the right place and subscribing to exactly what each screen needs.

## Four kinds of state

| Kind | Examples | Where it lives |
|------|----------|----------------|
| **Local UI state** | Is this row expanded? Text in a search box | `useState` in the component |
| **Shared client state** | Habits and check-ins, current user's settings, cart | A store (Context + reducer, or Zustand) |
| **Server state** | Data owned by a backend: recipes, profile, feed | A cache (TanStack Query, lesson 5) |
| **Navigation state** | Which screen, which params | The router (lessons 1–3) |

Most "state management is hard" pain comes from mixing these: server data copied into a global
store and then going stale, or a search box's text living in a global store and re-rendering the
whole app on every keystroke. Keep each kind in its own place, and most of the difficulty goes
away.

## Context + useReducer: the built-in option

A reducer puts all the ways state can change in one pure function:

```ts title="src/habitsReducer.ts"
export type Habit = { id: string; name: string; doneDates: string[]; archived: boolean };

export type Action =
  | { type: 'added'; id: string; name: string }
  | { type: 'toggled'; id: string; day: string }
  | { type: 'renamed'; id: string; name: string }
  | { type: 'archived'; id: string };

export function habitsReducer(state: Habit[], action: Action): Habit[] {
  switch (action.type) {
    case 'added':
      return [...state, { id: action.id, name: action.name, doneDates: [], archived: false }];
    case 'toggled':
      return state.map((h) =>
        h.id !== action.id
          ? h
          : {
              ...h,
              doneDates: h.doneDates.includes(action.day)
                ? h.doneDates.filter((d) => d !== action.day)
                : [...h.doneDates, action.day],
            },
      );
    case 'renamed':
      return state.map((h) => (h.id === action.id ? { ...h, name: action.name } : h));
    case 'archived':
      return state.map((h) => (h.id === action.id ? { ...h, archived: true } : h));
  }
}
```

Provide it through Context, in the root layout so every screen can reach it:

```tsx title="src/HabitsProvider.tsx"
import { createContext, ReactNode, useContext, useReducer, Dispatch } from 'react';
import { Action, Habit, habitsReducer } from './habitsReducer';

const HabitsState = createContext<Habit[] | null>(null);
const HabitsDispatch = createContext<Dispatch<Action> | null>(null);

export function HabitsProvider({ children }: { children: ReactNode }) {
  const [habits, dispatch] = useReducer(habitsReducer, []);
  return (
    <HabitsDispatch.Provider value={dispatch}>
      <HabitsState.Provider value={habits}>{children}</HabitsState.Provider>
    </HabitsDispatch.Provider>
  );
}

export function useHabits() {
  const v = useContext(HabitsState);
  if (!v) throw new Error('useHabits must be used inside HabitsProvider');
  return v;
}

export function useHabitsDispatch() {
  const v = useContext(HabitsDispatch);
  if (!v) throw new Error('useHabitsDispatch must be used inside HabitsProvider');
  return v;
}
```

Splitting state and dispatch into **two contexts** means components that only dispatch (an "Add"
button) never re-render when habits change — `dispatch` is stable for the provider's lifetime.

**The limitation:** every component that calls `useHabits()` re-renders on *every* change to the
habits array, even if it only displays one habit's name. With a Today tab, a Stats tab, a detail
screen and a settings screen all mounted, toggling one checkbox re-renders all four. For a small
app, that's fine. When it isn't, you want **selectors**.

## Zustand: a store with selectors

Zustand is a small state library (one dependency-free package) whose key feature is that components
subscribe to a **slice** of the store and re-render only when that slice changes.

```bash
npx expo install zustand
```

```ts title="src/useHabitStore.ts"
import { create } from 'zustand';
import { Action, Habit, habitsReducer } from './habitsReducer';

type HabitStore = {
  habits: Habit[];
  dispatch: (action: Action) => void;
};

export const useHabitStore = create<HabitStore>()((set) => ({
  habits: [],
  dispatch: (action) => set((s) => ({ habits: habitsReducer(s.habits, action) })),
}));
```

The same reducer is reused — it's just a pure function. Now each component selects what it needs:

```tsx
// Re-renders only when THIS habit's name changes
function HabitTitle({ id }: { id: string }) {
  const name = useHabitStore((s) => s.habits.find((h) => h.id === id)?.name);
  return <Text>{name}</Text>;
}

// Re-renders only when the count changes
function ActiveCount() {
  const count = useHabitStore((s) => s.habits.filter((h) => !h.archived).length);
  return <Text>{count} active habits</Text>;
}

// Never re-renders because of habit changes: dispatch is stable
function AddButton() {
  const dispatch = useHabitStore((s) => s.dispatch);
  return <Button title="Add" onPress={() => dispatch({ type: 'added', id: String(Date.now()), name: 'New' })} />;
}
```

### Selecting several values

A selector that builds a **new object or array** every time will re-render on every store change,
because the result is never `===` the previous one. Zustand provides `useShallow` to compare
shallowly instead:

```tsx
import { useShallow } from 'zustand/react/shallow';

const activeIds = useHabitStore(useShallow((s) => s.habits.filter((h) => !h.archived).map((h) => h.id)));
```

`activeIds` is a new array each time, but `useShallow` compares its elements, so the component only
re-renders when the list of IDs actually changes — not when a habit is toggled.

### Reading state outside React

Zustand stores are usable outside components, which is handy in notification handlers or navigation
callbacks:

```ts
const { habits } = useHabitStore.getState();
useHabitStore.getState().dispatch({ type: 'archived', id: '42' });
```

## Worked example: the reducer under test

Because the reducer is pure, it can be tested without rendering anything. These tests were run with
Jest in this course's SDK 57 check project and passed:

```ts title="src/habitsReducer.test.ts"
import { habitsReducer } from './habitsReducer';
import { useHabitStore } from './useHabitStore';

test('toggling twice returns to the original done state', () => {
  let s = habitsReducer([], { type: 'added', id: 'a', name: 'Read' });
  s = habitsReducer(s, { type: 'toggled', id: 'a', day: '2026-10-10' });
  expect(s[0].doneDates).toEqual(['2026-10-10']);
  s = habitsReducer(s, { type: 'toggled', id: 'a', day: '2026-10-10' });
  expect(s[0].doneDates).toEqual([]);
});

test('untouched habits keep their identity (cheap memo checks)', () => {
  let s = habitsReducer([], { type: 'added', id: 'a', name: 'Read' });
  s = habitsReducer(s, { type: 'added', id: 'b', name: 'Walk' });
  const next = habitsReducer(s, { type: 'renamed', id: 'a', name: 'Read 20 pages' });
  expect(next[1]).toBe(s[1]);
  expect(next[0]).not.toBe(s[0]);
});

test('the Zustand store applies the same reducer', () => {
  useHabitStore.getState().dispatch({ type: 'added', id: 'z', name: 'Stretch' });
  expect(useHabitStore.getState().habits.map((h) => h.name)).toContain('Stretch');
});
```

The second test checks a property that matters for performance: unchanged habits keep the same
object identity, so `memo`ised rows and selectors see "no change" for them.

## How It Actually Works

**Context:** when a provider's `value` changes (by `Object.is`), React walks down from the provider
and schedules a re-render for every component that called `useContext` for that context. There's no
way to subscribe to part of a context value; the unit of subscription is the whole value.

**Zustand:** the store is a plain JavaScript closure holding state and a `Set` of listener
functions. `useHabitStore(selector)` uses React's `useSyncExternalStore` hook: it subscribes a
listener to the store, and on every `set` React calls your selector against the new state and
compares the result with the previous result (`Object.is`, or shallowly with `useShallow`). Only if
the selected value changed does React re-render that component. `useSyncExternalStore` also
guarantees that all components see the same version of the store during one render, avoiding
"tearing" with concurrent rendering. The cost is that every selector runs on every store update, so
keep selectors cheap (avoid sorting 5,000 items inside one; derive and memoise instead).

## Common mistakes

- **Putting server data in a global store** and hand-writing loading flags and refetch logic —
  that's what lesson 5's query cache is for.
- **One giant context** for everything — every consumer re-renders on any change.
- **Selectors that return new objects without `useShallow`** — re-render on every update.
- **Selecting the whole store** (`useHabitStore()` with no selector) — subscribes to everything.
- **Mutating state inside the reducer** (`state.push(...)`) — React and Zustand both rely on new
  references to detect change.
- **Keeping text-input state in a global store** — re-renders across the app on every keystroke.

## Exercise

1. Move the Level 1 habit tracker's state into the Zustand store, using the reducer above.
2. Add a Stats tab showing "habits done today" and "longest current streak", each as its own
   component with its own selector.
3. Prove the selectors work: add `console.log('render <name>')` to each component, toggle a habit,
   and confirm only the components whose selected value changed log. Then remove `useShallow` from
   one array selector and observe the difference.
4. Write two more reducer tests: renaming a missing ID leaves state unchanged (by identity of each
   habit), and archived habits keep their history.
