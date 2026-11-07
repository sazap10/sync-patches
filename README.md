# Sync target SDK patches

A [Morphe](https://github.com/MorpheApp) patch bundle that raises the target SDK of Sync for Reddit from 33 to 36 and fixes the code that would crash at that target.

It is meant to be applied together with [patcheddit](https://github.com/wchill/patcheddit), which provides the patches that make Sync work again (Spoof client and the Sync fixes). This bundle does not touch any of that.

Sync shut down in 2023. The patches are written against its last release, `v23.06.30-13:39` (versionCode 23033) of `com.laurencedawson.reddit_sync`.

The source only works once the repository is public and has at least one release, because Morphe Manager reads
`patches-bundle.json` from the `main` branch and the release workflow fills it in.

## How to use

1. In Morphe Manager, add a patch source with the repository URL `https://github.com/sazap10/sync-patches`, or open
   <https://morphe.software/add-source?github=sazap10/sync-patches> on the phone.
   Keep patcheddit as a source too.
2. Patch the unpatched Sync APK. Morphe Manager applies the selected patches from every source in a single run, so the
   output contains both patcheddit's fixes and this bundle's "Target SDK 36" patch. The patch is selected by default.
3. Install the result.

### Switching from an installed Sync

Morphe Manager signs the APK it builds. Android installs it over the existing Sync only when the signing certificate
is the same.

- If your current Sync was built by the same Morphe Manager with the same keystore, the new build updates it in place
  and keeps your data.
- If the signature differs (for example the original Play Store build, or a build from another manager or keystore),
  Android refuses the update. You have to uninstall Sync first, which deletes its data. Sync sets
  `android:allowBackup="false"`, so Android's backup does not cover it. Save anything you need (settings, accounts,
  filters) before uninstalling.

## Patches

<!-- PATCHES_START EXPANDED -->

The release workflow replaces this section with the generated patch list.

<!-- PATCHES_END -->

## What "Target SDK 36" changes

Compatible with Sync for Reddit `v23.06.30-13:39`. Sync for Reddit Pro is listed as experimental and untested.

- Sets `targetSdkVersion` to 36 in `AndroidManifest.xml`. `minSdkVersion` stays at 23.
- Rewrites every runtime `registerReceiver` call in the app and its bundled libraries (45 call sites in 23033) to go
  through a small helper that adds `RECEIVER_NOT_EXPORTED` or `RECEIVER_EXPORTED` on Android 13 and later. Without a
  flag, Android 14 throws a `SecurityException` for apps that target 34 or higher. Receivers that get broadcasts from
  other apps are exported: Play Billing purchase updates, Play Core listeners, ML Kit `DOWNLOAD_COMPLETE` and the
  Mobile Ads debug toggle. All others only receive broadcasts from Sync itself or from the system and are not
  exported. Calls that already pass flags keep them. The helper only adds a flag when neither bit is set. The rewrite
  runs after all other patches, so code added by other bundles is covered too.
- Adds `FOREGROUND_SERVICE_DATA_SYNC` and gives every service that can call `startForeground()` a
  `foregroundServiceType`. Sync's own code never starts a foreground service. The two that can are bundled library
  services: WorkManager's `SystemForegroundService` (already `dataSync`) and Play Core's
  `ExtractionForegroundService` (disabled in the manifest, now typed `dataSync`).
- Sets `android:enableOnBackInvokedCallback="false"` on `<application>`. Predictive back is on by default at target 36,
  and while it is on Android does not call `onBackPressed()` or deliver `KEYCODE_BACK`, which Sync relies on for back
  and swipe-back.
- Fixes the sort and "more actions" sheets that Sync anchors under the toolbar. Sync removes their
  `BottomSheetBehavior` and leaves them at the top of the dialog. Enforced edge-to-edge breaks them in two ways:
  - The navigation bar is transparent, so Material 1.9's bottom sheet insets listener now receives insets and calls
    `BottomSheetBehavior.from()` on those sheets, which throws `IllegalArgumentException: The view is not associated
    with BottomSheetBehavior`. The listener now returns the insets unchanged for sheets without a behavior.
  - The dialog content now starts at the top of the screen, so the sheet covered the toolbar. For sheets without a
    behavior, the patch sets the sheet's top margin to the toolbar's bottom edge, measured on screen and converted
    into the dialog's coordinates. This follows the status bar height, display cutouts and orientation.

  Sheets that keep their behavior are not changed and still get Material's edge-to-edge handling.

## What "Log crashes" does

This patch is off by default. Enable it in Morphe Manager when you need to report a crash and have no adb.

It installs an uncaught-exception handler at the start of `RedditApplication.onCreate`. When Sync crashes, the
handler writes a text file and then passes the crash on to the previous handler, so Android still closes the app as
usual. Each crash gets its own file named `sync-crash-<yyyyMMdd-HHmmss>.txt`, containing the time, the thread, Sync's
version, the device model, the Android and target SDK levels, and the full stack trace with every cause. It does not
record accounts, tokens, intents or extras.

On Android 10 and later the files go to the public Download folder through MediaStore, which needs no permission. Open
the Files app and go to Downloads, then SyncCrashes. On Android 9 and earlier they go to
`Android/data/<package>/files/SyncCrashes`.

## Known issues

- **Edge-to-edge.** From target 35 Android draws apps edge to edge, and at target 36 the opt-out attribute is ignored.
  Sync's layouts were not built for that, so some screens may draw under the status bar, navigation bar or display
  cutout. This bundle does not change layouts. It only fixes the bottom sheet crash described above.
- **Barely tested on a device.** A Pixel 8 on Android SDK 37 confirmed the receiver fix and found the bottom sheet
  crash. The rest has not been exercised on a device.
- **Sync for Reddit Pro** was not available while writing these patches. The receiver rewrite matches API calls and
  package names rather than obfuscated class names, so it should apply, but it is untested.
- Morphe Desktop's developer option "verify with SDK" fails on this app with ten cross-DEX errors for OkHttp's
  GraalVM-only `okhttp3/internal/graal/OkHttpFeature` class. The same errors appear with patcheddit alone and with the
  unmodified app's OkHttp, and the check does not exist in Morphe Manager. It is not caused by this bundle.

### Suspected, not confirmed

These Android 14 to 16 changes apply to apps with a high target SDK. A static look at the 23033 APK found no confirmed
crash from them, but they need checking on a device.

- Mutable `PendingIntent`s with implicit intents throw on target 34. Every `PendingIntent` that Sync creates is either
  immutable or uses an explicit intent. Ads and Play services code was not traced in full.
- Implicit intents only reach exported components on target 34. Sync's only non-exported components with intent
  filters are `HomeActivity` (opened through exported launcher aliases) and two `BOOT_COMPLETED` receivers, which the
  system delivers to.
- Dynamically loaded code must be read-only on target 34. The Google Mobile Ads SDK loads dex files with
  `DexClassLoader`. If that fails it would affect ads only, which patcheddit's "Disable ads" removes from the UI.
- OpenJDK 17 changes on target 34: stricter regex group references and stricter `UUID.fromString` parsing. Any of
  Sync's string handling could hit these and it was not audited.
- Non-SDK interface lists marked `max-target-t` block reflection on hidden APIs for apps that target 34 or higher.
  Bundled libraries generally handle the failure.
- Background activity launch rules for `PendingIntent` creators and senders tighten on targets 34 and 35. Notification
  taps are user actions and are not affected.
- Exact alarms: WorkManager calls `AlarmManager.setExact`. Sync already targets 33, so this is not new at 36.
- Android 16 "safer intents" and intent redirection protections.

## Building locally

Requirements: JDK 21, an Android SDK (`ANDROID_HOME`), and a GitHub token that can read GitHub Packages, because the
Morphe Gradle plugin comes from `maven.pkg.github.com/MorpheApp/registry`.

```sh
GITHUB_ACTOR=<user> GITHUB_TOKEN=<token> ./gradlew :patches:buildAndroid
```

The bundle is written to `patches/build/libs/patches-<version>.mpp`. To try it with patcheddit using
[Morphe Desktop](https://github.com/MorpheApp/morphe-desktop):

```sh
java -jar morphe-desktop-<version>-all.jar patch \
  --patches patches/build/libs/patches-<version>.mpp \
  --patches patches-<patcheddit-version>.mpp \
  --keystore <keystore> -o sync-patched.apk <unpatched-sync>.apk
```

Keep APKs and keystores out of the repository. `.gitignore` excludes `*.apk`, `apk/` and common keystore files.

## Releases

`.github/workflows/release.yml` runs semantic-release on every push to `main`. Commits must follow Conventional
Commits: `fix:` makes a patch release and `feat:` a minor release. A release builds the `.mpp`, attaches it to a GitHub
release and commits `patches-bundle.json`, which is the file Morphe Manager reads to find new versions.

Releases are unsigned until you add a signing key. Morphe Manager accepts unsigned bundles. To sign them:

1. Run `./gradlew generateSigningKey`. It writes `morphe-signing.pub` and `morphe-signing.key`.
2. Commit `morphe-signing.pub`. Never commit `morphe-signing.key` (it is in `.gitignore`).
3. Add the content of `morphe-signing.key` as the `MORPHE_SIGNING_KEY` Actions secret, keep a backup, and delete the
   file.

If you later replace the key, Morphe Manager asks users to confirm the new key before it installs the next release.

## License

GPLv3. See [LICENSE](LICENSE) and [NOTICE](NOTICE).
