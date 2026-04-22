---
name: vpbank-ios-build-simulator-debug
description: Debug VPBank iOS build and simulator failures caused by CocoaPods linkage conflicts, missing Core/Resoures-style modules, arm64 simulator slice gaps, and duplicate framework embeddings.
version: 1.0.0
author: Hermes Agent
license: MIT
metadata:
  hermes:
    tags: [ios, flutter, cocoapods, simulator, xcode, vpbank, xcframework, debugging]
    related_skills: [ios-foundations, ios-feature-module-workspace-debug, systematic-debugging]
---

# VPBank iOS Build & Simulator Debug

## When to use
Use this skill when the VPBank iOS/Flutter app:
- fails on `flutter build ios`
- fails on simulator launch
- reports CocoaPods static framework errors
- reports `Unable to find module dependency: Core`
- reports `Multiple commands produce ... Core.framework`
- needs Apple Silicon simulator compatibility triage

## Core workflow

### 1) Classify the failure first
Look for the first real error:
- `The 'Pods-Runner' target has transitive dependencies that include statically linked binaries` → linkage mismatch
- `Unable to find module dependency: 'Core'` → module/workspace/import or framework resolution issue
- `Multiple commands produce ... Core.framework` → duplicate embed/copy phases
- simulator-only failure mentioning arm64 → missing arm64 simulator slice, likely x86_64-only framework

Do not jump to code edits until the failure is classified.

### 2) Fix CocoaPods linkage mismatch
If VPBank podspecs contain `s.static_framework = true` while Podfile uses `use_frameworks!`, remove the static framework flag from custom podspecs.

Typical files:
- `ios/OpenSSLVPBank/OpenSSLVPBank.podspec`
- `ios/eIDSDKVPBank/eIDSDKVPBank.podspec`
- `ios/CoreSDKVPBank/CoreSDKVPBank.podspec`

Keep Podfile on dynamic frameworks unless there is a separate, tested reason to change it.

### 3) Clean CocoaPods and Flutter state
After podspec changes:
```bash
cd ios
rm -rf Pods Podfile.lock
cd ..
flutter clean
```

Then rebuild.

### 4) Handle simulator architecture gaps
If the framework only supports x86_64 simulator slices and not arm64 simulator:
- add `EXCLUDED_ARCHS[sdk=iphonesimulator*] = 'arm64'` where needed
- ensure the simulator builds under Rosetta on Apple Silicon
- apply this consistently in podspecs and Podfile post_install when required by the project

Typical affected files in this repo:
- `ios/CoreSDKVPBank/CoreSDKVPBank.podspec`
- `ios/OpenSSLVPBank/OpenSSLVPBank.podspec`
- `ios/eIDSDKVPBank/eIDSDKVPBank.podspec`
- `packages/biometrics/ios/biometrics.podspec`
- `packages/ekyc/ios/ekyc.podspec`
- `packages/face_auth/ios/face_auth.podspec`
- `ios/Podfile`

### 5) Remove duplicate framework embeds
If the same framework is copied by both:
- manual Runner embed phase
- CocoaPods `[CP] Embed Pods Frameworks`

remove the manual embed from `ios/Runner.xcodeproj` and let CocoaPods manage the copy.

A common target is `Core.xcframework`, but the same rule applies to OpenSSL/eIDSDK if they are duplicated.

### 6) Verify using the right command
For build verification:
```bash
flutter build ios --debug --no-codesign
```

For simulator verification:
```bash
fvm flutter run
# or
fvm flutter run -d <SIMULATOR_ID>
```

## Recommended order of operations
1. Read the exact build error
2. Remove `s.static_framework = true` from custom VPBank podspecs if present
3. Clean Pods and Podfile.lock
4. Run `flutter clean`
5. Rebuild iOS
6. If simulator-only failure remains, exclude arm64 simulator where needed
7. If duplicate framework copy remains, remove manual Runner embed phase entries
8. Re-run simulator launch

## Practical notes from the VPBank case
- Dynamic CocoaPods worked better than forcing `use_frameworks! :linkage => :static`
- Static linkage caused plugin compatibility issues in this repo
- `Core.xcframework` had no arm64 simulator slice, so x86_64-only simulator build was required
- Manual embed removal fixed duplicate framework output conflicts
- When adding a new example screen into an existing sample app (e.g. TestFramework + ODSDK), first run `xcodebuild -list` to verify the project is readable before building. If `xcodebuild -list` fails with a parse error, fix the `.pbxproj` syntax first (a missing `;` or a malformed section can make Xcode report the project as damaged).
- If `xcodebuild` reports `Multiple commands produce ... <SDK>.framework`, inspect `PBXCopyFilesBuildPhase`, `PBXFrameworksBuildPhase`, `PBXTargetDependency`, and external sample projects referenced by `PBXContainerItemProxy` for duplicate embed/copy sources. Use the exact framework name in the error to locate duplicate sources quickly.
- In projects that aggregate several sibling SDKs (e.g. EnterpriseEKYCSDK, EnterpriseESignSDK, ODSDK, Test Embed XCFramework), a successful new example can still fail build due to pre-existing duplicate embeds in other SDK targets; removing duplicates only from the app target may not be enough.
- For isolated demo work, temporarily remove unrelated target dependencies/reference chains from the app target so you can prove the example screen builds with the minimum dependency set, then reintroduce the extra SDK targets one by one.
- When `xcodebuild` destination lookup fails, prefer an explicit simulator `id` from `xcodebuild -showBuildSettings` or the destinations list instead of a bare simulator name.

## Verification checklist
- `pod install` succeeds
- `flutter build ios --debug --no-codesign` succeeds
- simulator launches successfully
- no `Unable to find module dependency: Core`
- no `Multiple commands produce` errors
- no duplicate framework embedding remains

## Related debugging rules
- Use `systematic-debugging` for root-cause discipline
- Use `ios-feature-module-workspace-debug` when the issue is about workspace/module dependency wiring
- Use `ios-foundations` for UIKit/Swift context if the failure is in app code rather than build configuration
