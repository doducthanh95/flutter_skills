---
name: pencil-macos-automation
description: Automate the Pencil macOS app with AppleScript/System Events and JXA for UI inspection, window management, menu actions, and import-based mockup workflows.
version: 1.0.0
author: Hermes Agent
license: MIT
metadata:
  hermes:
    tags: [macos, applescript, jxa, automation, pencil, ui-automation]
    related_skills: [telegram-desktop-macos-automation, ios-simulator-automation-qa]
---

# Pencil macOS Automation

Use this skill when you need to control the Pencil.app desktop app on macOS to build wireframes/mockups, import SVG mockups, or inspect/manipulate the UI via Accessibility scripting.

## What this skill covers
- Detecting whether Pencil is running.
- Bringing Pencil to the front and normalizing window state.
- Inspecting menus, windows, and accessibility hierarchy.
- Clicking menu items like New File, Import Image/SVG/Figma…, Show/Hide UI.
- Using SVG/mockup import as a reliable alternative to manual drag-and-drop.
- Recovering from common macOS automation pitfalls like hidden/offscreen windows.

## Workflow

### 1) Confirm Pencil is running
Use shell to check the process:
```bash
pgrep -fl 'Pencil|pencil'
```
If needed, launch it:
```bash
open -a Pencil
```

### 2) Bring Pencil to the front and normalize the window
Use AppleScript:
```applescript
tell application "Pencil" to activate
tell application "System Events"
  tell process "Pencil"
    set frontmost to true
    set position of window 1 to {0, 40}
    set size of window 1 to {1440, 960}
  end tell
end tell
```

### 3) Inspect the UI before clicking
Prefer menu inspection first:
```applescript
tell application "System Events"
  tell process "Pencil"
    return name of every menu bar item of menu bar 1
  end tell
end tell
```
This is useful to verify menu names such as:
- File
- Edit
- View
- Window
- Help

To enumerate File menu items:
```applescript
tell application "System Events"
  tell process "Pencil"
    repeat with mi in menu items of menu 1 of menu bar item "File" of menu bar 1
      log name of mi
    end repeat
  end tell
end tell
```

### 4) Prefer import-based workflow for layout creation
For reproducible mockups, create an SVG/PNG first and import it into Pencil instead of trying to drag every primitive by hand.
Typical flow:
1. Generate a mockup SVG locally.
2. Open it.
3. In Pencil, use File → Import Image/SVG/Figma…

Example AppleScript to open the import dialog:
```applescript
tell application "System Events"
  tell process "Pencil"
    click menu item "Import Image/SVG/Figma…" of menu 1 of menu bar item "File" of menu bar 1
  end tell
end tell
```

### 5) If the UI looks “missing”, check for hidden/offscreen windows
Pencil may keep a second window offscreen or show a hidden editor shell. Inspect all windows:
```applescript
tell application "System Events"
  tell process "Pencil"
    repeat with w in windows
      log (name of w) & " pos=" & (position of w as text) & " size=" & (size of w as text)
    end repeat
  end tell
end tell
```
If you see an offscreen window like a large canvas at negative coordinates, leave it alone and work with the visible standard window.

### 6) Use JXA when AppleScript type coercion gets annoying
JXA is helpful for structured inspection of accessibility properties:
```bash
osascript -l JavaScript <<'JXA'
var se = Application('System Events');
var p = se.processes['Pencil'];
console.log(JSON.stringify(p.windows().map(function(w){
  return {name:w.name(), pos:w.position(), size:w.size(), role:w.role(), subrole:w.subrole()};
}), null, 2));
JXA
```
Use this when you need reliable introspection of buttons, groups, and window geometry.

## Practical findings
- Pencil exposes standard macOS accessibility windows and menu bar items.
- File menu includes useful actions such as:
  - New File
  - Open…
  - Import Image/SVG/Figma…
  - Export Selection to…
- View menu includes Show/Hide UI, which can help if the interface is cluttered.
- The app may create multiple windows, including one that is partly offscreen; verify window coordinates before interacting.
- The standard window often has minimal visible controls via accessibility, so menu-driven automation is more reliable than trying to drag from a deeply nested canvas tree.

## Recommended pattern for login mockups
For a login screen, the most robust workflow is:
1. Generate the login screen as SVG.
2. Import the SVG into Pencil.
3. Use Pencil only for minor adjustments, annotations, or arranging components.

This is usually faster and more reliable than manually dragging every shape from the library.

## Pitfalls
- macOS Accessibility permissions may be required for System Events scripting.
- Menu item names must match exactly, including punctuation and ellipsis characters.
- Some AppleScript properties can return `missing value`; use `try/on error` or JXA when needed.
- Clicking UI elements too early can fail if Pencil is still rendering; add short delays after launching or opening dialogs.
- Offscreen windows can confuse automation; inspect `position` and `size` before interacting.
- Pencil window geometry can serialize oddly in AppleScript output (for example `pos=4040 | size=1440960`); when that happens, use JXA to read/set `position` and `size` instead of trusting the AppleScript text form.
- If the Pencil project or import dialog does not appear, normalize the main window with JXA first, then retry the menu action.

## Recovery snippet when the canvas appears invisible or offscreen
Use JXA to bring the standard window into view:
```bash
osascript -l JavaScript <<'JXA'
var se = Application('System Events');
var p = se.processes['Pencil'];
p.frontmost = true;
if (p.windows().length > 0) {
  var w = p.windows()[0];
  w.position = [80, 80];
  w.size = [1440, 960];
}
console.log(JSON.stringify(p.windows().map(function(w){
  return {name:w.name(), position:w.position(), size:w.size(), subrole:w.subrole()};
}), null, 2));
JXA
```

## Verification
After each major step, verify with a quick check:
- `pgrep -fl Pencil`
- menu bar enumeration
- window list with positions and sizes
- screenshot/manual visual confirmation when necessary
- JXA window dump if the UI seems missing

## Example: open Pencil and show the import dialog
```bash
open -a Pencil
osascript <<'APPLESCRIPT'
tell application "Pencil" to activate
tell application "System Events"
  tell process "Pencil"
    set frontmost to true
    click menu item "Import Image/SVG/Figma…" of menu 1 of menu bar item "File" of menu bar 1
  end tell
end tell
APPLESCRIPT
```

---
This skill encodes the reusable approach discovered while automating Pencil: inspect the live app first, normalize windows, prefer menu-driven actions, and use SVG import for fast mockup creation.
