---
name: ios-xcode-project-rename
description: Rename an iOS/macOS Xcode project or module from one product name to another, including folders, workspace/project files, Swift file names, bundle identifiers, and project references.
version: 1.0.0
tags: [ios, xcode, rename, swift, project-structure]
---

# iOS Xcode Project Rename

Use this skill when renaming an existing Xcode project/module from one name to another, e.g. `Loan` → `Deposit`.

## When to use
- The project root folder name changes
- The `.xcworkspace` / `.xcodeproj` names should change
- Public Swift types, package constants, and translation keys need to match the new name
- You want to verify there are no lingering old-name references

## Recommended workflow

1. Inspect the current tree and search for old-name references.
   - Find files and folders containing the old name.
   - Search inside source, README, and project files.

2. Rename the filesystem items in this order:
   - root folder
   - `.xcodeproj`
   - `.xcworkspace`
   - top-level Swift files that encode the product name
   - translation / DI / coordinator files if they contain the old name

3. Update Swift source content.
   - `PACKAGE_NAME`
   - `SDK` class name
   - coordinator / router names
   - translation extensions like `trLoan` → `trDeposit`
   - domain / enum / bundle identifiers

4. Update Xcode metadata files.
   - `project.pbxproj`
   - `contents.xcworkspacedata`
   - `xcuserdata/.../xcschememanagement.plist`

5. Update docs and external links.
   - README title
   - Git remote URLs
   - integration/settings links

6. Verify thoroughly.
   - Search for the old name again in the renamed root
   - Confirm the new name appears consistently
   - Check for old-name references in comments inside `pbxproj`

## Practical notes

### Common rename targets
- `Loan.swift` → `Deposit.swift`
- `LoanCoordinator.swift` → `DepositCoordinator.swift`
- `LoanDI.swift` → `DepositDI.swift`
- `Loan.xcodeproj` → `Deposit.xcodeproj`
- `Loan.xcworkspace` → `Deposit.xcworkspace`

### Common text replacements
- `LoanSDK` → `DepositSDK`
- `LoanCoordinator` → `DepositCoordinator`
- `LoanRouter` → `DepositRouter`
- `trLoan` → `trDeposit`
- `PACKAGE_NAME = "loan"` → `PACKAGE_NAME = "deposit"`
- `packageName = "loan"` → `packageName = "deposit"`
- bundle id `...Loan` → `...Deposit`

## Pitfalls
- `project.pbxproj` often contains many old-name comments and target display names even after file rename.
- `xcuserdata/.../xcschememanagement.plist` may still reference the old scheme name.
- `contents.xcworkspacedata` usually points to the old project path and must be updated.
- README and GitLab/GitHub URLs often still reference the old repo slug.
- Renaming only Swift symbols without renaming the files can leave the project inconsistent.

## Verification checklist
- [ ] No `OldName` or `oldname` references remain in source
- [ ] Workspace points to the new `.xcodeproj`
- [ ] Project file references the new target/product names
- [ ] Public Swift APIs use the new feature name
- [ ] README and links are updated

## Useful commands / tools
- Use file search to locate files by name and content.
- Use targeted rename/edit tools rather than manual text editing when possible.
- Always verify after renaming with a final search for the old name.
