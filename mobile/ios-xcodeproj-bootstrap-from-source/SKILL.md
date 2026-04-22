---
name: ios-xcodeproj-bootstrap-from-source
description: Bootstraps a new iOS .xcodeproj for an existing source folder, using a framework target and Xcode filesystem-synchronized groups in the style of VPBank/Authen.
version: 1.0.0
tags: [ios, xcodeproj, pbxproj, framework, bootstrap, filesystem-synchronized, authen, vpbank]
---

# iOS Xcodeproj Bootstrap From Source

Use this skill when you have an existing iOS feature/module source folder and need to create a new `.xcodeproj` from scratch, especially when the code already follows an Authen/VPBank-style framework module layout.

## When to use
- A folder contains Swift/XIB/asset source but no `.xcodeproj`.
- You want a single framework target that automatically picks up files from the folder.
- You want to mirror the Authen project style without manually listing every source file.
- You need a reusable workflow for generating a valid `.xcodeproj`, scheme, and Info.plist.

## Recommended approach
1. Inspect the source tree first.
2. Determine whether the module should be a framework target or app target.
3. Identify required sibling dependencies like `Core.framework` and `Resoures.framework`.
4. Create a minimal but valid `.xcodeproj` using filesystem-synchronized groups.
5. Add a shared scheme.
6. Add or reuse `Info.plist`.
7. Verify with `xcodebuild -list` before trying a full build.

## Core principles
- Prefer `PBXFileSystemSynchronizedRootGroup` for source folders that should auto-sync.
- Keep the target count minimal unless tests are explicitly needed.
- Do not hand-author a huge `PBXFileReference` list when the folder layout can be synchronized.
- Keep the project file syntactically valid before chasing compiler errors.
- Verify the project parses with `xcodebuild -list` first.

## Step-by-step workflow

### 1) Inspect source and compare with a known-good module
Look at a similar module such as Authen:
- project root
- framework target name
- shared scheme layout
- Info.plist placement
- dependency framework references
- filesystem-synchronized root group usage

Capture:
- root folder name
- entry files like `Module.swift`, `ModuleCoordinator.swift`, `ModuleDI.swift`, `ModuleFactory.swift`
- dependency frameworks imported in source files
- resource files that need to be excluded from sync or added explicitly

### 2) Choose the target shape
For most feature modules in this repo:
- target type: `com.apple.product-type.framework`
- one primary target only
- optional test target only if explicitly needed

Typical build settings:
- `DEFINES_MODULE = YES`
- `BUILD_LIBRARY_FOR_DISTRIBUTION = YES`
- `SKIP_INSTALL = YES`
- deployment target aligned with repo convention
- `SWIFT_VERSION = 5.0`

### 3) Create the project skeleton
A good minimal layout is:
```text
ModuleName/
├── ModuleName.xcodeproj/
│   └── xcshareddata/xcschemes/ModuleName.xcscheme
├── Info.plist
├── Common/
├── Features/
├── Translate/
└── ...source files...
```

If the folder already contains source, do not move files unless necessary.

### 4) Use filesystem-synchronized root groups
This is the preferred pattern for large source folders with many files.

Recommended structure inside `project.pbxproj`:
- `PBXFileSystemSynchronizedRootGroup` for the source root
- `PBXFileSystemSynchronizedBuildFileExceptionSet` for exceptions like `Info.plist` and scripts
- one `PBXNativeTarget` for the framework
- one `PBXFrameworksBuildPhase` for dependency frameworks
- empty `PBXSourcesBuildPhase` is acceptable when the filesystem-synced target owns the files

### 5) Add dependency frameworks explicitly
If the source imports sibling frameworks, add them to the project as framework file references and framework build phase entries.

Examples:
- `Core.framework`
- `Resoures.framework`

Important:
- Keep module names aligned with actual built product names.
- Do not assume folder name equals product name.

### 6) Create a shared scheme
The scheme should point to the framework target and be shared so Xcode and CLI builds can use it.

Typical scheme contents:
- build framework target
- launch/profile reference the same target
- shared scheme stored under `xcshareddata/xcschemes/`

### 7) Add Info.plist
If the module needs an explicit plist, place it at the project root or module root.
A minimal ATS plist is often enough when the framework itself does not need app lifecycle keys.

### 8) Verify the project parses
Before building, run:
```bash
xcodebuild -list -project ModuleName.xcodeproj
```
This catches broken `pbxproj` syntax, missing references, and scheme issues early.

### 9) Build only after parse success
If the project lists correctly, then run a build:
```bash
xcodebuild -scheme ModuleName -project ModuleName.xcodeproj -configuration Debug build
```
Or, if the module is meant to be used in a workspace:
```bash
xcodebuild -workspace Workspace.xcworkspace -scheme ModuleName -configuration Debug build
```

## Common pitfalls
- Writing an invalid `project.pbxproj` by hand.
- Forgetting to create a shared scheme.
- Using the wrong framework product name in the scheme.
- Not adding sibling framework references even though the code imports them.
- Putting `Info.plist` into synchronized files when it should be excluded.
- Creating a test target when the user only asked for the main module.
- Assuming a workspace is required when a clean framework project is enough.

## Verification checklist
- [ ] `.xcodeproj` exists
- [ ] shared scheme exists
- [ ] `xcodebuild -list -project ...` succeeds
- [ ] target product name matches the scheme buildable reference
- [ ] framework dependencies are listed if imported by source
- [ ] `Info.plist` is present and excluded from filesystem sync if needed
- [ ] source folder auto-syncs correctly

## Practical note from the OD/Authen case
A reliable pattern was:
- compare the module against Authen first, not against generic Xcode defaults
- before recreating anything, check whether an `.xcodeproj` already exists and whether `xcodebuild -list` succeeds
- create a single framework target when the folder is a reusable feature/module
- use filesystem-synchronized root groups instead of hand-listing every source file
- keep `Info.plist` as an exception set entry when it should not be auto-synced
- mirror sibling framework links exactly (commonly `Core.framework` and `Resoures.framework` in VPBank modules)
- verify with `xcodebuild -list` before any full build attempt
- if the initial pbxproj draft is malformed, rewrite the file cleanly instead of patching many broken fragments

## When to stop and inspect manually
Stop and inspect the source/project if:
- the module imports unexpected frameworks
- the module needs a test target or workspace integration
- resources must be explicitly linked instead of auto-synced
- the folder layout differs significantly from Authen-style modules
