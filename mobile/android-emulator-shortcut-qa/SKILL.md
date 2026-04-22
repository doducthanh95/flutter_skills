---
name: android-emulator-shortcut-qa
description: Automate exploratory QA on Android apps running in an emulator using adb, UIAutomator dumps, screenshots, and app relaunches. Useful for testing login screens, shortcut buttons, and navigation flows when UI labels are partially hidden or taps trigger overlays.
version: 1.0.0
tags: [android, emulator, qa, adb, uiautomator, screenshots, exploratory-testing]
---

# Android Emulator Shortcut QA

## When to use
Use this skill when you need to test shortcut buttons or login-screen interactions in an Android app running on an emulator, especially when:
- labels are not fully visible in accessibility output
- taps may open overlays, webviews, or external apps
- you need to compare before/after state for each tap
- the app is a Flutter app or a mixed Flutter/native app

## Core workflow

### 1) Identify the installed package and launch target
- Find the app package with:
  - `adb shell pm list packages | grep -i '<app-name-or-brand>'`
- Confirm the launcher activity and main intent with:
  - `adb shell dumpsys package <package> | grep -n -A8 -B4 'android.intent.action.MAIN'`
- Launch the app explicitly if needed:
  - `adb shell am start -n <package>/<activity>`

### 2) Capture the current UI tree
- Dump the accessibility tree:
  - `adb shell uiautomator dump /sdcard/ui.xml`
  - `adb shell cat /sdcard/ui.xml`
- Parse:
  - `text="..."`
  - `content-desc="..."`
  - bounds for clickable elements
- Use the UI tree first when available; it is more reliable than guessing coordinates.

### 3) Take a screenshot for visual verification
- Capture a screenshot:
  - `adb exec-out screencap -p > /tmp/screen.png`
- If the UI tree is incomplete or ambiguous, compare screenshots before/after each tap.
- When possible, use vision analysis on the screenshot to identify shortcut areas and confirm navigation.

### 4) Tap one control at a time and reset between tests
For exploratory QA, do not chain many taps without resetting state.
Recommended loop:
1. start from the login screen
2. tap one shortcut/button
3. wait 1–3 seconds
4. dump UI tree again
5. capture screenshot again
6. record what changed
7. force-stop and relaunch the app before the next tap

Useful reset commands:
- `adb shell am force-stop <package>`
- `adb shell am start -n <package>/<activity>`

### 5) Summarize outcomes by action type
Classify each button into one of these outcomes:
- opens a new in-app screen
- opens a dialog / bottom sheet
- opens a webview
- launches an external app
- performs no visible action
- crashes / hangs / stays on same screen

## Practical heuristics
- If `uiautomator` exposes only generic nodes, use bounds from the tree and tap by coordinate.
- If taps appear to do nothing, compare screenshots hashes or use a fresh relaunch to avoid hidden state.
- If a tap opens an overlay, the UI tree may radically change; re-dump immediately.
- If content-desc text is available, prefer it over raw coordinates.
- For bottom-row shortcuts, the visible label may be in a parent container while the actual tappable child is an `ImageView`.

## Suggested command sequence
1. `adb devices`
2. `adb shell dumpsys package <package> | grep -n -A8 -B4 'android.intent.action.MAIN'`
3. `adb shell am start -n <package>/<activity>`
4. `adb shell uiautomator dump /sdcard/ui.xml`
5. `adb shell cat /sdcard/ui.xml`
6. `adb exec-out screencap -p > /tmp/screen.png`
7. Tap one element with `adb shell input tap X Y`
8. Repeat steps 4–7 for each shortcut, relaunching between taps

## Pitfalls
- Assuming the launcher package name is the same as the Flutter applicationId
- Relying only on screenshot similarity; some taps change hidden state without obvious visual differences
- Forgetting to restart the app between tests, which can make later taps depend on earlier state
- Tapping the label area instead of the actual clickable child node
- Using `uiautomator` too late after navigation; the original state may already be gone
- Expecting accessibility output to be clean: Flutter screens often expose generic `View`/`ImageView` nodes, while the useful labels live in `content-desc`
- Depending on remote vision analysis as a single source of truth; if vision fails or is unavailable, fall back to adb screenshots plus UIAutomator tree inspection

## Useful fallback techniques
If vision analysis is unavailable or the UI tree is sparse:
1. Capture a screenshot with `adb exec-out screencap -p`
2. Dump the UI tree immediately after launch/tap with `adb shell uiautomator dump`
3. Inspect `content-desc`, `text`, and `bounds` for clickable nodes
4. Tap by coordinate using the center of each bounds box
5. Compare pre/post screenshots or image hashes to detect subtle changes
6. Force-stop and relaunch between buttons to keep each test independent

## Verification
A good test pass usually includes:
- package name and launcher activity confirmed
- UI tree and screenshot captured before each tap
- each shortcut tested independently from a clean app start
- outcome recorded for each button
- any ambiguous buttons re-tested with a different tap point or a relaunch

## Example note
On a VPBank NEOBizPlus Android emulator session, this workflow was useful to discover the actual package (`com.vpbank.mobileappcmp`), launch the app directly, inspect the login screen, and verify that shortcut areas like Hỗ trợ, Tỷ giá, Lãi suất, and eKYC were present in the accessibility tree even when the screenshot/vision path was flaky. The tree exposed labels such as `content-desc="Tên đăng nhập\nMật khẩu"`, `content-desc="ĐĂNG NHẬP"`, and the shortcut row labels, which made coordinate-based testing reliable.
