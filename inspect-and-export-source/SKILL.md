---
name: inspect-and-export-source
description: "Procedural skill to discover, preview, and export source files from a repository while excluding large binaries, build artifacts and generated frameworks. Useful for code review, quick audits, and safe content export."
version: 1.0.0
author: Hermes Agent
license: MIT
metadata:
  hermes:
    tags: [code-inspection, export, repo, tar, git, ripgrep]
    related_skills: [codebase-inspection]
prerequisites:
  commands: [git, tar, sed, rg (ripgrep) optional]
---

# Inspect & Export Source (exclude binaries/assets)

This skill captures a robust, reproducible workflow to list, preview, and export source files from a repository while avoiding large binary/artifact directories (xcframeworks, AARs, .so, .jar, images, .git internals). It is tuned for interactive agent use where printing all files to the console must be limited.

## When to use
- You need the project source (code files only) for review, analysis, or to produce a small archive to share.
- You must avoid large binaries and build outputs that bloat exports or cause timeouts.
- You want a safe preview (first N lines) rather than streaming whole repo to console.

## Steps
1. Find tracked source files (preferred):
   - Use `git ls-files` with file patterns to get only tracked files of code extensions.
     Example:
     ```bash
     git ls-files '*.dart' '*.kt' '*.java' '*.swift' '*.xml' '*.gradle' '*.g.dart' > /tmp/source_list.txt
     wc -l /tmp/source_list.txt
     ```
   - Why git? It avoids untracked build outputs and local caches.

2. If the current folder is not a git repository or the target is a subfolder/module, fall back to a local recursive scan:
   - Prefer a script (Python, Ruby, Perl) or `rg --files` to collect source files from the exact module root.
   - Exclude build artifacts and binaries explicitly:
     - `build/`, `.git/`, `Pods/`, `.gradle/`, `*.xcframework`, `*.framework`, `*.so`, `*.jar`, `*.aar`, images, and dSYMs.
   - Example with Python when you need precise control:
     ```bash
     python3 - <<'PY'
     from pathlib import Path
     root = Path('/path/to/module')
     exts = {'.swift', '.m', '.mm', '.h', '.dart', '.kt', '.java', '.xml', '.gradle'}
     files = []
     for p in root.rglob('*'):
         s = str(p)
         if any(x in s for x in ['/build/', '/.git/', '/Pods/', '.xcframework/', '.framework/']):
             continue
         if p.is_file() and (p.suffix in exts or p.name.endswith('.g.dart')):
             files.append(p)
     files.sort()
     print('\n'.join(str(p) for p in files))
     PY
     ```

3. Create an export archive of those files (tar.gz):
   ```bash
   tar --files-from=/tmp/source_list.txt -czf /tmp/source_code_export.tar.gz
   ls -lh /tmp/source_code_export.tar.gz
   ```
   - This produces a compact archive containing only the listed files.

4. Preview file list and contents safely:
   - List the files:
     ```bash
     sed -n '1,200p' /tmp/source_list.txt
     ```
   - Print headers + first N lines of each file (limit to avoid huge output):
     ```bash
     count=0
     while IFS= read -r file; do
       echo "--- FILE: $file ---"
       sed -n '1,200p' "$file" || true
       echo "\n"
       count=$((count+1))
       if [ $count -ge 200 ]; then break; fi
     done < /tmp/source_list.txt
     ```
   - Adjust `sed -n '1,200p'` and `count` limits to your needs.

5. Alternative: Use ripgrep (rg) when `git` is not suitable:
   ```bash
   rg --files --hidden --glob '!.git' -g '!**/*.png' -g '!**/*.so' -g '!**/*.jar' -g '!**/*.aar' -g '!**/*.xcframework' -g "'**/*.dart'" -g "'**/*.kt'" -g "'**/*.java'" -g "'**/*.swift'" -g "'**/*.xml'" -g "'**/*.gradle'" -g "'**/*.g.dart'" > /tmp/source_list.txt
   ```
   - Be careful with `rg` globs; exclude folders like node_modules, .gradle, build, ios/Pods, android/.gradle, packages/*/build.

## Verification
- Confirm archive size and file count:
  ```bash
  wc -l /tmp/source_list.txt
  ls -lh /tmp/source_code_export.tar.gz
  ```
- Spot check files by opening a few with `sed -n '1,200p'`.

## Pitfalls & Recommendations
- Always exclude generated binaries and framework bundles (.xcframework, .aar, .so, .jar, images). Including them will bloat the archive and may exceed agent timeouts.
- Prefer `git ls-files` to avoid capturing build artifacts and local caches.
- When printing file contents to console, cap lines and number of files to avoid hitting timeouts or creating giant chat responses.
- If the repo path is not obvious, locate the real module first with `search_files` or a quick `terminal` scan before reading/exporting. In Flutter monorepos, the plugin source is often under `packages/<plugin>/android/src/...`, while the app-level Gradle may point to it via `../packages/<plugin>/android/local-maven-repo`.
- If repo is very large, split work by language (e.g., only `--suffix=dart` for Flutter), or export per-module.
- Watch for secrets: scan exported files for credentials and signing keys (e.g., search for `storePassword`, `keyPassword`, API keys). If found, treat as sensitive and rotate.

## Follow-ups
- Run pygount or similar tool for LOC/language breakdown (see related skill: codebase-inspection).
- Optionally, create a filtered archive that excludes specific large directories.

## Example usage (single script)
```bash
# 1. collect tracked source
git ls-files '*.dart' '*.kt' '*.java' '*.swift' '*.xml' '*.gradle' '*.g.dart' > /tmp/source_list.txt
# 2. create archive
tar --files-from=/tmp/source_list.txt -czf /tmp/source_code_export.tar.gz
# 3. preview (first 200 lines per file, first 200 files)
count=0
while IFS= read -r file; do
  echo "--- FILE: $file ---"
  sed -n '1,200p' "$file" || true
  echo "\n"
  count=$((count+1))
  if [ $count -ge 200 ]; then break; fi
done < /tmp/source_list.txt
```

---

Notes: this skill encodes the safe, pragmatic approach we used: prefer git-tracked files, generate an explicit file list, export only those files into a tar, preview in bounded chunks, and avoid printing raw binaries. Use when you need reliable, repeatable source exports for review or analysis.
