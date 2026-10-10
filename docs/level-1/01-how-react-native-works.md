---
description: "What happens between your JSX and a native iOS or Android view, how Expo fits in, and how to create a TypeScript project and run it on your phone."
---

# 01 · How React Native Works & Setting Up Expo

The most useful mental model for React Native is this: **your React code describes a tree of
components, and React Native keeps a tree of real native views in sync with it.** Nothing is drawn
in a browser. When you write

```tsx
<View style={{ padding: 16 }}>
  <Text>Hello</Text>
</View>
```

React Native creates a `UIView` containing a text view on iOS, and an Android `ViewGroup`
containing a `TextView`-style view on Android. Scrolling, text rendering, the keyboard and
accessibility all come from the operating system. That is why a React Native app feels native — and
why some web habits (CSS files, `<div>`, `onClick`) simply don't exist here.

## Where your code runs

A React Native app is a native app with a JavaScript engine embedded in it. By default that engine
is **Hermes**, a JavaScript engine built by Meta specifically for React Native: it compiles your
JavaScript to bytecode at build time so the app doesn't parse source text on launch.

There are three places work happens, and you will meet them again and again:

| Place | What runs there |
|-------|-----------------|
| **JS thread** | Your components, hooks, state updates, business logic, `fetch` callbacks |
| **UI (main) thread** | Native view creation and updates, touch delivery, scrolling, native animations |
| **Background threads** | Layout calculation (Yoga), image decoding, native module work |

If the JS thread is busy for 200 ms, your app can't respond to a tap with new JavaScript for 200 ms —
but a native scroll view keeps scrolling, because scrolling lives on the UI thread. Much of Level 3
and Level 4 is about using that split well.

## What Expo is (and isn't)

React Native itself is the runtime and the core components. **Expo** is a toolchain and module
collection built around it:

- `create-expo-app` scaffolds a project with sensible defaults (TypeScript, Hermes, the New
  Architecture).
- The **Expo CLI** (`npx expo start`) runs the Metro bundler and serves your JavaScript to devices.
- **Expo Go** is a pre-built app from the App Store / Play Store that can load your JavaScript
  without you compiling any native code. It only contains the native modules Expo ships with.
- **Expo SDK** packages (`expo-camera`, `expo-location`, `expo-sqlite`, …) wrap native APIs with a
  consistent TypeScript interface.
- **Development builds** and **EAS Build** (Level 3 and 4) let you compile your own native app when
  you need modules Expo Go doesn't contain.

The React Native team itself recommends starting new apps with a framework, and Expo is the one it
names. You are not locked in: an Expo project is a normal React Native project, and
`npx expo prebuild` generates the native `ios/` and `android/` folders whenever you want them.

## Setting up

You need **Node.js** (an active LTS release) and a phone. A simulator is optional at this stage.

```bash
npx create-expo-app@latest habit-app --template blank-typescript
cd habit-app
npx expo start
```

`npx expo start` prints a QR code. Install **Expo Go** on your phone, then scan the code (with the
Camera app on iOS, or from inside Expo Go on Android). Your phone and computer need to be on the
same network; if they can't reach each other (corporate Wi-Fi, VPNs), `npx expo start --tunnel`
routes through a tunnel instead.

!!! note "Expo Go and SDK versions"
    Each Expo Go release supports one SDK version. If Expo Go says your project's SDK is
    incompatible, either update the project (`npx expo install expo@latest` then
    `npx expo install --fix`) or move to a development build (Level 3, lesson 9). When this lesson
    was written, a fresh `blank-typescript` project pinned Expo SDK 57 with react-native 0.86.

The blank TypeScript template gives you this `package.json` (versions will differ for you):

```json
{
  "main": "index.ts",
  "scripts": {
    "start": "expo start",
    "android": "expo start --android",
    "ios": "expo start --ios",
    "web": "expo start --web"
  }
}
```

`index.ts` registers your root component:

```ts title="index.ts"
import { registerRootComponent } from 'expo';
import App from './App';

registerRootComponent(App);
```

`registerRootComponent` calls `AppRegistry.registerComponent('main', () => App)` — the hook that
the native side uses to find and mount your root component.

## Your first real screen

Replace `App.tsx` with something that uses state, so you can see Fast Refresh at work:

```tsx title="App.tsx"
import { useState } from 'react';
import { StatusBar } from 'expo-status-bar';
import { Pressable, StyleSheet, Text, View } from 'react-native';

export default function App() {
  const [count, setCount] = useState(0);

  return (
    <View style={styles.container}>
      <Text style={styles.title}>Glasses of water today</Text>
      <Text style={styles.count}>{count}</Text>
      <Pressable
        style={({ pressed }) => [styles.button, pressed && styles.buttonPressed]}
        onPress={() => setCount((c) => c + 1)}
        accessibilityRole="button"
      >
        <Text style={styles.buttonText}>+1 glass</Text>
      </Pressable>
      <StatusBar style="auto" />
    </View>
  );
}

const styles = StyleSheet.create({
  container: { flex: 1, alignItems: 'center', justifyContent: 'center', gap: 16 },
  title: { fontSize: 18, color: '#334155' },
  count: { fontSize: 64, fontWeight: '700' },
  button: { backgroundColor: '#4f46e5', paddingHorizontal: 24, paddingVertical: 12, borderRadius: 999 },
  buttonPressed: { opacity: 0.7 },
  buttonText: { color: 'white', fontSize: 16, fontWeight: '600' },
});
```

Save the file and the change appears on your phone within a second or two, **without losing the
counter value**. That is Fast Refresh: Metro re-sends only the changed module and React re-renders
while preserving hook state (as long as you didn't change the order or number of hooks).

## How It Actually Works

When you save `App.tsx`:

1. **Metro**, the bundler started by `expo start`, notices the file change, transforms it with Babel
   (TypeScript types are stripped, JSX becomes `jsx()` calls) and sends the new module to the
   connected app over a WebSocket.
2. The app's JavaScript engine swaps the module in place, and React re-renders the affected
   components.
3. React's **reconciler** compares the new element tree with the previous one and produces a list
   of changes — "update this text", "change this style".
4. In the **New Architecture**, those changes go to **Fabric**, React Native's renderer. Fabric
   keeps a C++ "shadow tree" mirroring your components, runs **Yoga** (a C++ Flexbox engine) to
   compute every view's position and size, then applies the minimal set of mutations to the real
   native views on the UI thread.
5. JavaScript talks to native code through **JSI** (the JavaScript Interface), which lets
   JavaScript hold direct references to C++ objects and call them synchronously. The old
   architecture serialized every message to JSON and sent it across an asynchronous "bridge"; the
   New Architecture removed that bottleneck, and in recent React Native versions it is the only
   architecture.

In a release build there is no Metro: your JavaScript is bundled ahead of time, compiled to Hermes
bytecode and shipped inside the app binary.

```mermaid
flowchart LR
  A[Your TSX] -->|Babel via Metro| B[JS bundle]
  B --> C[Hermes engine]
  C -->|React reconciler| D[Fabric shadow tree]
  D -->|Yoga layout| E[Native views on UI thread]
```

## Common mistakes

- **Expecting HTML elements.** `<div>` and `<p>` don't exist; using them crashes with "View config
  not found". Use `View` and `Text`.
- **Running `npm install` for native packages.** Use `npx expo install <pkg>` — it picks the
  version compatible with your SDK. A mismatched native module version is a common cause of
  crashes at startup.
- **Phone can't connect to Metro.** Phone and laptop on different networks, a VPN, or a firewall
  blocking the port. Try `--tunnel` before debugging anything else.
- **Assuming Expo Go is the app you ship.** It's a development convenience. Your users install
  your own binary, built with EAS Build or Xcode/Android Studio.
- **Mixing up "React Native version" and "Expo SDK version".** You upgrade the SDK, and it brings
  the matching React Native and React versions with it.

## Exercise

1. Create a `blank-typescript` project and run it in Expo Go on your phone.
2. Build the water counter above, then add a "Reset" button that only appears when `count > 0`.
3. Change the title colour and watch Fast Refresh keep your count. Now add a new `useState` call
   *above* the existing one and save — what happens to the count, and why does changing hook order
   force a full remount?
4. Run `npx expo start` and press `j` to open React Native DevTools. Find your `App` component in
   the Components tab and read its state.
