---
description: "Project: a Field Notes app that captures geotagged photo notes offline into SQLite and syncs them with retries and backoff when the network returns."
---

# 10 · Project — Field Notes with Camera, Location & Offline Sync

Field researchers, inspectors, delivery drivers and hikers share a problem: they need to capture
information where the signal is worst. This project builds **Field Notes**, an offline-first app:
write a note, attach a photo, tag it with your location, and keep working with no network. Notes
queue locally and sync to a server automatically — with retries and backoff — when connectivity
returns.

It pulls together Level 3: device APIs (camera, location, haptics), permissions done right, a
development build with config plugins, file storage, and app lifecycle — plus Level 2's SQLite.

## Requirements

- Note list (newest first) with a sync badge per note: *pending*, *synced*, *failed (retrying)*.
- New-note screen: text, optional photo from the camera, optional location tag (with an honest
  "approximate" label when only a stale fix was available).
- Photos are copied out of the cache into the app's document directory so the OS can't delete them.
- Everything is saved locally **first**; syncing never blocks saving.
- Sync runs when the app becomes active and when the network comes back; failures retry with
  exponential backoff; a note is never uploaded twice for the same version.
- Works with camera or location permission denied — those parts are simply unavailable.

## Setup

```bash
npx create-expo-app@latest field-notes
cd field-notes
npx expo install expo-camera expo-location expo-haptics expo-sqlite expo-file-system \
  @react-native-community/netinfo expo-dev-client
```

Add the permission strings through config plugins (lesson 3) and build a development build
(lesson 9):

```json title="app.json (excerpt)"
{
  "expo": {
    "scheme": "fieldnotes",
    "plugins": [
      "expo-router",
      ["expo-camera", { "cameraPermission": "Field Notes uses the camera so you can attach a photo to a note." }],
      ["expo-location", { "locationWhenInUsePermission": "Field Notes tags each note with where you wrote it." }]
    ]
  }
}
```

The camera needs a physical device. Location can be simulated on a simulator or emulator.

## Step 1 — schema

```ts title="src/db.ts"
import type { SQLiteDatabase } from 'expo-sqlite';

export type SyncState = 'pending' | 'synced' | 'failed';

export type NoteRow = {
  id: string;
  body: string;
  created_at: number;
  updated_at: number;
  latitude: number | null;
  longitude: number | null;
  accuracy: number | null;
  place: string | null;
  location_stale: number;     // 0/1
  photo_uri: string | null;
  sync_state: SyncState;
  attempts: number;
  next_attempt_at: number;    // epoch ms; 0 = as soon as possible
  synced_version: number;     // updated_at value the server has
};

const MIGRATIONS = [
  `CREATE TABLE notes (
     id TEXT PRIMARY KEY NOT NULL,
     body TEXT NOT NULL,
     created_at INTEGER NOT NULL,
     updated_at INTEGER NOT NULL,
     latitude REAL, longitude REAL, accuracy REAL, place TEXT,
     location_stale INTEGER NOT NULL DEFAULT 0,
     photo_uri TEXT,
     sync_state TEXT NOT NULL DEFAULT 'pending',
     attempts INTEGER NOT NULL DEFAULT 0,
     next_attempt_at INTEGER NOT NULL DEFAULT 0,
     synced_version INTEGER NOT NULL DEFAULT 0
   );
   CREATE INDEX idx_notes_sync ON notes(sync_state, next_attempt_at);`,
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

`synced_version` stores which edit the server has. If a note is edited after syncing,
`updated_at > synced_version` and it becomes pending again — that's how the app avoids both lost
edits and duplicate uploads.

## Step 2 — the sync policy, as pure functions

The decisions — *which notes to send now* and *when to retry* — are pure functions, so they can be
tested without a phone, a network or a database.

```ts title="src/syncPolicy.ts"
import type { NoteRow } from './db';

export const MAX_BACKOFF_MS = 30 * 60_000;

/** 30s, 60s, 2min, 4min… capped at 30 minutes. */
export function backoffMs(attempts: number): number {
  if (attempts <= 0) return 0;
  return Math.min(30_000 * 2 ** (attempts - 1), MAX_BACKOFF_MS);
}

export function needsSync(n: Pick<NoteRow, 'updated_at' | 'synced_version'>): boolean {
  return n.updated_at > n.synced_version;
}

/** Notes that need syncing and whose retry time has arrived, oldest first, at most `limit`. */
export function pickBatch(notes: NoteRow[], now: number, limit = 10): NoteRow[] {
  return notes
    .filter((n) => needsSync(n) && n.next_attempt_at <= now)
    .sort((a, b) => a.created_at - b.created_at)
    .slice(0, limit);
}

export type Outcome = { ok: true } | { ok: false; retryable: boolean };

/** Fields to write back after one upload attempt. */
export function afterAttempt(n: NoteRow, outcome: Outcome, now: number): Partial<NoteRow> {
  if (outcome.ok) {
    return { sync_state: 'synced', attempts: 0, next_attempt_at: 0, synced_version: n.updated_at };
  }
  const attempts = n.attempts + 1;
  return {
    sync_state: 'failed',
    attempts,
    // Non-retryable (e.g. HTTP 400) waits the maximum; a person should look at it.
    next_attempt_at: now + (outcome.retryable ? backoffMs(attempts) : MAX_BACKOFF_MS),
  };
}
```

These tests were run with Jest in the course's SDK 57 check project and passed:

```ts title="src/syncPolicy.test.ts"
import { afterAttempt, backoffMs, pickBatch, MAX_BACKOFF_MS } from './syncPolicy';
import type { NoteRow } from './db';

const note = (over: Partial<NoteRow>): NoteRow => ({
  id: 'n', body: '', created_at: 0, updated_at: 1, latitude: null, longitude: null, accuracy: null,
  place: null, location_stale: 0, photo_uri: null, sync_state: 'pending', attempts: 0,
  next_attempt_at: 0, synced_version: 0, ...over,
});

test('backoff doubles and caps', () => {
  expect([0, 1, 2, 3, 4].map(backoffMs)).toEqual([0, 30_000, 60_000, 120_000, 240_000]);
  expect(backoffMs(20)).toBe(MAX_BACKOFF_MS);
});

test('pickBatch skips synced notes and notes still backing off, oldest first', () => {
  const now = 1_000_000;
  const notes = [
    note({ id: 'synced', updated_at: 5, synced_version: 5 }),
    note({ id: 'waiting', created_at: 1, next_attempt_at: now + 1 }),
    note({ id: 'newer', created_at: 30 }),
    note({ id: 'older', created_at: 10 }),
    note({ id: 'edited-after-sync', created_at: 20, updated_at: 9, synced_version: 5 }),
  ];
  expect(pickBatch(notes, now).map((n) => n.id)).toEqual(['older', 'edited-after-sync', 'newer']);
});

test('success records the synced version; failure schedules a retry', () => {
  const n = note({ updated_at: 42, attempts: 2 });
  expect(afterAttempt(n, { ok: true }, 0)).toMatchObject({ sync_state: 'synced', synced_version: 42, attempts: 0 });
  expect(afterAttempt(n, { ok: false, retryable: true }, 1000)).toMatchObject({ sync_state: 'failed', attempts: 3, next_attempt_at: 1000 + 120_000 });
});
```

## Step 3 — keep photos safe

```ts title="src/photos.ts"
import { Directory, File, Paths } from 'expo-file-system';

const photosDir = new Directory(Paths.document, 'photos');

/** Copies a camera capture from the cache into permanent app storage and returns its URI. */
export function persistPhoto(cacheUri: string, noteId: string): string {
  if (!photosDir.exists) photosDir.create({ intermediates: true });
  const target = new File(photosDir, `${noteId}.jpg`);
  if (target.exists) target.delete();
  new File(cacheUri).copy(target);
  return target.uri;
}
```

## Step 4 — repository and sync engine

```ts title="src/notesRepo.ts"
import type { SQLiteDatabase } from 'expo-sqlite';
import type { NoteRow } from './db';

export function listNotes(db: SQLiteDatabase) {
  return db.getAllAsync<NoteRow>('SELECT * FROM notes ORDER BY created_at DESC');
}

export async function insertNote(db: SQLiteDatabase, n: Omit<NoteRow, 'sync_state' | 'attempts' | 'next_attempt_at' | 'synced_version'>) {
  await db.runAsync(
    `INSERT INTO notes (id, body, created_at, updated_at, latitude, longitude, accuracy, place, location_stale, photo_uri)
     VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, ?)`,
    n.id, n.body, n.created_at, n.updated_at, n.latitude, n.longitude, n.accuracy, n.place, n.location_stale, n.photo_uri,
  );
}

export async function applyPatch(db: SQLiteDatabase, id: string, patch: Partial<NoteRow>) {
  const keys = Object.keys(patch) as (keyof NoteRow)[];
  if (keys.length === 0) return;
  const sets = keys.map((k) => `${k} = ?`).join(', ');      // keys come from our own code, never user input
  await db.runAsync(`UPDATE notes SET ${sets} WHERE id = ?`, ...keys.map((k) => patch[k] as string | number | null), id);
}
```

```ts title="src/sync.ts"
import type { SQLiteDatabase } from 'expo-sqlite';
import { listNotes, applyPatch } from './notesRepo';
import { afterAttempt, pickBatch, type Outcome } from './syncPolicy';
import type { NoteRow } from './db';

const API = process.env.EXPO_PUBLIC_API_URL; // e.g. http://192.168.1.20:4000 during development

async function upload(note: NoteRow): Promise<Outcome> {
  if (!API) return { ok: false, retryable: true };
  try {
    const res = await fetch(`${API}/notes/${encodeURIComponent(note.id)}`, {
      method: 'PUT', // idempotent: sending the same version twice is harmless
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        id: note.id, body: note.body, createdAt: note.created_at, version: note.updated_at,
        location: note.latitude == null ? null : { lat: note.latitude, lng: note.longitude, accuracy: note.accuracy, place: note.place },
        hasPhoto: !!note.photo_uri,
      }),
    });
    if (res.ok) return { ok: true };
    return { ok: false, retryable: res.status >= 500 || res.status === 429 };
  } catch {
    return { ok: false, retryable: true }; // network error
  }
}

let running = false;

/** Uploads due notes. Safe to call often: concurrent calls are ignored. */
export async function syncNow(db: SQLiteDatabase): Promise<number> {
  if (running) return 0;
  running = true;
  let sent = 0;
  try {
    const batch = pickBatch(await listNotes(db), Date.now());
    for (const note of batch) {
      const outcome = await upload(note);
      await applyPatch(db, note.id, afterAttempt(note, outcome, Date.now()));
      if (outcome.ok) sent += 1;
    }
  } finally {
    running = false;
  }
  return sent;
}
```

Using `PUT /notes/:id` with the note's own ID makes uploads **idempotent**: if the app crashes after
the server stored a note but before the app recorded success, the retry overwrites the same record
instead of creating a duplicate. Photo upload is left as an exercise (multipart upload with
`File.upload` or a presigned URL); the payload marks `hasPhoto` so a server knows one is coming.

## Step 5 — trigger sync on reconnect and foreground

```ts title="src/useAutoSync.ts"
import { useEffect } from 'react';
import { AppState } from 'react-native';
import NetInfo from '@react-native-community/netinfo';
import { useSQLiteContext } from 'expo-sqlite';
import { syncNow } from './sync';

export function useAutoSync(onSynced: () => void) {
  const db = useSQLiteContext();
  useEffect(() => {
    const run = () => syncNow(db).then((n) => { if (n > 0) onSynced(); });
    run();
    const netSub = NetInfo.addEventListener((s) => { if (s.isConnected && s.isInternetReachable !== false) run(); });
    const appSub = AppState.addEventListener('change', (s) => { if (s === 'active') run(); });
    const timer = setInterval(run, 60_000); // picks up notes whose backoff expired
    return () => { netSub(); appSub.remove(); clearInterval(timer); };
  }, [db, onSynced]);
}
```

## Step 6 — screens

```tsx title="app/_layout.tsx"
import { Stack } from 'expo-router';
import { SQLiteProvider } from 'expo-sqlite';
import { migrate } from '../src/db';

export default function RootLayout() {
  return (
    <SQLiteProvider databaseName="fieldnotes.db" onInit={migrate}>
      <Stack>
        <Stack.Screen name="index" options={{ title: 'Field Notes' }} />
        <Stack.Screen name="new" options={{ title: 'New note', presentation: 'modal' }} />
      </Stack>
    </SQLiteProvider>
  );
}
```

```tsx title="app/index.tsx"
import { useCallback, useState } from 'react';
import { FlatList, Pressable, Text, View } from 'react-native';
import { router, useFocusEffect } from 'expo-router';
import { useSQLiteContext } from 'expo-sqlite';
import { Image } from 'expo-image';
import { listNotes } from '../src/notesRepo';
import { needsSync } from '../src/syncPolicy';
import { useAutoSync } from '../src/useAutoSync';
import type { NoteRow } from '../src/db';

function Badge({ note }: { note: NoteRow }) {
  const label = !needsSync(note) ? 'Synced' : note.sync_state === 'failed' ? 'Retrying' : 'Pending';
  const color = label === 'Synced' ? '#16a34a' : label === 'Retrying' ? '#b45309' : '#64748b';
  return <Text style={{ color, fontSize: 12, fontWeight: '700' }}>{label}</Text>;
}

export default function NotesList() {
  const db = useSQLiteContext();
  const [notes, setNotes] = useState<NoteRow[]>([]);
  const reload = useCallback(() => { listNotes(db).then(setNotes); }, [db]);

  useFocusEffect(reload);
  useAutoSync(reload);

  return (
    <View style={{ flex: 1 }}>
      <FlatList
        data={notes}
        keyExtractor={(n) => n.id}
        contentContainerStyle={{ padding: 12, gap: 10 }}
        ListEmptyComponent={<Text style={{ padding: 24, textAlign: 'center' }}>No notes yet. Tap + to capture one — it works offline.</Text>}
        renderItem={({ item }) => (
          <View style={{ flexDirection: 'row', gap: 12, backgroundColor: 'white', borderRadius: 12, padding: 12 }}
            accessible accessibilityLabel={`${item.body}. ${item.place ?? 'No location'}. ${needsSync(item) ? 'Not yet synced' : 'Synced'}`}>
            {item.photo_uri && <Image source={{ uri: item.photo_uri }} style={{ width: 56, height: 56, borderRadius: 8 }} />}
            <View style={{ flex: 1, gap: 2 }}>
              <Text numberOfLines={2} style={{ fontSize: 16 }}>{item.body}</Text>
              <Text style={{ color: '#64748b', fontSize: 13 }}>
                {new Date(item.created_at).toLocaleString()}
                {item.place ? ` · ${item.place}${item.location_stale ? ' (approx.)' : ''}` : ''}
              </Text>
              <Badge note={item} />
            </View>
          </View>
        )}
      />
      <Pressable onPress={() => router.push('/new')} accessibilityRole="button" accessibilityLabel="New note"
        style={{ position: 'absolute', right: 20, bottom: 32, width: 60, height: 60, borderRadius: 30, backgroundColor: '#0f766e', alignItems: 'center', justifyContent: 'center' }}>
        <Text style={{ color: 'white', fontSize: 30 }}>+</Text>
      </Pressable>
    </View>
  );
}
```

```tsx title="app/new.tsx"
import { useState } from 'react';
import { Button, KeyboardAvoidingView, Modal, Platform, ScrollView, Text, TextInput } from 'react-native';
import { router } from 'expo-router';
import { useSQLiteContext } from 'expo-sqlite';
import * as Haptics from 'expo-haptics';
import { Image } from 'expo-image';
import { CaptureScreen } from '../src/CaptureScreen';  // from lesson 2
import { locateForNote, type Tag } from '../src/locate'; // from lesson 2
import { persistPhoto } from '../src/photos';
import { insertNote } from '../src/notesRepo';

const newId = () => `${Date.now().toString(36)}-${Math.random().toString(36).slice(2, 8)}`;

export default function NewNote() {
  const db = useSQLiteContext();
  const [body, setBody] = useState('');
  const [photo, setPhoto] = useState<string | null>(null);
  const [tag, setTag] = useState<Tag | null>(null);
  const [locating, setLocating] = useState(false);
  const [cameraOpen, setCameraOpen] = useState(false);

  async function addLocation() {
    setLocating(true);
    setTag(await locateForNote()); // null if permission missing or no fix
    setLocating(false);
  }

  async function save() {
    if (!body.trim()) return;
    const id = newId();
    const now = Date.now();
    await insertNote(db, {
      id, body: body.trim(), created_at: now, updated_at: now,
      latitude: tag?.latitude ?? null, longitude: tag?.longitude ?? null, accuracy: tag?.accuracy ?? null,
      place: tag?.label ?? null, location_stale: tag?.stale ? 1 : 0,
      photo_uri: photo ? persistPhoto(photo, id) : null,
    });
    Haptics.notificationAsync(Haptics.NotificationFeedbackType.Success);
    router.back();
  }

  return (
    <KeyboardAvoidingView style={{ flex: 1 }} behavior={Platform.OS === 'ios' ? 'padding' : undefined}>
      <ScrollView contentContainerStyle={{ padding: 16, gap: 12 }} keyboardShouldPersistTaps="handled">
        <TextInput value={body} onChangeText={setBody} placeholder="What did you observe?" multiline
          style={{ minHeight: 120, fontSize: 16, textAlignVertical: 'top', backgroundColor: 'white', borderRadius: 10, padding: 12 }} />
        {photo && <Image source={{ uri: photo }} style={{ width: '100%', aspectRatio: 4 / 3, borderRadius: 12 }} />}
        <Button title={photo ? 'Retake photo' : 'Add photo'} onPress={() => setCameraOpen(true)} />
        <Button title={locating ? 'Locating…' : tag ? 'Update location' : 'Add location'} onPress={addLocation} disabled={locating} />
        {tag && <Text>{tag.label ?? `${tag.latitude.toFixed(4)}, ${tag.longitude.toFixed(4)}`}{tag.stale ? ' (approximate)' : ''}</Text>}
        {tag === null && !locating && <Text style={{ color: '#64748b' }}>Location is optional.</Text>}
        <Button title="Save note" onPress={save} disabled={!body.trim()} />
      </ScrollView>
      <Modal visible={cameraOpen} animationType="slide" onRequestClose={() => setCameraOpen(false)}>
        <CaptureScreen onCaptured={(uri) => { setPhoto(uri); setCameraOpen(false); }} />
      </Modal>
    </KeyboardAvoidingView>
  );
}
```

## Step 7 — a tiny server to sync against

You need something to sync *to*. For development, this small Node server (no dependencies) stores
notes in memory and is enough to watch the sync engine work:

```js title="server/index.mjs"
import { createServer } from 'node:http';

const notes = new Map();

createServer(async (req, res) => {
  const match = req.url?.match(/^\/notes\/([\w-]+)$/);
  if (req.method === 'PUT' && match) {
    let raw = '';
    for await (const chunk of req) raw += chunk;
    let note;
    try { note = JSON.parse(raw); } catch { res.writeHead(400).end('bad json'); return; }
    const existing = notes.get(match[1]);
    const newer = !existing || existing.version <= note.version;
    if (newer) notes.set(match[1], note); // a repeat of the same version is harmless; older ones are ignored
    console.log(`${newer ? 'stored' : 'ignored stale'} ${match[1]} v${note.version} (${notes.size} notes)`);
    res.writeHead(200, { 'Content-Type': 'application/json' }).end(JSON.stringify({ ok: true }));
    return;
  }
  if (req.method === 'GET' && req.url === '/notes') {
    res.writeHead(200, { 'Content-Type': 'application/json' }).end(JSON.stringify([...notes.values()]));
    return;
  }
  res.writeHead(404).end();
}).listen(4000, () => console.log('listening on :4000'));
```

Run it with `node server/index.mjs` and start the app with your computer's LAN address
(`EXPO_PUBLIC_API_URL=http://192.168.1.20:4000 npx expo start`). When this course ran the server and
sent the same version-2 `PUT` twice with `curl`, then an older version 1, the server logged
`stored n1 v2 (1 notes)` twice, then `ignored stale n1 v1 (1 notes)`, and `GET /notes` still
returned version 2 — the idempotency the app relies on. A body that isn't JSON got a `400`.

## Verify it

1. Airplane mode on. Create three notes, one with a photo and one with a location. All show
   **Pending**.
2. Fully close and reopen the app — notes and photos are still there.
3. Airplane mode off. Within moments the badges change to **Synced** and the server logs each note.
4. Stop the server, create a note, and watch it go to **Retrying**; restart the server and it syncs on
   a later attempt (up to the backoff interval, or immediately when you foreground the app).
5. Deny location permission and confirm the note can still be saved.

## How It Actually Works

Saving is a **local transaction**: the note goes into SQLite and the photo file into the document
directory before anything touches the network. Sync is a separate loop driven by three triggers —
NetInfo's connectivity events, `AppState` becoming `active`, and a one-minute timer — all calling
`syncNow`, which uses a module-level `running` flag so overlapping triggers don't upload the same
note twice at once. `pickBatch` selects notes where `updated_at > synced_version` and whose
`next_attempt_at` has passed; each upload result becomes a patch via `afterAttempt`, so a crash
mid-batch loses at most one attempt's bookkeeping, and the idempotent `PUT` makes repeating that
attempt harmless. While the app is in the background, nothing runs — mobile OSes suspend apps
quickly — which is why foregrounding triggers a sync. (Truly background sync uses
`expo-background-task`, where the OS decides when, and how rarely, your task runs.)

## Common mistakes

- **Syncing before saving** — a failed request loses the user's note.
- **Non-idempotent uploads** (`POST /notes` creating a new record each time) — retries create
  duplicates.
- **Retrying immediately in a loop** — drains battery and hammers the server; back off.
- **Leaving photos in the cache directory.**
- **Treating every error the same** — a 400 won't fix itself by retrying.
- **Assuming `isConnected` means the server is reachable** — captive portals and dead Wi-Fi exist;
  the sync result is the real test.

## Exercise

1. Add **photo upload**: after the JSON `PUT` succeeds, upload the image with `new File(uri).upload(...)`
   to `PUT /notes/:id/photo`, and track photo sync separately so a failed photo upload doesn't mark the
   note itself as unsynced.
2. Add **editing**: changing a synced note's text bumps `updated_at`, turns the badge back to Pending,
   and re-syncs. Write a test for `needsSync` covering this case.
3. Use the `usePowerSave` hook from lesson 8 (if you built it) to pause the one-minute timer while
   power saving is on.
4. Add a "Sync now" button with pull-to-refresh that calls `syncNow` and shows how many notes were sent.
