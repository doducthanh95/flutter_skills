---
name: copilot-vscode-observer
version: 1.0.0
tags: [vscode, copilot, observability, automation]
description: |
  Capture and monitor GitHub Copilot / Copilot Chat activity in VS Code by tailing VS Code log files and filtering Copilot-related messages. Useful for debugging extension behaviour, token/network errors, and collecting reproducible traces of Copilot suggestions and token fetches.
---

# Copilot VSCode Observer

## When to use
- You want to record Copilot/extension activity from VS Code for post-mortem analysis.
- Investigating Copilot token errors, network failures, or extension host crashes.
- Need an automated background logger that filters and rotates logs to avoid huge files.

## What this skill does
- Locates VS Code logs under ~/Library/Application Support/Code/logs
- Collects recent snippets from matching files (renderer.log, network.log, GitHub.copilot-chat logs, any file with 'copilot' in name)
- Starts a background tail -F that filters for Copilot-related keywords and appends timestamped lines into ~/copilot_observer/copilot_activity.log
- Rotates the log when it exceeds 20 MB

## Script (installed at ~/copilot_observer/monitor_copilot_logs.sh)
- The skill ships a single bash script that:
  1. Creates ~/copilot_observer and copilot_activity.log
  2. Finds and appends recent content from matching logs (tail -n 500)
  3. Starts tail -F on matching files, filtering lines with regex /copilot|copilot-chat|copilotcli|copilot_internal|Copilot|ccreq|copilotmd|copilot_token/
  4. Adds timestamps and source file path to each recorded line
  5. Rotates the output log at 20MB

## Usage
1. Install the script (copy to ~/copilot_observer/monitor_copilot_logs.sh) and make it executable:
   chmod +x ~/copilot_observer/monitor_copilot_logs.sh
2. Start it in background:
   bash ~/copilot_observer/monitor_copilot_logs.sh &
3. Verify it's running:
   pgrep -fl monitor_copilot_logs.sh
4. View the collected logs:
   tail -n 200 ~/copilot_observer/copilot_activity.log

## Verification
- After starting, the log should contain recent renderer/network/copilot-chat lines and then live entries as VS Code writes logs.
- Look for entries about token fetches, MCP server starts, extension host restarts, or Copilot chat activations.

## Pitfalls and notes
- This approach reads VS Code logs only (client-side). It does not capture the actual suggestions shown inline in editors unless the extension writes them to logs.
- For capturing inline completions, additional integration with the VS Code extension API or an editor-side plugin is required (not covered here).
- Log files may be large; rotation is important. Adjust rotation threshold if needed.
- On non-macOS systems the VS Code log path differs — adapt the script accordingly.

## Update guidance
- If the Copilot extension changes log filenames or keyword patterns, update the grep regex and file match list.
- If you want JSON structured logs, wrap the output in a JSON object per line (timestamp, file, message).

## Files created by this skill
- ~/copilot_observer/monitor_copilot_logs.sh (script)
- ~/copilot_observer/copilot_activity.log (collected output)

