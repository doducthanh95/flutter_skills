---
name: flutter-android-plugin-binary-swap-debug
description: Debug Flutter Android plugin build failures after swapping a binary AAR/XCFramework dependency. Covers missing artifacts, API/package drift, minSdk mismatches, and install-time ADB issues.
version: 1.0.0
tags: [flutter, android, plugin, aar, binary-dependency, debugging]
---

# Flutter Android plugin binary-swap debugging

Use this skill when a Flutter app or Flutter plugin fails after replacing an Android binary dependency (AAR/JAR/Maven artifact), especially in a local plugin under packages/<plugin>/android.

## Typical symptoms
- Gradle fails with "Could not find :<artifact>:" or missing `*.aar` in `android/libs`
- Kotlin compile errors from unresolved classes or methods after swapping SDK binaries
- `startEkyc(...)` / similar API signature no longer matches the new SDK
- Flutter fix suggests higher `minSdkVersion`
- Build compiles, but install fails on emulator/device with ADB/storage issues

## Core principle
Do not patch blindly. For binary swaps, first verify the actual binary contents and exported API, then align code to the new signature, then rebuild, then validate install/runtime.

## Investigation flow

### 1) Confirm the binary is really present where Gradle expects it
Check the plugin/module build file and the filesystem path it references.

Common checks:
- `implementation(name: 'Something', ext: 'aar')` means the file must exist at `android/libs/Something.aar`
- `flatDir { dirs project(':plugin').file('libs') }` must point to a real libs folder
- **The `libs` directory itself may not exist yet** - you must create it before copying the AAR

If the file is missing:
1. Create the directory if needed: `mkdir -p packages/<plugin>/android/libs`
2. Copy the correct AAR into the exact directory: `cp path/to/SDK.aar packages/<plugin>/android/libs/`
3. Verify: `ls -lh packages/<plugin>/android/libs/`

Common mistake: assuming the libs folder exists because it's referenced in build.gradle - it won't exist until you create it and place the binary there.

### 2) Inspect the binary before editing code
Use `zipfile`, `jar tf`, and `javap` to discover the real API.

Recommended commands:
- `jar tf EnterpriseOnboardingSDK.aar`
- extract `classes.jar`
- `javap -classpath classes.jar -public <FQCN>`

Look for:
- the real package name of classes
- the real companion/object methods
- whether methods are properties vs setters vs static functions
- exact parameter names and order for Kotlin callers

Do not trust old source imports or previous versions of the SDK.

### 3) Compare against old call sites in the repo
Search other apps/modules in the mono-repo that use the same SDK or a sibling package.

Useful patterns:
- search for `EkycManager`, `startEkyc(`, `shouldShowLogAPI`, `isBypassCheck`
- compare the working call site to the broken one
- prefer working in-repo examples over guessing

### 4) Fix compile errors one by one
When the compiler says:
- `Unresolved reference`: wrong package/class name or missing binary
- `No parameter with name ...`: API signature changed; use positional args or updated names
- `No value passed for parameter ...`: update the call to match the current method signature
- `Null cannot be a value of a non-null type`: adjust null handling before calling the SDK

Make the smallest possible change and rebuild after each correction.

### 5) Watch for minSdk / Android compatibility after compile issues
If Gradle reports the SDK requires a higher Android API:
- update `minSdkVersion` in the app module
- prefer the minimum version required by the binary, not a guess
- verify the plugin documentation or build output before changing app-wide settings

### 6) Validate install/runtime separately from compilation
A successful compile does not mean the app is runnable.

If install fails with ADB errors like:
- `Requested internal only, but not enough space`
- `INSTALL_FAILED_VERSION_DOWNGRADE`
- signature / package conflicts

Then the problem is on the device/emulator side, not the code.

Common fixes:
- uninstall the old app from the device/emulator
- free emulator storage or use a different emulator
- bump versionCode/build number if a downgrade is involved

## Practical commands

Build/run loop:
```bash
flutter run -d <device> --debug
```

Inspect a binary AAR:
```bash
python3 - <<'PY'
import zipfile
aar = 'path/to/SDK.aar'
with zipfile.ZipFile(aar) as z:
    print(z.namelist())
PY
```

Extract `classes.jar` and inspect API:
```bash
jar tf classes.jar | grep 'EkycManager\|YourClass'
javap -classpath classes.jar -public 'com.example.YourClass'
```

## Verification checklist
- [ ] The `android/libs` directory exists (create it if needed: `mkdir -p android/libs`)
- [ ] Binary file exists at the exact path Gradle expects
- [ ] Ran `flutter clean` after placing the new AAR
- [ ] New package/class names match the binary
- [ ] Method signatures match the binary, not old source code
- [ ] `minSdkVersion` satisfies the SDK requirement
- [ ] `flutter run` reaches install step
- [ ] Device/emulator has enough storage and no stale conflicting install

## Pitfalls
- **The `libs` directory may not exist at all** - Gradle will fail with "Could not find :<artifact>:" even though build.gradle references it correctly; you must create the directory and place the AAR manually
- Old source code may still compile in a different app/module, so search the mono-repo for a working reference
- Kotlin named arguments can break when the SDK changed parameter names; positional arguments may be safer after inspection
- AARs often contain transitive native/resources dependencies; missing the main AAR path is only the first failure
- Emulator install failures can appear after a clean compile and should not be confused with code errors
- Running `flutter clean` is essential after adding/replacing AARs to clear stale Gradle caches

## When to use this skill
Use it whenever a Flutter plugin module is swapped to a new Android binary SDK and build/run starts failing in stages: missing artifact → compile mismatch → minSdk mismatch → install/runtime issue.
