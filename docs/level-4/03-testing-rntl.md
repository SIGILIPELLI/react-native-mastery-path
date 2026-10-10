---
description: "Test React Native apps with jest-expo and RNTL 14: async render, queries by role and label, user events, mocking native modules, data screens."
---

# 03 · Unit & Component Tests with Jest and RNTL

You've already written pure-function tests throughout this course: the streak calculator, the
habits reducer, the Zod schema, the sync policy. They're the cheapest, fastest tests you can have, and
the course deliberately pushed logic into pure functions to make them possible. This lesson adds the
next layer: **component tests** that render React Native components in Node, interact with them the
way a user would, and assert on what the user would see — without a simulator.

## Setup (and two things that tripped up this course)

```bash
npx expo install jest-expo jest @testing-library/react-native -- --save-dev
```

```json title="package.json (excerpt)"
{
  "scripts": { "test": "jest" },
  "jest": { "preset": "jest-expo" }
}
```

`jest-expo` is a Jest preset that configures Babel the way Metro does, sets up React Native's mocks,
and mocks the native side of Expo modules so they can be imported in Node.

When this course set this up in an SDK 57 project, two extra installs were needed, and the error
messages said so:

1. **`@react-native/jest-preset`** — `jest-expo` 57 failed at startup with a message asking for this
   peer dependency. Installing the version matching the project's React Native minor
   (`@react-native/jest-preset@~0.86.0` for react-native 0.86) fixed it.
2. **`test-renderer`** — React Native Testing Library 14 replaced the deprecated React Test Renderer
   with the `test-renderer` package as a peer dependency. RNTL's migration guide recommends the
   `test-renderer` minor that matches your React minor; the project used React 19.2, so
   `test-renderer@~1.2.0`.

Your versions will differ — read the error messages, check the peer dependencies
(`npm view <package> peerDependencies`), and install what they ask for.

With that setup, every test file in this lesson was run: four test files, six tests, all passing.

## RNTL 14: render is async

Most tutorials you'll find were written for RNTL 12 or 13, where `render(...)` was synchronous. In
**RNTL 14**, `render`, `renderHook`, `fireEvent` and `act` return promises (it adopted React 19's
async rendering model), so you `await` them:

```tsx
await render(<HabitRow {...props} />);   // RNTL 14
```

Forget the `await` and queries fail with "`render` function has not been called" — exactly what
happened the first time these tests were run for this lesson.

## Query like a user

RNTL encourages finding elements the way users (and screen readers) do. In order of preference:

| Query | Finds by | Example |
|-------|----------|---------|
| `getByRole` | accessibility role + name | `getByRole('button', { name: 'Add' })` |
| `getByLabelText` | `accessibilityLabel` | `getByLabelText('New habit name')` |
| `getByPlaceholderText` | input placeholder | `getByPlaceholderText('Search recipes')` |
| `getByText` | visible text | `getByText('No habits yet.')` |
| `getByDisplayValue` | current `TextInput` value | `getByDisplayValue('Walk')` |
| `getByTestId` | `testID` prop | last resort |

Variants: `getBy…` throws if not found, `queryBy…` returns `null` (for asserting absence),
`findBy…` waits asynchronously (for content that appears after a fetch). Using roles and labels
means your tests double as accessibility checks: if a test can't find the "Add" button by role, a
screen reader user probably can't either (L3-07).

## User events

`userEvent` simulates realistic interaction sequences — focus, key presses, press in and out:

```tsx
const user = userEvent.setup();
await user.type(screen.getByLabelText('New habit name'), 'Walk');
await user.press(screen.getByRole('button', { name: 'Add' }));
```

Prefer it over the lower-level `fireEvent`, which dispatches a single event and skips the steps a real
interaction goes through.

## Worked example 1: the add-habit bar

These tests target the `AddHabitBar` from the Level 1 project. They were run with Jest 30,
jest-expo 57 and RNTL 14.1 for this lesson, and both passed:

```tsx title="src/AddHabitBar.test.tsx"
import { render, screen, userEvent } from '@testing-library/react-native';
import { AddHabitBar } from './AddHabitBar';
import { validateHabitName } from './habits';

const colors = { bg: '#fff', surface: '#fff', text: '#000', muted: '#666', border: '#ccc', primary: '#00f', success: '#0a0' };

test('shows a validation error and does not call onAdd for a duplicate', async () => {
  const user = userEvent.setup();
  const onAdd = jest.fn();
  const existing = [{ id: '1', name: 'Read', createdAt: '2026-10-01', doneDates: [] }];
  await render(<AddHabitBar colors={colors} bottomInset={0} validate={(n) => validateHabitName(n, existing)} onAdd={onAdd} />);

  await user.type(screen.getByLabelText('New habit name'), 'read');
  await user.press(screen.getByRole('button', { name: 'Add' }));

  expect(screen.getByText('You already track that habit.')).toBeOnTheScreen();
  expect(onAdd).not.toHaveBeenCalled();
});

test('adds a trimmed name and clears the input', async () => {
  const user = userEvent.setup();
  const onAdd = jest.fn();
  await render(<AddHabitBar colors={colors} bottomInset={0} validate={(n) => validateHabitName(n, [])} onAdd={onAdd} />);

  const input = screen.getByLabelText('New habit name');
  await user.type(input, '  Walk  ');
  await user.press(screen.getByRole('button', { name: 'Add' }));

  expect(onAdd).toHaveBeenCalledWith('Walk');
  expect(input).toHaveDisplayValue('');
});
```

Matchers like `toBeOnTheScreen` and `toHaveDisplayValue` come with RNTL — no extra setup needed in
recent versions.

## Worked example 2: accessibility state and memoised rows

```tsx title="src/HabitRow.test.tsx"
import { render, screen, userEvent } from '@testing-library/react-native';
import { HabitRow } from './HabitRow';

const colors = { bg: '#fff', surface: '#fff', text: '#000', muted: '#666', border: '#ccc', primary: '#00f', success: '#0a0' };

test('exposes checkbox role, name and checked state, and toggles by id', async () => {
  const user = userEvent.setup();
  const onToggle = jest.fn();
  await render(
    <HabitRow id="h1" name="Drink water" streak={3} doneToday colors={colors} onToggle={onToggle} onDelete={jest.fn()} />,
  );

  const row = screen.getByRole('checkbox', { name: 'Drink water, 3 day streak' });
  expect(row).toBeChecked();
  expect(screen.getByText('3-day streak')).toBeOnTheScreen();

  await user.press(row);
  expect(onToggle).toHaveBeenCalledWith('h1');
});
```

This test fails if someone removes `accessibilityRole`, changes the label, or forgets
`accessibilityState` — regressions a visual check wouldn't catch.

## Mocking native modules

`jest-expo` mocks the native part of Expo modules, but you often want control over what they return,
or to assert they were called:

```tsx title="src/CompleteWithHaptics.test.tsx"
import { Pressable, Text } from 'react-native';
import { render, screen, userEvent } from '@testing-library/react-native';
import * as Haptics from 'expo-haptics';

jest.mock('expo-haptics', () => ({
  notificationAsync: jest.fn(),
  NotificationFeedbackType: { Success: 'success' },
}));

function CompleteButton({ onDone }: { onDone: () => void }) {
  return (
    <Pressable accessibilityRole="button" onPress={() => { onDone(); Haptics.notificationAsync(Haptics.NotificationFeedbackType.Success); }}>
      <Text>Complete</Text>
    </Pressable>
  );
}

test('completing triggers a success haptic', async () => {
  const user = userEvent.setup();
  await render(<CompleteButton onDone={jest.fn()} />);
  await user.press(screen.getByRole('button', { name: 'Complete' }));
  expect(Haptics.notificationAsync).toHaveBeenCalledWith('success');
});
```

Mock at the **module boundary** (`expo-haptics`, your API client, your own native module) — never
mock React Native components themselves, or you end up testing your mocks.

## Worked example 3: a data screen with TanStack Query

For screens that fetch, give each test a fresh `QueryClient` with retries off, and mock `fetch`:

```tsx title="src/RecipeTitle.test.tsx"
import { Text } from 'react-native';
import { QueryClient, QueryClientProvider, useQuery } from '@tanstack/react-query';
import { render, screen } from '@testing-library/react-native';

function RecipeTitle({ id }: { id: string }) {
  const q = useQuery({
    queryKey: ['recipe', id],
    queryFn: async () => {
      const res = await fetch(`https://example.test/recipes/${id}`);
      if (!res.ok) throw new Error(`HTTP ${res.status}`);
      return (await res.json()) as { name: string };
    },
  });
  if (q.isPending) return <Text>Loading…</Text>;
  if (q.isError) return <Text>Couldn't load recipe</Text>;
  return <Text accessibilityRole="header">{q.data.name}</Text>;
}

function renderWithClient(ui: React.ReactElement) {
  const client = new QueryClient({ defaultOptions: { queries: { retry: false, gcTime: Infinity } } });
  return render(<QueryClientProvider client={client}>{ui}</QueryClientProvider>);
}

afterEach(() => jest.restoreAllMocks());

test('shows the recipe name after loading', async () => {
  jest.spyOn(global, 'fetch').mockResolvedValue({ ok: true, status: 200, json: async () => ({ name: 'Dal fry' }) } as Response);
  await renderWithClient(<RecipeTitle id="52785" />);
  expect(screen.getByText('Loading…')).toBeOnTheScreen();
  expect(await screen.findByRole('header', { name: 'Dal fry' })).toBeOnTheScreen();
});

test('shows an error state on HTTP 500', async () => {
  jest.spyOn(global, 'fetch').mockResolvedValue({ ok: false, status: 500, json: async () => ({}) } as Response);
  await renderWithClient(<RecipeTitle id="1" />);
  expect(await screen.findByText("Couldn't load recipe")).toBeOnTheScreen();
});
```

`retry: false` matters: with Query's default retries, the error test would wait through several
retry delays and could time out. `gcTime: Infinity` avoids Jest complaining about timers left running
after the test.

## What to test where

| Layer | Tool | Examples from this course |
|-------|------|---------------------------|
| Pure logic | Jest | streaks, reducers, schemas, sync policy, `loadRecipe` |
| Components | Jest + RNTL | forms, rows, empty/error states, a11y roles |
| Whole app on a device | Maestro/Detox (next lesson) | sign in → add habit → see it after restart |

Most of your tests should be in the first two layers: fast, deterministic, runnable on every commit.

## How It Actually Works

Jest runs in Node, where there is no iOS or Android. `jest-expo` and React Native's Jest preset
replace native host components with mocks that render to a JavaScript tree instead of native views,
and replace native modules with JavaScript stubs. RNTL then renders your component with
`test-renderer`, a React renderer whose "host" is an in-memory tree of host elements (`View`, `Text`,
`TextInput`) with their props. Queries walk that tree: `getByRole('checkbox', { name })` finds host
elements whose `accessibilityRole` (or `role`) matches and whose accessible name — computed from
`accessibilityLabel` or text content — matches. `userEvent.press` calls the same press-in, press-out
and press handlers that the real responder system would, wrapped in `act()` so React flushes all
state updates and effects before your assertions run. Because React 19 renders asynchronously, RNTL 14
awaits that flushing — hence the async API.

## Common mistakes

- **Forgetting `await render(...)`** with RNTL 14.
- **Querying by `testID` first** — tests pass while accessibility breaks.
- **Mocking React Native components** instead of module boundaries.
- **Shared `QueryClient` between tests** — cached data leaks from one test to the next.
- **Query retries on in tests** — slow, flaky error tests.
- **Snapshot tests of whole screens** as the main strategy — they break on every style tweak and
  rarely catch real bugs.

## Exercise

1. Set up `jest-expo` and RNTL in your app, resolving any peer-dependency errors the way described
   above.
2. Port the three worked examples to your codebase and get them passing.
3. Write a component test for the Level 2 habit form: submitting with "reminder" on but no time shows
   the error under the time field, and focus moves to it.
4. Write a test for the recipe detail screen's offline path by mocking `getRecipe` to reject and
   `readFavourite` to resolve a saved recipe; assert the "Offline copy" note appears.
