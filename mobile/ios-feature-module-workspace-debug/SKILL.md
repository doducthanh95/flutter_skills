---
name: ios-feature-module-workspace-debug
description: Debug and fix iOS feature/module targets that depend on Core/Resoures or similar sibling frameworks, including workspace setup, protocol conformance fixes, and build verification.
version: 1.0.0
tags: [ios, xcodebuild, workspace, framework, module, debugging, swift]
---

# iOS Feature Module Workspace Debug

Use this skill when an iOS module/framework target:
- imports sibling frameworks like `Core` or `Resoures`
- builds in Xcode but fails from the raw `.xcodeproj`
- has protocol conformance errors after adding module references
- needs a repeatable fix-and-verify workflow

## Typical symptoms
- `Unable to find module dependency: 'Core'`
- `Unable to find module dependency: 'Resoures'`
- protocol conformance errors after linking dependencies
- build succeeds only after opening the correct workspace

## Workflow

### 1) Verify the build entry point
Prefer building from the workspace, not the raw project, when the feature depends on sibling frameworks.

Check:
- `.xcworkspace` exists
- it includes the feature project and dependency projects
- the scheme exists inside the workspace
- the referenced subprojects actually open without parse errors

Example:
```bash
xcodebuild -workspace Feature.xcworkspace -scheme Feature -configuration Debug -sdk iphoneos build
```

If the raw project fails with missing modules, that is usually expected until the workspace is used.

### 1b) Validate project file integrity before chasing compiler errors
A malformed `.pbxproj` can surface as a project parse error or a JSON/plist-like failure before any Swift compilation starts.

If Xcode reports the dependency project is damaged:
- inspect the project file for unmatched braces / missing `objects = {` / truncated sections
- re-open the project directly in Xcode to confirm it parses
- only after parsing succeeds, continue with module resolution or build fixes

### 1c) Watch for duplicate binary embeddings
When a feature project and a sibling SDK both embed the same `.xcframework`/`.framework`, Xcode may fail with:
- `Multiple commands produce ...`
- duplicate `Headers`, `Modules`, `Info.plist`, or resource files
- duplicate output frameworks in `ProcessXCFramework`

Fix by keeping exactly one source of truth for each binary:
- keep the framework in one project only
- remove the duplicate `PBXBuildFile`, `PBXFileReference`, and embed/link entries from the other project
- prefer the upstream SDK project as the single owner of its own dependencies

This is especially common with shared binaries like `OpenSSL`, `eIDSDK`, or bundled resource frameworks.

### 2) Confirm module import names
When a framework is referenced in Xcode, the Swift import name must match the built product name.

Examples:
- `import Core`
- `import Resoures`

Do not guess the module name; confirm the built framework name from the sibling project’s `productName` / `productReference` in its `project.pbxproj`.

### 3) Fix protocol conformances introduced by the dependency
When using shared protocols from `Core`, inspect the exact protocol signature before editing.

Common cases:
- protocol requires `init()`
- stored properties must have defaults if no custom initializer exists
- `language` and `packageName` must be non-optional and initialized
- nested state enums with associated values may need `Equatable` conformance on payload types

Examples:
```swift
final class ViTranslate: Translate {
    var language = EnumLanguage.vn
    var packageName = "loan"

    required init() {
        self.language = .vn
        self.packageName = "loan"
    }

    func register() -> [String: String] {
        [:]
    }
}
```

If an enum contains associated values:
```swift
struct HomeEntity: Equatable { }
```

### 4) Keep the architecture consistent
For a feature module, keep the layers aligned:
- `Presentation/` → ViewController, ViewModel
- `Domain/` → Entity, Repository protocol, UseCase
- `Data/` → Repository implementation, DTO/model
- `DI/` → feature composition root
- `Coordinator/Router` → navigation entry point

### 5) Verify the actual fix
Always re-run the workspace build after changes:
```bash
xcodebuild -workspace Feature.xcworkspace -scheme Feature -configuration Debug -sdk iphoneos build
```

If the build still fails:
- inspect the last compiler errors first
- fix protocol conformance or module name issues before touching architecture
- avoid editing generated frameworks or build artifacts

## Common pitfalls
- Building `.xcodeproj` directly when dependencies live in sibling projects
- Wrong module import string because target product name differs from folder name
- Leaving placeholder code like `<#code#>` in protocol initializers
- Forgetting `Equatable` for state enums with associated values
- Assuming a workspace is enough without checking the scheme name

## Verification checklist
- workspace exists and includes dependent projects
- imports match the built module names
- `xcodebuild -workspace ... build` succeeds
- no placeholder code remains
- framework target produces a buildable `.framework`

## Notes from the Loan module case
- `Loan` built successfully only after using `Loan.xcworkspace`
- `Core` and `Resoures` were resolved as sibling framework dependencies
- `Translate` conformers needed explicit `language` and `packageName` setup
- `HomeViewModel.State` needed `HomeEntity: Equatable`
