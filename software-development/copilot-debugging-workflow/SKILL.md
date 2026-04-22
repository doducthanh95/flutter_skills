---
name: copilot-debugging-workflow
description: Triage GitHub Copilot / Copilot for Xcode / IDE Copilot failures by classifying log signatures first, then applying the right recovery path before touching code.
version: 1.0.0
author: Hermes Agent
license: MIT
metadata:
  hermes:
    tags: [copilot, debugging, logs, ide, authentication, network, lsp]
    related_skills: [systematic-debugging, subagent-driven-development]
---

# Copilot Debugging Workflow

## When to Use

Use this skill when Copilot, Copilot Chat, or IDE inline completion is broken, flaky, slow, or producing transport/auth errors.

Typical targets:
- GitHub Copilot for Xcode
- GitHub Copilot in VS Code / JetBrains / Android Studio
- Copilot CLI / chat integrations
- Local Copilot observer logs

## Core Principle

Classify the failure mode from logs before changing code.

Copilot problems are often one of these:
1. Request cancellation / superseded request
2. Auth / token / credential failure
3. Network transport failure
4. Extension or language-server state corruption
5. Actual product bug in completion or chat logic

Do not treat all of them as the same issue.

## Fast Triage Order

### 1) Read the exact log line

Look for the first real error, not the cascade after it.

Common signatures from the field:
- `Request was superseded by a new request` → usually benign cancellation from rapid edits, cursor movement, or another completion request taking over.
- `ERR_HTTP2_STREAM_CANCEL`, `read ECONNRESET`, `The pending stream has been canceled` → transport/network/session interruption.
- `either GITHUB_PAT, GITHUB_OAUTH_TOKEN, or GITHUB_OAUTH_TOKEN+VSCODE_COPILOT_CHAT_TOKEN must be set` → missing credentials in the environment.
- `Max retry for getting suggestions reached` → downstream symptom; inspect the earlier auth/network error.

### 2) Separate symptom from cause

If you see repeated inlineCompletion failures, ask:
- Is the editor generating requests too fast?
- Is auth missing or expired?
- Is the network unstable?
- Did the extension restart or lose its stream?

### 3) Verify environment first

For auth-related issues, check:
- GitHub sign-in state
- Required tokens in the runtime environment
- Whether the process is running under a test harness or automation context
- Whether the token provider can actually reach GitHub

### 4) Only then inspect the product/code path

If logs point to a stable transport/auth problem, fix that first.
If logs point to a real code bug, then switch to normal systematic debugging.

## Recovery Playbook

### Case A: `Request was superseded by a new request`

Likely cause:
- Rapid typing
- Cursor movement
- Another completion request replacing the previous one

Action:
- Reproduce with slower input
- Check whether the issue disappears when requests are less frequent
- Usually not a code bug unless it happens constantly under normal use

### Case B: `ERR_HTTP2_STREAM_CANCEL` / `read ECONNRESET`

Likely cause:
- Network reset
- Broken HTTP/2 stream
- Auth/session fetch interruption

Action:
- Re-check connectivity
- Re-authenticate
- Restart the extension or language server
- Verify no proxy/VPN/firewall interference
- Reproduce outside the current session if possible

### Case C: Missing GitHub token / auth env vars

Likely cause:
- Tooling launched without required credentials
- Test environment not set up for Copilot auth

Action:
- Confirm the expected auth variables or login flow
- For automation/test runners, ensure the documented token path is available
- Do not keep retrying completion; retries will just repeat the same auth failure

## Copilot Log Reading Heuristics

When logs are verbose:
- Find the first `error` or `unhandledRejection`
- Search upward for the first related `info` or `window/logMessage`
- Ignore repeated retries unless they change the error code/message
- The earliest root error usually matters more than the final "max retry" line

If multiple errors repeat:
- Group them by signature
- Count how many are cancellation vs auth vs transport
- Treat repeated same-signature errors as one root issue, not many independent bugs

## Minimal Verification Checklist

Before declaring victory:
- The original error signature no longer appears
- The extension re-authenticates cleanly, if auth was the issue
- Inline completion or chat actually returns results
- The problem stays fixed after a restart

## Pitfalls

- Chasing `Max retry` instead of the original auth/network failure
- Treating request-cancellation as a bug in completion logic
- Editing code when the environment is simply unauthenticated
- Assuming network errors are product bugs
- Forgetting that the IDE may issue overlapping requests by design

## Best Practice Summary

1. Read the exact log signature.
2. Classify the failure: cancellation, auth, network, or real code bug.
3. Fix the environment first when the log points there.
4. Use systematic debugging only after the failure is correctly classified.
5. Re-verify with a clean restart and a fresh request.

## Related Tools

- `Library/Logs/GitHubCopilot/*` for raw Copilot logs
- Copilot observer scripts for local monitoring and summarization
- `systematic-debugging` for non-Copilot bugs

## Practical Rule

If the log says the request was canceled or the stream was reset, do not jump to code changes.
First prove whether the problem is auth, transport, or request churn.
