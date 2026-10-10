---
description: "Capstone: take one React Native app to a release candidate with tests, accessibility, performance, EAS, OTA, monitoring and a preflight script."
---

# 10 · Capstone — Ship a Production-Ready App

The capstone brings the whole course together. You'll take **one** app — the habit tracker you've grown
since Level 1, or an app of your own with similar scope — and make it a genuine **release candidate**:
something you could submit to both stores tomorrow with confidence. The deliverable isn't a new
feature; it's the evidence that the app is ready.

## The app

**Habits** — a habit tracker with:

- Tabs: **Today**, **Stats**, **Settings** (Expo Router, L2-02).
- Habits and check-ins in **SQLite** with migrations (L2-06); streaks computed by the tested pure
  functions from L1-10.
- **Add/edit habit** form with Zod validation (L2-07) in a modal route.
- **Daily reminder** with a proper permission flow (L3-03, L3-04).
- **Swipe-to-complete** with haptics and an accessible alternative (L3-06, L3-07).
- **Celebration animation** that respects Reduce Motion (L3-05).
- Optional **sync** to a server using the Field Notes sync engine (L3-10) — choose whether your RC
  includes it, and say so in the README.

If you'd rather build something else, keep the same ingredients: multiple routes, local persistence
with migrations, a form, a permission, a notification or device API, a list that can grow large, and
an animation.

## Part 1 — architecture review

Write a one-page `ARCHITECTURE.md` answering:

1. **Where does each kind of state live?** (L2-04's table: local UI, shared client, server, navigation,
   persisted.) Name the actual hooks/stores/tables.
2. **What are the data flows for "check off a habit"** from tap to SQLite to every screen that shows it?
3. **What happens offline?** Every screen: works, degrades (how?), or unavailable (why?).
4. **What's the navigation tree?** A Mermaid diagram like L2-01's, including modals and where auth
   would go.

Reviewers (including you in six months) should be able to understand the app from this page.

## Part 2 — the quality gates

### Tests

- Unit tests for all pure logic: streaks (including month/year boundaries and DST), reducers, schemas,
  sync policy if included.
- Component tests (RNTL 14, L4-03) for: the add-habit form's validation; a habit row's
  role/name/checked state; the empty state; one data screen's loading and error states.
- One or two **E2E flows** (L4-04): add a habit → complete it → relaunch → still completed; and the
  reminder permission-denied path.

Run `npx jest --coverage` and look at what's *not* covered: is any untested branch something that
could lose user data? Test that first.

### Accessibility

Complete the L3-07 exercise for the whole app with VoiceOver **and** TalkBack, at the largest text size,
with Reduce Motion on. Record issues and fixes in `ACCESSIBILITY.md`.

### Performance

Seed 3 years of history (about 1,100 days × 6 habits) through a debug-only "seed data" button, make a
**preview/release build**, and on the slowest device available:

- Cold start to interactive Today screen — measure with a stopwatch or a simple timestamp log.
- Scroll the history log (FlashList, L4-02) — watch the perf monitor.
- Type in any search box — no input lag.

Record what you measured, on which device and build type, in `PERFORMANCE.md`. Real numbers from your
device only; if you couldn't test on Android, write that down.

### Security and privacy

- Run `npx expo export --platform android` and `grep -a` the bundle for anything secret (L4-05).
- Deep links and notification payloads go through `safeInternalPath`.
- `PRIVACY.md` lists every piece of data the app stores or sends, matching your App Privacy / Data
  Safety answers (L4-07) and your analytics `EventMap` (L4-09).

## Part 3 — release engineering

- `app.config.ts` with development and production variants (L3-09).
- `eas.json` with development, preview and production profiles; remote app versions with
  auto-increment (L4-06).
- `runtimeVersion: { policy: 'fingerprint' }` and `expo-updates` configured with channels (L4-08); the
  in-app update banner.
- Sentry (or equivalent) with source maps for builds **and** updates, release tags, privacy scrubbing
  (L4-09). Route-level `ErrorBoundary` on every screen that loads data.
- `RELEASE.md` runbook (L4-07) — and actually walk through it once with a preview build.

## Part 4 — an automated preflight check

Some release mistakes are mechanical and easy to catch with a script that runs in CI before every
production build. This one reads your resolved Expo config and fails on the problems this course has
warned about:

```js title="scripts/preflight.mjs"
// Usage: node scripts/preflight.mjs   (run from the project root)
import { execFileSync } from 'node:child_process';

const config = JSON.parse(
  execFileSync('npx', ['expo', 'config', '--json', '--type', 'public'], { encoding: 'utf8' }),
);

const problems = [];
const warn = [];

if (!/^\d+\.\d+\.\d+$/.test(config.version ?? '')) problems.push(`version "${config.version}" is not MAJOR.MINOR.PATCH`);
if (!config.ios?.bundleIdentifier) problems.push('ios.bundleIdentifier is missing');
if (!config.android?.package) problems.push('android.package is missing');
if (!config.scheme) warn.push('no URL scheme: deep links and some auth flows will not work');

const rv = config.runtimeVersion;
if (!rv) warn.push('runtimeVersion is not set: OTA updates are not configured');
else if (typeof rv === 'object' && rv.policy !== 'fingerprint') warn.push(`runtimeVersion policy "${rv.policy}": remember to bump it for every native change`);

// Permission strings: plugins that need one must have a specific, non-default sentence.
const PERMISSION_OPTIONS = { 'expo-camera': 'cameraPermission', 'expo-location': 'locationWhenInUsePermission' };
for (const entry of config.plugins ?? []) {
  const [name, options = {}] = Array.isArray(entry) ? entry : [entry];
  const key = PERMISSION_OPTIONS[name];
  if (!key) continue;
  const text = options[key];
  if (!text) problems.push(`${name}: no ${key} string (the library default is generic)`);
  else if (text.length < 30 || !/\b(so|to)\b/i.test(text)) warn.push(`${name}: "${text}" may be too vague for App Review`);
}

// Public env vars whose names suggest secrets.
for (const name of Object.keys(process.env)) {
  if (name.startsWith('EXPO_PUBLIC_') && /SECRET|PRIVATE|PASSWORD|TOKEN/i.test(name)) {
    problems.push(`${name} looks like a secret but EXPO_PUBLIC_ values ship inside the app bundle`);
  }
}

for (const w of warn) console.log(`warn  ${w}`);
for (const p of problems) console.log(`FAIL  ${p}`);
console.log(problems.length ? `\n${problems.length} problem(s) found.` : '\npreflight passed.');
process.exit(problems.length ? 1 : 0);
```

`npx expo config --json --type public` prints the fully resolved config — after `app.config.ts` logic
and plugins — so the script checks what will actually be built. This course ran the script against the
small `Field Notes` config from L3-09 (which has no `runtimeVersion`, and whose app.json had been
given an `expo-location` plugin with the short string "Tag notes with a place."), with a deliberately
badly named variable set:

```text
$ EXPO_PUBLIC_STRIPE_SECRET=x node scripts/preflight.mjs
warn  runtimeVersion is not set: OTA updates are not configured
warn  expo-location: "Tag notes with a place." may be too vague for App Review
FAIL  EXPO_PUBLIC_STRIPE_SECRET looks like a secret but EXPO_PUBLIC_ values ship inside the app bundle

1 problem(s) found.
```

The exit code was 1, so a CI step running it would stop the release. Extend it with your own checks:
minimum target SDK, required `ErrorBoundary` exports, presence of `RELEASE.md`.

## Deliverables

1. The app's repository with `ARCHITECTURE.md`, `ACCESSIBILITY.md`, `PERFORMANCE.md`, `PRIVACY.md`,
   `RELEASE.md` and a README explaining how to run it, test it and build it.
2. Passing unit/component tests and at least one passing E2E flow (or a clear note on why you couldn't
   run E2E and what you'd need).
3. `preflight.mjs` passing.
4. A preview build installed on at least one real device, with your measurements.
5. A short **release-readiness memo**: what you're confident about, what you didn't verify, and what
   you'd do before submitting. Honesty here is the skill — "I couldn't test on iOS" is a perfectly good
   sentence; a fabricated "tested on all devices" is not.

## How It Actually Works

A release candidate is a claim: "this exact binary and bundle behave correctly for real users." Each
part of the capstone gathers evidence for one aspect of that claim. Unit and component tests check
logic and UI behaviour in Node, fast and deterministic. E2E tests check that the compiled binary — the
native modules, permissions, storage and navigation together — works on a real OS. Accessibility and
performance checks run on devices because screen readers, text scaling and frame budgets only exist
there. Preflight checks the build configuration itself, because many production incidents in mobile
apps come from configuration (a missing permission string, a wrong runtime version) rather than code.
Monitoring closes the loop after release, turning real-world failures into new tests.

## Common mistakes

- **Polishing features instead of gathering evidence** — the capstone is about readiness.
- **Measuring performance on a development build or only on a flagship phone.**
- **Tests that only cover the happy path** — the bugs that hurt users are in error and offline paths.
- **Privacy docs that don't match what the code sends.**
- **Overclaiming in the readiness memo.**

## Exercise

Complete the capstone. Then, as a final reflection, list three things you'd do differently if you
started the app again today — about architecture, testing or release — and one thing from this course
you'll carry into every future mobile project.
