---
name: ios-simulator-automation-qa
description: Automate exploratory QA on iOS apps running in Simulator using simctl, Accessibility (AXUIElement), and screenshots. Useful for launching apps, finding buttons/text fields, filling forms, pressing controls, and capturing evidence.
version: 1.0.0
tags: [ios, simulator, qa, automation, accessibility, simctl, xcode]
---

# iOS Simulator Automation QA

Use this skill when you need to open an iOS app on Simulator, inspect its UI, and test taps/presses on buttons, links, and text fields with reproducible evidence.

## When to use
- Need to launch a specific app bundle on Simulator.
- Need to verify visible buttons/inputs and press them.
- Need to test login/search/form flows quickly without writing a full UI test target.
- Need screenshots or a lightweight exploratory QA pass on a running app.

## Core workflow

### 1) Boot Simulator and identify the app
- Find the available devices:
  ```bash
  xcrun simctl list devices available
  ```
- Boot the chosen device and open Simulator:
  ```bash
  xcrun simctl boot <UDID>
  open -a Simulator
  ```
- Find the app bundle identifier from installed apps:
  ```bash
  xcrun simctl listapps booted | grep -iE 'appname|brand|keyword'
  ```
- Launch the app:
  ```bash
  xcrun simctl launch booted <bundle_id>
  ```

### 2) Inspect the live UI tree with Accessibility
- Use a small Swift script with `AppKit` + `ApplicationServices` to access the Simulator window via AXUIElement.
- Enumerate:
  - `AXWindow`
  - `AXGroup`
  - `AXButton`
  - `AXTextField`
  - `AXStaticText`
  - `AXImage`
- Prefer the accessibility tree over raw screenshots when you need stable refs for controls.

Typical checks:
- buttons by `AXDescription` or `AXTitle`
- text fields by role `AXTextField`
- interactive images that expose `AXPress`

### 3) Interact with controls
- Press buttons with `AXUIElementPerformAction(..., kAXPressAction)`.
- Fill text fields with `AXUIElementSetAttributeValue(..., kAXValueAttribute, ...)`.
- After every meaningful action, re-read the tree or take a screenshot to confirm the state change.

### 4) Capture evidence
- Take a Simulator screenshot after each key state:
  ```bash
  xcrun simctl io booted screenshot /tmp/step.png
  ```
- Use screenshots to document issues, navigation results, and whether the app changed state.

### 5) Verify outcomes
For each action, verify one of:
- new screen appears
- dialog/alert opens
- field value is accepted
- error message appears
- button is inert / no response
- app crashes or freezes

## Useful AX helper patterns

### Find pressable elements
- Search the full AX tree for:
  - `role == "AXButton"`
  - `role == "AXTextField"`
  - `description` containing visible labels
- Some views expose pressable actions even if they are not classic buttons (for example, image links).

### Example Swift helpers
```swift
func press(_ el: AXUIElement) -> Bool {
    AXUIElementPerformAction(el, kAXPressAction as CFString) == .success
}

func setValue(_ el: AXUIElement, _ text: String) -> Bool {
    AXUIElementSetAttributeValue(el, kAXValueAttribute as CFString, text as CFTypeRef) == .success
}
```

### Example launch and screenshot flow
```bash
xcrun simctl launch booted com.example.app
sleep 3
xcrun simctl io booted screenshot /tmp/app-home.png
```

## Pitfalls
- Do not rely only on `simctl listapps`; some apps show the display name but the bundle id is the true launch key.
- Accessibility permissions may be required for scripts that inspect UI elements.
- Some simulator controls are nested deeply inside `AXGroup` nodes; recurse through children.
- Some visible items are `AXImage` or `AXStaticText` but still support press actions.
- Screenshot coordinates/window positions can be confusing; prefer AX role/description for locating elements.
- If the app appears blank, confirm the correct simulator device and bundle id first.

## Recommended QA sequence
1. Launch app.
2. Read the root screen accessibility tree.
3. Identify primary buttons/fields.
4. Fill test data if needed.
5. Press the main CTA.
6. Re-check the tree after each action.
7. Capture screenshots for any issue or important transition.

## What to record in findings
- device model/runtime
- bundle id
- exact button/field label
- steps to reproduce
- expected vs actual behavior
- screenshot path
- any console/log output if available

## Good fit examples
- login screen testing
- onboarding flow smoke testing
- QR / eKYC / payment entry flows
- checking whether visible links/buttons are wired correctly
- quick exploratory QA on a simulator build
