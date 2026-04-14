## Plan: Rename openpi/phospho → openpi/gibbonbot in fork

**TL;DR**: The `jtopaz1/openpi` fork at pinned commit `2c7f6eef` has the custom training module at `src/openpi/phospho/`, but `modal/pi05/app.py` imports from `openpi.gibbonbot` (renamed in commit `322e351b` without updating the fork). Fix by renaming the directory in the fork, pushing, and updating the pinned commit hash.

**Steps**

### Phase 1: Update the openpi fork (in the jtopaz1/openpi repo)

1. Clone https://github.com/jtopaz1/openpi.git and checkout commit `2c7f6eef0cf2bd2d547ccdffe8e115b846fcec42`
2. Rename directory `src/openpi/phospho/` → `src/openpi/gibbonbot/`
   - Contains 2 files: `compute_norm_stats.py`, `train.py`
   - No `__init__.py` — implicit namespace package, no extra file needed
3. Update the self-referencing comments inside both files:
   - `train.py` line comment: `# from openpi.phospho.train import train_with_config` → `# from openpi.gibbonbot.train import train_with_config`
   - `compute_norm_stats.py` line comment: `# from openpi.phospho.compute_norm_stats import compute_norm_with_config` → `# from openpi.gibbonbot.compute_norm_stats import compute_norm_with_config`
4. Verify no other files in the openpi repo reference `openpi.phospho` (grep to confirm — based on web inspection, the two comment lines above are the only references)
5. Commit and push to `jtopaz1/openpi` (e.g. branch `main` or a new branch)
6. Note the new commit SHA

### Phase 2: Update GibbonBotPlatform (this repo)

7. In `modal/pi05/app.py` line 59, update the pinned commit hash from `2c7f6eef0cf2bd2d547ccdffe8e115b846fcec42` to the new commit SHA (*depends on step 6*)
8. Optionally: clean up the commented-out clone on line 56-57 referencing `gibbonbotai/openpi` (same stale commit hash)

**Relevant files**

- **openpi fork** `src/openpi/phospho/` — rename to `src/openpi/gibbonbot/`
  - `src/openpi/phospho/compute_norm_stats.py` — update comment on line referencing `openpi.phospho`
  - `src/openpi/phospho/train.py` — update comment on line referencing `openpi.phospho`
- `modal/pi05/app.py` (this repo) — update pinned commit hash on line 59

**Verification**

1. After step 5: `git clone` the fork, checkout new commit, run `python -c "from openpi.gibbonbot.compute_norm_stats import compute_norm_with_config; from openpi.gibbonbot.train import train_with_config"` to confirm imports work (requires openpi deps installed)
2. After step 7: Redeploy pi0.5 Modal app and trigger a test training to confirm the `ModuleNotFoundError` is resolved
3. Quick sanity: `grep -r "openpi.phospho" modal/` should return no matches in this repo (it currently doesn't — the imports were already renamed)

**Decisions**

- Only the openpi fork needs directory/file changes; no import changes needed in GibbonBotPlatform (the imports at lines 494-495 already use `openpi.gibbonbot`)
- The two `.py` files have no internal imports referencing `phospho` — only doc comments need updating
- No `__init__.py` exists in the `phospho/` directory, so no extra file is needed after rename
