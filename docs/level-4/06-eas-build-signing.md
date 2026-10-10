---
description: "Build signed release binaries with EAS: eas.json profiles, iOS certificates and profiles, Android keystores, Play App Signing and versions."
---

# 06 · Building with EAS & App Signing

To install an app on someone else's phone, or submit it to a store, you need a **signed release
binary**: an `.ipa` for iOS, an `.aab` (Android App Bundle) for Google Play. Producing one involves
native toolchains, certificates, provisioning profiles, keystores and version numbers — the part of
mobile development most likely to eat a day. **EAS Build** (Expo Application Services) runs those
builds on hosted macOS and Linux machines and manages credentials for you. You can also build locally;
the concepts are the same.

!!! note "What this lesson could and couldn't run"
    EAS Build needs an Expo account, and store-signed iOS builds need a paid Apple Developer Program
    membership. Those weren't used to write this lesson, so no build logs are shown. `npm view eas-cli
    version` reported 24.12.1 when this was written; commands and flags below follow the EAS
    documentation for that era — check `eas build --help` for your version.

## Setup

```bash
npm install --global eas-cli     # or prefix commands with npx eas-cli
eas login
eas init                         # links the project to an EAS project; writes extra.eas.projectId
eas build:configure              # creates eas.json
```

## Build profiles: eas.json

```json title="eas.json"
{
  "cli": { "version": ">= 16.0.0", "appVersionSource": "remote" },
  "build": {
    "base": {
      "node": "22.13.0",
      "env": { "APP_VARIANT": "production" }
    },
    "development": {
      "extends": "base",
      "developmentClient": true,
      "distribution": "internal",
      "env": { "APP_VARIANT": "development" }
    },
    "preview": {
      "extends": "base",
      "distribution": "internal",
      "channel": "preview",
      "android": { "buildType": "apk" }
    },
    "production": {
      "extends": "base",
      "channel": "production",
      "autoIncrement": true
    }
  },
  "submit": { "production": {} }
}
```

- **development** — a development build (L3-09) for your team.
- **preview** — a release-mode build installable directly on testers' devices (an `.apk` on Android;
  on iOS, an ad hoc build for registered devices). Use it for QA and performance testing (lesson 1).
- **production** — store builds: `.aab` for Android by default, App Store–signed `.ipa` for iOS.
- `channel` connects builds to over-the-air update channels (lesson 8).
- `APP_VARIANT` feeds the `app.config.ts` from L3-09, giving development builds a different name and
  bundle identifier.

Build with:

```bash
eas build --profile preview --platform android
eas build --profile production --platform all
```

EAS uploads your project (respecting `.gitignore`/`.easignore`), runs `expo prebuild` on its build
machine, installs dependencies, compiles, signs, and gives you a download link and an install QR code.

## Versions: the two numbers

Each platform has a **user-facing version** and an **internal build number**:

| | User-facing | Internal (must increase every upload) |
|---|---|---|
| iOS | `CFBundleShortVersionString` ← `expo.version` (`1.4.0`) | `CFBundleVersion` ← `ios.buildNumber` |
| Android | `versionName` ← `expo.version` | `versionCode` ← `android.versionCode` (integer) |

Stores reject an upload whose internal number isn't higher than the last one. With
`"appVersionSource": "remote"` and `"autoIncrement": true`, EAS stores the build numbers on its servers
and bumps them on every production build, so you never collide — you only change `version` when you
release a new user-facing version.

## Signing, explained

**Why sign at all?** A signature proves who built the binary and that it hasn't been modified. The OS
and stores refuse unsigned or tampered apps, and updates must be signed by the same identity as the
installed version.

### iOS

- A **distribution certificate** (with its private key) identifies your Apple Developer team.
- A **provisioning profile** ties together the app's bundle identifier, the certificate, the
  entitlements (push, associated domains…), and — for ad hoc/internal builds — the list of allowed
  device UDIDs.
- EAS can create and store both for you on first build (you sign in with your Apple ID during
  `eas build`), or you can upload your own (`eas credentials`).
- Adding a capability like push or associated domains changes entitlements, so the provisioning
  profile must be regenerated — EAS detects config changes from plugins and handles this.

### Android

- An APK/AAB is signed with a key from a **keystore**. If you lose the key used for an app that's on
  the Play Store without **Play App Signing**, you can never publish an update to that app again.
- **Play App Signing** (the default for new apps) splits this: Google holds the **app signing key**
  and re-signs what it distributes; you sign uploads with an **upload key**. If the upload key is lost
  or leaked, it can be reset through Play Console support.
- EAS generates and stores an upload keystore on first build, or you upload yours.

**Back up your credentials** regardless of who manages them: `eas credentials` lets you download
them. Treat them like production database passwords.

## Environment variables and secrets in builds

Remember lesson 5: anything your **JavaScript** reads with `EXPO_PUBLIC_` ends up in the bundle. Build
environments also have variables that only the build needs — for example a token to download a
private npm package or upload source maps to a crash reporter. EAS stores per-environment variables
(`development`, `preview`, `production`) with a visibility setting:

```bash
eas env:create --environment production --name SENTRY_AUTH_TOKEN --value "…" --visibility secret
eas env:list --environment production
```

Secret-visibility variables are available to the build process but must not be embedded in the app.
Check the EAS environment-variable docs for how profiles select environments in your CLI version.

## Local builds

You don't have to build in the cloud:

```bash
eas build --profile production --platform android --local   # runs the EAS build steps on your machine
npx expo run:android --variant release                      # plain Gradle release build
npx expo run:ios --configuration Release                    # Xcode release build (macOS only)
```

Local builds need the full toolchain (Xcode for iOS — which only runs on macOS — and the Android SDK
plus a JDK), and signing credentials available locally.

## Worked example: from zero to a testable preview build

1. `eas init` and commit the `projectId` it adds.
2. Convert `app.json` to `app.config.ts` with the `APP_VARIANT` logic from L3-09.
3. Add the `eas.json` above.
4. `eas build --profile preview --platform android` → install the `.apk` on a mid-range Android phone
   via the QR code.
5. Run the performance checks from lesson 1 on this build, not on a development build.
6. When it's good: `eas build --profile production --platform all`.

## How It Actually Works

An EAS build is a scripted version of what you'd do by hand. The CLI archives your project and uploads
it; a fresh VM (macOS with Xcode for iOS, Linux with the Android SDK for Android) unpacks it, sets the
build's environment variables, installs Node dependencies, runs `expo prebuild` to generate the native
projects from your config and plugins, then runs `pod install` and `xcodebuild archive` (iOS) or a
Gradle `bundleRelease`/`assembleRelease` task (Android). Release builds bundle your JavaScript with
Metro and compile it to Hermes bytecode as part of that native build. Finally, the binary is signed
with credentials fetched from EAS's credential store (or your uploaded ones) and uploaded as a build
artifact. Because the VM is fresh each time, a build that works on EAS but not locally (or vice versa)
almost always means an undeclared difference: a Node version, an environment variable, or a file
excluded from the upload.

## Common mistakes

- **Losing the Android signing key** for an app without Play App Signing.
- **Not incrementing build numbers** — store uploads rejected.
- **Secrets in `EXPO_PUBLIC_` variables** to "make the build work".
- **Testing performance on development builds** instead of preview builds.
- **Uncommitted `projectId` or config changes** that exist only on one developer's machine.
- **Different Node versions locally and on EAS** — pin `node` in `eas.json`.

## Exercise

1. Write `eas.json` with development, preview and production profiles for your app, with distinct
   bundle identifiers for development.
2. If you have an Expo account, run a preview build for Android and install it on a device; if not,
   run `npx expo run:android --variant release` locally. Note the build's version and build number.
3. Explain, in three sentences, what happens if you lose your Android upload key with Play App
   Signing enabled versus without it.
4. List every environment variable your app uses and mark each as `public` (safe in the bundle) or
   `secret` (build-time only, or server-side).
