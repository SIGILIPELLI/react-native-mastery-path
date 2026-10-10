---
description: "Know when your React Native app breaks: route error boundaries, Sentry crash reporting with source maps and release tags, and lean analytics."
---

# 09 · Crash Reporting, Analytics & Monitoring

Once your app is in users' hands, you can't open DevTools on their phones. Without monitoring, you
learn about crashes from one-star reviews, days late and without a stack trace. A production app needs
three things: **graceful failure** (a broken screen shouldn't kill the app), **crash and error
reporting** with readable stack traces tied to the exact release, and a small amount of **product
analytics** collected respectfully.

## Layer 1: error boundaries

In a release build, an uncaught error during rendering unmounts the entire React tree — the user sees
a blank screen or the app closes. **Error boundaries** catch render errors in a subtree and show a
fallback instead.

Expo Router makes this per-route: export an `ErrorBoundary` from any route file or layout, and errors
in that route render it instead of crashing the app:

```tsx title="app/recipe/[id].tsx (excerpt)"
import { Pressable, Text, View } from 'react-native';
import type { ErrorBoundaryProps } from 'expo-router';

export function ErrorBoundary({ error, retry }: ErrorBoundaryProps) {
  return (
    <View style={{ flex: 1, padding: 24, gap: 12, justifyContent: 'center' }}>
      <Text style={{ fontSize: 18, fontWeight: '700' }}>This recipe couldn't be shown.</Text>
      <Text style={{ color: '#64748b' }}>{__DEV__ ? error.message : 'The problem has been reported.'}</Text>
      <Pressable onPress={retry} accessibilityRole="button"><Text style={{ color: '#4f46e5' }}>Try again</Text></Pressable>
    </View>
  );
}

export default function RecipeScreen() {
  // …
  return null;
}
```

The rest of the app — tabs, other screens — keeps working. Note what error boundaries **don't**
catch: errors in event handlers, in `async` code (a rejected promise in `useEffect`), and native
crashes. Those need handling in place (try/catch, query error states) and a crash reporter.

## Layer 2: crash and error reporting with Sentry

Several services do this (Sentry, Bugsnag, Firebase Crashlytics, and others). This lesson uses
**Sentry** because it has an Expo config plugin and captures JS errors and native crashes in one tool.
The SDK installed for SDK 57 in this course's check project was `@sentry/react-native` 7.11.

```bash
npx expo install @sentry/react-native
```

`npx expo install` adds the config plugin to `app.json`. Initialise it as early as possible:

```tsx title="app/_layout.tsx"
import * as Sentry from '@sentry/react-native';
import * as Updates from 'expo-updates';
import { Stack } from 'expo-router';

Sentry.init({
  dsn: process.env.EXPO_PUBLIC_SENTRY_DSN,   // a DSN identifies where to send events; it isn't a secret
  enabled: !__DEV__,                         // don't flood the project from development
  sendDefaultPii: false,                     // don't attach IP addresses and similar by default
  tracesSampleRate: 0.1,                     // performance traces for 10% of sessions
  environment: Updates.channel ?? 'development',
});

// Tie every event to the exact OTA update running (lesson 8).
if (Updates.updateId) Sentry.setTag('expo-update-id', Updates.updateId);
if (Updates.isEmergencyLaunch) Sentry.captureMessage(`Emergency launch: ${Updates.emergencyLaunchReason ?? 'unknown'}`);

function RootLayout() {
  return <Stack />;
}

export default Sentry.wrap(RootLayout);
```

`Sentry.wrap` adds a top-level error boundary and touch/navigation breadcrumbs. Report handled errors
too, where you catch them:

```ts
try {
  await syncNow(db);
} catch (e) {
  Sentry.captureException(e, { tags: { area: 'sync' } });
}
```

### Source maps: make stack traces readable

A release bundle is minified and compiled to Hermes bytecode; a raw stack trace says something like
`index.android.bundle:1:48213`. **Source maps** map that back to `src/sync.ts:42`. Sentry's Metro
helper adds debug IDs to bundles so maps can be matched:

```js title="metro.config.js"
const { getSentryExpoConfig } = require('@sentry/react-native/metro');

module.exports = getSentryExpoConfig(__dirname);
```

During EAS builds, the config plugin uploads source maps for the embedded bundle when a
`SENTRY_AUTH_TOKEN` is available as a **secret** build environment variable (lesson 6) — never an
`EXPO_PUBLIC_` one. For OTA updates, you upload source maps for each update after `eas update`;
follow Sentry's Expo guide for the exact command for your SDK version. Without this step, every OTA
update's crashes are unreadable.

### Privacy in crash reports

Crash reports can capture more than you think: breadcrumbs of navigation, console logs, request URLs.

- Keep `sendDefaultPii: false` unless you have a reason and have disclosed it.
- Use `setUser({ id })` with an opaque internal ID — not an email or name.
- Scrub sensitive data in `beforeSend` / `beforeBreadcrumb` (tokens in URLs, note text).
- Declare crash reporting in the App Privacy and Data Safety forms (lesson 7).

## Layer 3: analytics, minimally

Analytics answers product questions ("Do people who set a reminder keep their streaks longer?"). Most
apps collect far more than they ever analyse. Start with a handful of named events and a typed helper
that makes it hard to send anything else:

```ts title="src/analytics.ts"
type EventMap = {
  habit_created: { has_reminder: boolean };
  habit_completed: { streak_after: number };
  reminder_enabled: { hour: number };
  sync_failed: { retryable: boolean };
};

type Sender = (name: string, props: Record<string, string | number | boolean>) => void;

let send: Sender = () => {};          // no-op until configured
let enabled = false;

export function configureAnalytics(sender: Sender, userConsented: boolean) {
  send = sender;
  enabled = userConsented;
}

export function track<E extends keyof EventMap>(name: E, props: EventMap[E]) {
  if (!enabled) return;
  send(name, props);                  // only typed, non-personal properties can reach here
}
```

```ts
track('habit_completed', { streak_after: 12 });
// track('habit_completed', { habit_name: 'Therapy' });   ← type error: personal data can't sneak in
```

The `EventMap` type is your **data inventory**: everything the app can send is listed in one file, which
makes the privacy declarations accurate and reviews easy. `configureAnalytics` takes whichever provider
SDK you choose (PostHog, Amplitude, Firebase Analytics, your own endpoint) — and a consent flag, which
some jurisdictions and Apple's tracking rules may require you to obtain before collecting.

## What to watch after a release

| Signal | Where | Healthy looks like |
|--------|-------|--------------------|
| Crash-free sessions/users | Sentry release health; App Store Connect; Play Console vitals | Stable or improving versus the previous release |
| ANRs (Android "app not responding") | Play Console Android vitals | Below Play's bad-behaviour thresholds |
| New error types | Sentry issues filtered by release / update ID | None, or understood |
| Emergency launches | Your `captureMessage` above | Zero |
| Key funnels | Analytics (habit created → completed) | No sudden drops after a release |

Set alerts for crash-rate spikes on new releases, so a bad rollout (lesson 7) or OTA update
(lesson 8) is halted in hours, not days.

## How It Actually Works

Sentry's React Native SDK has two halves. The **JavaScript** half hooks the global error handler
(`ErrorUtils.setGlobalHandler`) and unhandled promise rejections, wraps your tree in an error boundary,
records breadcrumbs (navigation, touches, console, fetch) in a ring buffer, and when an error occurs,
builds an event with the stack trace and breadcrumbs and sends it — or queues it on disk if offline.
The **native** half installs the platform crash handlers (signal and exception handlers on iOS, an
uncaught-exception handler on Android, plus ANR detection), which write a crash report to disk at the
moment of the crash — the process is dying, so it can't send anything — and upload it on the next
launch.

Source maps work through **debug IDs**: at build time, the Metro plugin injects a unique ID into the
bundle and into its source map. Error events include the debug ID of the bundle that threw, and Sentry
finds the uploaded map with the same ID to symbolicate the stack frames. Native crashes are symbolicated
the same way with dSYM files (iOS) and ProGuard/R8 mapping files (Android), which the config plugin
uploads during EAS builds.

## Common mistakes

- **No error boundaries** — one broken screen blanks the whole app.
- **Crash reporting without source maps**, especially for OTA updates.
- **`SENTRY_AUTH_TOKEN` as an `EXPO_PUBLIC_` variable** — it would ship in the bundle.
- **Reporting from development** — noise that hides real production issues.
- **Emails, names or note contents in user context, breadcrumbs or analytics events.**
- **Tracking everything "just in case"** — privacy risk and inaccurate store declarations.
- **No alerts** — dashboards nobody looks at don't stop bad rollouts.

## Exercise

1. Add an `ErrorBoundary` to the recipe detail route, throw an error deliberately in a release build,
   and confirm the rest of the app keeps working.
2. Create a free Sentry project (if you're comfortable doing so), add the SDK with the `wrap` and tags
   above, trigger a test error in a preview build, and check whether the stack trace is readable. If
   it isn't, fix source-map upload.
3. Write the `EventMap` for your app with no more than six events, and justify each one in a comment.
4. Write `beforeBreadcrumb` that drops breadcrumbs for URLs containing `token=` and redacts the body of
   any console breadcrumb longer than 200 characters.
