---
title: "React Native Interview Questions and Answers (179 Questions)"
description: "179 React Native interview questions with answers and explanations, from beginner to advanced, organised by level. Free, from the React Native Mastery Path."
---

# React Native Interview Questions and Answers

**179 questions** with the correct answer and a short explanation, organised from beginner (Level 1) to advanced (Level 4). Every question comes from a lesson in the free React Native Mastery Path, so you can open the lesson to go deeper.

!!! tip "Practise, don't just read"
    Each lesson also ends with an interactive quiz, and the [final quiz](quiz.md) tests all four levels. Try to answer each question below before reading the answer.

## Level 1 · Foundations with Expo

### [01 · How React Native Works &amp; Setting Up Expo](level-1/01-how-react-native-works.md)

**Q1.** What does React Native create when you render &lt;View&gt;&lt;Text&gt;Hello&lt;/Text&gt;&lt;/View&gt;?

**Answer:** Real native views (a UIView on iOS, a ViewGroup on Android) kept in sync with the component tree

React Native keeps a tree of real platform views in sync with your React tree; nothing is drawn in a browser.

**Q2.** Which thread keeps a native ScrollView scrolling even while your JavaScript is busy for 200 ms?

**Answer:** The UI (main) thread, because scrolling is handled natively

Scrolling lives on the UI thread, so it does not need JavaScript. Taps that need new JS state do wait for the JS thread.

**Q3.** What is Expo Go?

**Answer:** A pre-built app that loads your JavaScript without you compiling native code, limited to the native modules Expo ships

Expo Go is great for starting out, but it only contains Expo's own native modules. Development builds remove that limit.

**Q4.** What is Hermes?

**Answer:** The JavaScript engine used by default in React Native, which compiles your JS to bytecode at build time

Hermes was built by Meta for React Native; precompiled bytecode means less parsing at launch.

### [02 · Core Components: View, Text, Image &amp; ScrollView](level-1/02-core-components.md)

**Q5.** What happens if you write &lt;View&gt;Hello&lt;/View&gt;?

**Answer:** It fails with 'Text strings must be rendered within a &lt;Text&gt; component'

In React Native every string must live inside a &lt;Text&gt;.

**Q6.** Given const unread = 0, what is wrong with &lt;View&gt;{unread &amp;&amp; &lt;Text&gt;new&lt;/Text&gt;}&lt;/View&gt;?

**Answer:** The expression evaluates to the number 0, which is rendered as a bare string and crashes on native

Make the condition a real boolean, for example unread &gt; 0 &amp;&amp;.

**Q7.** Why must a remote &lt;Image source={{ uri }} /&gt; be given a width and height?

**Answer:** React Native cannot know the size of a remote image at build time, so without a size it has zero size

Bundled assets via require() have dimensions known to Metro; remote images do not.

**Q8.** Which props make nested &lt;Text&gt; inside &lt;Text&gt; inherit the parent's font size and colour?

**Answer:** None needed: nested Text inherits style from its parent, the only place React Native styles inherit

React Native has no CSS cascade except for nested Text.

### [03 · Styling with StyleSheet](level-1/03-stylesheet-styling.md)

**Q9.** What unit do numeric style values like fontSize: 16 use?

**Answer:** Density-independent pixels (dp on Android, points on iOS)

The same number gives the same physical size on low- and high-density screens.

**Q10.** Which style prop correctly adds a border?

**Answer:** borderWidth together with borderColor (and optionally borderStyle)

The CSS border shorthand does not exist in React Native.

**Q11.** What does style={[styles.card, selected &amp;&amp; styles.cardSelected]} do?

**Answer:** Applies the card style, and also the selected style when selected is truthy (later entries win, falsy entries are ignored)

Style arrays replace CSS modifier classes.

**Q12.** How do you implement dark mode styling in React Native?

**Answer:** In JavaScript, for example with useColorScheme and a theme object, since there are no media queries

There are no pseudo-classes or media queries; they are done with state and hooks.

### [04 · Flexbox Layout the React Native Way](level-1/04-flexbox-layout.md)

**Q13.** What is the default flexDirection in React Native?

**Answer:** column

Phones are tall, so the default differs from the web's row.

**Q14.** Your root View has no flex: 1 and its background only covers half the screen. Why?

**Answer:** Without flex: 1 it is only as tall as its content

flex: 1 makes a view fill the available space.

**Q15.** Siblings with flex: 2 and flex: 1 share the remaining space in what ratio?

**Answer:** 2:1

flex with a positive number n behaves like flexGrow n, flexShrink 1 and flexBasis 0.

**Q16.** Which property positions children along the cross axis?

**Answer:** alignItems

justifyContent works on the main axis; alignItems on the cross axis.

### [05 · Touch &amp; Feedback with Pressable](level-1/05-pressable-touch.md)

**Q17.** Why is Pressable preferred over TouchableOpacity in new code?

**Answer:** It is the single modern component, with pressed state functions, Android ripple and hitSlop, replacing the older Touchable family

Pressable replaces the older components you still see in old code.

**Q18.** For a long press, what happens after delayLongPress elapses?

**Answer:** onLongPress fires and onPress does not

Press-and-hold triggers onLongPress instead of onPress.

**Q19.** A 20-point icon is hard to tap. What is the idiomatic fix that does not change layout?

**Answer:** Add hitSlop to extend the touch area invisibly

Platform guidance suggests at least about 44x44 points (Apple) or 48x48 dp (Android).

**Q20.** What does android_ripple do on iOS?

**Answer:** It is ignored

Combine it with a pressed style for iOS.

### [06 · TextInput &amp; Simple Forms](level-1/06-textinput-forms.md)

**Q21.** Which keyboard type shows a numeric pad without a decimal point?

**Answer:** keyboardType="number-pad"

decimal-pad adds the decimal separator.

**Q22.** Why should an email field set autoCapitalize="none"?

**Answer:** iOS would otherwise capitalise the first letter and login can fail

Also turn off autoCorrect for emails and usernames.

**Q23.** How do you move focus to the next field when the user presses the return key?

**Answer:** Use refs and call nextRef.current?.focus() in onSubmitEditing, with returnKeyType="next"

You handle submit behaviour with refs.

**Q24.** What does onChangeText give you?

**Answer:** The new text as a string directly

onChange gives a native event whose nativeEvent.text holds the same thing.

### [07 · Lists with FlatList &amp; SectionList](level-1/07-flatlist-sectionlist.md)

**Q25.** Why use FlatList instead of mapping 2,000 items inside a ScrollView?

**Answer:** FlatList virtualizes, rendering only items near the screen and saving memory

Mounting thousands of native views is slow and can get the app killed.

**Q26.** When is it wrong to use the item's index as keyExtractor?

**Answer:** When items can be inserted, removed or reordered, because React will reuse the wrong row's state

Use a stable unique ID.

**Q27.** Why should a vertical FlatList not be nested inside a vertical ScrollView?

**Answer:** The outer ScrollView gives it unlimited height so it renders everything and virtualization is lost

Use ListHeaderComponent and ListFooterComponent for extra content.

**Q28.** Which prop shows a message when the list has no items?

**Answer:** ListEmptyComponent

It renders when data is empty.

### [08 · Safe Areas, Platform Differences &amp; Screen Size](level-1/08-safe-area-platform.md)

**Q29.** What is the safe area?

**Answer:** The part of the screen not covered by system UI such as the notch, status bar and home indicator

Use react-native-safe-area-context to respect it.

**Q30.** Which hook reads safe-area insets in react-native-safe-area-context?

**Answer:** useSafeAreaInsets

It is handy for headers and footers that paint under system UI.

**Q31.** Which is the right way to branch on platform for a small value?

**Answer:** Platform.select({ ios: ..., android: ... }) or Platform.OS === 'ios'

For larger differences use .ios.tsx and .android.tsx files.

**Q32.** Which hook gives the current screen size and updates on rotation?

**Answer:** useWindowDimensions

The hook re-renders when dimensions change.

### [09 · Debugging React Native Apps](level-1/09-debugging.md)

**Q33.** How do you open React Native DevTools from the terminal running Expo?

**Answer:** Press j

It connects Chrome DevTools to the Hermes engine.

**Q34.** In a LogBox error overlay, how should you read the component stack?

**Answer:** Bottom-up: the bottom component rendered the one above it, so the cause is near the top entry

The component stack often tells you more than the JS stack.

**Q35.** What happened to the old 'Remote JS Debugging' mode?

**Answer:** It is gone from current React Native; DevTools attaches to Hermes on the device instead

Running JS in Chrome changed timing and hid real bugs.

**Q36.** A bug appears only in a release build. What is the correct first move?

**Answer:** Reproduce it in a release build and read native logs and crash reports, since release differs from development

Release builds minify, optimise and skip development checks.

### [10 · Project — Habit Tracker](level-1/10-project-habit-tracker.md)

**Q37.** Today is 2026-03-05 and doneDates are 03-05, 03-04 and 03-03. What does this print?

```js
const dayKey=d=>`${d.getFullYear()}-${String(d.getMonth()+1).padStart(2,'0')}-${String(d.getDate()).padStart(2,'0')}`;
const addDays=(k,n)=>{const [y,m,d]=k.split('-').map(Number);return dayKey(new Date(y,m-1,d+n));};
function currentStreak(doneDates,today){
  const done=new Set(doneDates);
  let cur=done.has(today)?today:addDays(today,-1);
  let s=0;
  while(done.has(cur)){s++;cur=addDays(cur,-1);}
  return s;
}

console.log(currentStreak(['2026-03-05','2026-03-04','2026-03-03'],'2026-03-05'));
```

**Answer:** 3

Three consecutive days ending today. Verified by running it in Node.

**Q38.** Today is 2026-03-05 and the habit was done on 03-04 and 03-03 but not yet today. What does this print?

```js
const dayKey=d=>`${d.getFullYear()}-${String(d.getMonth()+1).padStart(2,'0')}-${String(d.getDate()).padStart(2,'0')}`;
const addDays=(k,n)=>{const [y,m,d]=k.split('-').map(Number);return dayKey(new Date(y,m-1,d+n));};
function currentStreak(doneDates,today){
  const done=new Set(doneDates);
  let cur=done.has(today)?today:addDays(today,-1);
  let s=0;
  while(done.has(cur)){s++;cur=addDays(cur,-1);}
  return s;
}

console.log(currentStreak(['2026-03-04','2026-03-03'],'2026-03-05'));
```

**Answer:** 2

The streak counts from yesterday if today is not done yet, so it is not broken until the day ends. Verified by running it in Node.

**Q39.** A habit was done on 2026-02-28 and 2026-03-01 (no leap day). With today = 2026-03-01, what does this print?

```js
const dayKey=d=>`${d.getFullYear()}-${String(d.getMonth()+1).padStart(2,'0')}-${String(d.getDate()).padStart(2,'0')}`;
const addDays=(k,n)=>{const [y,m,d]=k.split('-').map(Number);return dayKey(new Date(y,m-1,d+n));};
function currentStreak(doneDates,today){
  const done=new Set(doneDates);
  let cur=done.has(today)?today:addDays(today,-1);
  let s=0;
  while(done.has(cur)){s++;cur=addDays(cur,-1);}
  return s;
}

console.log(currentStreak(['2026-02-28','2026-03-01'],'2026-03-01'));
```

**Answer:** 2

addDays goes through new Date(y, m-1, d+delta), so month boundaries are handled. Verified by running it in Node.

**Q40.** Why does the project store dates as local YYYY-MM-DD strings instead of Date objects?

**Answer:** It avoids time-zone surprises from comparing Date objects created at different times of day

A day key is the same however and whenever it is created.

## Level 2 · Navigation, State & Data

### [01 · Navigation Concepts: Stacks, Tabs &amp; Modals](level-2/01-navigation-concepts.md)

**Q41.** What happens to the List screen when you push Details onto a stack?

**Answer:** List stays mounted underneath, keeping its state and scroll position

Screens under the top of a stack stay mounted.

**Q42.** What is the usual structure for a tab navigator?

**Answer:** Each tab holds its own stack so each keeps an independent history

Switching tabs keeps each tab's place.

**Q43.** What is a modal screen for?

**Answer:** A self-contained task presented over everything else, such as composing or adding an item, which is dismissed rather than navigated back from

Modals rise from the bottom and are dismissed downward.

**Q44.** Why does a stack keeping screens mounted matter for state design?

**Answer:** A state change can re-render screens the user cannot see, so subscribe narrowly

This idea returns in the app-state lesson.

### [02 · File-Based Routing with Expo Router](level-2/02-expo-router.md)

**Q45.** In Expo Router, what does a file app/settings/notifications.tsx define?

**Answer:** The route /settings/notifications

Every file in app/ is a route.

**Q46.** What does _layout.tsx define?

**Answer:** The navigator (for example Stack or Tabs) for its folder

Layouts default-export a navigator.

**Q47.** Which element shows unmatched URLs?

**Answer:** +not-found.tsx

It is a special file in app/.

**Q48.** What is the difference between router.push and router.replace?

**Answer:** push adds a screen to the stack; replace swaps the current screen without growing history

Use replace for flows like sign-in where back should not return.

### [03 · Route Params, Deep Links &amp; Typed Routes](level-2/03-params-deep-links.md)

**Q49.** What does the file app/habit/[id].tsx mean?

**Answer:** A dynamic route where id is a parameter, such as /habit/42

Read it with useLocalSearchParams.

**Q50.** If you navigate with params: { count: 3 }, what type is count on the receiving screen?

**Answer:** string, because URL params are strings

The generic is a promise to TypeScript, not a runtime check; parse and validate.

**Q51.** What belongs in route params?

**Answer:** Identifiers and view state such as an id, tab, filter or sort

Objects in URLs bloat them and go stale.

**Q52.** Why prefer useLocalSearchParams over useGlobalSearchParams?

**Answer:** Global params re-render background screens whenever the user navigates anywhere

Screens stay mounted, so global params cause extra renders.

### [04 · App State: Context, useReducer &amp; Zustand](level-2/04-app-state.md)

**Q53.** What does this print?

```js
function reducer(state,a){
  switch(a.type){
    case 'added': return [...state,{id:a.id,name:a.name,doneDates:[],archived:false}];
    case 'toggled': return state.map(h=>h.id!==a.id?h:{...h,doneDates:h.doneDates.includes(a.day)?h.doneDates.filter(d=>d!==a.day):[...h.doneDates,a.day]});
    default: return state;
  }
}

let s=[];s=reducer(s,{type:'added',id:'a',name:'Run'});s=reducer(s,{type:'toggled',id:'a',day:'d1'});s=reducer(s,{type:'toggled',id:'a',day:'d1'});s=reducer(s,{type:'toggled',id:'a',day:'d2'});console.log(JSON.stringify(s[0].doneDates));
```

**Answer:** ["d2"]

Toggling twice removes d1; d2 is then added. Verified by running it in Node.

**Q54.** Where should a search box's text live?

**Answer:** In local useState in the component, not in a global store

Putting keystrokes in global state re-renders the whole app.

**Q55.** Why do Zustand selectors help in React Native apps?

**Answer:** A component re-renders only when the selected slice changes, which matters because screens stay mounted

Subscribe to exactly what each screen needs.

**Q56.** Which kind of state belongs in a data-fetching cache such as TanStack Query?

**Answer:** Server state owned by a backend

Do not copy server data into a global store where it goes stale.

### [05 · Fetching Data with TanStack Query](level-2/05-data-fetching.md)

**Q57.** What does the queryKey identify?

**Answer:** The cached data; components using the same key share one cache entry and request

Include every input of the query in the key, such as the search term.

**Q58.** Why must a queryFn throw on a non-OK response?

**Answer:** fetch does not reject on HTTP errors like 500, so you must throw for Query to see an error

Check res.ok and throw.

**Q59.** What does the API quirk 'meals: null' for no results mean for your client?

**Answer:** Normalise it to an empty array in your API layer

Keep API oddities out of components.

**Q60.** What does staleTime control?

**Answer:** How long fetched data is considered fresh before a background refetch is allowed

Cached data renders instantly while stale data refetches in the background.

### [06 · Local Persistence: AsyncStorage, SecureStore &amp; SQLite](level-2/06-local-persistence.md)

**Q61.** Where should an auth token be stored?

**Answer:** In SecureStore (the Keychain / Keystore)

AsyncStorage is unencrypted; treat it like a text file.

**Q62.** Which store suits records you filter, sort and join?

**Answer:** SQLite via expo-sqlite

AsyncStorage is for small key-value data.

**Q63.** Why does the prefs example merge saved JSON with defaultPrefs?

**Answer:** Adding a new preference in a later version does not break users who saved the old shape

Defensive loading with try/catch also protects against corrupt JSON.

**Q64.** What is the purpose of a version in a key like 'prefs:v1' or in SQLite user_version?

**Answer:** It lets a breaking change migrate data from one schema to the next

Versioned migrations keep existing users' data safe.

### [07 · Forms &amp; Validation with React Hook Form and Zod](level-2/07-forms-validation.md)

**Q65.** What does z.coerce.number() do in a form schema?

**Answer:** Turns the string from a TextInput into a number, then validates it

TextInput values are always strings.

**Q66.** What is .refine used for?

**Answer:** Cross-field or custom rules, with a path so the error attaches to the right field

For example: end date must be after start date.

**Q67.** Why does comparing 'YYYY-MM-DD' strings with &gt;= work?

**Answer:** The format sorts lexicographically in date order

A neat benefit of ISO-style dates.

**Q68.** What do z.input and z.output give you?

**Answer:** Two types: the raw form values and the parsed result

For example string vs number after coercion.

### [08 · Images, Fonts &amp; Assets](level-2/08-images-fonts-assets.md)

**Q69.** What does an @2x and @3x file next to image.png do?

**Answer:** Metro picks the variant matching the device density automatically

Layout uses the 1x size in points.

**Q70.** Why can't you write require(`./icons/${name}.png`)?

**Answer:** require paths must be static strings that Metro resolves at bundle time; use a map of requires

Build an object of static requires.

**Q71.** What does cachePolicy="memory-disk" give you in expo-image?

**Answer:** Decoded images in memory and files on disk so scrolling back or reopening does not refetch

Plus placeholders like BlurHash and transitions.

**Q72.** What is a BlurHash?

**Answer:** A short string that decodes to a blurry preview shown while the real image loads

Generate it on the server when the image is uploaded.

### [09 · The Keyboard, Modals &amp; Bottom Sheets](level-2/09-keyboard-modals.md)

**Q73.** On iOS, what does the software keyboard do to your layout?

**Answer:** Nothing: the keyboard covers it and you must move content yourself

Use KeyboardAvoidingView or insets.

**Q74.** Which KeyboardAvoidingView behavior is the most predictable on iOS?

**Answer:** padding

It adds bottom padding equal to the keyboard height.

**Q75.** What is keyboardVerticalOffset for?

**Answer:** Compensating for anything above the view, usually the navigation header

Wrong values leave the composer hidden or floating.

**Q76.** Why is edge-to-edge relevant to keyboard handling on Android?

**Answer:** The app draws behind system bars and the keyboard is reported as an inset, so you often must handle it explicitly

Test on both platforms, ideally on a real device.

### [10 · Project — Offline-Friendly Recipe Book](level-2/10-project-recipe-book.md)

**Q77.** Why are favourites stored with the full recipe JSON in SQLite?

**Answer:** So a favourited recipe opens offline

Network first, local fallback.

**Q78.** What does the recipe detail deep link recipebook://recipe/52785 require?

**Answer:** A URL scheme configured in app.json and a route app/recipe/[id].tsx

Expo Router gives every screen a URL.

**Q79.** What is an optimistic update for the favourite heart?

**Answer:** Update the UI immediately and roll back if the write fails

It feels instant.

**Q80.** Why is loadRecipe written as a pure function?

**Answer:** It can be tested without React or a device

Pure logic is the cheapest layer to test.

## Level 3 · Native Capabilities & Polish

### [01 · Under the Hood: Threads, JSI &amp; the New Architecture](level-3/01-architecture-new-arch.md)

**Q81.** Roughly how long is the per-frame budget at 60 Hz?

**Answer:** About 16.7 ms

Higher refresh rates leave less.

**Q82.** How did the old bridge communicate between JS and native?

**Answer:** By serialising calls to JSON and flushing them asynchronously in a queue

This caused overhead and prevented synchronous calls.

**Q83.** What does JSI let JavaScript do in the New Architecture?

**Answer:** Hold direct references to native objects and call them, including synchronously, without JSON serialisation

JSI underpins Fabric and TurboModules.

**Q84.** Since which React Native version is the New Architecture the default?

**Answer:** 0.76 (late 2024)

Older libraries may still need compatibility work.

### [02 · Device APIs: Camera, Location &amp; Haptics](level-3/02-device-apis.md)

**Q85.** Which can you NOT test properly on the iOS Simulator?

**Answer:** Camera and haptics (use a real device)

Simulators are not phones.

**Q86.** What is a good use of haptics?

**Answer:** Tie them to meaningful state changes such as a habit completed, using the semantic types

Users can disable system haptics, in which case calls do nothing.

**Q87.** What does getLastKnownPositionAsync give you?

**Answer:** A cheap, instant but possibly old position, good for a first guess

getCurrentPositionAsync with accuracy Balanced gets a fresh fix.

**Q88.** What battery trade-off applies to location?

**Answer:** Higher accuracy and continuous updates use more battery, so pick the lowest accuracy that works

Choose accuracy for the feature.

### [03 · Permissions Done Right](level-3/03-permissions.md)

**Q89.** On iOS, how many times is a permission's system prompt shown?

**Answer:** At most once; later requests return denied immediately

After that only Settings can change it.

**Q90.** What does canAskAgain: false mean?

**Answer:** The system will not show a prompt again, so send the user to Settings

Explain why and link to Settings.

**Q91.** When should you ask for notification permission?

**Answer:** When the user turns on a reminder, with context, not at first launch

Ask at the moment of value.

**Q92.** What happens on iOS if an app requests a permission without a usage description string?

**Answer:** It crashes (and App Review rejects vague strings)

Set strings in config plugins.

### [04 · Local &amp; Push Notifications](level-3/04-notifications.md)

**Q93.** What is a local notification?

**Answer:** One scheduled on the device by your app, with no server needed

Remote push comes from a server.

**Q94.** What do Android notification channels do?

**Answer:** Group notifications so users can control them separately in system settings

Create a channel before requesting permission on Android 13+.

**Q95.** By default, what happens to a notification that arrives while your app is open?

**Answer:** It is not shown unless you set a notification handler

Use setNotificationHandler once at startup.

**Q96.** Where can remote push be tested?

**Answer:** On a physical device, using a development build rather than Expo Go on Android

iOS also needs an Apple Developer account for APNs credentials.

### [05 · Animations with Animated &amp; Reanimated](level-3/05-animations.md)

**Q97.** Which properties can the Animated native driver animate?

**Answer:** Non-layout properties such as transform and opacity

Layout properties need the JS driver.

**Q98.** What are Reanimated worklets?

**Answer:** Small JavaScript functions run on the UI thread in a separate JS runtime

useAnimatedStyle's callback is one.

**Q99.** What is a shared value?

**Answer:** State both threads can read and write, used to drive animations

Created with useSharedValue.

**Q100.** Which function describes a spring-style move of a shared value?

**Answer:** withSpring

Others include withTiming, withDecay, withSequence.

### [06 · Gestures with React Native Gesture Handler](level-3/06-gestures.md)

**Q101.** Why does Gesture Handler feel smoother than the built-in responder system?

**Answer:** It recognises gestures natively on the UI thread, so a busy JS thread does not make gestures sticky

Paired with Reanimated, the loop avoids JS.

**Q102.** What must wrap your app to use Gesture Handler?

**Answer:** GestureHandlerRootView as close to the root as possible

With flex: 1.

**Q103.** How do you attach a gesture to a view?

**Answer:** Build it with Gesture.Pan() etc. and pass it to GestureDetector

The 2.x builder API.

**Q104.** Why should you let npx expo install choose the Gesture Handler version?

**Answer:** It picks the version matching your SDK, which may differ from npm's latest major

APIs differ between majors.

### [07 · Accessibility in React Native](level-3/07-accessibility.md)

**Q105.** What does a screen reader announce for a focused element, roughly in order?

**Answer:** Label, role, state, then optional hint

Give each interactive element all four.

**Q106.** Which prop sets the spoken name of an icon-only button?

**Answer:** accessibilityLabel

It defaults to text content.

**Q107.** Which prop describes checked or disabled?

**Answer:** accessibilityState

Keep it in sync with the visual state.

**Q108.** What should you do for users who enable Reduce Motion?

**Answer:** Reduce or skip non-essential animation, for example by checking the setting

Respect the OS accessibility settings.

### [08 · Writing a Native Module with the Expo Modules API](level-3/08-native-modules.md)

**Q109.** When should you write a native module?

**Answer:** When you need a platform capability that no maintained library provides

Check Expo SDK and React Native Directory first.

**Q110.** What does the Expo Modules API let you write?

**Answer:** Native modules in Swift and Kotlin with a declarative DSL

It generates the JSI bindings for you.

**Q111.** What does create-expo-module --local do?

**Answer:** Scaffolds a module inside your app's modules/ folder, autolinked

Local modules live in your repo.

**Q112.** Why does a native module require a rebuild?

**Answer:** Native code changes are compiled into the binary and cannot be hot-reloaded or sent over the air

Use a development build.

### [09 · Development Builds, Prebuild &amp; Config Plugins](level-3/09-dev-builds-prebuild.md)

**Q113.** What is a development build?

**Answer:** Your own debug app containing exactly your native dependencies, with the same dev tools as Expo Go

Use expo-dev-client.

**Q114.** What does npx expo prebuild do?

**Answer:** Generates the ios/ and android/ projects from a template, app config and config plugins

Continuous Native Generation lets you not commit them.

**Q115.** When do you rebuild the development build?

**Answer:** When the native side changes: dependencies or config

JS changes use Fast Refresh.

**Q116.** What is a config plugin?

**Answer:** Code that edits native project files such as Info.plist or the manifest during prebuild

Library authors and you can write them.

### [10 · Project — Field Notes with Camera, Location &amp; Offline Sync](level-3/10-project-field-notes.md)

**Q117.** In Field Notes, why is every note saved locally first?

**Answer:** Saving must never depend on the network; syncing happens afterwards

Offline-first design.

**Q118.** Why are photos copied out of the cache into the document directory?

**Answer:** The OS can delete cache files; documents persist

Persist what matters.

**Q119.** What is exponential backoff in sync retries?

**Answer:** Waiting progressively longer after each failure before retrying

It avoids hammering a struggling server.

**Q120.** How do you avoid uploading the same note version twice?

**Answer:** Track sync state and version per note and make the upload idempotent

Never upload twice for the same version.

## Level 4 · Production & Shipping

### [01 · Performance: Finding &amp; Fixing Jank](level-4/01-performance-profiling.md)

**Q121.** Where should you measure performance?

**Answer:** In a release build on a low-to-mid-range Android phone

Development builds are much slower.

**Q122.** JS frame rate drops but UI frame rate is fine. What does that suggest?

**Answer:** JavaScript is doing too much, such as heavy renders or big parsing

Taps feel delayed; scrolling stays smooth.

**Q123.** Which tool shows why a component re-rendered?

**Answer:** React DevTools Profiler with 'Record why each component rendered'

Open DevTools with j.

**Q124.** A whole list re-renders when one item changes. Typical fix?

**Answer:** memo the row, use stable callbacks and keep item identity stable

Measure again after the change.

### [02 · Big Lists: Tuning FlatList &amp; Using FlashList](level-4/02-list-performance.md)

**Q125.** What does getItemLayout enable?

**Answer:** Skips measurement and makes scrollToIndex exact, for fixed or computable row heights

It tells the list size and offset up front.

**Q126.** What does windowSize control?

**Answer:** How many screen-heights of content stay mounted (default 21)

Lower saves memory but may show blanks on fast flings.

**Q127.** How does FlashList differ from FlatList?

**Answer:** It recycles views instead of mounting and unmounting them

Set recyclingKey and use item types.

**Q128.** What is the first step before tuning list props?

**Answer:** Fix the rows: memoise, keep them shallow, size images and avoid heavy render work

No setting rescues a slow row.

### [03 · Unit &amp; Component Tests with Jest and RNTL](level-4/03-testing-rntl.md)

**Q129.** In RNTL 14, how do you call render?

**Answer:** await render(...), because render is async

Forgetting await gives 'render function has not been called'.

**Q130.** What does the jest-expo preset do?

**Answer:** Configures Babel like Metro, sets up React Native mocks and mocks native Expo modules

Tests run in Node.

**Q131.** In which order should you prefer queries?

**Answer:** By role, label and text as users find them, before testID

Query like a user.

**Q132.** What do component tests NOT tell you?

**Answer:** Whether the compiled app launches, native modules link or the keyboard covers a button

That needs E2E.

### [04 · End-to-End Testing with Maestro &amp; Detox](level-4/04-e2e-testing.md)

**Q133.** How does Maestro drive the app?

**Answer:** From outside through the accessibility layer, with YAML flows

Black box.

**Q134.** How does Detox synchronise with the app?

**Answer:** It tracks the app's busy state and waits until idle

Grey box.

**Q135.** Why write few E2E tests?

**Answer:** They are slower and more fragile, so cover journeys that must never break

Sign-in, core create flow, purchase.

**Q136.** What is testID for?

**Answer:** A stable handle for elements without stable text, mapped to the accessibility identifier or resource-id

Prefer labels and visible text first.

### [05 · Mobile Security: Secrets, Storage &amp; the Network](level-4/05-security.md)

**Q137.** What happens to a value set as EXPO_PUBLIC_MAPS_KEY?

**Answer:** It is inlined into the JS bundle and visible to anyone who unpacks the app

The lesson found it as plain text in the Hermes bytecode.

**Q138.** Where must private keys such as payment secrets live?

**Answer:** On your server, which the app calls after authenticating the user

Hiding a button does not protect an endpoint.

**Q139.** Where does mobile security ultimately live?

**Answer:** On the server, because the attacker controls the device

The app protects user data on the device.

**Q140.** What should you do with deep-link parameters?

**Answer:** Validate them as untrusted input

Treat links like any external input.

### [06 · Building with EAS &amp; App Signing](level-4/06-eas-build-signing.md)

**Q141.** What does an iOS .ipa vs Android .aab refer to?

**Answer:** The signed release binaries for the App Store and Google Play

Play uses Android App Bundles.

**Q142.** What is the eas.json preview profile typically used for?

**Answer:** Internal release-mode builds for QA and performance testing

An APK on Android.

**Q143.** What does appVersionSource: remote mean?

**Answer:** EAS manages build numbers remotely

With autoIncrement for production.

**Q144.** Why keep your keystore and credentials safe?

**Answer:** Losing signing credentials can block updates to an existing app (Play App Signing reduces the risk)

Let EAS manage credentials.

### [07 · Shipping to the App Store &amp; Google Play](level-4/07-store-submission.md)

**Q145.** What does TestFlight give you?

**Answer:** A beta stage where internal testers install immediately and external testers need a lightweight review

The closest thing to production.

**Q146.** What must apps with account creation also provide?

**Answer:** A way to delete the account in the app

Apple and Google both require a path.

**Q147.** What do you give reviewers if your app needs login?

**Answer:** A demo account in the review notes

Reviewers must reach all features.

**Q148.** Why use staged rollouts?

**Answer:** So a bad release reaches only a fraction of users first

Watch crashes and ramp up.

### [08 · Over-the-Air Updates &amp; Release Strategy](level-4/08-ota-updates.md)

**Q149.** What can an OTA update change?

**Answer:** The JavaScript bundle and its assets

Native changes need a new binary.

**Q150.** What is runtime version for?

**Answer:** A build only accepts updates with the same runtime version, protecting against JS/native mismatch

Mismatch would crash.

**Q151.** What does the fingerprint policy do?

**Answer:** Hashes everything affecting the native build so the runtime version changes only when native code changes

A JS-only change does not change it.

**Q152.** What are OTA updates NOT meant for?

**Answer:** Changing the app's primary purpose or swapping in a different app after review

Stay within store rules.

### [09 · Crash Reporting, Analytics &amp; Monitoring](level-4/09-monitoring.md)

**Q153.** What do route ErrorBoundaries catch?

**Answer:** Errors thrown while rendering a route's subtree

Handle the others in place and with a reporter.

**Q154.** Why upload source maps to Sentry?

**Answer:** So production stack traces map back to your readable source

Tag releases to match builds.

**Q155.** What is a sensible analytics approach?

**Answer:** Collect a small amount, respectfully and with consent where required

Declare it in privacy labels.

**Q156.** What does __DEV__ allow in the error screen?

**Answer:** Show details in development and a friendly message in release

Do not leak internals to users.

### [10 · Capstone — Ship a Production-Ready App](level-4/10-capstone.md)

**Q157.** What is the capstone's deliverable?

**Answer:** Evidence that the app is a release candidate: tests, accessibility, performance, release setup and monitoring

It is about readiness.

**Q158.** What goes in ARCHITECTURE.md?

**Answer:** Where state lives, data flows, offline behaviour and the navigation tree

One page a reviewer can follow.

**Q159.** What should E2E flows cover in the capstone?

**Answer:** Add habit, complete it, relaunch and still completed; and the permission-denied reminder path

Few, valuable flows.

**Q160.** What does a preflight script help with?

**Answer:** Checking the release gates automatically before you submit

Catch mistakes early.

## Mixed questions across all levels

**Q161.** A React Native app feels native mainly because...

**Answer:** its component tree is mapped to real platform views, and scrolling, text and accessibility come from the OS

Components map to native views; that is also why div, CSS files and onClick do not exist.

**Q162.** A tap handler updates state but the UI responds 300 ms late, while scrolling stays smooth. Which is the likely cause?

**Answer:** The JS thread is busy (heavy render or computation); scrolling runs on the UI thread

JS frame rate dropping with UI fine points to JavaScript work.

**Q163.** Which pair is the correct fix for 'my background only covers half the screen' and 'my list doesn't scroll'?

**Answer:** Give the root View and the list's parent flex: 1 so they receive a bounded height

Without flex: 1 a view is only as tall as its content.

**Q164.** What is the safest way to render a count badge only when unread &gt; 0?

**Answer:** {unread &gt; 0 &amp;&amp; &lt;Text&gt;{unread}&lt;/Text&gt;}

A 0 left of &amp;&amp; is rendered as a bare string and crashes inside a View.

**Q165.** Which approach gives native-feeling press feedback on both platforms?

**Answer:** Pressable with a pressed style (iOS) and android_ripple (Android), plus accessibilityRole and label

Pressable replaces the older Touchable family.

**Q166.** A search screen stores the text in a global store and the whole app re-renders on every keystroke. What is the better design?

**Answer:** Keep the text in local useState and put only shared data in the store, with narrow selectors

Match each kind of state to the right place.

**Q167.** Which store holds a refresh token?

**Answer:** expo-secure-store (Keychain/Keystore)

AsyncStorage is unencrypted; treat it like a text file.

**Q168.** Why does data fetched with TanStack Query appear instantly when you return to a screen?

**Answer:** It is cached by query key and renders from the cache while stale data refetches in the background

Server state is a cache you do not own.

**Q169.** A form field holds the string '3' but your schema needs a number. What does the schema do?

**Answer:** z.coerce.number() converts then validates it

TextInput values are always strings.

**Q170.** What is the correct pattern for an offline-friendly detail screen?

**Answer:** Try the network, fall back to the locally stored copy and tell the user it is an offline copy

Store the full record when favouriting.

**Q171.** Which statement about the New Architecture is true?

**Answer:** JSI lets JS call native objects directly without JSON serialisation, enabling synchronous calls

Fabric and TurboModules build on JSI.

**Q172.** An iOS user denied the camera permission last week. What can your app do?

**Answer:** Explain the benefit and send the user to Settings, because the system will not prompt again

canAskAgain false means Settings only.

**Q173.** Which animation is safe to run with useNativeDriver: true?

**Answer:** A transform scale or an opacity fade

The native driver supports non-layout properties.

**Q174.** A swipe-to-complete gesture must stay smooth while JS is busy. What do you use?

**Answer:** Gesture Handler plus Reanimated, so recognition and animation run on the UI thread

Callbacks are worklets.

**Q175.** What is the right accessibility setup for an icon-only delete button?

**Answer:** accessibilityRole="button" with accessibilityLabel="Delete" and a large enough touch target

Label, role, state, then hint.

**Q176.** A teammate adds a library with native code. How do you test it?

**Answer:** Create a development build; Expo Go only includes Expo's own native modules

Rebuild only when native dependencies or config change.

**Q177.** A bug is only in the release build. What is your first step?

**Answer:** Reproduce it in a release build and check native crash logs and the Sentry report with source maps

Development and release differ a lot.

**Q178.** Which JS-only fix can ship over the air, and which cannot?

**Answer:** A copy or logic fix can; a new native module or permission string cannot

Runtime version ties updates to compatible binaries.

**Q179.** Which belongs in the app bundle?

**Answer:** Only keys that are safe to be public; real secrets stay on the server

EXPO_PUBLIC_ values are inlined into the bundle and readable.

## Practical coding questions

### Habit statistics: current streak, longest streak and weekly rate

Implement currentStreak, longestStreak and weeklyRate for habits whose check-ins are local YYYY-MM-DD strings. Handle duplicates, month boundaries and leap days. This is plain JavaScript logic — the kind you would put in a pure module and unit-test with Jest before wiring it into a React Native screen.

Starter code:

```
const dayKey=d=>`${d.getFullYear()}-${String(d.getMonth()+1).padStart(2,'0')}-${String(d.getDate()).padStart(2,'0')}`;
const addDays=(k,n)=>{const [y,m,d]=k.split('-').map(Number);return dayKey(new Date(y,m-1,d+n));};

// TODO: implement currentStreak(doneDates, today), longestStreak(doneDates), weeklyRate(doneDates, today)
//  - currentStreak: consecutive days ending today, or yesterday if today is not done yet
//  - longestStreak: longest run of consecutive days anywhere (ignore duplicates; handle month/leap-year boundaries)
//  - weeklyRate: percentage (rounded) of the last 7 days, today included, that were done
function summarize(h, today) {
  return `${h.name}: current ${currentStreak(h.doneDates, today)}, longest ${longestStreak(h.doneDates)}, week ${weeklyRate(h.doneDates, today)}%`;
}
const habits = [
  { name: 'Read', doneDates: ['2026-02-26','2026-02-27','2026-02-28','2026-03-01','2026-03-02','2026-03-04','2026-03-05'] },
  { name: 'Run', doneDates: ['2026-03-05'] },
  { name: 'Stretch', doneDates: [] },
  { name: 'Water', doneDates: ['2026-03-03','2026-03-04','2026-03-05','2026-03-05'] },
];
for (const h of habits) console.log(summarize(h, '2026-03-05'));
console.log(longestStreak(['2024-02-28','2024-02-29','2024-03-01']));
```

Expected output:

```text
Read: current 2, longest 5, week 86%
Run: current 1, longest 1, week 14%
Stretch: current 0, longest 0, week 0%
Water: current 3, longest 3, week 43%
3
```

??? success "Show a solution"

    ```
    const dayKey=d=>`${d.getFullYear()}-${String(d.getMonth()+1).padStart(2,'0')}-${String(d.getDate()).padStart(2,'0')}`;
    const addDays=(k,n)=>{const [y,m,d]=k.split('-').map(Number);return dayKey(new Date(y,m-1,d+n));};

    function currentStreak(doneDates, today) {
      const done = new Set(doneDates);
      let cur = done.has(today) ? today : addDays(today, -1);
      let s = 0;
      while (done.has(cur)) { s++; cur = addDays(cur, -1); }
      return s;
    }
    function longestStreak(doneDates) {
      const days = [...new Set(doneDates)].sort();
      let best = 0, run = 0, prev = null;
      for (const d of days) {
        run = prev !== null && addDays(prev, 1) === d ? run + 1 : 1;
        best = Math.max(best, run);
        prev = d;
      }
      return best;
    }
    function weeklyRate(doneDates, today) {
      const done = new Set(doneDates);
      let n = 0;
      for (let i = 0; i < 7; i++) if (done.has(addDays(today, -i))) n++;
      return Math.round((n / 7) * 100);
    }
    function summarize(h, today) {
      return `${h.name}: current ${currentStreak(h.doneDates, today)}, longest ${longestStreak(h.doneDates)}, week ${weeklyRate(h.doneDates, today)}%`;
    }
    const habits = [
      { name: 'Read', doneDates: ['2026-02-26','2026-02-27','2026-02-28','2026-03-01','2026-03-02','2026-03-04','2026-03-05'] },
      { name: 'Run', doneDates: ['2026-03-05'] },
      { name: 'Stretch', doneDates: [] },
      { name: 'Water', doneDates: ['2026-03-03','2026-03-04','2026-03-05','2026-03-05'] },
    ];
    for (const h of habits) console.log(summarize(h, '2026-03-05'));
    console.log(longestStreak(['2024-02-28','2024-02-29','2024-03-01']));
    ```

    Day keys are built with new Date(y, m-1, d+delta), which rolls over months and leap days correctly. longestStreak sorts unique keys and extends a run while the next day follows the previous one. Verified by running the solution in Node; React Native itself was not needed because this logic is deliberately independent of the UI.

### Offline sync queue with exponential backoff

Implement backoff and SyncQueue so notes saved offline upload later with retries, never upload the same id:version twice, and a re-saved note becomes a new version. Time is passed in as a number so the behaviour is deterministic; there is no real network here.

Starter code:

```
const BASE = 1000, CAP = 8000;
// TODO: backoff(attempt) = min(BASE * 2^attempt, CAP)
// TODO: class SyncQueue(upload) with save(id, body), async tick(now), and a log array
//  - save increments the note's version if the id already exists (new version = new pending upload)
//  - tick(now) tries every note that is not synced and whose nextAt <= now
//  - never upload the same id:version twice (track sent keys)
//  - on failure: attempts++, state 'failed', nextAt = now + backoff(attempts-1), log "t=<now> <id>:<v> failed (retry at <nextAt>)"
//  - on success: log "t=<now> <id>:<v> synced"
let calls = 0;
const failures = { a: 3 };
const q = new SyncQueue(async (id) => { calls++; if (failures[id] > 0) { failures[id]--; throw new Error('offline'); } });
q.save('a', 'first'); q.save('b', 'hello');
for (const t of [0, 500, 1000, 3000, 7000, 8000]) await q.tick(t);
q.save('b', 'hello edited');
await q.tick(8000);
await q.tick(9000);
console.log(q.log.join('\n'));
console.log('calls', calls, 'states', [...q.notes.values()].map(n => `${n.id}v${n.version}:${n.state}`).join(' '));
console.log([0,1,2,3,4,5].map(backoff).join(','));
```

Expected output:

```text
t=0 a:1 failed (retry at 1000)
t=0 b:1 synced
t=1000 a:1 failed (retry at 3000)
t=3000 a:1 failed (retry at 7000)
t=7000 a:1 synced
t=8000 b:2 synced
calls 6 states av1:synced bv2:synced
1000,2000,4000,8000,8000,8000
```

??? success "Show a solution"

    ```
    const BASE = 1000, CAP = 8000;
    function backoff(attempt) { return Math.min(BASE * 2 ** attempt, CAP); }

    class SyncQueue {
      constructor(upload) { this.upload = upload; this.notes = new Map(); this.sent = new Set(); this.log = []; }
      save(id, body) {
        const prev = this.notes.get(id);
        this.notes.set(id, { id, body, version: prev ? prev.version + 1 : 1, attempts: 0, nextAt: 0, state: 'pending' });
      }
      async tick(now) {
        for (const n of this.notes.values()) {
          if (n.state === 'synced' || n.nextAt > now) continue;
          const key = `${n.id}:${n.version}`;
          if (this.sent.has(key)) { n.state = 'synced'; continue; }
          try {
            await this.upload(n.id, n.version, n.body);
            this.sent.add(key); n.state = 'synced';
            this.log.push(`t=${now} ${key} synced`);
          } catch (e) {
            n.attempts++; n.state = 'failed'; n.nextAt = now + backoff(n.attempts - 1);
            this.log.push(`t=${now} ${key} failed (retry at ${n.nextAt})`);
          }
        }
      }
    }
    let calls = 0;
    const failures = { a: 3 };
    const q = new SyncQueue(async (id) => { calls++; if (failures[id] > 0) { failures[id]--; throw new Error('offline'); } });
    q.save('a', 'first'); q.save('b', 'hello');
    for (const t of [0, 500, 1000, 3000, 7000, 8000]) await q.tick(t);
    q.save('b', 'hello edited');
    await q.tick(8000);
    await q.tick(9000);
    console.log(q.log.join('\n'));
    console.log('calls', calls, 'states', [...q.notes.values()].map(n => `${n.id}v${n.version}:${n.state}`).join(' '));
    console.log([0,1,2,3,4,5].map(backoff).join(','));
    ```

    Each failure doubles the wait up to a cap, nextAt gates retries, and the sent set keyed by id:version makes uploads idempotent. This is a simulation with a fake uploader and fake clock, not a real device or network; verified by running it in Node.

---

More free practice: the [React Native Mastery Path](index.md) curriculum, the [Interview Prep Pack](https://learnmasterypath.com/interview-pack/) and the [free resume ATS check](https://learnmasterypath.com/resume/).
