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

### `docs/` directory collision

**When:** After merging docs repo  
**Symptom:** Both sdk-typescript and the docs repo want to use `docs/` at root  
**Why:** sdk-typescript had a `docs/` directory with dev docs (DEPENDENCIES.md, TESTING.md, etc.). The original plan wanted `docs/` as a symlink to `site/src/content/docs`.  
**Resolution:** Renamed TS dev docs to `dev-docs/`. Skipped the symlink for now — `site/src/content/docs` is the canonical path.

### Docs repo merge: conflict-free

**When:** Merging docs repo with `--allow-unrelated-histories`  
**Result:** Zero conflicts. `designs/` landed at top level naturally (it wasn't moved into `site/`). `site/` contains the full Astro site.  
**Note:** The docs repo is 98MB (images/assets) — the largest of the four repos.

### MCP server hatch default env missing pytest

**When:** MCP CI test job  
**Symptom:** `pytest: not found` (exit code 127)  
**Why:** The `[tool.hatch.envs.default.scripts]` section defines a `test` script that runs `pytest`, but the default environment didn't declare `features = ["dev"]` to pull in the dev optional dependencies (which include pytest).  
**Fix:** Add `[tool.hatch.envs.default]` with `features = ["dev"]` to `strands-mcp/pyproject.toml`.  
**Impact on real migration:** This is a pre-existing issue in the mcp-server repo — their CI probably used a different env or installed deps differently. Needs fixing regardless.

### MCP tag collisions (silent loss)

**When:** Fetching mcp-server tags into the monorepo  
**Symptom:** MCP's `v0.1.x` and `v0.2.x` tags were silently skipped because Python SDK already has tags with those names  
**Why:** Git won't overwrite existing tags during fetch  
**Fix:** Manually created `mcp/v*` namespaced tags using commit SHAs from the standalone clone. 2 of 11 tags pointed to commits not in main's history (likely squash-merged PRs) and couldn't be created.  
**Impact on real migration:** Need the tag-mapping script from the plan. Also need to decide if orphaned tag commits matter (they reference work that was squash-merged, so the code is there, just not at that exact SHA).

### Path filter doesn't self-trigger on workflow file changes

**When:** Pushed a change to `.github/workflows/python-ci.yml` only  
**Symptom:** Python CI didn't run  
**Why:** Path filter `strands-py/**` correctly excluded `.github/workflows/` changes  
**Impact on real migration:** Workflow-only PRs won't trigger the associated CI. This is arguably correct behavior, but worth documenting. Could add `.github/workflows/python-*` to the Python CI paths filter if we want workflow changes to self-test.

---

## What Worked Well

- `git merge --allow-unrelated-histories` cleanly combined histories — only `.gitignore` conflict across 3 merges (Python, TypeScript, docs)
- TypeScript's existing subdirectory structure (`strands-ts/`, `strands-wasm/`, etc.) needed minimal changes
- Path-based CI triggers correctly limited which workflows ran (Python CI didn't trigger for TS-only paths)
- History is fully preserved and filterable: `git log -- strands-ts/` shows only TS commits
- Docs merge was completely conflict-free despite 574 files

---

## Open Questions

1. ~~**setuptools-scm fix strategy**~~ — **RESOLVED**: Use `root = ".."` + `tag_regex` + custom `git_describe_command`. Keeps VCS automation, works locally and in CI.
2. ~~**Python tag prefix going forward**~~ — **RESOLVED**: `tag_regex = "^python/v(?P<version>.+)$"` combined with `git_describe_command --match python/v*` handles the namespaced tags correctly.
3. ~~**Shared dependencies**~~ — **RESOLVED**: The root `pyproject.toml` is intentionally a monorepo dev-tools config (ruff, pyright, pytest). It's not published and already has monorepo-aware paths (`strands-py/src/**`). No conflict with `strands-py/pyproject.toml`.
4. ~~**griffe API compat check**~~ — **RESOLVED**: Run griffe from repo root with `--search strands-py/src` instead of using `working-directory`. Both current and `--against "main"` baseline resolve the same path.
5. ~~**Site build CI**~~ — **RESOLVED**: The docs repo gitignored `package-lock.json`, so `npm ci` fails. Fix: use `npm install` for now, un-gitignore the lock file so it can be committed. For the real migration, generate and commit `site/package-lock.json` for reproducible builds. The site builds and tests independently of the root workspace `package.json`.
6. **`docs/` directory collision** — sdk-typescript brought a `docs/` dir with dev docs. Renamed to `dev-docs/` to avoid conflict.
