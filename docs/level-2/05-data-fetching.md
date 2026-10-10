---
description: "Fetch and cache server data with TanStack Query in React Native: query keys, staleTime, refetch on focus, mutations and optimistic updates."
---

# 05 · Fetching Data with TanStack Query

Fetching data with `useEffect` and `useState` works for one screen. Then you need a loading state,
an error state, a retry button, pull-to-refresh, a cache so the detail screen doesn't re-download what
the list already had, a refresh when the user returns to the app, and something sensible when the
phone goes offline in a tunnel. **TanStack Query** (formerly React Query) handles all of that by
treating server data as a **cache** rather than as state you own.

```bash
npx expo install @tanstack/react-query
```

## The model: queries are cached by key

```tsx
import { useQuery } from '@tanstack/react-query';

const { data, isPending, isError, error, refetch, isRefetching } = useQuery({
  queryKey: ['recipes', 'search', term],
  queryFn: () => searchRecipes(term),
});
```

- **`queryKey`** identifies the data. Any component anywhere that uses the same key shares the same
  cache entry and the same in-flight request.
- **`queryFn`** fetches it and must throw on failure (`fetch` doesn't throw on HTTP 500 — you must).
- The result is **cached**. Navigate away and back, and the cached data renders instantly while a
  background refetch (if the data is stale) updates it.

## A typed API client

This course uses [TheMealDB](https://www.themealdb.com/api.php)'s free development key (`1`),
which is intended for development and education. Its search endpoint returns `{"meals": null}`
when nothing matches — an API quirk your client should normalise.

```ts title="src/api.ts"
const BASE = 'https://www.themealdb.com/api/json/v1/1';

export type Recipe = {
  id: string;
  name: string;
  category: string | null;
  area: string | null;
  thumbnail: string;
  instructions: string;
  ingredients: { name: string; measure: string }[];
};

type RawMeal = Record<string, string | null>;

export function toRecipe(m: RawMeal): Recipe {
  const ingredients: Recipe['ingredients'] = [];
  for (let i = 1; i <= 20; i++) {
    const name = m[`strIngredient${i}`]?.trim();
    if (name) ingredients.push({ name, measure: m[`strMeasure${i}`]?.trim() ?? '' });
  }
  return {
    id: m.idMeal ?? '',
    name: m.strMeal ?? 'Untitled',
    category: m.strCategory ?? null,
    area: m.strArea ?? null,
    thumbnail: m.strMealThumb ?? '',
    instructions: m.strInstructions ?? '',
    ingredients,
  };
}

async function get<T>(path: string, signal?: AbortSignal): Promise<T> {
  const res = await fetch(`${BASE}${path}`, { signal });
  if (!res.ok) throw new Error(`HTTP ${res.status} for ${path}`);
  return (await res.json()) as T;
}

export async function searchRecipes(term: string, signal?: AbortSignal): Promise<Recipe[]> {
  const data = await get<{ meals: RawMeal[] | null }>(`/search.php?s=${encodeURIComponent(term)}`, signal);
  return (data.meals ?? []).map(toRecipe);
}

export async function getRecipe(id: string, signal?: AbortSignal): Promise<Recipe | null> {
  const data = await get<{ meals: RawMeal[] | null }>(`/lookup.php?i=${encodeURIComponent(id)}`, signal);
  return data.meals?.[0] ? toRecipe(data.meals[0]) : null;
}
```

TheMealDB spreads ingredients over `strIngredient1…20` and `strMeasure1…20`. `toRecipe` reshapes that
into a sensible array once, at the edge, so no component has to know the API's layout.

Run with Node against the live API while this lesson was written, `searchRecipes('dal')` returned a
single recipe (`52785 Dal fry`, 16 ingredients) and `searchRecipes('zzzzqq')` returned an empty
array rather than crashing on `null`. The API's data changes over time, so your results may differ.

## Wiring it up for mobile

The web version of TanStack Query refetches stale data when the browser tab regains focus and when
the network comes back. A phone has no browser tab: you tell Query when the **app** comes to the
foreground, using React Native's `AppState`.

```tsx title="src/queryClient.tsx"
import { ReactNode, useEffect } from 'react';
import { AppState, AppStateStatus, Platform } from 'react-native';
import { focusManager, QueryClient, QueryClientProvider } from '@tanstack/react-query';

export const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      staleTime: 60_000,        // data counts as fresh for a minute: no refetch on remount
      gcTime: 30 * 60_000,      // unused cache entries are kept for 30 minutes
      retry: 2,
    },
  },
});

function onAppStateChange(status: AppStateStatus) {
  if (Platform.OS !== 'web') focusManager.setFocused(status === 'active');
}

export function QueryProvider({ children }: { children: ReactNode }) {
  useEffect(() => {
    const sub = AppState.addEventListener('change', onAppStateChange);
    return () => sub.remove();
  }, []);
  return <QueryClientProvider client={queryClient}>{children}</QueryClientProvider>;
}
```

For network status, the usual approach is to connect Query's `onlineManager` to
`@react-native-community/netinfo` (`onlineManager.setEventListener(...)` with a NetInfo listener),
so queries pause while offline and resume on reconnect instead of burning retries.

Wrap the root layout:

```tsx title="app/_layout.tsx"
import { Stack } from 'expo-router';
import { QueryProvider } from '../src/queryClient';

export default function RootLayout() {
  return (
    <QueryProvider>
      <Stack />
    </QueryProvider>
  );
}
```

## Worked example: search and detail screens

```tsx title="app/index.tsx"
import { useState } from 'react';
import { ActivityIndicator, FlatList, Pressable, Text, TextInput, View } from 'react-native';
import { useQuery } from '@tanstack/react-query';
import { Link } from 'expo-router';
import { searchRecipes } from '../src/api';
import { useDebounced } from '../src/useDebounced';

export default function SearchScreen() {
  const [text, setText] = useState('dal');
  const term = useDebounced(text.trim(), 350);

  const query = useQuery({
    queryKey: ['recipes', 'search', term],
    queryFn: ({ signal }) => searchRecipes(term, signal),
    enabled: term.length >= 2,
    placeholderData: (previous) => previous, // keep old results visible while typing
  });

  return (
    <View style={{ flex: 1 }}>
      <TextInput value={text} onChangeText={setText} placeholder="Search recipes" autoCorrect={false}
        style={{ margin: 16, padding: 12, borderRadius: 10, backgroundColor: '#f1f5f9', fontSize: 16 }} />
      {query.isError ? (
        <View style={{ padding: 16, gap: 8 }}>
          <Text>Couldn't load recipes: {query.error.message}</Text>
          <Pressable onPress={() => query.refetch()}><Text style={{ color: '#4f46e5' }}>Try again</Text></Pressable>
        </View>
      ) : query.isPending && term.length >= 2 ? (
        <ActivityIndicator style={{ marginTop: 32 }} />
      ) : (
        <FlatList
          data={query.data ?? []}
          keyExtractor={(r) => r.id}
          refreshing={query.isRefetching}
          onRefresh={() => query.refetch()}
          ListEmptyComponent={term.length >= 2 ? <Text style={{ padding: 16 }}>No recipes for “{term}”.</Text> : null}
          renderItem={({ item }) => (
            <Link href={{ pathname: '/recipe/[id]', params: { id: item.id } }} asChild>
              <Pressable style={{ padding: 16 }}>
                <Text style={{ fontSize: 16, fontWeight: '600' }}>{item.name}</Text>
                <Text style={{ color: '#64748b' }}>{[item.area, item.category].filter(Boolean).join(' · ')}</Text>
              </Pressable>
            </Link>
          )}
        />
      )}
    </View>
  );
}
```

```ts title="src/useDebounced.ts"
import { useEffect, useState } from 'react';

export function useDebounced<T>(value: T, ms: number): T {
  const [debounced, setDebounced] = useState(value);
  useEffect(() => {
    const t = setTimeout(() => setDebounced(value), ms);
    return () => clearTimeout(t);
  }, [value, ms]);
  return debounced;
}
```

```tsx title="app/recipe/[id].tsx"
import { ScrollView, Text } from 'react-native';
import { Stack, useLocalSearchParams } from 'expo-router';
import { useQuery, useQueryClient } from '@tanstack/react-query';
import { getRecipe, Recipe } from '../../src/api';

export default function RecipeScreen() {
  const { id } = useLocalSearchParams<{ id: string }>();
  const qc = useQueryClient();

  const { data, isPending, isError } = useQuery({
    queryKey: ['recipe', id],
    queryFn: ({ signal }) => getRecipe(id, signal),
    // Seed from any search result already in the cache, so the screen renders instantly.
    initialData: () =>
      qc.getQueriesData<Recipe[]>({ queryKey: ['recipes', 'search'] })
        .flatMap(([, list]) => list ?? [])
        .find((r) => r.id === id),
  });

  if (isPending) return <Text style={{ padding: 16 }}>Loading…</Text>;
  if (isError || !data) return <Text style={{ padding: 16 }}>Recipe not found.</Text>;

  return (
    <ScrollView contentContainerStyle={{ padding: 16, gap: 12 }}>
      <Stack.Screen options={{ title: data.name }} />
      {data.ingredients.map((i) => (
        <Text key={i.name}>• {i.measure} {i.name}</Text>
      ))}
      <Text style={{ lineHeight: 22 }}>{data.instructions}</Text>
    </ScrollView>
  );
}
```

`queryFn` receives an `AbortSignal`. Passing it to `fetch` means a request for "da" is cancelled
when the user keeps typing "dal" and the "da" query is no longer used.

## Mutations and optimistic updates

For writes, `useMutation`. An optimistic update changes the cache immediately and rolls back if the
server says no:

```tsx
const qc = useQueryClient();

const toggleFavourite = useMutation({
  mutationFn: (id: string) => api.toggleFavourite(id),          // your server call
  onMutate: async (id) => {
    await qc.cancelQueries({ queryKey: ['favourites'] });       // don't let a refetch overwrite us
    const previous = qc.getQueryData<string[]>(['favourites']);
    qc.setQueryData<string[]>(['favourites'], (ids = []) =>
      ids.includes(id) ? ids.filter((x) => x !== id) : [...ids, id]);
    return { previous };
  },
  onError: (_err, _id, context) => qc.setQueryData(['favourites'], context?.previous), // roll back
  onSettled: () => qc.invalidateQueries({ queryKey: ['favourites'] }),                // re-sync
});
```

On a phone with a slow connection, optimistic updates are the difference between an app that feels
instant and one where every tap shows a spinner.

## How It Actually Works

A `QueryClient` holds a **QueryCache**: a map from a hashed query key to a `Query` object. Each
`Query` stores the data, the error, timestamps and a list of **observers** — one per mounted
`useQuery` with that key. When a component mounts, its observer subscribes to the query; if the data
is missing or older than `staleTime`, the query fetches. Duplicate observers share one request
(**deduplication**). When the last observer unsubscribes (all screens using the key unmount), the
query becomes inactive, and after `gcTime` it's garbage-collected.

Triggers for background refetches of stale data are: a new observer mounting, `focusManager`
reporting focus (which you wired to `AppState`), `onlineManager` reporting reconnection, and an
`invalidateQueries` call. Because screens stay mounted in a stack, observers on background screens
still exist — so the list underneath a detail screen is kept fresh too, and a returning user sees
current data without a loading spinner.

## Common mistakes

- **Not throwing on HTTP errors** — `fetch` resolves on 404 and 500; your query "succeeds" with an
  error page as data.
- **Missing variables in the query key** (`['recipes']` when the query depends on `term`) — every
  search shows the first search's results.
- **`staleTime: 0` (the default) everywhere** — every remount refetches; on a phone that's battery and
  data.
- **Copying query data into `useState`** — it stops updating. Read from the query.
- **Forgetting `focusManager` on native** — "refetch when I come back to the app" silently never
  happens.
- **Fetching on every keystroke** — debounce.

## Exercise

1. Build the search and detail screens above and confirm the detail screen opens instantly from a
   search result.
2. Add a "Random recipe" button on the search screen using `/random.php`. Should its `staleTime` be
   different? (Hint: what does "fresh" mean for random data?)
3. Turn on airplane mode, pull to refresh, and observe the error UI. Install
   `@react-native-community/netinfo`, connect it to `onlineManager`, and repeat — what changes?
4. Write a unit test for `toRecipe` that feeds it a raw meal with ingredients 1–3 filled and 4–20
   empty or `null`, and asserts exactly three ingredients come out.
