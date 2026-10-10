---
description: "Pass data between screens with dynamic segments and search params, open screens from custom-scheme and universal links, and enable typed routes."
---

# 03 · Route Params, Deep Links & Typed Routes

Every Expo Router screen has a URL. That one fact gives you three features: passing data between
screens (params in the URL), opening any screen from outside the app (deep links), and type-safe
navigation (the router knows every valid URL at build time). This lesson covers all three, plus the
judgement call of *what* belongs in a URL.

## Dynamic segments and search params

```
app/habit/[id].tsx              → /habit/42
app/habit/[id]/history.tsx      → /habit/42/history
app/tag/[...path].tsx           → /tag/health/sleep   (catch-all: path = ['health', 'sleep'])
```

Read them with `useLocalSearchParams`:

```tsx
import { useLocalSearchParams } from 'expo-router';

export default function HabitHistory() {
  const { id, range } = useLocalSearchParams<{ id: string; range?: '7d' | '30d' }>();
  // /habit/42/history?range=30d  →  id = '42', range = '30d'
}
```

Dynamic segments (`[id]`) and query-string values (`?range=30d`) arrive in the same object. Navigate
with either a string or an object:

```tsx
router.push(`/habit/${id}/history?range=30d`);
router.push({ pathname: '/habit/[id]/history', params: { id, range: '30d' } });
```

The object form handles URL encoding for you — use it whenever a value comes from user input.

!!! warning "Params are strings"
    Everything in a URL is a string. `params: { count: 3 }` arrives as `'3'`. The type parameter on
    `useLocalSearchParams<…>()` is a promise you make to TypeScript, not a runtime check — parse
    and validate values you depend on.

```tsx
const params = useLocalSearchParams<{ id: string; range?: string }>();
const range = params.range === '30d' ? '30d' : '7d'; // default anything unexpected
const id = Number.parseInt(params.id, 10);
if (!Number.isFinite(id)) return <NotFound />;
```

### Local vs global params

`useLocalSearchParams` returns params for **this screen's** route and only updates when this screen
is focused. `useGlobalSearchParams` returns params for whichever route is currently focused anywhere
in the app, and re-renders on every navigation. Screens stay mounted in stacks (lesson 1), so with
global params a background screen re-renders whenever the user navigates — use local params unless
you have a specific reason.

## What belongs in a URL

Put **identifiers and view state** in params: an ID, a tab, a filter, a sort order. Don't put
**objects** in params (a whole habit serialised to JSON): it bloats URLs, goes stale, and breaks
when the screen is opened from a deep link that doesn't have the object. Pass the ID; have the
detail screen read the habit from your store or cache (lessons 4 and 5).

## Deep links

### Custom scheme links

The `scheme` in `app.json` (`"scheme": "habitapp"`) registers a URL scheme with the OS. Then
`habitapp://habit/42` opens the app on that route. Test it without writing code:

```bash
# iOS Simulator
xcrun simctl openurl booted "habitapp://habit/42"

# Android emulator or USB device
adb shell am start -W -a android.intent.action.VIEW -d "habitapp://habit/42"

# Either, via the Expo CLI helper (prints the right command for Expo Go vs a dev build)
npx uri-scheme open "habitapp://habit/42" --ios
```

In Expo Go, your app runs inside Expo Go's own scheme (`exp://…`), so custom-scheme links to your
app need a development build (Level 3, lesson 9). Links you build in code with
`Linking.createURL('/habit/42')` (from `expo-linking`) produce the right form for whichever
environment you're running in.

### Universal links / App Links

Custom schemes have weaknesses: any app can claim the same scheme, and they don't work if the app
isn't installed. **Universal links** (iOS) and **Android App Links** use ordinary `https://` URLs
on a domain you own. If the app is installed, the OS opens it; otherwise the link opens your
website. Setting them up requires:

1. Hosting verification files on your domain:
   `https://example.com/.well-known/apple-app-site-association` (iOS) and
   `https://example.com/.well-known/assetlinks.json` (Android, containing your signing
   certificate's SHA-256 fingerprint).
2. Declaring the domain in `app.json`:

```json title="app.json (excerpt)"
{
  "expo": {
    "ios": { "associatedDomains": ["applinks:example.com"] },
    "android": {
      "intentFilters": [{
        "action": "VIEW",
        "autoVerify": true,
        "data": [{ "scheme": "https", "host": "example.com", "pathPrefix": "/habit" }],
        "category": ["BROWSABLE", "DEFAULT"]
      }]
    }
  }
}
```

This needs your own domain, an Apple Developer team ID and a signed build, so it can't be done in
Expo Go. With Expo Router, once the OS hands your app `https://example.com/habit/42`, the path
`/habit/42` is matched against your routes exactly like an internal link.

## Typed routes

Turn on typed routes in `app.json`:

```json title="app.json (excerpt)"
{ "expo": { "experiments": { "typedRoutes": true } } }
```

When you run `npx expo start`, Expo generates a type declaration listing every route, and `href`
props and `router.push` become type-checked:

```tsx
router.push('/habit/42');                                        // ✅
router.push('/habits/42');                                       // ❌ type error: no such route
router.push({ pathname: '/habit/[id]', params: { id: '42' } });  // ✅
router.push({ pathname: '/habit/[id]', params: {} });            // ❌ missing id
```

Renaming a route file then turns every stale link in the codebase into a compile error rather than a
"page not found" in production.

## Worked example: a shareable, validated detail route

```tsx title="app/habit/[id]/history.tsx"
import { Link, Stack, useLocalSearchParams } from 'expo-router';
import { Share, Pressable, Text, View } from 'react-native';
import * as Linking from 'expo-linking';

const RANGES = ['7d', '30d', '365d'] as const;
type Range = (typeof RANGES)[number];

function parseRange(value: string | undefined): Range {
  return (RANGES as readonly string[]).includes(value ?? '') ? (value as Range) : '7d';
}

export default function HabitHistory() {
  const params = useLocalSearchParams<{ id: string; range?: string }>();
  const range = parseRange(params.range);
  const id = params.id;

  async function share() {
    const url = Linking.createURL(`/habit/${id}/history`, { queryParams: { range } });
    await Share.share({ message: `My habit history: ${url}` });
  }

  return (
    <View style={{ flex: 1, padding: 16, gap: 12 }}>
      <Stack.Screen options={{ title: `History · ${range}` }} />
      <View style={{ flexDirection: 'row', gap: 8 }}>
        {RANGES.map((r) => (
          <Link key={r} href={{ pathname: '/habit/[id]/history', params: { id, range: r } }} replace asChild>
            <Pressable
              accessibilityRole="tab"
              accessibilityState={{ selected: r === range }}
              style={{ paddingHorizontal: 14, paddingVertical: 8, borderRadius: 999, backgroundColor: r === range ? '#4f46e5' : '#e2e8f0' }}
            >
              <Text style={{ color: r === range ? 'white' : '#0f172a', fontWeight: '600' }}>{r}</Text>
            </Pressable>
          </Link>
        ))}
      </View>
      <Text>Showing {range} of history for habit {id}.</Text>
      <Pressable onPress={share} accessibilityRole="button"><Text style={{ color: '#4f46e5' }}>Share this view</Text></Pressable>
    </View>
  );
}
```

Two decisions to notice: the range lives in the URL, so a shared link reopens the same view; and
switching range uses `replace`, so tapping 7d → 30d → 365d doesn't fill the back stack with three
history screens.

## How It Actually Works

When a URL arrives — from the OS at launch (`Linking.getInitialURL()`), from the OS while running
(a `url` event), or from your own `router.push` — Expo Router runs the same pipeline:

1. Strip the scheme and host, leaving a path and query (`/habit/42/history?range=30d`).
2. Match the path against the route tree built from `app/`: static segments first, then dynamic
   `[param]` segments, then catch-alls, so `/habit/new` matches `habit/new.tsx` before
   `habit/[id].tsx` if both exist.
3. Build a **navigation state** for that match, including all parent navigators — the root stack,
   the group, the nested stack — so the screen opens with a sensible back stack. A layout can export
   `unstable_settings = { initialRouteName: 'index' }` to make sure a deep-linked detail screen has
   its list underneath it.
4. Dispatch that state (or the difference from the current state) to the root navigator.

Typed routes are generated by scanning the same `app/` directory: Expo writes a `.d.ts` file
(in the `.expo/types` folder, included by `expo-env.d.ts`) declaring a union of every valid static
path and every dynamic pathname with its required params. Nothing changes at runtime; it's purely a
compile-time contract.

## Common mistakes

- **Trusting param types** — `useLocalSearchParams<{ id: number }>()` still gives you a string.
- **Serialising objects into params** — pass IDs.
- **`useGlobalSearchParams` everywhere** — needless re-renders of background screens.
- **Testing deep links only in Expo Go** — scheme behaviour differs; use a development build.
- **Deep-linked detail screen with no way back** — set `initialRouteName` so a parent screen exists.
- **Forgetting that universal links need the verification file served over HTTPS** with the correct
  content type and no redirects.

## Exercise

1. Add `app/habit/[id]/edit.tsx` and a "tags" catch-all route `app/tag/[...path].tsx` that shows the
   path segments as breadcrumbs.
2. Turn on `typedRoutes`, restart Metro, and deliberately break a link to see the type error.
3. Open `habitapp://habit/2/history?range=30d` on a simulator or emulator in a development build (or,
   if you only have Expo Go, use `Linking.createURL` to print the Expo Go URL and open that). Confirm
   the range tabs reflect the URL and that back goes to a sensible screen.
