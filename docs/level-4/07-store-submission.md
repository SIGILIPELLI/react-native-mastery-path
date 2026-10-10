---
description: "Get through App Store and Google Play review: listings, privacy labels and data safety, common rejections, TestFlight and staged rollouts."
---

# 07 · Shipping to the App Store & Google Play

Building a signed binary (lesson 6) gets you a file. Getting it onto strangers' phones means passing
two review processes with different rules, filling in store listings and privacy declarations, and
rolling out carefully enough that a bad release doesn't reach everyone at once. None of this is
React Native–specific — which is exactly why it catches React Native developers off guard.

!!! note "Accounts and policies"
    You need an **Apple Developer Program** membership (paid annually) and a **Google Play Console**
    developer account (one-time registration fee). Store policies change regularly; this lesson
    describes the durable process and points at the policies to read — check Apple's App Review
    Guidelines and Google Play's Developer Program Policies for current rules before submitting.
    No submission was performed for this lesson.

## Before you submit: the checklist

- Release build tested on real iOS and Android devices (not just simulators), including a small
      phone and a mid-range Android device.
- Every permission string explains the user benefit (L3-03) and every requested permission is
      actually used.
- Sign-in works for reviewers: provide a **demo account** in review notes if your app needs login.
- If users can create accounts in the app, they can also **delete their account** in the app
      (Apple requires this; Google requires an account-deletion path too).
- A reachable **privacy policy URL**.
- No placeholder content, test screens or "coming soon" features in the build.
- Crash reporting configured (lesson 9) so you see problems from the first real user.
- Icons, splash and screenshots for every required device size.

## Apple: App Store Connect

1. **Create the app record** in App Store Connect with the same bundle identifier as your build.
2. **Upload the build.** With EAS: `eas submit --platform ios --latest` (it can create an App Store
   Connect API key for you), or upload the `.ipa` with Apple's Transporter app.
3. **TestFlight.** Uploaded builds appear in TestFlight after processing. Internal testers (your team)
   can install immediately; **external testers** require a lightweight beta review first. Use this
   stage — it's the closest thing to production.
4. **App Privacy ("nutrition label").** Declare which data you collect, whether it's linked to the
   user's identity, and whether it's used for tracking. This includes data collected by SDKs you embed
   (analytics, crash reporting, ads). If you track users across other companies' apps, you must use
   the **App Tracking Transparency** prompt.
5. **Privacy manifests.** Apple requires apps and certain SDKs to declare in a privacy manifest the
   reasons they use some APIs ("required reason APIs" such as reading file timestamps or user
   defaults). Expo and many libraries ship their own manifests; you can add entries via
   `ios.privacyManifests` in your app config. Missing declarations show up as warnings or rejections
   after upload.
6. **Listing:** name, subtitle, description, keywords, screenshots per device size, support URL,
   age rating questionnaire, category.
7. **Submit for review.** Choose manual or automatic release after approval, or a **phased release**
   that rolls the update out to automatic-update users gradually over several days.

Frequent rejection reasons worth knowing (see the App Review Guidelines for the full list):

- **Crashes or broken features** on the reviewer's device — often an untested iPad or a missing
  backend.
- **Incomplete information**: no demo account, features hidden behind sign-in.
- **Minimum functionality**: an app that's just a website in a WebView.
- **Payments**: selling digital goods or features consumed in the app generally requires Apple's
  in-app purchase system (rules vary by region and change; read the current guideline 3.1).
- **Vague permission strings** or requesting permissions without a feature that uses them.

## Google: Play Console

1. **Create the app** in Play Console and complete the setup tasks it lists.
2. **Upload an `.aab`** to a testing track first. With EAS: `eas submit --platform android --latest`
   (requires a Google service account key with Play Console access; the very first upload of a new app
   may need to be done manually in the console).
3. **Testing tracks:** **internal** (up to a small group, available within minutes), **closed** (invited
   testers), **open** (anyone can join). New personal developer accounts may be required to run a
   closed test with a minimum number of testers for a period before production access — check the
   current requirement in Play Console.
4. **Data safety form:** declare data collected and shared, purposes, encryption in transit, and
   whether users can request deletion. Like Apple's label, it must include SDK behaviour.
5. **Content rating questionnaire, target audience, ads declaration, app access** (demo credentials).
6. **Target API level:** Google Play requires new apps and updates to target a recent Android API
   level. Expo SDK upgrades raise the target for you; an app stuck on an old SDK eventually can't ship
   updates.
7. **Production release with staged rollout:** release to a percentage of users (for example 5%), watch
   crash and ANR rates in Play Console's Android vitals, then increase. You can **halt** a staged
   rollout if something goes wrong.

## Versioning your releases

Keep a simple rule: `version` (user-facing) follows semantic meaning — `1.4.0` for features, `1.4.1`
for fixes — and EAS auto-increments the build numbers (lesson 6). Tag the git commit of every store
release (`git tag v1.4.0`), so you can always rebuild or hotfix exactly what users have.

## Worked example: a release runbook

```text title="RELEASE.md"
1. Freeze: merge only fixes to the release branch.
2. Bump expo.version (e.g. 1.4.0). Commit, tag v1.4.0.
3. eas build --profile production --platform all
4. eas submit --platform ios --latest      → TestFlight internal
   eas submit --platform android --latest  → Play internal testing
5. Smoke test on: small iPhone, large iPhone, mid-range Android, tablet (if supported).
   Script: sign in (demo account) → add habit → toggle → restart → still there → deep link → push.
6. Promote: TestFlight external / Play closed track for 2–3 days. Watch crash reports.
7. Submit for App Review (iOS, phased release) and promote to Production at 10% (Android).
8. Watch crash-free sessions and ANRs. Increase rollout 10% → 50% → 100% if healthy.
9. If a JS-only bug appears: OTA fix to the production channel (lesson 8).
   If a native bug appears: halt rollout (Android) / pause phased release (iOS), hotfix 1.4.1.
```

Writing this down once turns releases from an event into a routine.

## How It Actually Works

When you upload an `.ipa`, App Store Connect **processes** it: it validates the signature and
entitlements against your provisioning profile, checks the privacy manifest and required icons, and
generates device-specific variants (app thinning) so each device downloads only the resources it
needs. Review is partly automated and partly human: a reviewer installs the build and exercises it.

For Android, the `.aab` you upload is not what users install. Google Play generates optimised
**split APKs** from the bundle — per CPU architecture, screen density and language — and signs them with
the app signing key it holds (Play App Signing, lesson 6). That's why the bundle format is required and
why your upload key isn't the key users' devices verify. Staged rollouts are a server-side setting:
Play decides per user whether they're offered the new version, which is why you can halt it instantly
without uploading anything.

## Common mistakes

- **No demo account** for an app that requires sign-in.
- **No in-app account deletion** when sign-up exists.
- **Privacy declarations that ignore SDKs** (analytics, crash reporting).
- **Going straight to 100% production** without a staged or phased rollout.
- **Treating TestFlight/internal testing as optional.**
- **Not tagging release commits.**
- **Letting the Android target API level fall behind**, blocking updates.

## Exercise

1. Write your app's `RELEASE.md` runbook, adapted from the worked example.
2. Draft the App Privacy and Data Safety answers for the Field Notes app (L3-10): it collects precise
   location, photos and notes, and syncs them to your server. Which data is "linked to the user"? Is any
   of it used for tracking?
3. Write the review notes you'd give Apple for the Field Notes app, including how a reviewer can test
   offline sync.
4. If you have accounts: upload a build to TestFlight internal testing and Play internal testing, and
   install it from both.
