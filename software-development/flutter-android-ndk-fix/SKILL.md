---
name: flutter-android-ndk-fix
description: Fixes common Flutter Android build failures caused by NDK inconsistencies or unsupported Gradle project layout. Encodes the trial-and-error steps used to diagnose and resolve NDK/source.properties and Gradle/NDK version mismatches.
version: 1.1.0
author: Hermes Agent
license: MIT
metadata:
  hermes:
    tags: [flutter, android, ndk, gradle, build-fix, troubleshooting]
---

# Flutter Android NDK & Gradle build fix

## When to use

- `flutter build apk` or `flutter build appbundle` fails with messages like:
  - "Your app is using an unsupported Gradle project. To fix this problem, create a new project by running `flutter create -t app <app-directory>`..."
  - "NDK at /.../ndk/<ver> did not have a source.properties file"
  - Errors from Gradle C/C++ toolchain related to NDK discovery

Use this skill when diagnosing Android build failures that mention NDK versions, ndkVersion lookup, or unsupported Gradle project layouts.

## Principles

- Prefer using a known-good NDK that is actually installed (source.properties must exist).
- Pin Flutter version with FVM if required for reproducible builds.
- If Flutter reports "unsupported Gradle project" for a minimal project, re-run `flutter create .` to generate missing Android build files and move your Dart code/pubspec.yaml in.
- Avoid leaving projects in a state where Gradle references an NDK folder that is broken or partially downloaded.

## Steps (ordered)

1. Reproduce the failure and capture the build log (save for analysis):

   cd <project>
   fvm flutter build apk -v 2>&1 | tee build_log.txt

   Inspect build_log.txt for the first-occurring error mentioning `ndk` or `source.properties`.

2. List installed NDKs on the machine and verify `source.properties` exists:

   ls -la $ANDROID_SDK_ROOT/ndk
   # or macOS default
   ls -la ~/Library/Android/sdk/ndk
   
   For each NDK folder, check:
   cat ~/Library/Android/sdk/ndk/<version>/source.properties

   A valid NDK has a non-empty source.properties file. If a folder is present but source.properties is missing or empty, it's a broken/partial install.

3. Fix options (choose one):

   A) Point Gradle to a working NDK by hardcoding ndkVersion in android/app/build.gradle(.kts):

   // Kotlin DSL (build.gradle.kts)
   // before
   ndkVersion = flutter.ndkVersion

   // after (use a version that exists on your machine)
   // Use specific NDK version available on system to avoid lookup issues
   ndkVersion = "27.0.12077973"

   B) Or set ndk.dir in android/local.properties to the full path of an installed NDK:

   ndk.dir=/Users/<you>/Library/Android/sdk/ndk/27.0.12077973

   C) Reinstall the NDK with sdkmanager or Android Studio SDK Manager:

   sdkmanager --install "ndk;27.0.12077973"

   Prefer method A or C for CI-friendly reproducibility. Setting ndkVersion to a non-installed version will fail on other machines/CI unless you also install that NDK there.

4. If Flutter warns about an unsupported Gradle project layout, regenerate Android boilerplate (safe if you have no native customizations or you backed them up):

   # creates missing android/gradle files, etc.
   fvm flutter create .

   Then re-run `fvm flutter pub get` and try the build again.

5. Use FVM to pin Flutter and run the full sequence (example):

   fvm use 3.32.6
   fvm flutter pub get
   fvm flutter analyze
   fvm flutter build apk --release -v 2>&1 | tee build_log.txt

6. If build still fails, inspect build_log.txt for the earliest cause and iterate. Common follow-ups:
   - Missing Android SDK build-tools → install via sdkmanager
   - Gradle plugin / Kotlin DSL mismatches → prefer Flutter-generated android files or update gradle/wrapper
   - Partial SDK downloads → delete broken NDK folder and reinstall

## Example session (what we did and found)

- Observed error in build log: "NDK at /Users/.../ndk/26.3.11579264 did not have a source.properties file" → indicates broken NDK install.
- Listed installed NDKs and found 27.0.12077973 exists and has source.properties.
- Edited android/app/build.gradle.kts to pin ndkVersion = "27.0.12077973".
- Edited android/local.properties to include ndk.dir pointing to installed NDK (optional but helpful):

  sdk.dir=/Users/<you>/Library/Android/sdk
  ndk.dir=/Users/<you>/Library/Android/sdk/ndk/27.0.12077973
  flutter.sdk=/Users/<you>/fvm/versions/3.32.6

- Ran: fvm flutter build apk --release -v and the build completed successfully, producing build/app/outputs/flutter-apk/app-release.apk.

- If Flutter historically reports "unsupported Gradle project", we ran `fvm flutter create .` which regenerated required android/ gradle files before rebuilding.

## Device / deploy notes (optional)

- To run the app on a connected Android device, ensure USB debugging is enabled and the device is trusted.

- To run the app on a connected iPhone (common pitfalls addressed here):

  1) Ensure the device is unlocked, Trust the computer when prompted, and enable Developer Mode on the iPhone (Settings → Privacy & Security → Developer Mode). Restart the device if required.

  2) On the Mac, open Xcode once and go to Window → Devices and Simulators. If Xcode prompts to "Use for Development" for that iPhone, follow the prompts. This often resolves CLi detection issues.

  3) Install helpers (optional but useful) via Homebrew:
     - brew install libimobiledevice usbmuxd ideviceinstaller
     Note: ifuse may require Linux; install only supported tools on macOS.

  4) Detect device id (CLI):

     # Flutter's machine-readable device listing (preferred)
     fvm flutter devices --machine

     This prints JSON — look for the object with "name": "ddthanh" (or your device name) and copy the "id" field (UDID), e.g. 00008120-000225003C58201E.

     Alternatively, if libimobiledevice is installed:
     idevice_id -l

  5) Run app on device (use FVM if the project uses it):

     fvm flutter run -d <device-id>

     If multiple devices are attached and interactive selection appears, choose the number corresponding to your iPhone.

  6) If the CLI run fails with generic errors but the device is visible in Xcode, open ios/Runner.xcworkspace in Xcode and run the app there — Xcode gives more actionable errors about signing, provisioning profiles, and developer mode.

  7) Ensure iOS usage descriptions are present in ios/Runner/Info.plist for camera and photo usage:

     <key>NSCameraUsageDescription</key>
     <string>App needs camera access to take photos.</string>
     <key>NSPhotoLibraryAddUsageDescription</key>
     <string>App needs to save photos to your library.</string>

  8) If flutter reports device but `fvm flutter run` still exits early with a generic error, run with --verbose and capture the log (useful for diagnosing provisioning, ios-deploy, or usbmuxd issues):

     fvm flutter run -d <device-id> --verbose 2>&1 | tee ios_run_log.txt

  9) For reproducible automation/CI, prefer Xcode-based provisioning or use `flutter build ipa` and distribute the IPA via TestFlight / Apple Configurator if local USB deploy continues to be flaky.

Notes and pitfalls:
- macOS permissions and Xcode trusting sometimes block CLI tools; opening Xcode and running the app once often resolves it.
- Do not rely solely on idevice_id on macOS if it's not installed; Flutter's `flutter devices --machine` is a reliable JSON source to programmatically extract device IDs.
- Keep documentation of required tooling (FVM version, NDK version) in repo README for new contributors and CI setup.



- To run the app on a connected iPhone, ensure Developer Mode is enabled on the device and the device trusts the Mac. Use libimobiledevice/idevice_id to help detect USB devices (brew install libimobiledevice ideviceinstaller). Then run:

  fvm flutter devices -v
  fvm flutter run -d <device-id>

- For iOS, remember to add usage keys to ios/Runner/Info.plist (NSCameraUsageDescription, NSPhotoLibraryAddUsageDescription) when the app uses camera/photo features.

## Verification

- Successful build should produce an APK at:
  build/app/outputs/flutter-apk/app-release.apk

- `fvm flutter analyze` should report no issues (or only unrelated warnings).

## Pitfalls & Notes

- Don't hardcode ndkVersion in shared repos without ensuring CI and other devs install the same NDK; instead document required NDK and provide CI script to install it.
- Deleting android/ and regenerating with `flutter create .` can lose native edits — always backup native changes before regenerating.
- A missing `source.properties` usually means the SDK/NDK download was interrupted; reinstall using sdkmanager/Android Studio for a clean install.

## When to update this skill

- If you discover a different root cause pattern for Flutter Android build failures, or if Flutter/Gradle tooling changes how ndkVersion is resolved, update the troubleshooting steps.
