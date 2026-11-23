---
title: "React Native Cheat Sheet: Quick Reference, Beginner to Advanced"
description: "A free React Native cheat sheet: 20 quick-reference tables from every level of the React Native Mastery Path, each linked to its full lesson."
---

# React Native Cheat Sheet

Quick-reference tables from **20 lessons** of the free React Native Mastery Path, in course order from beginner to advanced. Each section links to the full lesson with explanations and runnable examples.

## Level 1 · Foundations with Expo

### [01 · How React Native Works &amp; Setting Up Expo](level-1/01-how-react-native-works.md)

*Where your code runs*

| Place | What runs there |
|-------|-----------------|
| **JS thread** | Your components, hooks, state updates, business logic, `fetch` callbacks |
| **UI (main) thread** | Native view creation and updates, touch delivery, scrolling, native animations |
| **Background threads** | Layout calculation (Yoga), image decoding, native module work |

### [02 · Core Components: View, Text, Image &amp; ScrollView](level-1/02-core-components.md)

*Text — the only place text can live*

| Prop | Use |
|------|-----|
| `numberOfLines={2}` | Truncate with an ellipsis after two lines |
| `ellipsizeMode="middle"` | Where the `…` goes (`head`, `middle`, `tail`, `clip`) |
| `selectable` | Let users long-press to copy |
| `allowFontScaling={false}` | Opt out of system font size — almost never do this; see lesson L3-07 |

### [04 · Flexbox Layout the React Native Way](level-1/04-flexbox-layout.md)

*The defaults that differ from CSS*

| Property | Web default | React Native default |
|----------|-------------|----------------------|
| `display` | `block` / `inline` | always `flex` |
| `flexDirection` | `row` | **`column`** |
| `alignContent` | `normal` (`stretch`) | `flex-start` |
| `flexShrink` | `1` | **`0`** |
| `position` | `static` | `relative` |
| box sizing | `content-box` | border-box behaviour (borders and padding are inside `width`) |

### [06 · TextInput &amp; Simple Forms](level-1/06-textinput-forms.md)

*Tell the keyboard what this field is*

| Prop | Example values | Effect |
|------|----------------|--------|
| `keyboardType` | `default`, `email-address`, `number-pad`, `decimal-pad`, `phone-pad`, `url` | Which keyboard layout appears |
| `inputMode` | `text`, `numeric`, `decimal`, `email`, `tel`, `url`, `search` | Web-style alternative to `keyboardType`; takes precedence if both are set |
| `autoCapitalize` | `none`, `sentences`, `words`, `characters` | Shift behaviour |
| `autoCorrect` | `false` for emails, usernames, codes | Stops "fixing" user input |
| `autoComplete` | `email`, `password`, `new-password`, `one-time-code`, `name`, `postal-code` | Autofill hint (Android and iOS) |
| `textContentType` | `emailAddress`, `password`, `newPassword`, `oneTimeCode` | iOS-specific autofill hint |
| `secureTextEntry` | `true` | Masks input; enables password autofill |
| `returnKeyType` | `next`, `done`, `go`, `search`, `send` | Label on the return key |
| `maxLength` | `4` | Hard character limit |
| `multiline` | `true` | Textarea behaviour (return inserts a newline) |

## Level 2 · Navigation, State & Data

### [01 · Navigation Concepts: Stacks, Tabs &amp; Modals](level-2/01-navigation-concepts.md)

*Native screens vs JavaScript screens*

| | Native stack | JS stack |
|---|---|---|
| Built on | Platform navigation controllers via `react-native-screens` (`UINavigationController`-style on iOS, fragments on Android) | Views animated by JavaScript/Reanimated |
| Feel | Exactly native transitions, headers, large titles, gestures | Close to native; fully customisable animations |
| Performance | Transitions run natively, unaffected by a busy JS thread | Good, but more work in JS |
| Customisation | Limited to what the platform offers | Anything |

### [04 · App State: Context, useReducer &amp; Zustand](level-2/04-app-state.md)

*Four kinds of state*

| Kind | Examples | Where it lives |
|------|----------|----------------|
| **Local UI state** | Is this row expanded? Text in a search box | `useState` in the component |
| **Shared client state** | Habits and check-ins, current user's settings, cart | A store (Context + reducer, or Zustand) |
| **Server state** | Data owned by a backend: recipes, profile, feed | A cache (TanStack Query, lesson 5) |
| **Navigation state** | Which screen, which params | The router (lessons 1–3) |

### [06 · Local Persistence: AsyncStorage, SecureStore &amp; SQLite](level-2/06-local-persistence.md)

| Store | Good for | Not for |
|-------|----------|---------|
| **AsyncStorage** | Small key–value data: preferences, onboarding-seen flag, last selected tab | Secrets; large or relational data; anything you query |
| **SecureStore** (`expo-secure-store`) | Small secrets: auth tokens, refresh tokens, an encryption key | Large values; general data |
| **SQLite** (`expo-sqlite`) | Real app data: records, history, anything you filter, sort or join | Tiny one-off flags (overkill) |
| **File system** (`expo-file-system`) | Images, downloads, exported files | Structured data |

## Level 3 · Native Capabilities & Polish

### [01 · Under the Hood: Threads, JSI &amp; the New Architecture](level-3/01-architecture-new-arch.md)

*Three kinds of threads*

| Thread | Runs | If it's blocked… |
|--------|------|------------------|
| **JS thread** | Hermes executing your bundle: React rendering, hooks, event handlers, timers, `fetch` callbacks | Taps get no JS response; JS-driven animations freeze; state doesn't update |
| **UI / main thread** | Native view creation and mutation, touch delivery, native scrolling, native animations, the platform's rendering | Everything visibly freezes; the OS may report "App Not Responding" on Android |
| **Background threads** | Yoga layout in Fabric, image decoding, network I/O, native module work dispatched off-main | Usually invisible unless you wait on the result |

### [02 · Device APIs: Camera, Location &amp; Haptics](level-3/02-device-apis.md)

*Accuracy costs battery and time*

| `Accuracy` | Typical use | Cost |
|------------|-------------|------|
| `Lowest` / `Low` | "Which city am I in?" | Cell/Wi-Fi only, very cheap |
| `Balanced` | Nearby places, tagging a note | Moderate |
| `High` | Turn-by-turn or precise check-in | GPS, more battery, slower first fix |
| `Highest` / `BestForNavigation` | Navigation apps | Highest drain |

### [07 · Accessibility in React Native](level-3/07-accessibility.md)

*The essential props*

| Prop | Purpose | Notes |
|------|---------|-------|
| `accessibilityRole` | `button`, `link`, `checkbox`, `switch`, `header`, `image`, `tab`, `search`, `adjustable`, `progressbar`… | Changes how it's announced and which gestures apply |
| `accessibilityLabel` | Spoken name | Defaults to the text content; set it for icon-only controls |
| `accessibilityState` | `{ disabled, selected, checked, busy, expanded }` | Keep in sync with visual state |
| `accessibilityValue` | `{ min, max, now }` or `{ text }` | For sliders, progress, steppers |
| `accessibilityHint` | Outcome of the action | Short; users can turn hints off |
| `accessibilityLiveRegion` (Android) | `polite` / `assertive` | Announce changes to this element |
| `accessibilityElementsHidden` (iOS) / `importantForAccessibility="no-hide-descendants"` (Android) | Hide from screen readers | For decorative content |

### [08 · Writing a Native Module with the Expo Modules API](level-3/08-native-modules.md)

*The module definition DSL*

| DSL | Meaning | JavaScript side |
|-----|---------|-----------------|
| `Name("DeviceHealth")` | Module name used by `requireNativeModule` | — |
| `Constant("x") { … }` | Value computed once | property |
| `Function("f") { … }` | **Synchronous** call — keep it fast | `module.f()` returns a value |
| `AsyncFunction("g") { … }` | Runs off the JS thread, resolves a promise | `await module.g()` |
| `Events("onChange")` | Events the module can emit | `module.addListener('onChange', cb)` |
| `OnStartObserving` / `OnStopObserving` | First listener added / last removed | — |

### [09 · Development Builds, Prebuild &amp; Config Plugins](level-3/09-dev-builds-prebuild.md)

*Expo Go vs a development build*

| | Expo Go | Development build |
|---|---|---|
| Native code | Expo's SDK modules only | Any native library, your own modules |
| App identity | "Expo Go" | Your name, icon, bundle ID, scheme |
| Permission strings, plist/manifest config | Expo Go's | Yours |
| Push notifications | Limited | Full, with your credentials |
| Rebuild needed when | Never | Native dependencies or config change |
| JS changes | Fast Refresh | Fast Refresh |

## Level 4 · Production & Shipping

### [01 · Performance: Finding &amp; Fixing Jank](level-4/01-performance-profiling.md)

*Step 2a — JavaScript: what's rendering, and why?*

| Finding | Fix |
|---------|-----|
| A whole list re-renders when one item changes | `memo` the row, stable `renderItem`/callbacks (`useCallback`), keep item identity stable |
| Every screen re-renders on any store change | Selectors (Zustand) or split contexts (L2-04) |
| Expensive derived data recomputed every render | `useMemo`, or compute once when data arrives |
| A component re-renders on every keystroke elsewhere | Move the input's state down into the input component |
| Inline objects/arrays passed to memoised children | Hoist constants or memoise |

### [02 · Big Lists: Tuning FlatList &amp; Using FlashList](level-4/02-list-performance.md)

*FlatList props that matter*

| Prop | What it does | Trade-off |
|------|--------------|-----------|
| `getItemLayout` | Tells the list each row's size and offset up front | Only possible with fixed (or computable) heights; skips measurement and makes `scrollToIndex` exact |
| `initialNumToRender` | Rows rendered in the first batch | Enough to fill the first screen — more slows initial render |
| `maxToRenderPerBatch` | Rows rendered per incremental batch while scrolling | Higher fills faster but blocks JS longer per batch |
| `windowSize` | How many screen-heights stay mounted (default 21) | Lower saves memory and work; too low shows blank areas on fast flings |
| `removeClippedSubviews` | Detaches off-screen native views | Saves UI work on long lists; can cause rendering glitches in some layouts — test |
| `updateCellsBatchingPeriod` | Delay between batches | Rarely needs changing |

### [03 · Unit &amp; Component Tests with Jest and RNTL](level-4/03-testing-rntl.md)

*Query like a user*

| Query | Finds by | Example |
|-------|----------|---------|
| `getByRole` | accessibility role + name | `getByRole('button', { name: 'Add' })` |
| `getByLabelText` | `accessibilityLabel` | `getByLabelText('New habit name')` |
| `getByPlaceholderText` | input placeholder | `getByPlaceholderText('Search recipes')` |
| `getByText` | visible text | `getByText('No habits yet.')` |
| `getByDisplayValue` | current `TextInput` value | `getByDisplayValue('Walk')` |
| `getByTestId` | `testID` prop | last resort |

### [04 · End-to-End Testing with Maestro &amp; Detox](level-4/04-e2e-testing.md)

*Two tools, two philosophies*

| | Maestro | Detox |
|---|---|---|
| Tests written in | YAML "flows" | JavaScript/TypeScript (Jest) |
| How it drives the app | From outside, through the accessibility layer — black box | Inside the app, with a native library linked into a test build — grey box |
| Synchronisation | Waits and retries automatically for elements to appear | Tracks the app's busy state (network, timers, animations, JS thread) and waits until idle |
| Setup | Install a CLI; works with any build | Native configuration and dedicated test builds |
| Best fit | Fast to start; QA engineers and developers alike; Expo-friendly | Large teams wanting tests in TypeScript next to code, deep sync with app state |

### [05 · Mobile Security: Secrets, Storage &amp; the Network](level-4/05-security.md)

*Storing tokens and user data*

| Data | Store | Why |
|------|-------|-----|
| Refresh/access tokens, encryption keys | `expo-secure-store` (Keychain / Keystore-backed) | Encrypted at rest; `WHEN_UNLOCKED_THIS_DEVICE_ONLY` keeps it off backups |
| Sensitive user content (health notes, journals) | SQLite, ideally encrypted (SQLCipher-capable builds) with the key in SecureStore | App sandbox alone doesn't protect against device backups or rooted devices |
| Preferences, non-sensitive cache | AsyncStorage / SQLite | Fine unencrypted |

### [06 · Building with EAS &amp; App Signing](level-4/06-eas-build-signing.md)

*Versions: the two numbers*

| | User-facing | Internal (must increase every upload) |
|---|---|---|
| iOS | `CFBundleShortVersionString` ← `expo.version` (`1.4.0`) | `CFBundleVersion` ← `ios.buildNumber` |
| Android | `versionName` ← `expo.version` | `versionCode` ← `android.versionCode` (integer) |

### [08 · Over-the-Air Updates &amp; Release Strategy](level-4/08-ota-updates.md)

*What an OTA update can and can't change*

| Can change (JS bundle + assets) | Can't change (native binary) |
|---|---|
| Screens, components, styles, copy | Adding or upgrading a native module or SDK |
| Business logic, bug fixes in JS | Permission strings, entitlements, `Info.plist`/manifest |
| Images and fonts loaded through the bundle | App icon, splash screen, bundle ID |
| Feature flags you ship in JS | Expo SDK / React Native upgrades |

### [09 · Crash Reporting, Analytics &amp; Monitoring](level-4/09-monitoring.md)

*What to watch after a release*

| Signal | Where | Healthy looks like |
|--------|-------|--------------------|
| Crash-free sessions/users | Sentry release health; App Store Connect; Play Console vitals | Stable or improving versus the previous release |
| ANRs (Android "app not responding") | Play Console Android vitals | Below Play's bad-behaviour thresholds |
| New error types | Sentry issues filtered by release / update ID | None, or understood |
| Emergency launches | Your `captureMessage` above | Zero |
| Key funnels | Analytics (habit created → completed) | No sudden drops after a release |

---

Go deeper with the full [React Native Mastery Path](index.md) curriculum or test yourself with the [interview questions](interview-questions.md).
