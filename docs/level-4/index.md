---
description: "React Native Level 4: profiling, FlashList, Jest and RNTL, Maestro and Detox, security, EAS Build and signing, store release, OTA and monitoring."
---

# Level 4 · Production & Shipping

An app on your phone isn't a product until strangers can install it from a store, it doesn't stutter
on a three-year-old Android phone, a bug fix can reach users without a week-long review, and you hear
about crashes before your reviews do. Level 4 is that last mile.

You'll learn to find and fix performance problems with real profilers rather than guesses, make long
lists fast, write component tests with React Native Testing Library and end-to-end tests with Maestro
and Detox, avoid the security mistakes specific to mobile apps, build and sign release binaries with
EAS, get through App Store and Google Play review, ship JavaScript fixes over the air safely, and wire
up crash reporting and analytics. The capstone takes one app through all of it to a release candidate.

**Prerequisites:** Levels 1–3. Store submission needs an **Apple Developer Program** membership (paid,
yearly) and a **Google Play Console** account (one-time fee), and EAS Build needs an Expo account. This
course can't create those for you or run builds against them, so those lessons describe each step
precisely and say where you need your own account. Everything that can run locally — profiling,
tests, builds of JavaScript bundles — was run while writing.

## Modules

1. **[Performance: Finding & Fixing Jank](01-performance-profiling.md)** — measure first: the
   performance monitor, React profiler, Hermes sampling profiles, and the usual culprits.
2. **[Big Lists: Tuning FlatList & Using FlashList](02-list-performance.md)** — the props that matter
   and when to switch to a recycling list.
3. **[Unit & Component Tests with Jest and RNTL](03-testing-rntl.md)** — jest-expo, testing by role
   and label, user events, mocking native modules.
4. **[End-to-End Testing with Maestro & Detox](04-e2e-testing.md)** — testing the real app on a
   simulator, and which tool fits which team.
5. **[Mobile Security: Secrets, Storage & the Network](05-security.md)** — what an attacker can read
   from your binary, token storage, TLS, deep-link and WebView risks.
6. **[Building with EAS & App Signing](06-eas-build-signing.md)** — build profiles, credentials,
   versioning and environment variables.
7. **[Shipping to the App Store & Google Play](07-store-submission.md)** — listings, privacy
   declarations, review guidelines, TestFlight and staged rollouts.
8. **[Over-the-Air Updates & Release Strategy](08-ota-updates.md)** — expo-updates, runtime versions,
   channels and safe rollouts.
9. **[Crash Reporting, Analytics & Monitoring](09-monitoring.md)** — error boundaries, Sentry, source
   maps and privacy-respecting analytics.
10. **[Capstone — Ship a Production-Ready App](10-capstone.md)** — take one app through the full
    release checklist.
