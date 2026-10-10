---
description: "Use the camera, GPS and haptics from React Native with expo-camera, expo-location and expo-haptics — including accuracy, battery and simulator caveats."
---

# 02 · Device APIs: Camera, Location & Haptics

The reason to build a native app instead of a website is often the hardware. Expo's SDK wraps the
common device APIs in typed modules that work the same way on iOS and Android. This lesson uses three
of them — camera, location and haptics — and focuses on the parts the docs' quick-starts skip:
battery cost, accuracy trade-offs, lifecycle, and what you can and can't test on a simulator.

```bash
npx expo install expo-camera expo-location expo-haptics
```

All three work in Expo Go for experimenting. Permission prompts are covered properly in the next
lesson; here we use each module's permission hook in its simplest form.

!!! warning "Simulators are not phones"
    The iOS Simulator has no camera (a `CameraView` shows nothing useful), simulates location from a
    preset or a GPX file, and has no haptic engine. Android emulators can feed a virtual camera
    scene and a mock location. Test camera and haptics on a real device.

## Haptics: the cheapest polish you'll ever add

```ts
import * as Haptics from 'expo-haptics';

Haptics.selectionAsync();                                           // tick: picker/segmented change
Haptics.impactAsync(Haptics.ImpactFeedbackStyle.Light);             // tap: toggle, small confirm
Haptics.impactAsync(Haptics.ImpactFeedbackStyle.Medium);            // drop a dragged item
Haptics.notificationAsync(Haptics.NotificationFeedbackType.Success); // task completed
Haptics.notificationAsync(Haptics.NotificationFeedbackType.Error);   // validation failed
```

Rules of thumb: tie haptics to **meaningful state changes** (habit completed, item deleted), not to
every tap; use the semantic types (`Success`, `Warning`, `Error`) rather than inventing patterns; and
remember users can disable system haptics, in which case these calls simply do nothing.

```tsx
async function completeHabit(id: string) {
  await markDone(id);
  Haptics.notificationAsync(Haptics.NotificationFeedbackType.Success);
}
```

## Location

```ts
import * as Location from 'expo-location';

const { status } = await Location.requestForegroundPermissionsAsync();
if (status !== 'granted') return;

// One reading
const pos = await Location.getCurrentPositionAsync({ accuracy: Location.Accuracy.Balanced });
console.log(pos.coords.latitude, pos.coords.longitude, pos.coords.accuracy); // accuracy in metres

// Cheap and instant, possibly old — great for a first guess
const last = await Location.getLastKnownPositionAsync({ maxAge: 5 * 60_000 });
```

### Accuracy costs battery and time

| `Accuracy` | Typical use | Cost |
|------------|-------------|------|
| `Lowest` / `Low` | "Which city am I in?" | Cell/Wi-Fi only, very cheap |
| `Balanced` | Nearby places, tagging a note | Moderate |
| `High` | Turn-by-turn or precise check-in | GPS, more battery, slower first fix |
| `Highest` / `BestForNavigation` | Navigation apps | Highest drain |

A cold GPS fix can take a while, especially indoors. A good pattern is "last known first, then
refine": show `getLastKnownPositionAsync` immediately, request a fresh position in the background,
and update when it arrives. Always look at `coords.accuracy` — a position with 2,000 m accuracy is
not a street address.

### Watching location

```ts
const sub = await Location.watchPositionAsync(
  { accuracy: Location.Accuracy.Balanced, distanceInterval: 25, timeInterval: 10_000 },
  (loc) => setPosition(loc.coords),
);
// later — always:
sub.remove();
```

`distanceInterval` (metres) and `timeInterval` (milliseconds) throttle updates. Forgetting
`sub.remove()` when a screen loses focus keeps the GPS running and drains the battery. Tracking in
the **background** (app not visible) needs a separate permission, `expo-task-manager`, platform
configuration, and a justification that app reviewers will scrutinise — only do it if the feature
truly requires it.

### Reverse geocoding

```ts
const [place] = await Location.reverseGeocodeAsync({ latitude, longitude });
const label = [place?.name, place?.city].filter(Boolean).join(', ');
```

This uses the platform's geocoder (Apple's on iOS, Google's via the system on Android); it needs a
network connection and is rate-limited, so cache results rather than calling it on every update.

## Camera

`expo-camera` provides `CameraView`, a native camera preview you place in your layout like any view:

```tsx title="src/CaptureScreen.tsx"
import { useRef, useState } from 'react';
import { Button, Image, Pressable, StyleSheet, Text, View } from 'react-native';
import { CameraView, CameraType, useCameraPermissions } from 'expo-camera';
import * as Haptics from 'expo-haptics';

export function CaptureScreen({ onCaptured }: { onCaptured: (uri: string) => void }) {
  const [permission, requestPermission] = useCameraPermissions();
  const [facing, setFacing] = useState<CameraType>('back');
  const [ready, setReady] = useState(false);
  const [preview, setPreview] = useState<string | null>(null);
  const camera = useRef<CameraView>(null);

  if (!permission) return <View />; // permission status still loading
  if (!permission.granted) {
    return (
      <View style={styles.center}>
        <Text style={styles.explain}>Field Notes uses the camera to attach a photo to a note.</Text>
        <Button title="Allow camera" onPress={requestPermission} />
      </View>
    );
  }

  async function capture() {
    if (!camera.current || !ready) return;
    const photo = await camera.current.takePictureAsync({ quality: 0.7, skipProcessing: false });
    Haptics.impactAsync(Haptics.ImpactFeedbackStyle.Medium);
    setPreview(photo.uri);
  }

  if (preview) {
    return (
      <View style={styles.fill}>
        <Image source={{ uri: preview }} style={styles.fill} />
        <View style={styles.row}>
          <Button title="Retake" onPress={() => setPreview(null)} />
          <Button title="Use photo" onPress={() => onCaptured(preview)} />
        </View>
      </View>
    );
  }

  return (
    <View style={styles.fill}>
      <CameraView ref={camera} style={styles.fill} facing={facing} onCameraReady={() => setReady(true)} />
      <View style={styles.row}>
        <Button title="Flip" onPress={() => setFacing((f) => (f === 'back' ? 'front' : 'back'))} />
        <Pressable onPress={capture} disabled={!ready} accessibilityRole="button" accessibilityLabel="Take photo" style={styles.shutter} />
      </View>
    </View>
  );
}

const styles = StyleSheet.create({
  fill: { flex: 1, backgroundColor: 'black' },
  center: { flex: 1, alignItems: 'center', justifyContent: 'center', padding: 24, gap: 12 },
  explain: { textAlign: 'center', fontSize: 16 },
  row: { flexDirection: 'row', justifyContent: 'space-around', alignItems: 'center', padding: 24, backgroundColor: 'black' },
  shutter: { width: 72, height: 72, borderRadius: 36, backgroundColor: 'white', borderWidth: 4, borderColor: '#cbd5e1' },
});
```

Notes from real use:

- Wait for `onCameraReady` before calling `takePictureAsync`, or the first capture can fail.
- The photo's `uri` points to a file in the app's **cache** directory, which the OS may clear. To
  keep a photo, copy it into your app's document directory with `expo-file-system` (the Level 3
  project does this).
- `quality` (0–1) controls JPEG compression; full-resolution photos from modern phones are large, so
  pick the lowest quality your feature can tolerate.
- **Unmount the camera when the screen loses focus.** A camera left running in a background tab keeps
  the hardware busy and the camera-in-use indicator lit. With Expo Router, render `CameraView` only
  when `useIsFocused()` is true.
- `CameraView` also scans barcodes and QR codes (`barcodeScannerSettings` and
  `onBarcodeScanned`) without an extra library.

## Worked example: "tag this note with where I am"

```ts title="src/locate.ts"
import * as Location from 'expo-location';

export type Tag = { latitude: number; longitude: number; accuracy: number | null; label: string | null; stale: boolean };

export async function locateForNote(): Promise<Tag | null> {
  const perm = await Location.getForegroundPermissionsAsync();
  if (!perm.granted) return null;

  // 1. instant guess
  const last = await Location.getLastKnownPositionAsync({ maxAge: 10 * 60_000, requiredAccuracy: 200 });

  // 2. fresh fix, but don't wait forever
  const fresh = await Promise.race([
    Location.getCurrentPositionAsync({ accuracy: Location.Accuracy.Balanced }),
    new Promise<null>((resolve) => setTimeout(() => resolve(null), 8_000)),
  ]).catch(() => null);

  const best = fresh ?? last;
  if (!best) return null;

  let label: string | null = null;
  try {
    const [place] = await Location.reverseGeocodeAsync(best.coords);
    label = [place?.name, place?.city].filter(Boolean).join(', ') || null;
  } catch {
    // offline: keep coordinates, add a label later
  }

  return { latitude: best.coords.latitude, longitude: best.coords.longitude, accuracy: best.coords.accuracy, label, stale: !fresh };
}
```

The `stale` flag lets the UI say "approximate location" honestly when only the last-known position
was available.

## How It Actually Works

Each of these packages is an **Expo module**: a TypeScript API on top of Swift (iOS) and Kotlin
(Android) code, called through JSI. `getCurrentPositionAsync` creates a native location request
(`CLLocationManager` on iOS; the fused location provider or the platform location manager on
Android), returns a promise, and resolves it on the JS thread when the OS reports a fix.

`CameraView` is a **native view component**: Fabric creates a native view backed by AVFoundation on
iOS and CameraX on Android. Preview frames go straight from the camera hardware to the GPU-backed
preview layer — they never pass through JavaScript, which is why the preview is smooth even if your
JS is busy. `takePictureAsync` asks the native camera session to capture a still, writes a JPEG to the
cache directory on a background thread, and resolves with its file URI. Haptics call
`UIFeedbackGenerator` on iOS and the `Vibrator`/haptic feedback APIs on Android.

## Common mistakes

- **Testing camera or haptics only on a simulator** and concluding they're broken.
- **Watching location without removing the subscription.**
- **`Accuracy.Highest` for a feature that needs a city name.**
- **Keeping photos in the cache directory** and losing them later.
- **Leaving `CameraView` mounted in a background tab.**
- **Haptics on every interaction** — it becomes noise and costs battery.

## Exercise

1. Add a haptic success tap when a habit is completed, and a light selection tick when the user
   changes tabs in a segmented control. Try it on a device and remove any haptic that feels like noise.
2. Build a "Where am I?" screen that shows last-known position immediately, then refines with a
   fresh `Balanced` fix, displaying accuracy in metres and a "stale" badge.
3. Build the capture screen and confirm the camera turns off (indicator light goes out) when you
   switch to another tab. If it doesn't, add `useIsFocused`.
4. Add QR scanning: show the decoded text in an `Alert`, and ignore repeated scans of the same code
   within two seconds.
