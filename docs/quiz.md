---
title: "Final Quiz"
description: "Test what you have learned across all four levels of React Native Mastery Path with a 19-question final quiz."
---

# Final Quiz

19 questions drawn from all four levels of this course. You need **70% (14 out of 19)** to pass —
you can retake it as many times as you like. Your best result is saved in this browser.

<div id="mp-quiz-root"></div>

<script type="application/json" id="mp-quiz-data">
[
  {
    "question": "A React Native app feels native mainly because...",
    "options": [
      "its component tree is mapped to real platform views, and scrolling, text and accessibility come from the OS",
      "it runs Swift and Kotlin translations of your JS at runtime",
      "it is compiled to a web page that looks like an app",
      "it draws everything itself on a canvas"
    ],
    "correctIndex": 0,
    "explanation": "Components map to native views; that is also why div, CSS files and onClick do not exist."
  },
  {
    "question": "A tap handler updates state but the UI responds 300 ms late, while scrolling stays smooth. Which is the likely cause?",
    "options": [
      "The GPU is faulty",
      "The UI thread is overloaded by shadows",
      "The JS thread is busy (heavy render or computation); scrolling runs on the UI thread",
      "Metro is too slow"
    ],
    "correctIndex": 2,
    "explanation": "JS frame rate dropping with UI fine points to JavaScript work."
  },
  {
    "question": "Which pair is the correct fix for 'my background only covers half the screen' and 'my list doesn't scroll'?",
    "options": [
      "Switch to ScrollView everywhere",
      "Give the root View and the list's parent flex: 1 so they receive a bounded height",
      "Use position: fixed",
      "Set display: block"
    ],
    "correctIndex": 1,
    "explanation": "Without flex: 1 a view is only as tall as its content."
  },
  {
    "question": "What is the safest way to render a count badge only when unread > 0?",
    "options": [
      "{unread || <Text />}",
      "Return the number directly in a View",
      "{unread && <Text>{unread}<\/Text>}",
      "{unread > 0 && <Text>{unread}<\/Text>}"
    ],
    "correctIndex": 3,
    "explanation": "A 0 left of && is rendered as a bare string and crashes inside a View."
  },
  {
    "question": "Which approach gives native-feeling press feedback on both platforms?",
    "options": [
      "A web hover style",
      "TouchableWithoutFeedback only",
      "Pressable with a pressed style (iOS) and android_ripple (Android), plus accessibilityRole and label",
      "onClick on a View"
    ],
    "correctIndex": 2,
    "explanation": "Pressable replaces the older Touchable family."
  },
  {
    "question": "A search screen stores the text in a global store and the whole app re-renders on every keystroke. What is the better design?",
    "options": [
      "Add more global state",
      "Use AsyncStorage",
      "Debounce the whole app",
      "Keep the text in local useState and put only shared data in the store, with narrow selectors"
    ],
    "correctIndex": 3,
    "explanation": "Match each kind of state to the right place."
  },
  {
    "question": "Which store holds a refresh token?",
    "options": [
      "AsyncStorage",
      "SQLite without encryption",
      "A bundled JSON file",
      "expo-secure-store (Keychain/Keystore)"
    ],
    "correctIndex": 3,
    "explanation": "AsyncStorage is unencrypted; treat it like a text file."
  },
  {
    "question": "Why does data fetched with TanStack Query appear instantly when you return to a screen?",
    "options": [
      "Navigation freezes the network",
      "It is cached by query key and renders from the cache while stale data refetches in the background",
      "Hermes caches fetch calls",
      "The server remembers you"
    ],
    "correctIndex": 1,
    "explanation": "Server state is a cache you do not own."
  },
  {
    "question": "A form field holds the string '3' but your schema needs a number. What does the schema do?",
    "options": [
      "Rejects every string",
      "z.coerce.number() converts then validates it",
      "Stores it as text",
      "Ignores it"
    ],
    "correctIndex": 1,
    "explanation": "TextInput values are always strings."
  },
  {
    "question": "What is the correct pattern for an offline-friendly detail screen?",
    "options": [
      "Only work online",
      "Try the network, fall back to the locally stored copy and tell the user it is an offline copy",
      "Throw away local data",
      "Always show a spinner"
    ],
    "correctIndex": 1,
    "explanation": "Store the full record when favouriting."
  },
  {
    "question": "Which statement about the New Architecture is true?",
    "options": [
      "It removes Hermes",
      "JSI lets JS call native objects directly without JSON serialisation, enabling synchronous calls",
      "It adds a faster JSON bridge",
      "It only affects Android"
    ],
    "correctIndex": 1,
    "explanation": "Fabric and TurboModules build on JSI."
  },
  {
    "question": "An iOS user denied the camera permission last week. What can your app do?",
    "options": [
      "Request it again repeatedly",
      "Silently ignore it",
      "Explain the benefit and send the user to Settings, because the system will not prompt again",
      "Re-prompt on every launch"
    ],
    "correctIndex": 2,
    "explanation": "canAskAgain false means Settings only."
  },
  {
    "question": "Which animation is safe to run with useNativeDriver: true?",
    "options": [
      "A transform scale or an opacity fade",
      "Animating height",
      "Animating flex",
      "Animating top"
    ],
    "correctIndex": 0,
    "explanation": "The native driver supports non-layout properties."
  },
  {
    "question": "A swipe-to-complete gesture must stay smooth while JS is busy. What do you use?",
    "options": [
      "LayoutAnimation only",
      "setInterval",
      "Gesture Handler plus Reanimated, so recognition and animation run on the UI thread",
      "setState in onTouchMove"
    ],
    "correctIndex": 2,
    "explanation": "Callbacks are worklets."
  },
  {
    "question": "What is the right accessibility setup for an icon-only delete button?",
    "options": [
      "Colour only",
      "Only an emoji",
      "accessibilityRole=\"button\" with accessibilityLabel=\"Delete\" and a large enough touch target",
      "No label"
    ],
    "correctIndex": 2,
    "explanation": "Label, role, state, then hint."
  },
  {
    "question": "A teammate adds a library with native code. How do you test it?",
    "options": [
      "Reload Expo Go",
      "Use the web",
      "Edit node_modules",
      "Create a development build; Expo Go only includes Expo's own native modules"
    ],
    "correctIndex": 3,
    "explanation": "Rebuild only when native dependencies or config change."
  },
  {
    "question": "A bug is only in the release build. What is your first step?",
    "options": [
      "Ignore it",
      "Add console.log in development",
      "Blame Metro",
      "Reproduce it in a release build and check native crash logs and the Sentry report with source maps"
    ],
    "correctIndex": 3,
    "explanation": "Development and release differ a lot."
  },
  {
    "question": "Which JS-only fix can ship over the air, and which cannot?",
    "options": [
      "A copy or logic fix can; a new native module or permission string cannot",
      "Both can",
      "Neither can",
      "Only permission strings can"
    ],
    "correctIndex": 0,
    "explanation": "Runtime version ties updates to compatible binaries."
  },
  {
    "question": "Which belongs in the app bundle?",
    "options": [
      "Admin tokens",
      "Payment secret keys",
      "Database passwords",
      "Only keys that are safe to be public; real secrets stay on the server"
    ],
    "correctIndex": 3,
    "explanation": "EXPO_PUBLIC_ values are inlined into the bundle and readable."
  }
]
</script>

## Coding challenges

Try these on your own computer, then compare your output. Nothing you write is sent anywhere.

### Habit statistics: current streak, longest streak and weekly rate

Implement currentStreak, longestStreak and weeklyRate for habits whose check-ins are local YYYY-MM-DD strings. Handle duplicates, month boundaries and leap days. This is plain JavaScript logic — the kind you would put in a pure module and unit-test with Jest before wiring it into a React Native screen.

**Starter code**

```javascript
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

**Expected output**

```text
Read: current 2, longest 5, week 86%
Run: current 1, longest 1, week 14%
Stretch: current 0, longest 0, week 0%
Water: current 3, longest 3, week 43%
3
```

<details><summary>Show solution and explanation</summary>

<pre><code>const dayKey=d=&gt;`${d.getFullYear()}-${String(d.getMonth()+1).padStart(2,&#x27;0&#x27;)}-${String(d.getDate()).padStart(2,&#x27;0&#x27;)}`;
const addDays=(k,n)=&gt;{const [y,m,d]=k.split(&#x27;-&#x27;).map(Number);return dayKey(new Date(y,m-1,d+n));};

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
    run = prev !== null &amp;&amp; addDays(prev, 1) === d ? run + 1 : 1;
    best = Math.max(best, run);
    prev = d;
  }
  return best;
}
function weeklyRate(doneDates, today) {
  const done = new Set(doneDates);
  let n = 0;
  for (let i = 0; i &lt; 7; i++) if (done.has(addDays(today, -i))) n++;
  return Math.round((n / 7) * 100);
}
function summarize(h, today) {
  return `${h.name}: current ${currentStreak(h.doneDates, today)}, longest ${longestStreak(h.doneDates)}, week ${weeklyRate(h.doneDates, today)}%`;
}
const habits = [
  { name: &#x27;Read&#x27;, doneDates: [&#x27;2026-02-26&#x27;,&#x27;2026-02-27&#x27;,&#x27;2026-02-28&#x27;,&#x27;2026-03-01&#x27;,&#x27;2026-03-02&#x27;,&#x27;2026-03-04&#x27;,&#x27;2026-03-05&#x27;] },
  { name: &#x27;Run&#x27;, doneDates: [&#x27;2026-03-05&#x27;] },
  { name: &#x27;Stretch&#x27;, doneDates: [] },
  { name: &#x27;Water&#x27;, doneDates: [&#x27;2026-03-03&#x27;,&#x27;2026-03-04&#x27;,&#x27;2026-03-05&#x27;,&#x27;2026-03-05&#x27;] },
];
for (const h of habits) console.log(summarize(h, &#x27;2026-03-05&#x27;));
console.log(longestStreak([&#x27;2024-02-28&#x27;,&#x27;2024-02-29&#x27;,&#x27;2024-03-01&#x27;]));
</code></pre>

<p>Day keys are built with new Date(y, m-1, d+delta), which rolls over months and leap days correctly. longestStreak sorts unique keys and extends a run while the next day follows the previous one. Verified by running the solution in Node; React Native itself was not needed because this logic is deliberately independent of the UI.</p>
</details>

### Offline sync queue with exponential backoff

Implement backoff and SyncQueue so notes saved offline upload later with retries, never upload the same id:version twice, and a re-saved note becomes a new version. Time is passed in as a number so the behaviour is deterministic; there is no real network here.

**Starter code**

```javascript
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

**Expected output**

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

<details><summary>Show solution and explanation</summary>

<pre><code>const BASE = 1000, CAP = 8000;
function backoff(attempt) { return Math.min(BASE * 2 ** attempt, CAP); }

class SyncQueue {
  constructor(upload) { this.upload = upload; this.notes = new Map(); this.sent = new Set(); this.log = []; }
  save(id, body) {
    const prev = this.notes.get(id);
    this.notes.set(id, { id, body, version: prev ? prev.version + 1 : 1, attempts: 0, nextAt: 0, state: &#x27;pending&#x27; });
  }
  async tick(now) {
    for (const n of this.notes.values()) {
      if (n.state === &#x27;synced&#x27; || n.nextAt &gt; now) continue;
      const key = `${n.id}:${n.version}`;
      if (this.sent.has(key)) { n.state = &#x27;synced&#x27;; continue; }
      try {
        await this.upload(n.id, n.version, n.body);
        this.sent.add(key); n.state = &#x27;synced&#x27;;
        this.log.push(`t=${now} ${key} synced`);
      } catch (e) {
        n.attempts++; n.state = &#x27;failed&#x27;; n.nextAt = now + backoff(n.attempts - 1);
        this.log.push(`t=${now} ${key} failed (retry at ${n.nextAt})`);
      }
    }
  }
}
let calls = 0;
const failures = { a: 3 };
const q = new SyncQueue(async (id) =&gt; { calls++; if (failures[id] &gt; 0) { failures[id]--; throw new Error(&#x27;offline&#x27;); } });
q.save(&#x27;a&#x27;, &#x27;first&#x27;); q.save(&#x27;b&#x27;, &#x27;hello&#x27;);
for (const t of [0, 500, 1000, 3000, 7000, 8000]) await q.tick(t);
q.save(&#x27;b&#x27;, &#x27;hello edited&#x27;);
await q.tick(8000);
await q.tick(9000);
console.log(q.log.join(&#x27;\n&#x27;));
console.log(&#x27;calls&#x27;, calls, &#x27;states&#x27;, [...q.notes.values()].map(n =&gt; `${n.id}v${n.version}:${n.state}`).join(&#x27; &#x27;));
console.log([0,1,2,3,4,5].map(backoff).join(&#x27;,&#x27;));
</code></pre>

<p>Each failure doubles the wait up to a cap, nextAt gates retries, and the sent set keyed by id:version makes uploads idempotent. This is a simulation with a fake uploader and fake clock, not a real device or network; verified by running it in Node.</p>
</details>
