---
name: example-and-release
description: How to finish a CleverTap Expo plugin release sync — update the CTExample app (app.json + package.json), append a row to the README compatibility matrix, write the CHANGELOG entry in the exact existing format, choose the plugin's semver bump, and verify with the compile-only build loop (yalc + expo prebuild + gradle/xcodebuild). Use at the end of a release sync.
---

# Example app + release finishing steps

The tail of a release sync: keep the example app current, document the change, bump
the plugin version, and verify it compiles. Match the EXISTING formats exactly —
downstream tooling and humans expect them.

## 1. CTExample app

`CTExample/` is the example Expo app; it consumes the plugin via **yalc**
(`"@clevertap/clevertap-expo-plugin": "file:.yalc/@clevertap/clevertap-expo-plugin"`
plus `"expo": { "autolinking": { "nativeModulesDir": ".." } }`).

- `CTExample/package.json` — bump `clevertap-react-native` to match the target
  `${RN_VERSION}`, and `expo` / `react-native` to match the target Expo SDK (use the
  react-native version that ships with that Expo SDK, from `expo-sdk-analysis`).
- `CTExample/app.json` — if the sync added a new feature flag / config field, reflect
  it in the plugin config block so the demo keeps exercising it. Don't change the test
  credentials.

## 2. README compatibility matrix

`README.md` has a table:

```
| CleverTap Expo Plugin version | Expo SDK version | React Native version | CleverTap React Native SDK version |
```

Append ONE new row for this release: `| <new plugin version> | <Expo SDK> | <react-native> | ${RN_VERSION} |`.
Also update any "Expo NN+ Migration" notes if the Expo jump introduced a new
`app.json` migration (e.g. the SDK-55 `notification.icon` → `android.notificationIcon`).

## 3. CHANGELOG.md

New entry at the **TOP**, matching the existing format exactly:

```markdown
### [Version X.Y.Z](https://github.com/CleverTap/clevertap-expo-plugin/releases/tag/X.Y.Z) (Month DD, YYYY)

#### Added
- Adds support for Expo SDK [<v>](https://expo.dev/changelog/sdk-<v>) and React Native [<rn>](...)
- Adds support for CleverTap React Native SDK [${RN_VERSION}](...)

#### Android Platform ####
  - <integration-step change, if any>

#### iOS Platform ####
  - <integration-step change, if any>
```

Use `${RELEASE_DATE}` for the date. Derive version-anchor links from the changelog
dates in `expo-diff.json` (no need to fetch). Only include `#### Android/iOS Platform`
subsections if there were integration-step changes on that platform.

## 4. Plugin semver bump (`package.json` `version`)

This is the single canonical version file the PR completeness check looks for. Choose:

- **patch** (1.0.x) — dependency version pins only; no new feature/integration step;
  no breaking Expo/RN change.
- **minor** (1.x.0) — a new feature flag, a new `compileOnly` dep, or a new additive
  integration step; backward compatible. (Most common.)
- **major** (x.0.0) — a removed feature/step, an Android `minSdk` or iOS
  deployment-target bump propagated to host apps, an `app.json` field removal that
  breaks existing configs, or a breaking `@expo/config-plugins` adoption.

The plugin's own version is independent of the native SDK versions it pins.

## 5. Verify — compile-only build loop

CI does this for you (the `build/expo` composite). Locally, to verify the plugin
compiles after edits:

```bash
npm run build           # compile the plugin (output to build/)
yalc publish            # publish to the local yalc store
cd CTExample
yalc add @clevertap/clevertap-expo-plugin
npm install
npx expo prebuild --clean       # regenerate native projects from app.json + plugin
cd android && ./gradlew :app:assembleDebug   # Android compile
cd ../ios && pod install && xcodebuild -workspace CTExample.xcworkspace -scheme CTExample -sdk iphonesimulator CODE_SIGNING_ALLOWED=NO build   # iOS compile
```

CI is **compile-only** (no emulator/simulator boot) to fit the 90-min job timeout —
it proves the plugin's generated native config compiles. For device-level behavior
testing (SDK init, event recording) use the `ctexample-testing` skill on an emulator
locally; that is NOT part of the automated sync gate.

After an iOS-affecting change, delete `CTExample/ios/Podfile.lock` before `pod install`
so the (transitively-floating) `CleverTap-iOS-SDK` re-resolves.

## Completion checklist

- [ ] `CTExample/package.json` (+ `app.json` if a field was added) updated.
- [ ] README compatibility matrix has a new row.
- [ ] `CHANGELOG.md` has a new dated entry at the top, correct format.
- [ ] `package.json` `version` bumped per semver.
- [ ] The plugin compiles (the CI build gate, or the loop above).
