---
description: "Project: an Expo Router recipe book with search and favourites tabs, TanStack Query, favourites saved to SQLite, and recipes that open offline."
---

# 10 · Project — Offline-Friendly Recipe Book

This project combines Level 2 into one app: a **recipe book** with two tabs — Search and
Favourites — and a recipe detail screen. Search results come from TheMealDB through TanStack Query.
Favourites are stored in SQLite **with the full recipe**, so a favourited recipe opens even with no
network. Everything is typed, the detail screen is deep-linkable, and the "load from network, fall
back to local" logic is a pure, tested function.

## Requirements

- **Search tab:** debounced search, loading/error/empty states, pull-to-refresh, two-column grid of
  `RecipeCard`s (lesson 8).
- **Favourites tab:** favourites from SQLite, newest first, instant updates when you favourite or
  unfavourite elsewhere.
- **Detail screen** `/recipe/[id]`: ingredients and instructions, a heart button in the header that
  toggles favourite with an optimistic update.
- **Offline:** if the network fails and the recipe is a favourite, show the saved copy with a small
  "Offline copy" note.
- Deep link `recipebook://recipe/52785` opens the detail screen with the tabs underneath.

## Structure

```
app/
├── _layout.tsx              # providers + root stack
├── (tabs)/
│   ├── _layout.tsx          # Search | Favourites
│   ├── index.tsx            # Search
│   └── favourites.tsx
└── recipe/[id].tsx          # detail
src/
├── api.ts                   # from lesson 5 (searchRecipes, getRecipe, Recipe)
├── db.ts                    # migrations
├── favourites.ts            # SQLite repo + query hooks
├── loadRecipe.ts            # network-then-local logic (pure, tested)
├── queryClient.tsx          # from lesson 5
├── useDebounced.ts          # from lesson 5
└── RecipeCard.tsx           # from lesson 8
```

Reuse `api.ts`, `queryClient.tsx` and `useDebounced.ts` from lesson 5 and `RecipeCard.tsx` from
lesson 8 unchanged.

## Step 1 — the database

```ts title="src/db.ts"
import type { SQLiteDatabase } from 'expo-sqlite';

const MIGRATIONS = [
  `CREATE TABLE favourites (
     id TEXT PRIMARY KEY NOT NULL,
     name TEXT NOT NULL,
     thumbnail TEXT NOT NULL,
     area TEXT,
     json TEXT NOT NULL,          -- the full Recipe, for offline viewing
     saved_at INTEGER NOT NULL
   );
   CREATE INDEX idx_favourites_saved ON favourites(saved_at DESC);`,
];

export async function migrate(db: SQLiteDatabase) {
  await db.execAsync('PRAGMA journal_mode = WAL;');
  const row = await db.getFirstAsync<{ user_version: number }>('PRAGMA user_version');
  for (let v = row?.user_version ?? 0; v < MIGRATIONS.length; v++) {
    await db.withTransactionAsync(async () => {
      await db.execAsync(MIGRATIONS[v]);
      await db.execAsync(`PRAGMA user_version = ${v + 1}`);
    });
  }
}
```

## Step 2 — network first, local fallback (pure and tested)

```ts title="src/loadRecipe.ts"
import type { Recipe } from './api';

export type LoadedRecipe = { recipe: Recipe; source: 'network' | 'offline' } | null;

type Deps = {
  fetchRemote: (id: string) => Promise<Recipe | null>;
  readLocal: (id: string) => Promise<Recipe | null>;
};

export async function loadRecipe(id: string, { fetchRemote, readLocal }: Deps): Promise<LoadedRecipe> {
  try {
    const remote = await fetchRemote(id);
    if (remote) return { recipe: remote, source: 'network' };
    const local = await readLocal(id); // API says "no such recipe" but we have a saved copy
    return local ? { recipe: local, source: 'offline' } : null;
  } catch (networkError) {
    const local = await readLocal(id);
    if (local) return { recipe: local, source: 'offline' };
    throw networkError; // nothing saved: surface the real error to the UI
  }
}
```

Taking the two data sources as parameters is what makes this testable without a network or a
database. These tests were run with Jest in the course's SDK 57 check project and passed:

```ts title="src/loadRecipe.test.ts"
import { loadRecipe } from './loadRecipe';
import type { Recipe } from './api';

const r = (name: string): Recipe => ({ id: '1', name, category: null, area: null, thumbnail: '', instructions: '', ingredients: [] });

test('prefers the network copy', async () => {
  const res = await loadRecipe('1', { fetchRemote: async () => r('fresh'), readLocal: async () => r('saved') });
  expect(res).toEqual({ recipe: r('fresh'), source: 'network' });
});

test('falls back to the saved copy when offline', async () => {
  const res = await loadRecipe('1', {
    fetchRemote: async () => { throw new TypeError('Network request failed'); },
    readLocal: async () => r('saved'),
  });
  expect(res?.source).toBe('offline');
  expect(res?.recipe.name).toBe('saved');
});

test('rethrows when offline and nothing is saved', async () => {
  await expect(
    loadRecipe('1', { fetchRemote: async () => { throw new TypeError('Network request failed'); }, readLocal: async () => null }),
  ).rejects.toThrow('Network request failed');
});

test('returns null when the recipe does not exist anywhere', async () => {
  expect(await loadRecipe('1', { fetchRemote: async () => null, readLocal: async () => null })).toBeNull();
});
```

## Step 3 — favourites repository and hooks

SQLite is the source of truth for favourites; TanStack Query caches the result in memory so every
screen that shows favourites shares one copy and updates together.

```ts title="src/favourites.ts"
import { useMutation, useQuery, useQueryClient } from '@tanstack/react-query';
import { useSQLiteContext, type SQLiteDatabase } from 'expo-sqlite';
import type { Recipe } from './api';

export type FavouriteSummary = { id: string; name: string; thumbnail: string; area: string | null };

export async function readFavourite(db: SQLiteDatabase, id: string): Promise<Recipe | null> {
  const row = await db.getFirstAsync<{ json: string }>('SELECT json FROM favourites WHERE id = ?', id);
  return row ? (JSON.parse(row.json) as Recipe) : null;
}

const KEY = ['favourites'] as const;

export function useFavourites() {
  const db = useSQLiteContext();
  return useQuery({
    queryKey: KEY,
    queryFn: () => db.getAllAsync<FavouriteSummary>('SELECT id, name, thumbnail, area FROM favourites ORDER BY saved_at DESC'),
    staleTime: Infinity, // only changes through our own mutations
  });
}

export function useIsFavourite(id: string) {
  const { data } = useFavourites();
  return !!data?.some((f) => f.id === id);
}

export function useToggleFavourite() {
  const db = useSQLiteContext();
  const qc = useQueryClient();
  return useMutation({
    mutationFn: async ({ recipe, makeFavourite }: { recipe: Recipe; makeFavourite: boolean }) => {
      if (makeFavourite) {
        await db.runAsync(
          'INSERT OR REPLACE INTO favourites (id, name, thumbnail, area, json, saved_at) VALUES (?, ?, ?, ?, ?, ?)',
          recipe.id, recipe.name, recipe.thumbnail, recipe.area, JSON.stringify(recipe), Date.now(),
        );
      } else {
        await db.runAsync('DELETE FROM favourites WHERE id = ?', recipe.id);
      }
    },
    onMutate: async ({ recipe, makeFavourite }) => {
      await qc.cancelQueries({ queryKey: KEY });
      const previous = qc.getQueryData<FavouriteSummary[]>(KEY);
      qc.setQueryData<FavouriteSummary[]>(KEY, (list = []) =>
        makeFavourite
          ? [{ id: recipe.id, name: recipe.name, thumbnail: recipe.thumbnail, area: recipe.area }, ...list.filter((f) => f.id !== recipe.id)]
          : list.filter((f) => f.id !== recipe.id),
      );
      return { previous };
    },
    onError: (_e, _v, ctx) => qc.setQueryData(KEY, ctx?.previous),
    onSettled: () => qc.invalidateQueries({ queryKey: KEY }),
  });
}
```

## Step 4 — layouts

```tsx title="app/_layout.tsx"
import { Stack } from 'expo-router';
import { SQLiteProvider } from 'expo-sqlite';
import { QueryProvider } from '../src/queryClient';
import { migrate } from '../src/db';

export const unstable_settings = { initialRouteName: '(tabs)' }; // deep links get the tabs underneath

export default function RootLayout() {
  return (
    <SQLiteProvider databaseName="recipes.db" onInit={migrate}>
      <QueryProvider>
        <Stack>
          <Stack.Screen name="(tabs)" options={{ headerShown: false }} />
          <Stack.Screen name="recipe/[id]" options={{ title: '' }} />
        </Stack>
      </QueryProvider>
    </SQLiteProvider>
  );
}
```

```tsx title="app/(tabs)/_layout.tsx"
import { Tabs } from 'expo-router/js-tabs';
import Ionicons from '@expo/vector-icons/Ionicons';

export default function TabsLayout() {
  return (
    <Tabs screenOptions={{ tabBarActiveTintColor: '#e11d48' }}>
      <Tabs.Screen name="index" options={{ title: 'Search', tabBarIcon: ({ color, size }) => <Ionicons name="search" color={color} size={size} /> }} />
      <Tabs.Screen name="favourites" options={{ title: 'Favourites', tabBarIcon: ({ color, size }) => <Ionicons name="heart" color={color} size={size} /> }} />
    </Tabs>
  );
}
```

Set `"scheme": "recipebook"` in `app.json` for the deep link.

## Step 5 — screens

```tsx title="app/(tabs)/index.tsx"
import { useState } from 'react';
import { ActivityIndicator, FlatList, Pressable, Text, TextInput, View } from 'react-native';
import { useQuery } from '@tanstack/react-query';
import { router } from 'expo-router';
import { searchRecipes } from '../../src/api';
import { useDebounced } from '../../src/useDebounced';
import { RecipeCard } from '../../src/RecipeCard';
import { useFavourites } from '../../src/favourites';

export default function Search() {
  const [text, setText] = useState('chicken');
  const term = useDebounced(text.trim(), 350);
  const favourites = useFavourites();
  const favIds = new Set(favourites.data?.map((f) => f.id));

  const q = useQuery({
    queryKey: ['recipes', 'search', term],
    queryFn: ({ signal }) => searchRecipes(term, signal),
    enabled: term.length >= 2,
    placeholderData: (prev) => prev,
  });

  return (
    <View style={{ flex: 1, backgroundColor: '#f8fafc' }}>
      <TextInput value={text} onChangeText={setText} placeholder="Search recipes" autoCorrect={false} returnKeyType="search"
        style={{ margin: 12, padding: 12, borderRadius: 10, backgroundColor: 'white', fontSize: 16 }} />
      {q.isError ? (
        <View style={{ padding: 16, gap: 8 }}>
          <Text>Couldn't reach the recipe server. Your favourites still work offline.</Text>
          <Pressable onPress={() => q.refetch()} accessibilityRole="button"><Text style={{ color: '#e11d48' }}>Try again</Text></Pressable>
        </View>
      ) : q.isPending && term.length >= 2 ? (
        <ActivityIndicator style={{ marginTop: 32 }} />
      ) : (
        <FlatList
          data={q.data ?? []}
          numColumns={2}
          keyExtractor={(r) => r.id}
          columnWrapperStyle={{ gap: 12 }}
          contentContainerStyle={{ padding: 12, gap: 12 }}
          refreshing={q.isRefetching}
          onRefresh={() => q.refetch()}
          keyboardDismissMode="on-drag"
          ListEmptyComponent={term.length >= 2 ? <Text style={{ padding: 16 }}>No recipes for “{term}”.</Text> : <Text style={{ padding: 16 }}>Type at least two letters.</Text>}
          renderItem={({ item }) => (
            <RecipeCard id={item.id} name={item.name} thumbnail={item.thumbnail} area={item.area}
              favourite={favIds.has(item.id)} onPress={() => router.push({ pathname: '/recipe/[id]', params: { id: item.id } })} />
          )}
        />
      )}
    </View>
  );
}
```

```tsx title="app/(tabs)/favourites.tsx"
import { FlatList, Text, View } from 'react-native';
import { router } from 'expo-router';
import { useFavourites } from '../../src/favourites';
import { RecipeCard } from '../../src/RecipeCard';

export default function Favourites() {
  const { data = [], isPending } = useFavourites();
  if (isPending) return null;
  return (
    <FlatList
      data={data}
      numColumns={2}
      keyExtractor={(f) => f.id}
      columnWrapperStyle={{ gap: 12 }}
      contentContainerStyle={{ padding: 12, gap: 12, flexGrow: 1 }}
      ListEmptyComponent={
        <View style={{ flex: 1, alignItems: 'center', justifyContent: 'center', padding: 32 }}>
          <Text style={{ fontSize: 16, textAlign: 'center' }}>No favourites yet. Tap the heart on any recipe to save it — saved recipes open offline.</Text>
        </View>
      }
      renderItem={({ item }) => (
        <RecipeCard id={item.id} name={item.name} thumbnail={item.thumbnail} area={item.area} favourite
          onPress={() => router.push({ pathname: '/recipe/[id]', params: { id: item.id } })} />
      )}
    />
  );
}
```

```tsx title="app/recipe/[id].tsx"
import { Pressable, ScrollView, StyleSheet, Text, View } from 'react-native';
import { Stack, useLocalSearchParams } from 'expo-router';
import { useQuery } from '@tanstack/react-query';
import { useSQLiteContext } from 'expo-sqlite';
import { Image } from 'expo-image';
import Ionicons from '@expo/vector-icons/Ionicons';
import { getRecipe } from '../../src/api';
import { loadRecipe } from '../../src/loadRecipe';
import { readFavourite, useIsFavourite, useToggleFavourite } from '../../src/favourites';

export default function RecipeDetail() {
  const { id } = useLocalSearchParams<{ id: string }>();
  const db = useSQLiteContext();
  const isFav = useIsFavourite(id);
  const toggle = useToggleFavourite();

  const q = useQuery({
    queryKey: ['recipe', id],
    queryFn: ({ signal }) => loadRecipe(id, { fetchRemote: (rid) => getRecipe(rid, signal), readLocal: (rid) => readFavourite(db, rid) }),
    retry: 1,
  });

  if (q.isPending) return <Text style={styles.message}>Loading…</Text>;
  if (q.isError) return <Text style={styles.message}>You're offline and this recipe isn't saved.</Text>;
  if (!q.data) return <Text style={styles.message}>Recipe not found.</Text>;

  const { recipe, source } = q.data;

  return (
    <ScrollView contentContainerStyle={styles.content}>
      <Stack.Screen
        options={{
          title: recipe.name,
          headerRight: () => (
            <Pressable
              onPress={() => toggle.mutate({ recipe, makeFavourite: !isFav })}
              hitSlop={10}
              accessibilityRole="button"
              accessibilityLabel={isFav ? 'Remove from favourites' : 'Save to favourites'}
            >
              <Ionicons name={isFav ? 'heart' : 'heart-outline'} size={24} color="#e11d48" />
            </Pressable>
          ),
        }}
      />
      <Image source={{ uri: recipe.thumbnail }} style={styles.hero} contentFit="cover" cachePolicy="memory-disk" />
      {source === 'offline' && <Text style={styles.offline}>Offline copy — saved with your favourites.</Text>}
      <Text style={styles.h2}>Ingredients</Text>
      {recipe.ingredients.map((i, idx) => (
        <View key={`${i.name}-${idx}`} style={styles.ingredient}>
          <Text style={styles.measure}>{i.measure}</Text>
          <Text style={styles.ingName}>{i.name}</Text>
        </View>
      ))}
      <Text style={styles.h2}>Method</Text>
      <Text style={styles.body}>{recipe.instructions}</Text>
    </ScrollView>
  );
}

const styles = StyleSheet.create({
  content: { padding: 16, gap: 8, paddingBottom: 48 },
  hero: { width: '100%', aspectRatio: 4 / 3, borderRadius: 16, backgroundColor: '#e2e8f0' },
  offline: { backgroundColor: '#fef3c7', color: '#92400e', padding: 8, borderRadius: 8, overflow: 'hidden' },
  h2: { fontSize: 18, fontWeight: '700', marginTop: 12 },
  ingredient: { flexDirection: 'row', gap: 12 },
  measure: { width: 110, color: '#64748b' },
  ingName: { flex: 1 },
  body: { fontSize: 16, lineHeight: 24 },
  message: { padding: 24, fontSize: 16 },
});
```

Ingredient keys use name plus index because the API can list the same ingredient twice.

## Verify it

1. Search, open a recipe, favourite it. Switch to the Favourites tab — it's already there (shared
   query cache, optimistic update).
2. Turn on airplane mode. Open the favourite: it loads with the "Offline copy" note. Open a
   non-favourite: you get the offline message, not a crash.
3. Fully close the app and reopen it — favourites survive (SQLite).
4. With a development build, open `recipebook://recipe/52785` — the detail screen opens with the
   tabs underneath, so back works. In Expo Go, use the `exp://` URL that `Linking.createURL` prints.

## How It Actually Works

Follow the "favourite" tap. `toggle.mutate` runs `onMutate` first: it cancels any in-flight
favourites query (so a slow read can't overwrite the optimistic value), snapshots the current list,
and writes the new list into the query cache. Every mounted observer of `['favourites']` — the
header heart via `useIsFavourite`, the Search tab's hearts, the Favourites tab — re-renders
immediately from the cache. Meanwhile `mutationFn` writes to SQLite on the native side. `onSettled`
invalidates the key, which re-runs the SQL query and replaces the optimistic list with the
database's truth. If the write failed, `onError` restores the snapshot first.

Offline, `getRecipe`'s `fetch` rejects with a network `TypeError`. `loadRecipe` catches it and asks
SQLite for the saved JSON. The query itself succeeds with `source: 'offline'`, so the UI renders
normally instead of entering an error state.

## Common mistakes

- **Saving only an ID for favourites** — then "offline favourites" can't render anything.
- **Two sources of truth** — e.g. keeping favourites in Zustand *and* SQLite. Pick SQLite and cache
  it with Query.
- **Not cancelling queries in `onMutate`** — a refetch finishing late flips the heart back.
- **Forgetting `initialRouteName`** — deep-linked detail screens have no back destination.
- **Assuming `fetch` errors mean "offline"** — it could be a server error. Here a server error with no
  saved copy is rethrown and shown, which is honest; a polished app would distinguish the two.

## Exercise

1. Add a **notes** field to each favourite (a migration adding a `note TEXT` column), editable on the
   detail screen with a `TextInput` and the keyboard handling from lesson 9.
2. Add a "Categories" row on the Search tab using `/categories.php`, cached with
   `staleTime: Infinity` (categories rarely change).
3. Persist the TanStack Query cache across launches with `@tanstack/query-async-storage-persister`
   so recent searches show instantly on a cold start. Decide which queries should *not* be persisted.
4. Add one more test to `loadRecipe.test.ts`: the network returns `null` but a local copy exists.
