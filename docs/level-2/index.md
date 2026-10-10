---
description: "React Native Level 2: navigation with Expo Router, deep links, app state with Zustand, TanStack Query, AsyncStorage, SQLite, forms and the keyboard."
---

# Level 2 · Navigation, State & Data

A single screen with in-memory state is a demo. A real app has many screens, remembers things
between launches, talks to a server, works on a train with no signal, and survives the keyboard.
Level 2 turns the habit tracker skills from Level 1 into the architecture of a real, multi-screen,
data-driven app.

You'll learn how mobile navigation differs from web routing (a stack of live screens, not a single
page), use Expo Router's file-based routes for stacks, tabs and modals, pass parameters and open
screens from links, split state into the right buckets (local UI state, shared client state, server
state, persisted state), fetch and cache server data with TanStack Query, store data on the device
with AsyncStorage, SecureStore and SQLite, build validated forms, manage images and fonts, and stop
the keyboard covering your inputs. The project combines everything into an offline-friendly recipe
book.

**Prerequisites:** Level 1, plus comfort with `async`/`await` and `useEffect`. If state management
patterns are new, Level 2 of the [React Mastery Path](https://sigilipelli.github.io/react-mastery-path/)
covers Context and `useReducer` in depth; this level focuses on how they apply on mobile.

## Modules

1. **[Navigation Concepts: Stacks, Tabs & Modals](01-navigation-concepts.md)** — why mobile
   navigation is a stack of mounted screens, and the navigators every app combines.
2. **[File-Based Routing with Expo Router](02-expo-router.md)** — layouts, groups, tabs, modals and
   `Link`, built from your folder structure.
3. **[Route Params, Deep Links & Typed Routes](03-params-deep-links.md)** — dynamic segments,
   search params, URL schemes, universal links and type-safe `href`s.
4. **[App State: Context, useReducer & Zustand](04-app-state.md)** — choosing where state lives and
   avoiding re-render storms.
5. **[Fetching Data with TanStack Query](05-data-fetching.md)** — caching, refetch on focus and
   reconnect, mutations and optimistic updates.
6. **[Local Persistence: AsyncStorage, SecureStore & SQLite](06-local-persistence.md)** — which
   store for which data, and schema migrations.
7. **[Forms & Validation with React Hook Form and Zod](07-forms-validation.md)** — schema-driven
   validation that scales past three fields.
8. **[Images, Fonts & Assets](08-images-fonts-assets.md)** — `expo-image`, resolution variants,
   custom fonts, icons and the splash screen.
9. **[The Keyboard, Modals & Bottom Sheets](09-keyboard-modals.md)** — keeping inputs visible and
   building overlay UI that feels native.
10. **[Project — Offline-Friendly Recipe Book](10-project-recipe-book.md)** — tabs, detail screens,
    search, favourites in SQLite and cached API data.
