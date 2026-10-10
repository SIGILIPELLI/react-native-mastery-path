---
description: "React Native Level 1: how RN renders native views, Expo setup, core components, StyleSheet, Flexbox, Pressable, TextInput, FlatList and debugging."
---

# Level 1 · Foundations with Expo

React Native looks like React, and most of what you know about components and hooks carries over
unchanged. What does not carry over is the platform underneath: there is no DOM, no CSS cascade, no
`<div>` or `<span>`, and the screen is a phone with a notch, a home indicator and a keyboard that
slides up over your content. Level 1 is about that gap.

You will set up an Expo project and run it on a real device, learn the handful of core components
everything else is built from, style and lay them out with React Native's subset of CSS and its
Flexbox defaults, handle touches and text input, render long lists efficiently, deal with safe areas
and platform differences, and learn to debug when something goes wrong. The project at the end
pulls it together into a habit tracker that runs on both iOS and Android.

**Prerequisites:** React fundamentals (components, props, `useState`, `useEffect`) and a little
TypeScript. If those are new, the
[React Mastery Path](https://sigilipelli.github.io/react-mastery-path/) Level 1 and the
[TypeScript Mastery Path](https://sigilipelli.github.io/typescript-mastery-path/) cover them.

## Modules

1. **[How React Native Works & Setting Up Expo](01-how-react-native-works.md)** — what actually
   happens between your JSX and a native view, and a working project on your phone.
2. **[Core Components: View, Text, Image & ScrollView](02-core-components.md)** — the building
   blocks, and the rules (like "all text lives in `<Text>`") that trip up web developers.
3. **[Styling with StyleSheet](03-stylesheet-styling.md)** — style objects, units, no cascade,
   and how to share styles without a stylesheet language.
4. **[Flexbox Layout the React Native Way](04-flexbox-layout.md)** — column-first defaults,
   `flex: 1`, and building real screen layouts.
5. **[Touch & Feedback with Pressable](05-pressable-touch.md)** — press states, hit slop, long
   presses and why touch feedback matters on mobile.
6. **[TextInput & Simple Forms](06-textinput-forms.md)** — controlled inputs, keyboard types,
   focus chains and submit behaviour.
7. **[Lists with FlatList & SectionList](07-flatlist-sectionlist.md)** — virtualization, keys,
   pull-to-refresh and infinite scroll.
8. **[Safe Areas, Platform Differences & Screen Size](08-safe-area-platform.md)** — notches,
   `Platform`, platform-specific files and responsive sizing.
9. **[Debugging React Native Apps](09-debugging.md)** — the dev menu, React Native DevTools,
   LogBox, network inspection and reading red-screen errors.
10. **[Project — Habit Tracker](10-project-habit-tracker.md)** — a complete single-screen app with
    a list, a form, touch feedback and a streak calculation.
