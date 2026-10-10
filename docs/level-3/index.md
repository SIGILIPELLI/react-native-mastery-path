---
description: "React Native Level 3: threads and the New Architecture, camera, location, permissions, notifications, Reanimated, gestures and native modules."
---

# Level 3 · Native Capabilities & Polish

Levels 1 and 2 used React Native as a cross-platform UI layer. Level 3 is where it becomes a
*mobile* app: it uses the camera and GPS, asks for permissions properly, sends notifications,
animates at the display's frame rate even while JavaScript is busy, responds to gestures, works with
screen readers, and — when no library does what you need — talks to Swift and Kotlin code you write
yourself.

It starts with the model that makes all of this make sense: which thread runs what, and how the New
Architecture (JSI, Fabric, TurboModules) connects JavaScript to native code. Then each lesson uses
that model: device APIs are native modules, Reanimated's worklets run on the UI thread, Gesture
Handler recognises touches natively, and your own Expo module is a TurboModule-style native module.
The level finishes with development builds and config plugins — the tools that let you add any native
code to an Expo app — and a field-notes app that uses the camera, location and offline sync.

**Prerequisites:** Levels 1 and 2. Several lessons need a **development build** (lesson 9 explains
how) or a physical device; each lesson says so where it applies. Some features — push delivery, a
real camera, GPS — can't be fully exercised on a simulator, and the lessons are explicit about that.

## Modules

1. **[Under the Hood: Threads, JSI & the New Architecture](01-architecture-new-arch.md)** — the
   runtime model behind every other lesson.
2. **[Device APIs: Camera, Location & Haptics](02-device-apis.md)** — Expo modules for hardware,
   with real-world caveats.
3. **[Permissions Done Right](03-permissions.md)** — asking at the right moment, handling "no",
   and the platform rules.
4. **[Local & Push Notifications](04-notifications.md)** — scheduling reminders, push tokens and
   handling taps.
5. **[Animations with Animated & Reanimated](05-animations.md)** — shared values, worklets and
   layout animations.
6. **[Gestures with React Native Gesture Handler](06-gestures.md)** — pan, swipe-to-complete and
   composing gestures with animations.
7. **[Accessibility in React Native](07-accessibility.md)** — roles, labels, focus, dynamic type
   and reduced motion.
8. **[Writing a Native Module with the Expo Modules API](08-native-modules.md)** — Swift and Kotlin
   functions callable from TypeScript.
9. **[Development Builds, Prebuild & Config Plugins](09-dev-builds-prebuild.md)** — leaving Expo Go
   without leaving Expo.
10. **[Project — Field Notes with Camera, Location & Offline Sync](10-project-field-notes.md)** —
    capture geotagged notes offline and sync them when the network returns.
