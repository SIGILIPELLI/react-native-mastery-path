---
description: "Write a native module with the Expo Modules API: scaffold a local module, implement Swift and Kotlin functions and events, call them from TS."
---

# 08 · Writing a Native Module with the Expo Modules API

Sooner or later you'll need a platform API that no library exposes the way you want — a system
setting, an SDK from a hardware vendor, a performance-critical routine. React Native's answer is a
**native module**: Swift/Objective-C on iOS, Kotlin/Java on Android, callable from JavaScript. The
**Expo Modules API** is the most approachable way to write one: a small declarative DSL in Swift and
Kotlin that generates the JSI bindings for you, works with the New Architecture, and doesn't require
writing any C++ or Objective-C.

!!! warning "What was and wasn't run for this lesson"
    The module skeleton below was generated with `create-expo-module` (version 57.0.1) and the
    TypeScript side was type-checked in the course's SDK 57 project. The Swift and Kotlin
    implementations follow that generated template and standard platform APIs, but compiling them
    needs Xcode and the Android toolchain, which weren't run for this lesson. Build it yourself with
    `npx expo run:ios` / `npx expo run:android` and treat compiler errors as part of the exercise.

## When to write one (and when not to)

Write a native module when you need something native and **no maintained library does it**. Check
the [Expo SDK](https://docs.expo.dev/versions/latest/) and the
[React Native Directory](https://reactnative.directory/) first; a maintained library is code you don't
have to keep compiling against every new iOS and Android release. Don't write one to make
JavaScript "faster" without profiling first (Level 4) — Hermes is fast, and crossing into native has
its own cost.

## Scaffold a local module

A **local module** lives inside your app's repository, in a `modules/` folder, and is autolinked like
any other package:

```bash
npx create-expo-module@latest device-health --local
```

The tool asks for a module name and an Android package (or takes `--name` and `--package` flags). It
generates:

```
modules/device-health/
├── expo-module.config.json        # which platforms and module classes to link
├── src/
│   ├── DeviceHealthModule.ts      # TypeScript binding (requireNativeModule)
│   └── DeviceHealth.types.ts      # event payload types
├── ios/
│   ├── DeviceHealthModule.swift
│   └── DeviceHealth.podspec
└── android/
    ├── build.gradle
    └── src/main/java/expo/modules/devicehealth/DeviceHealthModule.kt
```

```json title="modules/device-health/expo-module.config.json"
{
  "platforms": ["apple", "android"],
  "apple": { "modules": ["DeviceHealthModule"] },
  "android": { "modules": ["expo.modules.devicehealth.DeviceHealthModule"] }
}
```

Native code can't run in Expo Go — Expo Go only contains Expo's own modules. You need a development
build (next lesson): `npx expo run:ios` or `npx expo run:android` compiles one locally, and it must be
rebuilt whenever native code changes. Fast Refresh only reloads JavaScript.

## The module definition DSL

Both platforms use the same vocabulary:

| DSL | Meaning | JavaScript side |
|-----|---------|-----------------|
| `Name("DeviceHealth")` | Module name used by `requireNativeModule` | — |
| `Constant("x") { … }` | Value computed once | property |
| `Function("f") { … }` | **Synchronous** call — keep it fast | `module.f()` returns a value |
| `AsyncFunction("g") { … }` | Runs off the JS thread, resolves a promise | `await module.g()` |
| `Events("onChange")` | Events the module can emit | `module.addListener('onChange', cb)` |
| `OnStartObserving` / `OnStopObserving` | First listener added / last removed | — |

Arguments and return values are converted automatically: numbers, strings, booleans, arrays,
dictionaries/maps, and `Record` types you define.

## Worked example: a "device health" module

We'll expose three things that are useful for an offline-first app: free disk space (to warn before
saving large photos), battery level, and whether the system's low-power/battery-saver mode is on —
with an event when it changes, so the app can pause background syncing.

### Swift (iOS)

```swift title="modules/device-health/ios/DeviceHealthModule.swift"
import ExpoModulesCore
import UIKit

public class DeviceHealthModule: Module {
  public func definition() -> ModuleDefinition {
    Name("DeviceHealth")

    Events("onPowerSaveChange")

    // Synchronous and cheap: reading a file-system attribute.
    Function("getFreeDiskBytes") { () -> Double in
      let url = URL(fileURLWithPath: NSHomeDirectory())
      let values = try url.resourceValues(forKeys: [.volumeAvailableCapacityForImportantUsageKey])
      return Double(values.volumeAvailableCapacityForImportantUsage ?? 0)
    }

    // Battery level in 0...1, or -1 when unknown (e.g. the Simulator).
    AsyncFunction("getBatteryLevelAsync") { () -> Double in
      return await MainActor.run {
        UIDevice.current.isBatteryMonitoringEnabled = true
        return Double(UIDevice.current.batteryLevel)
      }
    }

    Function("isPowerSaveEnabled") { () -> Bool in
      ProcessInfo.processInfo.isLowPowerModeEnabled
    }

    OnStartObserving {
      NotificationCenter.default.addObserver(
        self,
        selector: #selector(self.powerStateChanged),
        name: Notification.Name.NSProcessInfoPowerStateDidChange,
        object: nil
      )
    }

    OnStopObserving {
      NotificationCenter.default.removeObserver(self, name: Notification.Name.NSProcessInfoPowerStateDidChange, object: nil)
    }
  }

  @objc
  private func powerStateChanged() {
    sendEvent("onPowerSaveChange", ["enabled": ProcessInfo.processInfo.isLowPowerModeEnabled])
  }
}
```

`UIDevice` APIs must be used on the main thread, hence `MainActor.run`. `throws` from
`resourceValues` propagate to JavaScript as a rejected call/exception automatically.

### Kotlin (Android)

```kotlin title="modules/device-health/android/src/main/java/expo/modules/devicehealth/DeviceHealthModule.kt"
package expo.modules.devicehealth

import android.content.BroadcastReceiver
import android.content.Context
import android.content.Intent
import android.content.IntentFilter
import android.os.BatteryManager
import android.os.PowerManager
import android.os.StatFs
import expo.modules.kotlin.exception.Exceptions
import expo.modules.kotlin.modules.Module
import expo.modules.kotlin.modules.ModuleDefinition

class DeviceHealthModule : Module() {
  private val context: Context
    get() = appContext.reactContext ?: throw Exceptions.ReactContextLost()

  private var receiver: BroadcastReceiver? = null

  override fun definition() = ModuleDefinition {
    Name("DeviceHealth")

    Events("onPowerSaveChange")

    Function("getFreeDiskBytes") {
      StatFs(context.filesDir.path).availableBytes.toDouble()
    }

    AsyncFunction("getBatteryLevelAsync") {
      val bm = context.getSystemService(Context.BATTERY_SERVICE) as BatteryManager
      val percent = bm.getIntProperty(BatteryManager.BATTERY_PROPERTY_CAPACITY)
      if (percent in 0..100) percent / 100.0 else -1.0
    }

    Function("isPowerSaveEnabled") {
      (context.getSystemService(Context.POWER_SERVICE) as PowerManager).isPowerSaveMode
    }

    OnStartObserving {
      val r = object : BroadcastReceiver() {
        override fun onReceive(c: Context, intent: Intent) {
          val pm = c.getSystemService(Context.POWER_SERVICE) as PowerManager
          sendEvent("onPowerSaveChange", mapOf("enabled" to pm.isPowerSaveMode))
        }
      }
      context.registerReceiver(r, IntentFilter(PowerManager.ACTION_POWER_SAVE_MODE_CHANGED))
      receiver = r
    }

    OnStopObserving {
      receiver?.let { context.unregisterReceiver(it) }
      receiver = null
    }
  }
}
```

The function names, argument types and return types match the Swift side exactly — that shared
contract is what the TypeScript declaration describes.

### TypeScript binding

```ts title="modules/device-health/src/DeviceHealth.types.ts"
export type PowerSaveChangePayload = { enabled: boolean };

export type DeviceHealthEvents = {
  onPowerSaveChange: (payload: PowerSaveChangePayload) => void;
};
```

```ts title="modules/device-health/src/DeviceHealthModule.ts"
import { NativeModule, requireNativeModule } from 'expo';
import { DeviceHealthEvents } from './DeviceHealth.types';

declare class DeviceHealthModule extends NativeModule<DeviceHealthEvents> {
  getFreeDiskBytes(): number;
  getBatteryLevelAsync(): Promise<number>;
  isPowerSaveEnabled(): boolean;
}

export default requireNativeModule<DeviceHealthModule>('DeviceHealth');
```

The `declare class` is a hand-written contract: TypeScript trusts it, so keep it in sync with the
native code. A mismatch shows up at runtime, not compile time.

### A hook for the app

```ts title="modules/device-health/src/usePowerSave.ts"
import { useEffect, useState } from 'react';
import DeviceHealth from './DeviceHealthModule';

export function usePowerSave(): boolean {
  const [enabled, setEnabled] = useState(() => DeviceHealth.isPowerSaveEnabled());
  useEffect(() => {
    const sub = DeviceHealth.addListener('onPowerSaveChange', (e) => setEnabled(e.enabled));
    return () => sub.remove();
  }, []);
  return enabled;
}
```

```tsx title="app/health.tsx"
import { useEffect, useState } from 'react';
import { Text, View } from 'react-native';
import DeviceHealth from '../modules/device-health/src/DeviceHealthModule';
import { usePowerSave } from '../modules/device-health/src/usePowerSave';

const gb = (bytes: number) => (bytes / 1024 ** 3).toFixed(1);

export default function Health() {
  const powerSave = usePowerSave();
  const [battery, setBattery] = useState<number | null>(null);
  const free = DeviceHealth.getFreeDiskBytes();

  useEffect(() => { DeviceHealth.getBatteryLevelAsync().then(setBattery); }, []);

  return (
    <View style={{ padding: 16, gap: 8 }}>
      <Text>Free storage: {gb(free)} GB</Text>
      <Text>Battery: {battery == null ? '…' : battery < 0 ? 'unknown' : `${Math.round(battery * 100)}%`}</Text>
      <Text>Power saving: {powerSave ? 'on — background sync paused' : 'off'}</Text>
    </View>
  );
}
```

Toggle Low Power Mode (iOS Control Center) or Battery Saver (Android quick settings) on a real
device and watch the last line update.

## How It Actually Works

At build time, Expo's **autolinking** scans `modules/` and `node_modules` for
`expo-module.config.json` files and adds each module to the native projects: a CocoaPods dependency
built from the `.podspec` on iOS, a Gradle project on Android, plus a generated list of module classes
the app registers at startup.

At runtime, the Expo Modules core creates a JSI host object for each module the first time JavaScript
calls `requireNativeModule('DeviceHealth')`. The `definition()` DSL is read once to build a table of
functions, constants and events. A JavaScript call to `getFreeDiskBytes()` becomes a direct JSI call
into that table; arguments are converted from JSI values to Swift/Kotlin types, the closure runs, and
the return value is converted back. `Function` closures run synchronously on the JS thread — so they
must be quick — while `AsyncFunction` closures are dispatched to a background queue and resolve a
JavaScript promise when done. `sendEvent` posts to the module's JS event emitter, which is why the
`NativeModule<Events>` type gives you a typed `addListener`.

## Common mistakes

- **Expecting native changes to Fast Refresh** — rebuild the development build.
- **Testing in Expo Go** — your module isn't in it.
- **Slow work in `Function`** — it blocks the JS thread; use `AsyncFunction`.
- **UI APIs off the main thread** (iOS `UIDevice`, `UIApplication`) — crashes or warnings.
- **Mismatched names or types** between Swift, Kotlin and the TypeScript `declare class`.
- **Observers never removed** — implement `OnStopObserving`.
- **Holding a strong reference to an Activity/Context** on Android beyond its lifetime — leaks.

## Exercise

1. Generate the module with `create-expo-module --local`, paste in the implementations, and build a
   development build on at least one platform. Fix any compiler errors you hit and note what they
   were.
2. Add `getThermalState()` returning `'nominal' | 'fair' | 'serious' | 'critical'` on iOS
   (`ProcessInfo.processInfo.thermalState`) and a best-effort equivalent on Android API 29+
   (`PowerManager.currentThermalStatus`), returning `'unknown'` on older Android.
3. Use `usePowerSave()` in the Level 2 recipe app to skip prefetching images while power saving is on.
4. Write a Jest test for a component that uses `usePowerSave` by mocking the module with
   `jest.mock('../modules/device-health/src/DeviceHealthModule', …)`.
