# Monorepo Convergence Notes

Running log of decisions, issues, and things we learn during this dry-run.

---

## Issues Encountered

### 1. `setuptools-scm` breaks when package is not at git root

**When:** Python CI (unit tests, lint) after push to mono-example  
**Symptom:** `LookupError: Error getting the version from source 'vcs': setuptools-scm was unable to detect version for .../strands-py`  
**Why:** The Python SDK uses hatchling + setuptools-scm for VCS-based versioning. After restructuring into `strands-py/`, the package is no longer at the git root, so setuptools-scm can't find tags to derive the version.

**Root cause (deeper):** Even with `root = ".."` set, setuptools-scm v10+ (via `vcs-versioning`) uses `git describe --match "*[0-9]*"` by default. In a monorepo with tags from multiple packages, this finds the *closest* tag regardless of package — e.g., `typescript/v1.2.0` instead of `python/v1.40.0`. When that tag doesn't match the `tag_regex`, the version parsing asserts.

**Resolution:** Three settings are needed together in `strands-py/pyproject.toml`:
```toml
[tool.hatch.version]
source = "vcs"
raw-options.root = ".."
raw-options.tag_regex = "^python/v(?P<version>.+)$"
raw-options.git_describe_command = "git describe --dirty --tags --long --abbrev=40 --match python/v*"
```

- `root = ".."` — look at git root one level up
- `tag_regex` — only match `python/v*` tags for version extraction
- `git_describe_command` — constrain `git describe` to only consider `python/v*` tags (the default `*[0-9]*` glob matches tags from all packages)

**Verified:** Locally resolves to `1.40.1.dev462` (correct distance from `python/v1.40.0`).

**Known cosmetic side-effects:**
- The `.devN` distance count includes commits from all paths (not just `strands-py/`), so it inflates between Python releases. Harmless — only affects dev installs.
- `+dirty` suffix triggers if any file in the monorepo is modified, even non-Python files.

**Impact on real migration:** This pyproject.toml change must be part of the restructure commit. It's backward-incompatible with the pre-monorepo layout (where `root = "."` and bare `v*` tags work), so it can only land at cutover time.

### 2. `griffe check --search src` path is now wrong

**When:** Python API Compatibility job  
**Symptom:** `ModuleNotFoundError: strands` — griffe can't find the module at the expected path  
**Why:** The workflow uses `working-directory: strands-py` and `--search src`, but griffe's `--against "main"` flag checks out the old main branch where `src/` was at root. The path mismatch means it can't load either the old or new version.  
**Fix options:**
- Run griffe from repo root with `--search strands-py/src`
- Adjust the `--against` comparison to use the restructured commit as baseline (not `main` from before the move)

**Impact on real migration:** The API compat check needs to be aware that the first run post-restructure will have no valid baseline. May need to skip or reset the baseline on the restructure commit.

### 3. Tag collisions between Python and TypeScript

**When:** During `git fetch` of both repos' tags  
**Symptom:** Both repos have `v0.1.0`–`v0.7.0`, `v1.0.0`, `v1.1.0`, `v1.2.0`. The first repo fetched "wins" the bare tag names; the second's are silently skipped.  
**Resolution:** Since sdk-python becomes the base repo (keeping its identity), its bare tags naturally take priority. TypeScript tags were manually created with `typescript/v*` prefix using commit SHAs from the TS clone. This must be done explicitly — you can't rely on fetch to bring in both sets.

**Impact on real migration:** Need a script that:
1. Records all TypeScript tag→commit mappings before the merge
2. Creates `typescript/v*` tags pointing to those commits after merge
3. Verifies no tags were lost

### 4. Untracked `.gitignore` blocks merge

**When:** Attempting `git merge --allow-unrelated-histories` with sdk-typescript  
**Symptom:** `error: The following untracked working tree files would be overwritten by merge: .gitignore`  
**Why:** The sandbox had an untracked `.gitignore` (from the original mono-example repo). Git refuses to overwrite untracked files during merge.  
**Resolution:** Committed the file before merging, then resolved the add/add conflict.

**Impact on real migration:** Not an issue for the real migration (sdk-python's .gitignore will already be tracked). But a reminder: ensure working tree is clean before merging.

---

## Decisions Made

### Tag naming convention
- Python bare tags (v1.0.0 – v1.40.0) stay untouched
- Python gets `python/v*` namespaced duplicates pointing to same commits
- TypeScript gets only `typescript/v*` namespaced tags (bare tags were overwritten by Python's)
- Going forward: only namespaced tags trigger releases

### `.github/` handling
- Both repos' `.github/` directories archived into their respective subdirs (`.github-archive/`)
- Fresh root-level `.github/workflows/` written with path-based triggers
- This avoids merge conflicts in workflows and gives us a clean slate

### TypeScript root workspace files
- `package.json`, `package-lock.json`, `.node-version`, `.prettierrc`, `.husky/` stay at repo root
- They naturally serve as the monorepo workspace config (the TS SDK's workspace already spans multiple subdirs)

### TypeScript CI: works out of the box
- The TS workspace structure was already monorepo-ready
- `npm ci` at root + `npm run test:all:coverage` just works
- No path adjustments needed for the TS side

---

## What Worked Well

- `git merge --allow-unrelated-histories` cleanly combined both histories with only `.gitignore` as a conflict
- TypeScript's existing subdirectory structure (`strands-ts/`, `strands-wasm/`, etc.) needed minimal changes
- Path-based CI triggers correctly limited which workflows ran (Python CI didn't trigger for TS-only paths)
- History is fully preserved and filterable: `git log -- strands-ts/` shows only TS commits

---

## Open Questions

1. ~~**setuptools-scm fix strategy**~~ — **RESOLVED**: Use `root = ".."` + `tag_regex` + custom `git_describe_command`. Keeps VCS automation, works locally and in CI.
2. ~~**Python tag prefix going forward**~~ — **RESOLVED**: `tag_regex = "^python/v(?P<version>.+)$"` combined with `git_describe_command --match python/v*` handles the namespaced tags correctly.
3. **Shared dependencies** — the TS repo has a root `pyproject.toml` (for the WASM Python projection). How does that interact with `strands-py/pyproject.toml`? Do tools get confused by two pyproject.toml files?
4. **griffe API compat check** — needs rework for monorepo paths. The `--against "main"` baseline won't have `strands-py/src/` until after cutover. May need to skip the first run or rebase the comparison target.
