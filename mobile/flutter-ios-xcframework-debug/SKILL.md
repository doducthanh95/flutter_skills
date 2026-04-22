---
name: Flutter iOS XCFramework Debug
description: |
  Debug Flutter iOS build failures in macOS/Xcode/CocoaPods setups, especially missing vendored XCFrameworks,
  broken .symlinks/plugins paths, and Podspec/xcframework copy script mismatches. Uses a systematic
  root-cause flow before editing podspecs or build settings.
version: 1.0.0
tags: [flutter, ios, xcode, cocoapods, xcframework, build-debug]
---

# Flutter iOS XCFramework Debug

## When to use
Use this skill when a Flutter iOS app:
- opens in Xcode but fails to build or run on simulator/device
- fails inside `[CP] Copy XCFrameworks`
- mentions missing `.xcframework` paths or `rsync ... No such file or directory`
- has plugin pods that depend on vendored frameworks
- needs triage on macOS using `Runner.xcworkspace`, `podspec`, and generated Pods scripts

## Core idea
Do not guess. First find the exact missing artifact or broken build script path.
Most failures here are not app-code issues; they come from:
- missing vendored framework folders in `packages/<plugin>/ios/Frameworks`
- podspecs referencing a framework or dependency that is not present
- stale CocoaPods scripts under `ios/Pods/Target Support Files/*`
- wrong Flutter version or stale DerivedData / Pods state

## Recommended flow

### 1) Reproduce with Xcodebuild or Flutter verbose
Prefer a direct build first:
```bash
cd <flutter-project>
# if project uses FVM
fvm flutter --version
fvm flutter build ios --simulator -v 2>&1 | tee /tmp/flutter_ios_build.log
```
Or build the workspace directly:
```bash
xcodebuild \
  -workspace ios/Runner.xcworkspace \
  -scheme Runner \
  -configuration Debug \
  -destination 'platform=iOS Simulator,name=iPhone 17 Pro' \
  build 2>&1 | tee /tmp/xcodebuild_ios.log
```

### 2) Read the first real error
- Find the earliest `error:` or failed script, not the tail noise.
- If the failure is in `[CP] Copy XCFrameworks`, inspect the related pod target.
- If the failure says `rsync ... (l)stat: No such file or directory`, it usually means the input xcframework path does not exist.

### 3) Inspect Podspecs for the failing plugin
Locate the plugin podspec under `packages/<plugin>/ios/*.podspec`.
Check for:
- `s.vendored_frameworks`
- `s.dependency` entries for other pods or xcframework-backed pods
- `FRAMEWORK_SEARCH_PATHS`
- `EXCLUDED_ARCHS`
- `s.platform`

Example:
```bash
search_files("*.podspec", target="files", path="<project>/packages")
```

### 4) Inspect generated CocoaPods scripts
Open the generated script under:
- `ios/Pods/Target Support Files/<pod>/<pod>-xcframeworks.sh`
- `ios/Pods/Target Support Files/<pod>/<pod>-xcframeworks-input-files.xcfilelist`
- `ios/Pods/Target Support Files/<pod>/<pod>-xcframeworks-output-files.xcfilelist`

These files tell you exactly which `.xcframework` path CocoaPods expects.
Compare that with the real file system under:
- `ios/.symlinks/plugins/<plugin>/ios/Frameworks/`

### 5) Verify the artifact actually exists
For each referenced xcframework, confirm:
- directory exists
- it contains `Info.plist`
- has `ios-arm64` and/or `ios-arm64_x86_64-simulator`
- the slice names match the pod script expectations

If the script expects:
```text
.../eIDSDK.xcframework/ios-arm64_x86_64-simulator/*
```
and that directory is missing, the build will fail even if other frameworks exist.

### 6) Decide the fix based on root cause
Common fixes:
- restore the missing xcframework from the upstream binary package
- correct the podspec path to the actual framework name
- remove a stale dependency to a framework no longer shipped
- update `FRAMEWORK_SEARCH_PATHS` if the framework is present but not found
- regenerate Pods if scripts are stale

Do not patch podspecs until you know whether the binary exists.

### 7) Clean and rebuild after a real fix
```bash
rm -rf ~/Library/Developer/Xcode/DerivedData/Runner-*
cd ios
pod deintegrate
pod install
cd ..
fvm flutter pub get
fvm flutter build ios --simulator -v
```

## Practical heuristics learned from a real repo
- The project may have multiple plugins with prebuilt frameworks in `packages/<plugin>/ios/Frameworks`.
- One plugin can compile fine while another fails in its xcframework copy step.
- A podspec can declare a dependency (for example `eIDSDK`) that is not present anywhere in the repo; this usually means the binary framework package was not checked in or the path changed.
- `ios/Pods/Target Support Files/<pod>/<pod>-xcframeworks.sh` is often the fastest way to identify the expected missing path.
- If Xcode can open the workspace but build fails only on simulator, it is usually a dependency packaging issue, not a simulator/device configuration issue.
- If the project uses FVM, prefer `fvm flutter ...` instead of `flutter ...` to avoid version drift.

## Example symptom -> diagnosis mapping
- `rsync: ... lstat: No such file or directory`
  -> missing path in `.symlinks/plugins/.../Frameworks/...xcframework`

- `PhaseScriptExecution [CP] Copy XCFrameworks` failed
  -> inspect pod-specific xcframework script and xcfilelists

- build works for some plugins but not one plugin target
  -> compare that plugin’s podspec and Frameworks folder against the working plugins

- simulator build fails after workspace opens successfully
  -> likely pod/binary packaging mismatch, not Xcode project corruption

## Verification
A successful resolution should end with:
- `xcodebuild ... build` exits 0
- no `[CP] Copy XCFrameworks` errors
- the app launches on the selected simulator/device

## Update this skill when
- you discover a new recurring CocoaPods/XCFramework failure pattern
- the repo uses a different packaging convention for plugin binaries
- the project changes from local vendored frameworks to remote binary pods or Swift Package Manager

---

# Addendum: Handling Podspecs that reference private GitHub repos
From a recent run in this repo we encountered a Pod `eIDSDK (1.0.1)` whose podspec on the public trunk points to a personal GitHub repo that is not available (https://github.com/duynghiavu/eIDSDK.git). This causes `pod install` to fail with `fatal: repository '.../eIDSDK.git/' not found`.

When you see that error:
- Run `pod spec cat eIDSDK` to confirm the podspec's `source` field.
- Locate which plugin depends on it (search for `s.dependency 'eIDSDK'`) — in this repo it's `packages/ekyc/ios/ekyc.podspec`.

Options to resolve:
1) Provide a local copy of the framework/pod and override in your Podfile:
   pod 'eIDSDK', :path => '../packages/ekyc/ios/eIDSDK'

2) Remove the dependency from the plugin podspec and vendor the xcframework inside the plugin (s.vendored_frameworks), if acceptable.

3) Ask the upstream owner to update the podspec on trunk to point to a valid repo or publish a new pod with correct source.

4) Use a private specs repo (Artifactory/Nexus/internal CocoaPods repo) and push a corrected eIDSDK podspec there, then add that spec repo in your Podfile.

Prefer local override (1) for immediate development.
