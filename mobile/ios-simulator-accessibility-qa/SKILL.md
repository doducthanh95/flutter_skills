---
name: ios-simulator-accessibility-qa
description: Automate and test iOS apps on the macOS Simulator using simctl, Accessibility (AXUIElement), and simulator screenshots when browser-based tools are unavailable.
version: 1.0.0
tags: [ios, simulator, accessibility, axuielement, simctl, qa, testing]
---

# iOS Simulator Accessibility QA

Use this skill when the user wants you to open an iOS app on the macOS Simulator, click through buttons, fill fields, and explore flows without relying on browser automation.

This is especially useful when:
- the app is already installed in Simulator
- the app has no web DOM or browser snapshot
- vision analysis is unreliable or unavailable
- you need to test tappable UI via Accessibility labels/roles

## Core approach

1. Identify the booted simulator and app bundle ID.
2. Launch the app with `xcrun simctl launch`.
3. Use Accessibility APIs to inspect the live UI tree.
4. Interact via AX actions (`AXPress`) and text input via `kAXValue`.
5. Capture screenshots with `xcrun simctl io <device> screenshot ...` after each meaningful step.
6. Re-read the accessibility tree after interactions to verify navigation/state changes.

## Step 1: Find the device and app

Typical commands:
```bash
xcrun simctl list devices booted
xcrun simctl listapps booted | grep -iE 'neobiz|bizplus|<app-name>'
```

If no device is booted:
```bash
xcrun simctl boot <UDID>
open -a Simulator
```

Launch the app:
```bash
xcrun simctl launch <UDID> <bundle_id>
```

## Step 2: Inspect the accessibility tree

Use a small Swift script with `AppKit` + `ApplicationServices` to read the Simulator process tree.

Useful attributes:
- `kAXWindowsAttribute`
- `kAXChildrenAttribute`
- `kAXRoleAttribute`
- `kAXTitleAttribute`
- `kAXDescriptionAttribute`
- `kAXSubroleAttribute`
- `kAXValueAttribute`

Look for these common roles:
- `AXButton`
- `AXTextField`
- `AXStaticText`
- `AXImage`
- `AXGroup`

Recommended strategy:
- recursively walk the tree
- prefer `description` if `title` is empty
- treat images with meaningful descriptions as tappable if `AXPress` works
- record frame values if you need spatial debugging

## Step 3: Tap and type through AX

Press a control:
```swift
AXUIElementPerformAction(element, kAXPressAction as CFString)
```

Fill a text field:
```swift
AXUIElementSetAttributeValue(element, kAXValueAttribute as CFString, "test value" as CFTypeRef)
```

If the field refuses the value, try:
- focusing it first via `AXPress`
- checking whether it is actually a secure field or custom control
- using a different accessibility node in the subtree

## Step 4: Re-check after every interaction

After each tap or input:
- capture a screenshot with `simctl io screenshot`
- re-read the AX tree
- note any new screen title, buttons, errors, or unexpected stalling

This is the best replacement when browser vision fails.

## Step 5: Common debugging patterns

### A. App opens but no visible UI labels
- inspect `AXGroup` children under the content area
- look for `AXImage` or `AXGenericElement` nodes with meaningful `description`
- many custom Flutter/React Native apps expose text via `description` rather than `title`

### B. Button appears as an image
- try pressing the node anyway if it has a meaningful `description`
- some custom components expose tappable images or generic elements

### C. Text fields are unnamed
- identify them by order and frame position
- use the login form pattern: first `AXTextField` = username, second = password

### D. No response after press
- re-dump the tree to verify state change
- wait a few seconds and capture another screenshot
- check whether a modal or navigation occurred offscreen

### E. Vision tooling fails
- fall back to `simctl` + AX tree + screenshots
- do not block on image AI analysis if the simulator is accessible through AX

## Practical helper flow

1. `xcrun simctl listapps booted | grep -i <app>`
2. `xcrun simctl launch <udid> <bundle_id>`
3. Swift AX script: list all buttons/text fields
4. Tap the target control(s)
5. Screenshot the result
6. Repeat for the next screen

## Notes from field use

- The Simulator window itself can be queried through Accessibility as an `AXWindow`.
- The content area is usually nested under an `AXGroup` with subrole `iOSContentGroup`.
- In some apps, a login screen may expose:
  - two unnamed `AXTextField` controls
  - a login button labeled via `description`
  - link-like `AXStaticText` nodes that still respond to `AXPress`
- If `browser_vision` or other image analysis fails with a connection error, screenshots can still be collected with `simctl` and inspected separately.

## Verification checklist

- app is booted and launched
- bundle identifier is confirmed
- AX tree shows the current screen
- buttons are pressed successfully
- text fields accept values
- a screenshot is captured after each major step
- navigation or state change is verified from the updated tree

## Pitfalls

- Don’t rely only on screenshot OCR; AX tree is usually more reliable.
- Don’t assume an element must be an `AXButton` to be tappable; some tappable custom controls show up as `AXImage` or `AXGenericElement`.
- Don’t stop after a press succeeds; confirm the screen actually changed.
- Don’t hardcode coordinates unless AX access fails.
- Don’t forget to run on the correct booted device/UDID; multiple simulators can be booted.
