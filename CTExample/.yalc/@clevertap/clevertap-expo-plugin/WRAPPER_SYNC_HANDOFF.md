# Wrapper-Sync Automation — CleverTap Expo Plugin (Handoff)

> Status: initial build (2026-06-18). Owner: @piyush-kukadiya. This doc is the
> "where is the Expo wrapper-sync right now" reference for a fresh Claude session
> or a new developer.

## What this is

A "robot" that keeps `@clevertap/clevertap-expo-plugin` up to date when a new
`clevertap-react-native` release and/or a new Expo SDK version ships. A maintainer
clicks **Actions → native-release-sync → Run workflow**, types the target
`clevertap_rn_version` and `expo_sdk_version`, and a headless Claude run (in the
central tooling repo) bumps versions, propagates native-integration-step changes,
updates the example app + compatibility matrix + CHANGELOG, compiles CTExample,
and opens a PR for review.

## Why Expo is different from the other wrappers

The React Native / Flutter / Cordova wrappers are native bridges — their sync
surfaces new SDK *methods*. **The Expo plugin is a config plugin: it surfaces NO
methods.** It pins dependency versions and generates native setup (manifest,
gradle, Podfile, iOS extensions) during `expo prebuild`. So the Expo sync is
**version + build-config + native-integration-step propagation**, driven by the
clevertap-react-native + Expo SDK versions (the required native Android/iOS SDK
versions are auto-resolved from the clevertap-react-native release). iOS native
SDK versions are intentionally UNPINNED here — they float transitively from
clevertap-react-native's podspec.

## The two repos

1. **This repo (`clevertap-expo-plugin`)** holds the thin dispatch workflow
   (`.github/workflows/native-release-sync.yml`), `CODEOWNERS`, the rewritten
   domain skills under `.claude/skills/`, and the plugin + CTExample that the
   robot edits.
2. **`CleverTap/clevertap-wrapper-tooling`** (the hub) holds all the reusable CI
   machinery: the reusable `sync.yml` conductor, composite actions, the Expo
   orchestrator prompt (`prompts/sync-orchestrator-expo.md`), the fact-finder
   (`tools/expo_diff.py`), and the build composite (`.github/actions/build/expo`).

`uses:` pins `@v1` (a moving tag). Note `uses:` does NOT follow org redirects.

## The Expo-specific pieces in the hub

- `tools/expo_diff.py` — deterministic fact-finder. Resolves the RN→native chain,
  diffs the Android version catalog (tomllib) + dependency blocks, diffs the iOS
  podspec, and **discovers** the plugin's pins by parsing `constants.ts` +
  `androidAppDepsTemplate.ts` and matching to the catalog **by Maven coordinate**
  (so `play-services-ads-identifier` is correctly NOT confused with the catalog's
  `play-services-ads`). Fails loud on uncertainty; never invents a version.
- `prompts/sync-orchestrator-expo.md` — single combined sync (both platforms in
  one pass). Auto-applies only HIGH-confidence coordinate matches; flags the rest.
- `.github/actions/build/expo/action.yml` — yalc-links the plugin into CTExample,
  runs `expo prebuild`, COMPILE-ONLY builds Android (`gradlew assembleDebug`) +
  iOS (`xcodebuild build`). No emulator/simulator boot (fits the 90-min timeout).
- `prompts/pr-description-expo.md` — renders the Expo PR body.

## Files the robot edits each run

`src/android_config/utility/constants.ts` (version pins),
`src/android_config/utility/androidAppDepsTemplate.ts` (+ `types/androidTypes.ts`,
for new compileOnly deps), `src/android_config/gradle/withCleverTapAndroidAppRootBuildGradle.ts`
(classpaths), `android/build.gradle` (push-templates pin),
`src/iOS_config/IOSConstants.ts` + `ios/ExpoAdapterCleverTap.podspec` (iOS
deployment target), `CTExample/{app.json,package.json}`, `README.md` (compat
matrix), `CHANGELOG.md`, `package.json` (plugin version).

## Required repo secrets

`CLEVERTAP_WRAPPER_SYNC_APP_ID`, `CLEVERTAP_WRAPPER_SYNC_PRIVATE_KEY`,
`ANTHROPIC_API_KEY`, `SLACK_WEBHOOK_URL` (same `clevertap-wrapper-sync` App as the
other wrappers).

## Testing (fork-based — see the hub's TESTING.md)

1. Fork this repo. Create a **button branch** (e.g. `task/setup-sync-automation`)
   holding the dispatch workflow; set it as the fork's default branch so the Run
   button appears. Ensure the fork has a **`develop`** branch (the baseline the
   robot edits; roll its versions back one step for a real test).
2. Install the App on the fork + add the 4 secrets.
3. First run with `skip_sync=true` ($0 — validates setup + build/expo).
4. Then a real run against `develop`; the robot auto-creates
   `task/release_<name>` from `develop` and opens a PR back against `develop`.
5. Iterate the hub via the moving `@v1` tag (`git tag -f v1 && git push -f origin v1`).

## Known traps (baked into the implementation)

- The PR-open condition in `sync.yml` had to include `sync_expo.outcome` — Expo's
  `sync_android`/`sync_ios` steps are skipped, so without it the PR would never open.
- `expo_diff.py` resolves CleverTap's combined git tags (e.g. `corev7.6.0_ptv2.2.0`)
  via the GitHub tags API; CI passes the App token (`GITHUB_TOKEN`) to avoid the
  anonymous rate limit. The clevertap-react-native / android-sdk default branch is
  `master`, not `main`.
- Versions come ONLY from `expo_diff.json`; WebFetch (Expo-only) is for reading
  integration-step docs, never for version numbers.
