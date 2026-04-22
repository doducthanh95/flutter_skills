---
name: flutter-plugin-local-maven-debug
description: Debug and fix Flutter plugin Android builds that depend on internal AARs published in a local Maven repo, with FVM/Flutter SDK path issues, AndroidX, and JDK compatibility.
version: 1.0.0
tags: [flutter, android, plugin, gradle, local-maven, debugging]
---

# Flutter Plugin Local Maven Debug

Use this skill when a Flutter plugin or app module depends on internal Android SDKs distributed as AARs/POMs in a local Maven repo and the build fails due to Gradle resolution, Flutter SDK path, AndroidX, or JDK incompatibilities.

## When to use
- A plugin/module has `local-maven-repo/` with internal artifacts.
- The build resolves AARs in one project but not another.
- `flutter.sdk` points to the wrong Flutter install.
- Gradle fails with JDK/bytecode errors like `Unsupported class file major version`.
- AndroidX/Jetifier errors appear after adding modern Android dependencies.
- `flutter run` works only after fixing the plugin build first.

## Core workflow

### 1) Map the actual module layout
- Find the plugin root and Android module.
- Identify the artifact coordinates used by the plugin.
- Compare with what exists in `android/local-maven-repo/`.
- Prefer Maven coordinates over `flatDir` when a `.pom` is present.

Example checks:
- `android/build.gradle`
- `android/settings.gradle`
- `android/local-maven-repo/.../*.pom`
- `android/src/main/...` plugin entrypoints

### 2) Inspect the current Gradle setup
Check for these patterns:
- `maven { url = uri("$rootDir/local-maven-repo") }`
- `implementation("group:artifact:version")`
- `compileOnly files("$flutterRoot/bin/cache/artifacts/engine/.../flutter.jar")`
- root project vs included subproject naming
- `android.useAndroidX=true`
- `android.enableJetifier=true`

### 3) Reproduce the failure in layers
Always reproduce in this order:
1. Gradle configuration / project discovery
2. Dependency resolution
3. Kotlin/Java compilation
4. `flutter run` or app launch

Use the smallest command that isolates the failure:
- `./gradlew projects`
- `./gradlew assembleDebug --stacktrace`
- `flutter run -d <device>`

### 4) Fix common root causes

#### A. JDK incompatibility
If you see:
- `Unsupported class file major version 67`

Then:
- switch to an older compatible JDK used by the project, commonly JDK 17 or 19
- verify with `java -version`
- export `JAVA_HOME` before rerunning Gradle

#### B. Wrong task path
If the module is a root Gradle project, do not call `:module:assembleDebug` unless `./gradlew projects` shows that subproject.
- Use `./gradlew assembleDebug` for single-module root projects.
- Use `./gradlew :app:assembleDebug` only when the project structure באמת has `:app`.

#### C. Wrong Flutter SDK path
If `local.properties` points to a stale Flutter install:
- read the plugin/app `local.properties`
- update `flutter.sdk` to the real FVM-managed path or installed Flutter path
- verify the path actually exists

#### D. Wrong Flutter engine jar path
If using `compileOnly files(...)` for Flutter embedding, ensure the path matches the actual SDK:
- `android-arm/flutter.jar`
- `android-arm64/flutter.jar`

Pick the one that exists in the target Flutter SDK cache.

#### E. AndroidX not enabled
If Gradle says AndroidX dependencies are present but AndroidX is disabled:
- add to `android/gradle.properties`
  - `android.useAndroidX=true`
  - `android.enableJetifier=true`

#### F. Local Maven repo not wired correctly
If the plugin ships AARs/POMs:
- add the repo under `repositories {}` in the plugin module
- if the host app also needs it, add the same repo in the host app Gradle config
- replace `implementation(name: 'X', ext: 'aar')` with the Maven coordinate if the repo has a matching POM

### 5) Verify after each fix
After each change, rerun the same command that failed.
- First aim for `./gradlew assembleDebug` success.
- Then run the app with `flutter run`.
- If runtime still fails, inspect logs separately; don’t assume build success means runtime success.

## Example fix pattern
For a plugin with local Maven artifacts:

1. Add repo:
```gradle
allprojects {
    repositories {
        maven { url = uri("$rootDir/local-maven-repo") }
        google()
        mavenCentral()
    }
}
```

2. Use Maven dependency:
```gradle
implementation "vpbank.corp:EnterpriseOnboardingMasterApp:1.0.0"
```

3. Enable AndroidX:
```properties
android.useAndroidX=true
android.enableJetifier=true
```

4. Point to the real Flutter SDK:
```properties
flutter.sdk=/Users/<you>/fvm/versions/<version>
```

5. Point `flutter.jar` to the existing engine folder:
```gradle
compileOnly files("$flutterRoot/bin/cache/artifacts/engine/android-arm64/flutter.jar")
```

6. Build:
```bash
export JAVA_HOME=/path/to/jdk19
./gradlew assembleDebug --stacktrace
```

7. Run app:
```bash
flutter run -d <device>
```

## Pitfalls
- Don’t assume `/opt/homebrew/bin/flutter` exists; verify the actual SDK path.
- Don’t hardcode `:module:assembleDebug` unless the module is a subproject.
- Don’t use `flatDir` if the repo already has proper POM metadata.
- Don’t ignore JDK major-version errors; fix the runtime JDK first.
- Don’t stop at build success; always verify `flutter run` too.

## What to check in a successful case
- `./gradlew projects` shows expected project structure.
- `./gradlew assembleDebug` completes successfully.
- The local Maven artifact resolves from the intended repo.
- `flutter run` launches the app without embedding-path errors.
