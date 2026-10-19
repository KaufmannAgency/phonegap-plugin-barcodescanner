# CLAUDE.md — phonegap-plugin-barcodescanner (Paketti fork)

Fork of [phonegap/phonegap-plugin-barcodescanner](https://github.com/phonegap/phonegap-plugin-barcodescanner)
(upstream archived; last release 8.1.0). Camera barcode/QR scanning for the orderer app.
System overview: `../pake-scripts/CLAUDE.md`.

## Branch and fork diff

- Single branch **`master`**. Forked at `bd82b3f` (2019-06-26, `8.1.1-dev`); no `upstream` remote.
- The whole fork is one commit, `369da18` (2022-10-20): `compile` → `implementation` in
  `src/android/barcodescanner.gradle` so it builds with Gradle 7+. No functional changes.
- Android scanning lives in the prebuilt `src/android/barcodescanner-release-2.1.5.aar` (ZXing-based,
  loaded via `flatDir`) plus `com.android.support:support-v4:$ANDROID_SUPPORT_V4_VERSION` (default `27.+`) —
  legacy support library, so the consuming app needs `cordova-plugin-androidx-adapter`.
- `barcodescanner.gradle` still declares `jcenter()`; it works with Gradle 8.13 but is deprecated and
  disappears in Gradle 9 — the first thing to fix if a newer cordova-android breaks the build.

## Consumer

`pake-front` (orderer app `fi.kaufmann.paketti.tilaaja`, pake module `paketti-front`):
`cordova/package.json` `"phonegap-plugin-barcodescanner": "github:KaufmannAgency/phonegap-plugin-barcodescanner"`
(no `#branch` → `master`; lock pins `369da18`) with `ANDROID_SUPPORT_V4_VERSION 27.+`.
Called through `window.cordova.plugins.barcodeScanner.scan` in `src/forms/BarcodeScanner.ts`
(used by the QR-code reader of the order forms).

## Compatibility

`plugin.xml` `<engines>`: cordova >= 7.1.0, cordova-android >= 6.3.0. In use with cordova-android 14.0.1
(Gradle 8.13, AGP 8.7.3, compileSdk 35). iOS/Windows code is untouched upstream; the app builds Android only.

## Toolchain

This repo is not built on its own — it is compiled inside the consuming app's Cordova Android project.

| tool | version | where pinned / why |
|---|---|---|
| cordova CLI | 13.0.0 | `npm install -g cordova@13.0.0` in the app's `do_init_*_mobile` (`pake-tools.sh`) |
| cordova-android | 14.0.1 (pake-front), 15.1.0 (pake-transporter), 11.0.0 (pake-specialist) | app `cordova/package.json` devDependencies |
| Gradle / AGP | 8.13 / 8.7.3 (cdv-android 14), 8.14.2 / 8.10.1 (cdv-android 15) | bundled by cordova-android; plugins must use `implementation`, not `compile` (Gradle 7+) |
| JDK | 21 (porter-front) / 18 (paketti-front packaging) | `export JAVA_HOME=$(/usr/libexec/java_home -v …)` in app `pake-tools.sh`; JDK 25 is too new for this Gradle |
| Android SDK | compileSdk 35 (front) / 36 (transporter), minSdk 24 | app `config.xml` + cordova-android defaults; `ANDROID_HOME` in `~/.zshenv` |
| node | nvm default (25.8.2) or 24.13.0 | no `.nvmrc`; `nvm use` lines in app `pake-tools.sh` are commented out |

## How a consuming app picks up a change

Consumers reference this repo as an npm git dependency in their `cordova/package.json`; their
`cordova/package-lock.json` **pins the exact commit SHA**, and `cordova/plugins/` + `cordova/platforms/` are
git-ignored copies. So pushing to the branch alone changes nothing in an app build.

1. Commit and push here, on the branch the consumer references (see table above).
2. In the consumer's `cordova/` dir (node from nvm — cordova CLI 13.0.0 is installed into node 24.13.0 / 25.8.2):
   `npm install <spec-from-package.json>` (e.g. `npm install github:KaufmannAgency/<repo>#<branch>`) to move
   the lock to the new commit, then `cordova plugin rm <plugin-id> && cordova plugin add <plugin-id>` to
   refresh `cordova/plugins/` and `platforms/android`. `pake init <module> --mobile --clean` also works but
   wipes `platforms/` (re-prompts for `pakeapp.keystore`).
   - Quick iteration without pushing: `cordova plugin add /Users/joni/Github/<repo>` (local path) —
     this rewrites the spec in `package.json`/`config.xml`, so restore the github spec before committing the app.
3. Build + install a debug APK on a USB device: `pake build,package,run <module> --mobile --named-env remote`
   (`-u <user>` sets `TEST_USER`, which **bypasses Imprivata**), then `pake log <module> --mobile` (adb logcat).
4. Commit the consumer's updated `cordova/package-lock.json` (and `package.json` if the spec changed).

`<module>` is `paketti-front` (repo `pake-front`) or `porter-front` (repo `pake-transporter`); the
`do_*_mobile` functions live in each app repo's `pake-tools.sh`. There are no tests in this repo.

Here: `npm install github:KaufmannAgency/phonegap-plugin-barcodescanner` in `pake-front/cordova`,
plugin id `phonegap-plugin-barcodescanner`, module `paketti-front`.
