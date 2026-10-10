---
description: "Ship JavaScript fixes over the air with expo-updates and EAS Update: runtime versions, fingerprints, channels, in-app checks and rollbacks."
---

# 08 · Over-the-Air Updates & Release Strategy

A store release takes time: build, upload, review (hours to days on iOS), then users updating at
their own pace. If you ship a bug in a JavaScript screen, waiting a week for a fix is painful. Because
most of a React Native app is a JavaScript bundle plus assets, you can deliver a new bundle directly
to installed apps — an **over-the-air (OTA) update** — without going through the store, as long as you
stay within the rules.

## What an OTA update can and can't change

| Can change (JS bundle + assets) | Can't change (native binary) |
|---|---|
| Screens, components, styles, copy | Adding or upgrading a native module or SDK |
| Business logic, bug fixes in JS | Permission strings, entitlements, `Info.plist`/manifest |
| Images and fonts loaded through the bundle | App icon, splash screen, bundle ID |
| Feature flags you ship in JS | Expo SDK / React Native upgrades |

Store policies matter too. Apple's guidelines allow downloading interpreted code like a JavaScript
bundle as long as it doesn't change the app's primary purpose or create a store-like experience for
executable code; Google Play has similar rules. Use OTA for fixes and incremental improvements that
would pass review anyway — not to swap in a different app after approval.

## The key idea: runtime version

An update's JavaScript must match the native code it runs on. If update 1.4.3's JS calls a native
module that only exists in binary 1.5.0, an app still on 1.4.0 would crash. **Runtime version** is the
compatibility label: a build accepts only updates with the **same** runtime version.

```json title="app.json (excerpt)"
{
  "expo": {
    "runtimeVersion": { "policy": "fingerprint" },
    "updates": { "url": "https://u.expo.dev/<your-project-id>" }
  }
}
```

Policies (from the config types in SDK 57): `appVersion` (runtime version = `expo.version` — you
must remember to bump `version` whenever native code changes), `nativeVersion`, `sdkVersion`, and
**`fingerprint`**, which computes a hash of everything that affects the native build — native
dependencies, config plugins, the native-relevant parts of your config — so it changes automatically
when, and only when, native code changes.

You can see this yourself with `@expo/fingerprint`. This course generated fingerprints for a small
SDK 57 project three times:

```bash
npx @expo/fingerprint fingerprint:generate > fp1.json   # baseline
# edit App.tsx (a JavaScript-only change)
npx @expo/fingerprint fingerprint:generate > fp2.json
# add ["expo-location", {...}] to "plugins" in app.json (a native change)
npx @expo/fingerprint fingerprint:generate > fp3.json
```

The baseline and the JavaScript-only change produced the **same** hash (`60bbe913…`); adding the
`expo-location` config plugin produced a **different** one (`1e437c33…`). That's exactly the
behaviour you want: JS fixes can ship as OTA updates to existing builds, while a native change forces a
new runtime version — and therefore a new store build.

## Channels and branches

EAS Update organises updates like this:

- A **build** is created with a **channel** (from its `eas.json` profile: `preview`, `production`).
- You publish updates to a **branch**.
- A channel points at a branch. By default, the `production` channel points at the `production`
  branch.

```bash
npx expo install expo-updates
eas update:configure                                   # adds the updates URL and runtime version
eas update --branch production --message "Fix crash when a recipe has no image"
eas update --branch preview --message "Try new onboarding copy"
```

Because channels point at branches, you can do useful things without republishing: point `production`
at a branch you've already verified in `preview`, or roll back by republishing a previous update. EAS
also supports **rolling out** an update to a percentage of users; see `eas update --help` and the EAS
Update docs for the options in your CLI version.

## When updates are applied

By default, the app checks for an update **on launch**, downloads it in the background, and applies it
on the **next** launch. Users with the app open don't get an abrupt reload. You can control this in-app:

```tsx title="src/UpdateBanner.tsx"
import { Pressable, Text, View } from 'react-native';
import * as Updates from 'expo-updates';

export function UpdateBanner() {
  const { isUpdatePending, isDownloading } = Updates.useUpdates();

  if (!Updates.isEnabled) return null;          // dev builds and Expo Go
  if (isDownloading) return null;               // stay quiet while downloading
  if (!isUpdatePending) return null;

  return (
    <View style={{ flexDirection: 'row', alignItems: 'center', gap: 12, padding: 12, backgroundColor: '#eef2ff' }}>
      <Text style={{ flex: 1 }}>An update is ready.</Text>
      <Pressable onPress={() => Updates.reloadAsync()} accessibilityRole="button">
        <Text style={{ color: '#4338ca', fontWeight: '700' }}>Restart</Text>
      </Pressable>
    </View>
  );
}
```

```ts title="src/checkForUpdate.ts"
import * as Updates from 'expo-updates';

/** Call when the app returns to the foreground, e.g. from an AppState listener. */
export async function checkAndDownload(): Promise<void> {
  if (!Updates.isEnabled) return;
  try {
    const result = await Updates.checkForUpdateAsync();
    if (result.isAvailable) await Updates.fetchUpdateAsync(); // sets isUpdatePending → banner shows
  } catch {
    // Offline or update server unreachable: try again next time. Never block the app on this.
  }
}
```

For a **critical** fix you might reload immediately after downloading on a screen where nothing would
be lost — but be careful: reloading discards in-memory state, including half-written forms.

## Rollback and safety

Things go wrong. Protections built in and practices to add:

- **Emergency launch.** If a downloaded update crashes before the app has finished loading, expo-updates
  falls back to the previous working bundle (or the one embedded in the binary).
  `Updates.isEmergencyLaunch` tells you this happened — report it to your crash tool (lesson 9).
- **Republish a known-good update** to the branch to roll forward to safety; EAS also has rollback
  commands — check the docs for your CLI version.
- **Publish to `preview` first**, install a preview build, test, then publish the same commit to
  `production`.
- **Tag updates with the git commit** (EAS records it) so crash reports map to code.
- **Upload source maps** for each update to your crash reporter, or stack traces are unreadable.

## A release strategy that uses both channels

1. **Native changes** (new SDK, new native module, permission changes) → new store build with a new
   runtime version, via lesson 7's process with phased/staged rollout.
2. **JS-only fixes** → OTA update to the matching runtime version, first to `preview`, then
   `production`, ideally to a percentage of users first.
3. **Feature launches** → ship the code dark behind a remote feature flag in a store build, then turn
   it on remotely; OTA becomes the fix channel rather than the launch channel.

## How It Actually Works

A release build contains an **embedded** bundle and assets plus an updates configuration (URL,
runtime version, channel). On launch, expo-updates' native code decides which bundle to run: the
newest downloaded update compatible with this runtime version, if one is stored and healthy, otherwise
the embedded one. It then (by default) asks the update server for the latest update for its channel
and runtime version. The server responds with a **manifest** listing the update's bundle and asset
URLs with hashes. Only assets that aren't already on the device are downloaded; each is verified
against its hash, stored in the app's storage, and recorded in a small SQLite database. The new
update becomes the launch candidate for next time. Because the decision happens in native code before
JavaScript starts, a bad update that crashes on startup can be detected (the launch didn't reach "JS
loaded successfully") and skipped on the following launch.

The fingerprint policy works because `@expo/fingerprint` hashes the *inputs* to the native build —
package versions of native modules, their native source directories, config plugins and the
native-relevant config — not your JavaScript, so it changes only when a new binary would be different.

## Common mistakes

- **Shipping an OTA update that needs a native change** — crashes on older binaries; use the
  fingerprint policy to make this impossible.
- **`appVersion` policy without bumping `version`** after a native change.
- **Publishing straight to production** without testing on a preview build.
- **Forcing reloads** that discard user input.
- **Using OTA to change the app's purpose** after review.
- **No source maps for updates** — unreadable crash reports.

## Exercise

1. Add `expo-updates` with the `fingerprint` policy. Run `npx @expo/fingerprint fingerprint:generate`
   before and after a JS-only change and after adding a native module; confirm the behaviour yourself.
2. Add `UpdateBanner` to your root layout and `checkAndDownload` to an `AppState` "active" listener.
3. If you have an EAS account: make a preview build, publish an update to the `preview` branch that
   changes a label, and watch the banner appear on the device after relaunching or foregrounding.
4. Write your team's rule for "store release vs OTA" in three bullet points.
