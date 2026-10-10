---
description: "Choose the right on-device store: AsyncStorage for small preferences, SecureStore for tokens, expo-sqlite for real data — with versioned schema migrations."
---

# 06 · Local Persistence: AsyncStorage, SecureStore & SQLite

A web app can lean on the server and treat the browser as a cache. A mobile app is expected to open
instantly, show yesterday's data in airplane mode, and remember the user's settings across
restarts — and an OS update or a low-storage event shouldn't wipe anything important. You need
on-device storage, and the right store depends on what you're storing.

| Store | Good for | Not for |
|-------|----------|---------|
| **AsyncStorage** | Small key–value data: preferences, onboarding-seen flag, last selected tab | Secrets; large or relational data; anything you query |
| **SecureStore** (`expo-secure-store`) | Small secrets: auth tokens, refresh tokens, an encryption key | Large values; general data |
| **SQLite** (`expo-sqlite`) | Real app data: records, history, anything you filter, sort or join | Tiny one-off flags (overkill) |
| **File system** (`expo-file-system`) | Images, downloads, exported files | Structured data |

## AsyncStorage: small, string-only, asynchronous

```bash
npx expo install @react-native-async-storage/async-storage
```

```ts title="src/prefs.ts"
import AsyncStorage from '@react-native-async-storage/async-storage';

export type Prefs = { reminderHour: number; weekStartsOn: 0 | 1; theme: 'system' | 'light' | 'dark' };

const KEY = 'prefs:v1';
export const defaultPrefs: Prefs = { reminderHour: 20, weekStartsOn: 1, theme: 'system' };

export async function loadPrefs(): Promise<Prefs> {
  try {
    const raw = await AsyncStorage.getItem(KEY);
    if (!raw) return defaultPrefs;
    return { ...defaultPrefs, ...(JSON.parse(raw) as Partial<Prefs>) };
  } catch {
    return defaultPrefs; // corrupt JSON or storage error: fall back, don't crash on launch
  }
}

export async function savePrefs(prefs: Prefs): Promise<void> {
  await AsyncStorage.setItem(KEY, JSON.stringify(prefs));
}
```

Details that matter:

- Values are **strings**. JSON-encode objects; parse defensively.
- Merging with `defaultPrefs` means adding a new preference in a later app version doesn't break
  users who saved the old shape.
- The key includes a version (`prefs:v1`), so a breaking change can migrate from `v1` to `v2`.
- It's **unencrypted**. Treat it like a text file anyone with device backups could read.

Zustand (lesson 4) has a `persist` middleware that can save a store to AsyncStorage automatically —
convenient for small stores like preferences.

## SecureStore: for secrets

```bash
npx expo install expo-secure-store
```

```ts title="src/session.ts"
import * as SecureStore from 'expo-secure-store';

const TOKEN_KEY = 'auth.refreshToken';

export async function saveRefreshToken(token: string) {
  await SecureStore.setItemAsync(TOKEN_KEY, token, {
    keychainAccessible: SecureStore.WHEN_UNLOCKED_THIS_DEVICE_ONLY,
  });
}

export const getRefreshToken = () => SecureStore.getItemAsync(TOKEN_KEY);
export const clearRefreshToken = () => SecureStore.deleteItemAsync(TOKEN_KEY);
```

On iOS values go into the **Keychain**; on Android they're encrypted with a key held in the
**Android Keystore**. `WHEN_UNLOCKED_THIS_DEVICE_ONLY` means the item is only readable while the
device is unlocked and is not migrated to a new phone via backups — a reasonable default for tokens.
Keep values small; it's designed for secrets, not data. (Level 4, lesson 5 goes deeper on mobile
security.)

## SQLite: real data

```bash
npx expo install expo-sqlite
```

SQLite is a full relational database in a single file inside your app's sandbox. `expo-sqlite`
exposes it with a promise-based API and a React provider:

```tsx title="app/_layout.tsx"
import { Stack } from 'expo-router';
import { SQLiteProvider } from 'expo-sqlite';
import { migrate } from '../src/db';

export default function RootLayout() {
  return (
    <SQLiteProvider databaseName="habits.db" onInit={migrate}>
      <Stack />
    </SQLiteProvider>
  );
}
```

`onInit` runs before any child renders, which makes it the right place for migrations.

### Migrations with `PRAGMA user_version`

Your schema *will* change after release, and users upgrade from any old version. SQLite stores an
integer in the database header, `user_version`, that you can use to track which migrations have run:

```ts title="src/db.ts"
import type { SQLiteDatabase } from 'expo-sqlite';

const MIGRATIONS: string[] = [
  // v1
  `CREATE TABLE habits (
     id TEXT PRIMARY KEY NOT NULL,
     name TEXT NOT NULL,
     created_at TEXT NOT NULL
   );
   CREATE TABLE checkins (
     habit_id TEXT NOT NULL REFERENCES habits(id) ON DELETE CASCADE,
     day TEXT NOT NULL,
     PRIMARY KEY (habit_id, day)
   );`,
  // v2: archiving habits
  `ALTER TABLE habits ADD COLUMN archived INTEGER NOT NULL DEFAULT 0;`,
  // v3: faster "what did I do on day X" queries
  `CREATE INDEX idx_checkins_day ON checkins(day);`,
];

export async function migrate(db: SQLiteDatabase): Promise<void> {
  await db.execAsync('PRAGMA journal_mode = WAL; PRAGMA foreign_keys = ON;');
  const row = await db.getFirstAsync<{ user_version: number }>('PRAGMA user_version');
  let version = row?.user_version ?? 0;
  while (version < MIGRATIONS.length) {
    const sql = MIGRATIONS[version];
    await db.withTransactionAsync(async () => {
      await db.execAsync(sql);
      await db.execAsync(`PRAGMA user_version = ${version + 1}`);
    });
    version += 1;
  }
}
```

The rules: **never edit a migration that has shipped** (users already ran it), only append new
ones; and run each migration with its version bump inside one transaction, so a crash halfway leaves
the database at the previous version rather than half-migrated.

### Queries

```ts title="src/habitRepo.ts"
import type { SQLiteDatabase } from 'expo-sqlite';

export type HabitRow = { id: string; name: string; created_at: string; archived: number; done_today: number };

export function listHabits(db: SQLiteDatabase, today: string) {
  return db.getAllAsync<HabitRow>(
    `SELECT h.*, EXISTS(SELECT 1 FROM checkins c WHERE c.habit_id = h.id AND c.day = ?) AS done_today
       FROM habits h
      WHERE h.archived = 0
      ORDER BY h.created_at`,
    today,
  );
}

export async function toggleCheckin(db: SQLiteDatabase, habitId: string, day: string) {
  const res = await db.runAsync('DELETE FROM checkins WHERE habit_id = ? AND day = ?', habitId, day);
  if (res.changes === 0) {
    await db.runAsync('INSERT INTO checkins (habit_id, day) VALUES (?, ?)', habitId, day);
  }
}

export function addHabit(db: SQLiteDatabase, id: string, name: string, createdAt: string) {
  return db.runAsync('INSERT INTO habits (id, name, created_at) VALUES (?, ?, ?)', id, name, createdAt);
}
```

Always pass values as **parameters** (`?`), never by string concatenation — that's how SQL injection
happens, even in a local database (a habit named `'); DROP TABLE habits; --` should be stored, not
executed).

## Worked example: a screen backed by SQLite

```tsx title="app/index.tsx"
import { useCallback, useState } from 'react';
import { FlatList, Pressable, Text } from 'react-native';
import { useFocusEffect } from 'expo-router';
import { useSQLiteContext } from 'expo-sqlite';
import { HabitRow, listHabits, toggleCheckin } from '../src/habitRepo';

const todayKey = () => new Date().toISOString().slice(0, 10); // demo only: UTC day; use local day keys (L1-10) in real code

export default function Today() {
  const db = useSQLiteContext();
  const [habits, setHabits] = useState<HabitRow[]>([]);

  const reload = useCallback(async () => setHabits(await listHabits(db, todayKey())), [db]);

  useFocusEffect(useCallback(() => { reload(); }, [reload]));

  async function toggle(id: string) {
    await toggleCheckin(db, id, todayKey());
    await reload();
  }

  return (
    <FlatList
      data={habits}
      keyExtractor={(h) => h.id}
      renderItem={({ item }) => (
        <Pressable onPress={() => toggle(item.id)} style={{ padding: 16 }}>
          <Text>{item.done_today ? '✅' : '⬜️'} {item.name}</Text>
        </Pressable>
      )}
    />
  );
}
```

Kill the app, reopen it, and the check-ins are still there.

## How It Actually Works

All three stores write into your app's **sandbox** — a private directory that other apps can't read
and that's deleted when the app is uninstalled.

- **AsyncStorage** is backed by a small native key–value store (an SQLite-backed store on Android,
  and serialized files on iOS in the classic implementation). Every call crosses from JavaScript to
  native and back asynchronously, which is why reading 500 keys one at a time is slow — batch with
  `multiGet`, or use a real database.
- **SecureStore** calls the platform's secret storage. The iOS Keychain is a system-managed
  encrypted database; Android stores an encrypted value in app storage, with the encryption key held
  inside the Keystore, which can be hardware-backed so the key never leaves secure hardware.
- **expo-sqlite** embeds the SQLite C library in your app. Your SQL goes through a native module to
  SQLite, which reads and writes the `.db` file. With **WAL mode** (`journal_mode = WAL`), writes go
  to a separate write-ahead log and readers aren't blocked by writers — good for an app that reads
  for the UI while writing check-ins. `PRAGMA user_version` is just four bytes in the database file
  header that SQLite never touches itself, which is why it's a convenient migration counter.

## Common mistakes

- **Tokens in AsyncStorage** — readable from unencrypted backups and on rooted devices.
- **Big JSON blobs in AsyncStorage** (an entire history as one key) — slow to read, and Android has
  size limits on its implementation; move to SQLite.
- **Editing a shipped migration** — existing users never run the edit, and new users get a different
  schema from old ones.
- **String-building SQL** with user input.
- **Assuming storage reads are instant** — show the UI after `onInit` completes, or a splash screen.
- **Forgetting `foreign_keys = ON`** — SQLite ignores `REFERENCES ... ON DELETE CASCADE` without it.

## Exercise

1. Move the habit tracker's data to SQLite using the migrations above.
2. Add migration v4: a `notes` table (`habit_id`, `day`, `text`) for a note per check-in, and show
   notes on the detail screen. Confirm an app that already ran v1–v3 upgrades cleanly (don't delete
   the app between runs).
3. Store preferences (reminder hour, week start) with AsyncStorage and load them before the first
   render, with a defaults merge.
4. Write a query that returns each habit's check-in count for the last 7 days in one SQL statement
   (hint: `LEFT JOIN` + `GROUP BY` + `day >= ?`).
