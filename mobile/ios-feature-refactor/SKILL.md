---
name: ios-feature-refactor
description: Refactor an iOS feature/module into a UIKit + Coordinator + ViewModel + UseCase + Repository structure, with DI and build verification.
version: 1.0.0
tags: [ios, swift, uikit, coordinator, mvvm, clean-architecture, refactor, feature-module]
---

# iOS Feature Refactor

Use this skill when refactoring an existing iOS module or framework into a consistent feature architecture:
- UIKit for presentation
- Coordinator for routing
- ViewModel for state and UI binding
- UseCase for business rules
- Repository for data abstraction
- Entity/Model separation between domain and data

This skill is based on the Loan module refactor workflow and is intended to be reusable across similar iOS feature modules.

## Typical target structure

```text
FeatureName/
  Presentation/
    FeatureViewController.swift
    FeatureViewModel.swift
  Domain/
    FeatureEntity.swift
    FeatureRepository.swift
    FeatureUseCase.swift
  Data/
    FeatureModel.swift
    FeatureRepositoryImpl.swift
  FeatureDI.swift
  FeatureCoordinator.swift
```

## Refactor workflow

1. Inspect the repo structure first
   - Identify actual tracked source files.
   - Detect whether the project is a framework target, app target, or module inside a larger workspace.
   - Find existing entry points: coordinator, DI container, feature screen, and any stubs.

2. Map the existing code to architecture roles
   - Screen classes → Presentation
   - Navigation logic → Coordinator
   - State/business logic → ViewModel
   - Data orchestration/API/storage → Repository
   - Pure business behavior → UseCase
   - Data transfer objects → Model
   - Business entities → Entity

3. Establish a single DI entry per feature
   - Prefer one `FeatureDI.shared` for the module.
   - Repositories should be lazy singletons when shared.
   - UseCases and ViewModels should usually be factory-created.
   - Keep construction logic out of view controllers.

4. Keep coordinators thin
   - Router resolves the coordinator by package/module name.
   - Coordinator maps route names to view controllers.
   - Avoid embedding business logic in navigation code.
   - Parameter-based setup should happen in the coordinator or screen factory.

5. Keep ViewControllers passive
   - Build UI and bind to ViewModel state only.
   - Avoid direct repository or API calls from the screen.
   - Prefer programmatic UIKit if the existing project is already code-driven.

6. Make ViewModels state-driven
   - Represent loading/loaded/error states explicitly.
   - Publish state changes to the view.
   - Use `@MainActor` for UI-facing state mutation.
   - Encapsulate async loading in a single method such as `load()`.

7. Keep repositories abstract
   - Define a protocol in Domain.
   - Implement it in Data.
   - Return Domain entities, not raw models, from the UseCase boundary.

8. Create domain/data mapping explicitly
   - Add `toEntity()` or similar conversion methods in data models.
   - Do not leak API response types into Presentation.

## Recommended patterns

### Coordinator
- `FeatureCoordinatorType` enum for route names.
- `FeatureCoordinator` implements `CoreCoordinator` or project-specific coordinator protocol.
- `FeatureRouter.navigator` resolves the active coordinator from the app routing system.

### DI
- `shared` singleton for the feature container.
- `makeFeatureUseCase()` factory method.
- `makeFeatureViewModel()` factory method.
- Keep imported managers/services behind the DI layer.

### ViewModel
- Use a small state enum:

```swift
enum State: Equatable {
    case idle
    case loading
    case loaded(FeatureEntity)
    case failed(String)
}
```

- Mutate state from one place only.
- Use a closure callback or observable mechanism for UI updates.

### ViewController
- Use `create(params:packageName:)` if the module already follows that factory convention.
- Build UI in `viewDidLoad()`.
- Bind state once.
- Trigger initial load explicitly if needed.

## Common pitfalls discovered

- External modules may be missing from a standalone build.
  - In the Loan module, `xcodebuild` failed because `Core` and `Resoures` were not present as resolved module dependencies.
  - Do not assume the module is self-contained.
  - Check workspace/package integration before debugging code.

- Generated or placeholder files may compile but not be wired into the target.
  - Verify target membership and scheme configuration.

- Old code may use inconsistent naming.
  - Normalize names like `loginUseCase` → `homeUseCase`.
  - Keep `Loan` naming consistent if the module was renamed from another feature.

- Factory methods may still receive unused parameters.
  - Preserve the signature if the project expects it, but keep the implementation minimal and typed.

## Verification steps

1. List project/scheme information
   - Use `xcodebuild -list -project <Project>.xcodeproj`

2. Build the target
   - Use `xcodebuild -scheme <Scheme> -project <Project>.xcodeproj -configuration Debug build`

3. If build fails, check for:
   - missing imported modules
   - missing target membership
   - incorrect framework/app product type
   - route names not matching factory keys
   - stale file names or mismatched class names

## When this skill is useful

- Converting an old feature into clean architecture layers
- Introducing a new screen flow with Coordinator + DI
- Cleaning a module that currently mixes UI, navigation, and data access
- Standardizing a feature module before adding tests

## Reusable checklist

- [ ] Coordinator owns routing
- [ ] ViewController owns UI only
- [ ] ViewModel owns state and async flow
- [ ] UseCase owns business orchestration
- [ ] Repository owns data abstraction
- [ ] Entity/Model split is explicit
- [ ] DI creates objects from the bottom up
- [ ] Build verified with Xcode once dependencies are integrated
