---
description: "Test the real React Native app end to end with Maestro YAML flows and Detox, choose stable selectors, handle permissions and network, and run E2E in CI."
---

# 04 · End-to-End Testing with Maestro & Detox

Component tests run in Node against mocks. They can't tell you that the app crashes on launch because
a native module is missing, that the keyboard covers the Save button on a small phone, that a deep
link opens the wrong screen, or that data really survives a restart. **End-to-end (E2E) tests**
drive the actual compiled app on a simulator, emulator or device, the way a user would.

They're slower and more fragile than unit tests, so you write **few** of them, for the journeys that
must never break: sign-in, the core "create" flow, purchase, offline save.

!!! note "What this lesson could run"
    E2E tests need a built app and a running simulator or emulator, which weren't available in the
    environment this course was written in. The flows below are written against the course's own
    apps and the tools' documented syntax, but no run output is shown — run them yourself, and expect
    to adjust selectors to your UI.

## Two tools, two philosophies

| | Maestro | Detox |
|---|---|---|
| Tests written in | YAML "flows" | JavaScript/TypeScript (Jest) |
| How it drives the app | From outside, through the accessibility layer — black box | Inside the app, with a native library linked into a test build — grey box |
| Synchronisation | Waits and retries automatically for elements to appear | Tracks the app's busy state (network, timers, animations, JS thread) and waits until idle |
| Setup | Install a CLI; works with any build | Native configuration and dedicated test builds |
| Best fit | Fast to start; QA engineers and developers alike; Expo-friendly | Large teams wanting tests in TypeScript next to code, deep sync with app state |

Both are mature choices. Many Expo teams start with Maestro because it needs no changes to the app.

## Make the app testable first

E2E tests need stable handles on elements. Use, in order of preference:

1. **Accessibility labels and visible text** — what users see; robust if copy is stable.
2. **`testID`** for elements without stable text (icons, list rows with dynamic content):

```tsx
<Pressable testID="add-habit-fab" accessibilityLabel="Add habit" onPress={…} />
<TextInput testID="habit-name-input" accessibilityLabel="Habit name" />
<View testID={`habit-row-${habit.id}`}>…</View>
```

`testID` maps to the accessibility identifier on iOS and the resource-id / view tag on Android, which
is what both tools look up. Don't put `testID` on everything; put it where text won't do.

## Maestro

Install the CLI (macOS/Linux) following [maestro.dev](https://maestro.dev/) instructions, build and
install your app on a simulator (a development or release build — not Expo Go), then write a flow:

```yaml title=".maestro/add-habit.yaml"
appId: com.example.habits
---
- launchApp:
    clearState: true
- tapOn: "Add habit"
- tapOn:
    id: "habit-name-input"
- inputText: "Drink water"
- tapOn: "Save habit"
- assertVisible: "Drink water"
# The core promise: data survives a restart
- stopApp
- launchApp
- assertVisible: "Drink water"
```

```bash
maestro test .maestro/add-habit.yaml
```

Maestro waits for elements automatically, so you rarely write explicit sleeps. Useful commands:
`scrollUntilVisible`, `swipe`, `back` (Android), `openLink` (deep links), `takeScreenshot`, and
`runFlow` to reuse a sign-in flow across tests. `maestro studio` opens an interactive inspector to
find selectors.

### Permissions and deep links in Maestro

```yaml title=".maestro/capture-note.yaml"
appId: com.example.fieldnotes
---
- launchApp:
    clearState: true
    permissions:
      camera: allow
      location: allow
- tapOn: "New note"
- inputText: "Fence post damaged near gate 3"
- tapOn: "Add location"
- takeScreenshot: after-location   # location text varies; capture it for review instead of asserting
- tapOn: "Save note"
- assertVisible: "Fence post damaged near gate 3"
- openLink: "fieldnotes://"
```

Granting permissions at launch keeps tests away from system dialogs, which differ by OS version. Test
the *denied* path in a separate flow with `permissions: { location: deny }`.

## Detox

Detox needs native setup: a `.detoxrc.js` describing how to build and launch test binaries, and
(for Expo) a config plugin to add its native test library during prebuild. Follow the Detox docs for
your React Native version — setup details change between releases. A test then looks like Jest:

```ts title="e2e/addHabit.test.ts"
import { by, device, element, expect } from 'detox';

describe('Add habit', () => {
  beforeAll(async () => {
    await device.launchApp({ newInstance: true, delete: true, permissions: { notifications: 'YES' } });
  });

  it('adds a habit and keeps it after relaunch', async () => {
    await element(by.id('add-habit-fab')).tap();
    await element(by.id('habit-name-input')).typeText('Drink water');
    await element(by.text('Save habit')).tap();
    await expect(element(by.text('Drink water'))).toBeVisible();

    await device.launchApp({ newInstance: true });
    await expect(element(by.text('Drink water'))).toBeVisible();
  });
});
```

Detox's selling point is **synchronisation**: before each action it waits until the app is idle — no
pending network requests, timers, animations or JS work — so tests don't need waits. The catch: an
infinite animation or a polling timer means the app is *never* idle, and tests hang. Disable such
animations in test builds, or tell Detox to ignore specific URLs or timers.

## Controlling the network

E2E tests that hit a real backend are slow and flaky. Options:

- Point the test build at a **local or staging server** with seeded data via an environment variable
  (`EXPO_PUBLIC_API_URL`, L3-10).
- Run a **mock server** (the tiny Node server from L3-10 is a start) and reset it between flows.
- Test offline behaviour by toggling airplane mode on a device/emulator, or by stopping the mock
  server — then assert your offline UI.

## E2E in CI

E2E tests run on macOS runners (for iOS simulators) or Linux runners with Android emulators. A
typical pipeline: build a release-like binary once (EAS Build has an `e2e`-style profile, or build
locally in CI), boot a simulator, install, run flows, upload screenshots and videos as artifacts on
failure. EAS also offers workflows for running Maestro tests against EAS builds; check current EAS
documentation for the exact configuration. Keep the E2E suite small enough to finish in minutes, and
run it on pull requests to `main` rather than every push.

## How It Actually Works

**Maestro** runs outside your app. On iOS it uses Apple's XCTest UI automation framework; on Android it
uses UIAutomator-based instrumentation. Both read the **accessibility tree** — labels, identifiers,
text, frames — which is why `accessibilityLabel` and `testID` make elements findable, and why an
accessible app is easier to test. Each command polls the hierarchy until the element appears or a
timeout passes, which gives tolerance to animations and loading without explicit waits.

**Detox** links a native library into a test build of your app. That library talks to the Detox test
runner over a WebSocket and, crucially, observes the app's internals: the main run loop, network
requests in flight, timers, animations and the React Native JS thread's work queue. An action like
`tap()` is only dispatched when all of those are idle, and then executed through the platform's own
test frameworks (EarlGrey on iOS, Espresso on Android). That inside view is what makes Detox
deterministic — and what makes it sensitive to anything that keeps the app permanently busy.

## Common mistakes

- **Too many E2E tests** — slow, flaky suites that people stop trusting. Cover critical journeys only.
- **Selectors tied to layout** (the third child of the second view) instead of labels or `testID`s.
- **Sleeping instead of asserting** — `sleep 3` is either too long or too short.
- **Depending on a live production backend.**
- **Not clearing state between flows** — one test's data leaks into the next.
- **Permission dialogs left to chance** — grant or deny explicitly at launch.
- **Testing in Expo Go** — test the binary you ship.

## Exercise

1. Add `testID`s to the habit app's add button, name input and rows.
2. Install Maestro, build your app for a simulator or emulator, and get `add-habit.yaml` passing,
   including the restart check.
3. Write a second flow for the denied-permission path of Field Notes: deny location, create a note, and
   assert it saves without a location.
4. Write down which three user journeys in your app deserve E2E tests and why the rest should be
   covered by component tests instead.
