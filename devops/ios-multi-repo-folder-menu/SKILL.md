---
name: ios-multi-repo-folder-menu
description: Build and maintain bash menu scripts for managing many Git repositories inside an iOS workspace folder on macOS, including branch checkout across repos and cloning repos into a blank folder from a manifest.
version: 1.0.0
author: Hermes Agent
license: MIT
metadata:
  hermes:
    tags: [bash, git, macOS, iOS, multi-repo, menu, clone, checkout]
---

# iOS Multi-Repo Folder Menu

Use this skill when the user wants a Bash menu script to manage many Git repos inside an iOS workspace folder on macOS.

Typical use cases:
- checkout a branch across all repos in a folder
- clone all repositories into a blank folder
- work on macOS default Bash 3.2
- avoid false positives from build artifacts or nested `.git` directories

## Core approach

1. Discover repo layout first
   - Inspect the workspace folder structure before coding assumptions.
   - In many iOS monorepos, the real git repos are immediate child directories like `account/`, `payment/`, `core/`.
   - Avoid recursively treating everything with `.git` content as a repo if build artifacts contain copied git metadata.
   - In the user’s IOS workspace, the source of truth is the script’s sibling folder tree; clone manifests can be generated automatically from those repos’ `origin` URLs.

2. Detect repos robustly
   - Use `git -C "$dir" rev-parse --is-inside-work-tree` to verify a repo.
   - Prefer scanning immediate child directories (`find folder -mindepth 1 -maxdepth 1 -type d`).
   - If the selected folder itself may be a repo, include it too.
   - For this workspace, top-level Git repos were confirmed under the IOS folder and excluded build/XCFramework artifacts.

3. Stay compatible with macOS Bash 3.2
   - Do not use `mapfile`.
   - Do not rely on associative arrays.
   - Prefer `while IFS= read -r ...` loops and simple arrays.
   - Avoid assuming newer Bash features when building interactive scripts for macOS.

4. Checkout branches safely
   - Check for local branch refs with `git show-ref --verify --quiet "refs/heads/$branch"`.
   - If not local, fetch and check `refs/remotes/origin/$branch`.
   - Use `git -C "$repo" ...` or `(cd "$repo" && git ...)` to avoid `pushd/popd` overhead.
   - `git rev-parse --verify` was less reliable than `show-ref` for this task in practice.

5. Clone many repos into a blank folder
   - Prefer auto-generating `repos.txt` from the source IOS folder by reading each repo’s `origin` remote URL.
   - If you do use a manifest, support lines like:
     - `https://host/org/repo.git`
     - `repo-name https://host/org/repo.git`
   - Before cloning, validate that the destination folder is empty.
   - Derive destination name from the explicit repo name or from the basename of the URL.
   - For this workspace, preserving the source repo names from the IOS tree was important.

6. Parallel clone + post-clone branch selection
   - Clone jobs can be run in parallel with a configurable concurrency limit (`CLONE_JOBS`).
   - A practical default is 4; allow the user to raise/lower it.
   - When cloning, prefer `git clone --branch dev --single-branch` if remote `dev` exists.
   - Otherwise clone normally and run `git -C "$dest" checkout dev` afterward.
   - Keep per-repo logs so concurrent output can be replayed cleanly.

7. User-facing polish
   - Colored logs and emoji/icons are helpful for menu scripts that operate on many repos.
   - Keep status lines short and consistent: queued, cloning, ok, fail, skip, warning.

## Recommended script structure

### Menu loop
- `print_header`
- `prompt_folder`
- `prompt_blank_folder`
- `checkout_branch_all_repos`
- `clone_all_repos`
- `main`

### Helper functions
- `is_git_repo(dir)`
- `list_git_repos(folder)`
- `run_git(repo, args...)`

## Suggested repo listing logic

```bash
is_git_repo() {
  git -C "$1" rev-parse --is-inside-work-tree >/dev/null 2>&1
}

list_git_repos() {
  local folder="$1" dir
  if is_git_repo "$folder"; then
    printf '%s\n' "$folder"
  fi
  while IFS= read -r -d '' dir; do
    is_git_repo "$dir" && printf '%s\n' "$dir"
  done < <(find "$folder" -mindepth 1 -maxdepth 1 -type d -print0 2>/dev/null)
}
```

## Suggested branch checkout logic

```bash
if git -C "$repo" show-ref --verify --quiet "refs/heads/$branch"; then
  git -C "$repo" checkout "$branch"
else
  git -C "$repo" fetch --all --prune
  if git -C "$repo" show-ref --verify --quiet "refs/remotes/origin/$branch"; then
    git -C "$repo" checkout -b "$branch" --track "origin/$branch"
  fi
fi
```

## Suggested clone-from-manifest logic

- Read manifest line by line.
- Strip comments and CRLF.
- Skip blank lines.
- Parse either:
  - `repo-name URL`
  - `URL`
- Check destination existence before cloning.
- Clone with `git clone "$url" "$dest"`.

## Pitfalls learned

- macOS default Bash 3.2 does not have `mapfile`.
- `git rev-parse --verify "$branch"` can be misleading for branch existence checks; `show-ref` is more reliable for this use case.
- A folder can contain many non-repo directories such as `build/`, `.DS_Store`, XCFramework artifacts, and archives — do not treat them as Git repos.
- Some repos may have missing `.git` directories if the tree is partially copied; always verify with `git -C`.
- Clone functionality needs a manifest or source mapping; the script cannot infer repository URLs from an empty folder alone.

## Verification checklist

Before delivering the script:
- `bash -n script.sh`
- Test in a temp folder with fake repos
- Test against the real workspace structure
- Confirm the menu runs on macOS Bash 3.2
- Confirm branch checkout works for both local and origin-tracking branches
- Confirm blank-folder validation blocks non-empty targets

## When to reuse this skill

Reuse this skill when the user asks for:
- a bash menu for multi-repo Git management
- checkout/pull/status across many iOS repos
- cloning many repos into a clean workspace directory
- shell scripts that must remain compatible with macOS default Bash
