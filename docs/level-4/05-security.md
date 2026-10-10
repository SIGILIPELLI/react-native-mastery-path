---
description: "Secure a React Native app: why EXPO_PUBLIC values ship in the bundle, token storage, TLS and pinning, deep-link validation and WebView risks."
---

# 05 · Mobile Security: Secrets, Storage & the Network

The most important fact about mobile security is uncomfortable: **your app runs on a device the
attacker controls.** Anyone can download your app from the store, unpack it, read its JavaScript
bundle and resources, run it on a rooted or jailbroken phone, intercept its network traffic and call
your API directly. Security therefore lives on the **server**; the app's job is to avoid making the
attacker's life easy and to protect the *user's* data on their own device.

The [Cybersecurity Mastery Path](https://sigilipelli.github.io/cybersecurity-mastery-path/) covers
security fundamentals broadly, and the mobile-testing lesson in the
[Ethical Hacking Mastery Path](https://sigilipelli.github.io/ethical-hacking-mastery-path/) shows the
attacker's view. This lesson is the React Native developer's checklist.

## 1. Nothing in the bundle is secret

Expo inlines environment variables prefixed with `EXPO_PUBLIC_` into your JavaScript at build time.
The name is the warning: they are **public**. To make the point concrete, this course exported a
release bundle with a fake key:

```tsx title="App.tsx (demonstration)"
const MAPS_KEY = process.env.EXPO_PUBLIC_MAPS_KEY;

export default function App() {
  // Imagine this key is passed to a maps SDK.
  globalThis.fetch?.(`https://maps.example.com/tiles?key=${MAPS_KEY}`);
  return null;
}
```

```bash
EXPO_PUBLIC_MAPS_KEY=demo-not-a-real-key-12345 npx expo export --platform android --clear
grep -a -c "demo-not-a-real-key-12345" dist/_expo/static/js/android/index-*.hbc
```

`expo export` produced a single Hermes bytecode file (about 1.4 MB for this tiny app), and `grep`
found the key in it, as plain text, once. Hermes bytecode is not encryption; anyone who unzips the
APK or IPA can do the same. The same applies to anything else in the bundle: API URLs, feature flags,
hard-coded "admin" checks.

**So:**

- Keys in the app must be keys that are **safe to be public** — restricted by the provider to your
  app's bundle ID / package name / signing certificate, with quotas (typical for maps and analytics
  SDKs).
- **Private keys** (payment provider secret keys, third-party API keys with billing, OpenAI-style
  keys, database credentials) must never ship in the app. Put them on your server and have the app
  call your server, which authenticates the user and calls the third party.
- Server-side **authorisation** decides what a user can do. Hiding a button in the app protects
  nothing; the API must check.

## 2. Storing tokens and user data

Recap of L2-06, from a security angle:

| Data | Store | Why |
|------|-------|-----|
| Refresh/access tokens, encryption keys | `expo-secure-store` (Keychain / Keystore-backed) | Encrypted at rest; `WHEN_UNLOCKED_THIS_DEVICE_ONLY` keeps it off backups |
| Sensitive user content (health notes, journals) | SQLite, ideally encrypted (SQLCipher-capable builds) with the key in SecureStore | App sandbox alone doesn't protect against device backups or rooted devices |
| Preferences, non-sensitive cache | AsyncStorage / SQLite | Fine unencrypted |

Token practices:

- Use **short-lived access tokens** with a refresh token, so a leaked access token expires quickly.
- On sign-out, **delete** tokens from SecureStore and clear cached user data (query cache, SQLite).
- Never put tokens in URLs (they end up in logs) or in `AsyncStorage`.
- For sign-in with third parties, use the system browser via **OAuth 2.0 + PKCE**
  (`expo-auth-session`/`expo-web-browser`), never a WebView that can read the user's password.

## 3. The network

- **HTTPS only.** iOS App Transport Security and Android's default network security config block
  cleartext HTTP in release builds; don't add blanket exceptions to make a dev server work in
  production.
- **Certificate pinning** — accepting only your server's specific certificate or public key — defends
  against interception by compromised or user-installed certificate authorities. It also means a
  certificate rotation can lock out every installed app until it updates, so it needs an operational
  plan (pin to multiple keys, including a backup). It requires native configuration or a library; for
  most apps, correct TLS plus server-side security is the right first step, and pinning is a deliberate
  decision for high-risk apps (banking, health).
- Assume requests can be **replayed and modified**: validate everything server-side, use idempotency
  keys for payments (like the `PUT /notes/:id` design in L3-10), rate-limit.

## 4. Deep links and notification payloads are input

Any app — or any web page — can open `yourapp://anything`. Treat route params and notification `data`
as **untrusted input**:

```ts title="src/safeNav.ts"
const ALLOWED_PREFIXES = ['/recipe/', '/habit/', '/settings'] as const;

/** Returns an internal path we are willing to navigate to, or null. */
export function safeInternalPath(raw: unknown): string | null {
  if (typeof raw !== 'string') return null;
  if (!raw.startsWith('/') || raw.startsWith('//')) return null;     // no external or protocol-relative URLs
  if (/[\s\\]|%2f|%5c/i.test(raw)) return null;                        // no encoded slashes or whitespace tricks
  return ALLOWED_PREFIXES.some((p) => raw === p || raw.startsWith(p)) ? raw : null;
}
```

This check was unit-tested for this lesson (Jest, passing):

```ts title="src/safeNav.test.ts"
import { safeInternalPath } from './safeNav';

test('allows known internal routes only', () => {
  expect(safeInternalPath('/recipe/52785')).toBe('/recipe/52785');
  expect(safeInternalPath('/settings')).toBe('/settings');
  expect(safeInternalPath('https://evil.example')).toBeNull();
  expect(safeInternalPath('//evil.example/recipe/1')).toBeNull();
  expect(safeInternalPath('/admin/delete-all')).toBeNull();
  expect(safeInternalPath('/recipe/..%2F..%2Fadmin')).toBeNull();
  expect(safeInternalPath(42)).toBeNull();
});
```

And never perform a **state-changing action directly from a link** (`yourapp://delete-account`,
`yourapp://transfer?to=…`) — open a screen that asks the user to confirm.

## 5. WebViews

`react-native-webview` embeds a browser. Risks: loading untrusted pages that can call into your app
via `onMessage`, or injecting JavaScript that exposes data. Rules: load only your own HTTPS origins
(`originWhitelist`), validate every message from the page as untrusted input, don't inject tokens into
pages, and open external links in the system browser instead.

## 6. Logging and analytics hygiene

- Strip debug logging from release builds (`if (__DEV__)`), and never log tokens, passwords, full
  request bodies or personal data — device logs are readable on Android with debugging enabled, and
  crash/analytics tools store what you send them (lesson 9).
- Don't put personal data in screen names, event names or URLs that analytics collect.

## 7. Dependencies and the supply chain

Every npm package runs with your app's privileges. Prefer maintained, widely used libraries; keep the
lockfile committed; run `npm audit` (and read it critically — many findings affect only build tools);
and review what a new native dependency asks for (permissions, network calls) before adding it.

## Should you obfuscate or detect root/jailbreak?

Obfuscation and root/jailbreak detection **raise the cost** of reverse engineering and tampering; they
don't make an app secure, and determined attackers bypass both. They're reasonable defence-in-depth
for high-risk apps, after the server-side fundamentals above are right — not instead of them.

## How It Actually Works

A release build compiles your JavaScript into a single Hermes bytecode file inside the app package
(the APK/AAB's assets, or the iOS app bundle). Bytecode keeps a **string table** of every string
literal the program uses — including inlined environment values — because the program needs them at
runtime. That's why `grep -a` found the key. Hermes bytecode can also be disassembled with publicly
available tools, so even logic is readable with effort.

SecureStore, by contrast, keeps values outside the app package entirely: on iOS in the Keychain, a
system database encrypted with keys tied to the device and (depending on accessibility class) the
user's passcode; on Android, encrypted with a key that lives in the Android Keystore, which on many
devices is backed by secure hardware so the raw key never enters app memory.

## Common mistakes

- **Secret API keys in `EXPO_PUBLIC_` variables** or anywhere in the app.
- **Authorisation enforced only in the UI.**
- **Tokens in AsyncStorage**, URLs or logs.
- **Navigating to whatever a deep link or push payload says.**
- **Clearing UI on sign-out but not caches and stored tokens.**
- **Blanket cleartext-HTTP exceptions** left in production config.
- **Pinning certificates without a rotation plan.**

## Exercise

1. Run `npx expo export --platform android` on your app and `grep -a` the `.hbc` file for your API base
   URL and any `EXPO_PUBLIC_` values. Is anything there that shouldn't be?
2. Move any secret third-party call behind a small server endpoint that authenticates the user.
3. Use `safeInternalPath` in the notification-routing hook from L3-04 and in a deep-link handler, and
   add two more test cases for paths you consider dangerous in your app.
4. Implement sign-out that deletes SecureStore tokens, clears the TanStack Query cache
   (`queryClient.clear()`) and wipes user tables in SQLite. Verify by signing in as a second user.
