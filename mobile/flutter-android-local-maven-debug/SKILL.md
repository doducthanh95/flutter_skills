---
name: flutter-android-local-maven-debug
description: Debug and ship Flutter apps that include native Android plugins backed by local Maven repos, FVM-managed Flutter SDKs, and heavy transitive AAR dependencies. Use when builds fail due to Java/Gradle mismatches, missing local Maven resolution, AndroidX/Jetifier, duplicate classes, or emulator install/storage errors.
version: 1.0.0
tags: [flutter, android, gradle, maven, fvm, debug, dependency-resolution]
---

# Flutter Android Local Maven Debug

## When to use
Use this skill when a Flutter app or plugin:
- depends on a native Android plugin with AARs stored in a local Maven repo
- is built with FVM and the Flutter SDK path matters
- fails on Gradle/Kotlin/Java compatibility
- fails because AndroidX/Jetifier is not enabled
- fails due to duplicate classes from transitive Android AARs
- builds successfully but fails to install on emulator because of storage limits

## Core workflow

### 1) Identify the real repo and SDK path
- Confirm the actual project root first.
- Confirm the Flutter SDK used by the repo (often FVM):
  - `fvm flutter --version`
  - `fvm flutter doctor -v`
- Check `local.properties` in both the app and plugin modules.
- Never assume `/opt/homebrew/bin/flutter` or system Flutter if FVM is present.

### 2) Reproduce from the app root, not from the plugin root
- For Flutter app runtime issues, run from the app root:
  - `fvm flutter run -d <device>`
- For Android module-only issues, run from the module directory when needed:
  - `./gradlew assembleDebug`
- If the Android module is a root project, use `assembleDebug`, not `:module:assembleDebug`.

### 3) Resolve local Maven repo wiring
If a plugin ships AARs in `android/local-maven-repo`:
- Add the repo in the plugin Android build file:
  - `maven { url = uri("$rootDir/local-maven-repo") }`
- Add the same repo in the app-level `android/build.gradle` if the app consumes the plugin transitively.
- Prefer Maven coordinates from the generated POM over `flatDir` + raw AAR when possible.
- If a dependency is published as a Maven artifact, depend on the coordinate, not `implementation(name: ...)`.
### 4) Fix environment mismatches first

Common blockers:
- `Unsupported class file major version 67`
  - usually Java 23+ with a Gradle/Kotlin stack that expects Java 17/19
  - switch to a compatible JDK, commonly JDK 17 or 19
- `android.useAndroidX` error
  - add to `gradle.properties`:
    - `android.useAndroidX=true`
    - `android.enableJetifier=true`
- Jetify / Java heap space ("Java heap space" during JetifyTransform)
  - Jetify can run out of heap when converting large engine JARs or many AARs. Increase Gradle JVM args to give more memory in `android/gradle.properties`:

    org.gradle.jvmargs=-Xmx6G -XX:MaxMetaspaceSize=512m -XX:+HeapDumpOnOutOfMemoryError

  - This has resolved "Java heap space" errors during `:compileKotlin` / JetifyTransform in real-world cases. If you still get OOM, try increasing Xmx further or running the Gradle task with `--no-daemon`.
- Flutter embed jar path missing
  - verify `flutter.sdk` in `local.properties`
  - ensure the engine jar exists under the SDK path referenced by the repo

### 5) Triage duplicate-class failures systematically
When Gradle reports duplicate classes:
1. Read the exact duplicated package/class names.
2. Run dependency insight on the app module:
   - `./gradlew :app:dependencyInsight --dependency <artifact> --configuration debugRuntimeClasspath`
3. Find which direct dependency and which transitive dependency bring in the same library.
4. Remove the direct dependency if it is already transitively included.
5. If needed, exclude the conflicting transitive module from one side.

Typical pattern:
- Plugin A brings `com.vpbank:liveness`
- Plugin B brings `com.fis.ekyc:liveness_corp`
- Both contain the same classes
- Keep only one source of truth; exclude the other side in the plugin or app dependency graph

### 6) Reduce APK size for emulator installs
If install fails with:
- `INSTALL_FAILED_INSUFFICIENT_STORAGE`
- `Requested internal only, but not enough space`

Do this:
- Check APK size:
  - `ls -lh build/app/outputs/flutter-apk/app-debug.apk`
- Inspect the largest embedded assets/libraries with a zip listing.
- Temporarily narrow ABIs to `arm64-v8a` if you are using an arm64 emulator.
- Disable ABI/language/density splits if they make the build or install path more complex.
- Prefer `flutter build apk --release` or `fvm flutter build apk --release` for a final install when debug APK is too large.
- If install still fails, use `adb install -r -g <apk>` after freeing emulator storage or using a clean emulator image.

### 7) Prefer release build verification when debug is blocked by device limits
If debug run fails only at install time:
- verify with release build first:
  - `fvm flutter build apk --release`
  - `adb install -r -g build/app/outputs/flutter-apk/app-release.apk`
- This confirms the dependency graph is valid even if the emulator cannot accept the debug APK.

## Recommended command sequence
1. `fvm flutter --version`
2. `fvm flutter devices`
3. `fvm flutter run -d <device>`
4. If Gradle fails, run `./gradlew assembleDebug --stacktrace`
5. If dependency conflicts appear, run `./gradlew :app:dependencyInsight --dependency <name> --configuration debugRuntimeClasspath`
6. If install fails due to storage, run `fvm flutter build apk --release`
7. Install with `adb install -r -g <apk>`

## Pitfalls
- Using the wrong Flutter SDK path in `local.properties`
- Using Java 23 with older Gradle/Kotlin stacks
- Adding the same native SDK both directly in the app and transitively from a plugin
- Keeping both `flatDir` and Maven metadata when the repo already has a proper POM
- Forgetting to update app-level repositories for plugin-provided local Maven artifacts
- Debug APKs becoming huge because native libraries and ML assets are bundled for every ABI
- Emulator storage exhaustion masked as generic install failure

## Verification
A successful end state usually means:
- `./gradlew assembleDebug` or `fvm flutter build apk --release` succeeds
- The APK installs successfully on the target device/emulator
- `dependencyInsight` shows no duplicate source for the same AAR/classes
- `local.properties` points to the correct FVM Flutter SDK
- `gradle.properties` enables AndroidX/Jetifier when AndroidX dependencies are present

## Notes from a real-world fix
- A Flutter app using `packages/ekyc` required adding `packages/ekyc/android/local-maven-repo` to the app-level repositories.
- A `face_auth` plugin depended on `com.vpbank:faceauthensdk:1.0.0` which transitively pulled `com.vpbank:liveness:1.0.0`; the app also pulled `liveness_corp` from eKYC, causing duplicate classes.
- Removing the direct app-level `faceauthensdk` dependency and excluding the transitive `liveness` from `face_auth` resolved the conflict.
- The emulator then failed on storage, so a release APK build/install was used to verify the app end-to-end.
