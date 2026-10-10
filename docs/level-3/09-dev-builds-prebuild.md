---
description: "Move beyond Expo Go: development builds, Continuous Native Generation, expo prebuild, and a config plugin that edits Info.plist and the manifest."
---

# 09 · Development Builds, Prebuild & Config Plugins

Expo Go is a fixed app containing a fixed set of native modules. The moment you add a library with
native code that Expo Go doesn't include — your own module from lesson 8, a keyboard controller, a
payments SDK — or you need your own URL scheme, icon, permission strings or push credentials, you
outgrow it. The answer isn't to "eject" from Expo. It's a **development build**: your own debug app,
containing exactly your native dependencies, with the same developer tools as Expo Go.

## Expo Go vs a development build

| | Expo Go | Development build |
|---|---|---|
| Native code | Expo's SDK modules only | Any native library, your own modules |
| App identity | "Expo Go" | Your name, icon, bundle ID, scheme |
| Permission strings, plist/manifest config | Expo Go's | Yours |
| Push notifications | Limited | Full, with your credentials |
| Rebuild needed when | Never | Native dependencies or config change |
| JS changes | Fast Refresh | Fast Refresh |

The daily loop is the same: run `npx expo start`, edit JavaScript, see changes instantly. You only
rebuild the development build when the **native** side changes.

## Creating a development build

Install the dev client, which adds the developer menu and the launcher UI to your own app:

```bash
npx expo install expo-dev-client
```

Then build it one of two ways:

**Locally** (needs Xcode for iOS, Android Studio/SDK for Android):

```bash
npx expo run:ios        # builds, installs on a simulator or a connected device, starts Metro
npx expo run:android
```

**In the cloud with EAS Build** (no local toolchain; Level 4 covers EAS in depth):

```json title="eas.json (excerpt)"
{
  "build": {
    "development": { "developmentClient": true, "distribution": "internal" }
  }
}
```

```bash
npx eas-cli build --profile development --platform android
```

The result is an installable app (an `.apk` for Android internal distribution, or an iOS build for
simulators or registered test devices). Open it, and it connects to your Metro server like Expo Go
does.

## Continuous Native Generation and `expo prebuild`

Where do the `ios/` and `android/` projects come from? In an Expo project you normally **don't commit
them at all**. Instead, **`npx expo prebuild`** generates them from:

1. A template for the current SDK,
2. Your `app.json` / `app.config.ts` (name, bundle identifier, icons, scheme, …),
3. The **config plugins** of every library you use, plus your own.

Expo calls this **Continuous Native Generation (CNG)**: the native projects are build artifacts,
regenerated whenever config changes, which is why upgrading the SDK doesn't mean hand-merging
hundreds of lines of Xcode and Gradle changes. `run:ios`, `run:android` and EAS Build all run prebuild
for you when the folders don't exist.

To see it for yourself, this course ran prebuild in an SDK 57 project with the following config (the
plugin file is shown in the next section):

```json title="app.json"
{
  "expo": {
    "name": "Field Notes",
    "slug": "field-notes",
    "scheme": "fieldnotes",
    "ios": { "bundleIdentifier": "com.example.fieldnotes" },
    "android": { "package": "com.example.fieldnotes" },
    "plugins": [
      ["expo-camera", { "cameraPermission": "Field Notes uses the camera so you can attach a photo to a note." }],
      ["./plugins/withReleaseFlags", { "analyticsEnabled": false }]
    ]
  }
}
```

```bash
npx expo prebuild --no-install --clean
```

`--no-install` skips CocoaPods and `--clean` deletes existing native folders first. It created
`ios/` (with `FieldNotes.xcodeproj`, a `Podfile` and the app sources) and `android/` (a Gradle
project), and switched the `ios`/`android` scripts in `package.json` from `expo start --ios` to
`expo run:ios`. The generated files contained, among other things:

```xml title="ios/FieldNotes/Info.plist (excerpt, as generated)"
<key>ITSAppUsesNonExemptEncryption</key>
<false/>
<key>NSCameraUsageDescription</key>
<string>Field Notes uses the camera so you can attach a photo to a note.</string>
```

```xml title="android/app/src/main/AndroidManifest.xml (excerpt, as generated)"
<uses-permission android:name="android.permission.CAMERA"/>
<meta-data android:name="com.example.ANALYTICS_ENABLED" android:value="false"/>
<data android:scheme="fieldnotes"/>
```

The camera permission string came from `expo-camera`'s config plugin, the URL scheme from `scheme`,
and the encryption flag and meta-data from our own plugin.

!!! tip "Should I commit `ios/` and `android/`?"
    With CNG, add them to `.gitignore` and treat `app.json` + plugins as the source of truth. If you
    hand-edit native files instead, you've opted into maintaining them yourself ("bare" workflow) and
    must not run `prebuild --clean` again, because it would overwrite your edits. Choose deliberately.

## Writing a config plugin

A config plugin is a function that takes the Expo config and returns a modified one. **Mods** like
`withInfoPlist` and `withAndroidManifest` let it change native files during prebuild:

```js title="plugins/withReleaseFlags.js"
const { withInfoPlist, withAndroidManifest, AndroidConfig } = require('expo/config-plugins');

/** Sets the export-compliance flag on iOS and a meta-data entry on Android. */
module.exports = function withReleaseFlags(config, { analyticsEnabled = false } = {}) {
  config = withInfoPlist(config, (cfg) => {
    cfg.modResults.ITSAppUsesNonExemptEncryption = false;
    return cfg;
  });

  config = withAndroidManifest(config, (cfg) => {
    const app = AndroidConfig.Manifest.getMainApplicationOrThrow(cfg.modResults);
    AndroidConfig.Manifest.addMetaDataItemToMainApplication(app, 'com.example.ANALYTICS_ENABLED', String(analyticsEnabled));
    return cfg;
  });

  return config;
};
```

`ITSAppUsesNonExemptEncryption = false` declares that the app only uses exempt encryption (such as
HTTPS), which saves answering the export-compliance question on every App Store Connect upload — set
it only if that's true for your app. Mods exist for most native files (`withAppBuildGradle`,
`withPodfile`, `withEntitlementsPlist`, `withStringsXml`, `withDangerousMod` for anything else), and
helpers like `AndroidConfig.Manifest` avoid hand-editing XML structures.

Plugins should be **idempotent**: running prebuild twice must produce the same result, so check before
adding entries. (Running prebuild a second time, without `--clean`, on the project above still left
exactly one `ANALYTICS_ENABLED` entry and one encryption key — the `AndroidConfig` helper replaces an
existing meta-data item instead of appending a duplicate.) And they run in Node at build time, not in your app — they can't read device state.

### Dynamic config

`app.json` is static. For environment-specific values, use `app.config.ts`:

```ts title="app.config.ts"
import type { ExpoConfig } from 'expo/config';

const IS_DEV = process.env.APP_VARIANT === 'development';

const config: ExpoConfig = {
  name: IS_DEV ? 'Field Notes (Dev)' : 'Field Notes',
  slug: 'field-notes',
  scheme: 'fieldnotes',
  ios: { bundleIdentifier: IS_DEV ? 'com.example.fieldnotes.dev' : 'com.example.fieldnotes' },
  android: { package: IS_DEV ? 'com.example.fieldnotes.dev' : 'com.example.fieldnotes' },
  plugins: ['expo-camera', ['./plugins/withReleaseFlags', { analyticsEnabled: !IS_DEV }]],
};

export default config;
```

Different bundle identifiers let the development build and the store build sit side by side on the
same phone.

## How It Actually Works

`expo prebuild` loads your config (evaluating `app.config.ts` if present), then resolves every entry
in `plugins` — library plugins are found through the package's `app.plugin.js`. Each plugin wraps
the config with mods. Prebuild then copies the SDK's native template, and runs the **mod chain**: for
each native file type, it reads the file (parsing plists and XML into objects), passes it through
every registered mod in order, and writes the result. Base mods provided by Expo handle the core
fields (name, identifiers, icons, scheme), and library and project plugins add their changes on top.

`expo-dev-client` adds native code that, at launch, shows a launcher to pick a Metro server (or
scans a QR code), loads the JavaScript bundle from it, and provides the developer menu. In a release
build, the dev client's UI is excluded and the app loads its embedded bundle instead.

## Common mistakes

- **Hand-editing `ios/`/`android/` and then running `prebuild --clean`** — edits lost.
- **Changing native config and forgetting to rebuild** — the old development build doesn't have it.
- **Using Expo Go "until it breaks"** for apps that clearly need native modules — switch early.
- **Non-idempotent plugins** that add duplicate manifest entries each run.
- **Committing generated native folders and also using CNG** — two sources of truth that drift.
- **Setting `ITSAppUsesNonExemptEncryption = false` without checking** what encryption your app uses.

## Exercise

1. Add `expo-dev-client` to your habit or recipe app and create a development build (locally or with
   EAS). Confirm Fast Refresh still works.
2. Write a config plugin that adds `LSApplicationQueriesSchemes` (iOS) with `["comgooglemaps"]` so
   the app may check whether Google Maps is installed. Run `npx expo prebuild --no-install` and find
   your change in `Info.plist`.
3. Convert `app.json` to `app.config.ts` with a development variant that has a different name and
   bundle identifier, and install both variants side by side.
4. Run prebuild twice in a row and `diff` the outputs to confirm your plugin is idempotent.
