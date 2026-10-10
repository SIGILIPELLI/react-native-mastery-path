---
description: "Debug React Native apps: the dev menu, React Native DevTools breakpoints and inspector, LogBox, network inspection and native crash logs."
---

# 09 · Debugging React Native Apps

React Native bugs come from more places than web bugs: your JavaScript, the React tree, the
network, the native layer underneath, and the gap between "works in development" and "works in a
release build." A good debugging routine checks them in that order, using the right tool for
each.

## The dev menu

In a development build or Expo Go, open the **dev menu**:

- **Physical device:** shake it.
- **iOS Simulator:** `Ctrl + Cmd + Z` (or Device → Shake).
- **Android emulator:** `Cmd + M` (macOS) / `Ctrl + M` (Windows/Linux).
- **From the terminal running `npx expo start`:** press `m`.

From there you can reload, toggle the **element inspector** (tap any element to see its styles and
box model), open the **performance monitor** (JS and UI frame rates), and open the debugger.

## React Native DevTools

Modern React Native ships **React Native DevTools**, a Chrome DevTools–based debugger that connects
to the Hermes engine in your app. Start it by pressing **`j`** in the terminal running Expo. You get:

- **Console** — your `console.log` output with object inspection (logs also print in the terminal).
- **Sources** — set breakpoints in your TypeScript source (source maps are wired up), step through,
  inspect scope.
- **Components** and **Profiler** — the React DevTools panels, to inspect props, state, hooks and
  render timings.
- **Memory** — heap snapshots, for chasing leaks (Level 4).

Use `debugger;` as a hard breakpoint while experimenting:

```tsx
function computeStreak(days: string[]) {
  debugger; // pauses here when DevTools is attached
  // ...
}
```

The older "Remote JS Debugging" mode (running your JS in Chrome instead of on the device) is gone
from current React Native; it changed timing and behaviour enough to hide real bugs.

## LogBox: warnings and errors in-app

Development builds show **LogBox**: yellow toasts for warnings, a full-screen red overlay for
uncaught errors and render errors. The error overlay shows a **component stack**, which is usually
more useful than the JS stack:

```
Error: Text strings must be rendered within a <Text> component.

This error is located at:
    in RCTView (created by View)
    in View (created by HabitRow)
    in HabitRow (created by HabitList)
```

Read it bottom-up: `HabitList` rendered `HabitRow`, whose `View` received a bare string.

You can silence known, third-party noise — narrowly:

```tsx
import { LogBox } from 'react-native';

LogBox.ignoreLogs(['Some specific library warning text']);
```

Don't use `LogBox.ignoreAllLogs()` to make warnings "go away" — warnings in React Native are
usually telling you about a real performance or correctness issue.

## Network inspection

When data looks wrong, first prove what the server actually sent. React Native DevTools includes a
Network panel in recent versions; failing that, log at the boundary:

```ts title="api.ts"
export async function getJSON<T>(url: string): Promise<T> {
  const started = Date.now();
  const res = await fetch(url, { headers: { Accept: 'application/json' } });
  if (__DEV__) console.log(`[net] ${res.status} ${url} (${Date.now() - started} ms)`);
  if (!res.ok) {
    const body = await res.text();
    throw new Error(`HTTP ${res.status} for ${url}: ${body.slice(0, 200)}`);
  }
  return (await res.json()) as T;
}
```

`__DEV__` is a global boolean that's `true` in development bundles and `false` in production ones,
so this logging disappears from release builds. Two network gotchas specific to mobile:

- **`localhost` on a phone is the phone.** To reach a dev server on your laptop, use the laptop's LAN
  IP (and on the Android emulator, `10.0.2.2` maps to the host machine).
- **Plain `http://` is blocked by default** on iOS (App Transport Security) and on modern Android for
  release builds. Use HTTPS, or configure exceptions only for development.

## Worked example: a debugging session

Symptom: tapping a habit sometimes toggles the *wrong* habit after searching.

1. **Reproduce reliably.** Search "read", tap the first result, clear the search. The checkmark is on
   a different row.
2. **Inspect state.** Open DevTools (`j`) → Components → `HabitList`. The `done` set contains the
   correct ID. So data is right; rendering is wrong.
3. **Inspect the row.** Select a `HabitRow` and look at its `key` in the Components panel: it's
   `"0"`, `"1"`, … — indexes, not IDs.
4. **Hypothesis:** each row keeps local `done` state; with index keys, React reuses row 0's state
   for whichever habit is at index 0 after filtering.
5. **Fix:** `keyExtractor={(h) => h.id}` — and better, lift `done` out of the row into the list's
   state keyed by ID.
6. **Prove it.** Repeat step 1. Add a regression test later (Level 4).

The pattern is general: reproduce → find whether **data** or **rendering** is wrong → narrow with
the right tool → fix the cause, not the symptom → prove the fix.

## When the JavaScript tools show nothing: native crashes

If the app closes instantly with no red screen, the crash is native (a missing native module, a bad
config, a permission string missing from `Info.plist`). Look at native logs:

```bash
# Android: stream device logs, filtered to errors and React Native
adb logcat '*:E' ReactNative:V ReactNativeJS:V

# iOS Simulator: stream the simulator's system log for your app's process
xcrun simctl spawn booted log stream --level error --predicate 'process == "YourAppName"'
```

On a physical iPhone, use the Console app on macOS (select the device) or Xcode's
Devices and Simulators window. A classic Expo case: using a module that isn't in Expo Go or not
rebuilding your development build after adding a native package — the error mentions a native module
that "cannot be found". The fix is to rebuild, not to debug JavaScript.

## How It Actually Works

React Native DevTools talks to Hermes over the **Chrome DevTools Protocol (CDP)**. When you press
`j`, the Expo CLI's dev server (built on Metro) acts as an **inspector proxy**: the app connects
to Metro over a WebSocket, DevTools connects to Metro, and Metro relays CDP messages between them.
Hermes implements the debugger domain of CDP — breakpoints, stepping, scopes — directly in the
engine running on your phone, so you debug the same JavaScript, on the same engine, with the same
timing your users get. Source maps produced by Metro map Hermes positions back to your `.tsx` lines.

LogBox is plain React: in development, React Native patches `console.error`/`console.warn` and the
global error handler (`ErrorUtils`) so that messages are collected and rendered as an overlay
component above your app. In production builds there is no LogBox — an uncaught JS error crashes
the app (or is caught by your error boundary and reported to a crash service, Level 4).

## Common mistakes

- **Debugging a release-only bug in development.** `__DEV__` checks, minification and missing dev
  tools change behaviour; reproduce with `npx expo run:ios --configuration Release` or a preview
  build.
- **`console.log` of huge objects in a list's render** — floods the bridge to DevTools and slows
  the app, making performance bugs worse while you investigate them.
- **Ignoring yellow warnings**, especially key warnings and "VirtualizedLists should never be
  nested".
- **Using `localhost` for an API on a physical device.**
- **Assuming a native crash is a JS bug** and adding try/catch everywhere.

## Exercise

1. Introduce three bugs on purpose in your Today screen: a bare string in a `View`, index keys on a
   filterable list, and a `fetch` to `http://localhost:3000` from a physical phone.
2. Diagnose each using the tool from this lesson that fits it best (LogBox component stack,
   DevTools Components panel, network logging) and write one sentence per bug: symptom → tool →
   cause.
3. Set a breakpoint inside an `onPress` handler, trigger it, and change a variable's value in the
   Scope pane before continuing. What does the UI show?
