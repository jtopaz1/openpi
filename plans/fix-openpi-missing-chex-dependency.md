## Plan: Add `chex` dependency to openpi fork

**TL;DR**: `chex` is imported by `src/openpi/models/utils/fsq_tokenizer.py` but is not listed in `pyproject.toml` dependencies. It used to be pulled in transitively by older JAX/Flax versions but no longer is. Add it explicitly.

### Steps (in the `jtopaz1/openpi` repo)

1. Open `pyproject.toml`
2. Add `"chex>=0.1.91"` to the `dependencies` list under `[project]`
3. Commit and push
4. Note the new commit SHA

### Then (in this repo)

5. In `modal/pi05/app.py` line 59, update the pinned commit hash to the new SHA from step 4

### Context

The import chain that fails:
```
openpi.gibbonbot.compute_norm_stats
  → openpi.training.config
    → openpi.models.tokenizer
      → openpi.models.utils.fsq_tokenizer
        → import chex  ← ModuleNotFoundError
```

`chex` is used in `fsq_tokenizer.py` for assertion helpers (`chex.assert_equal_shape`, `chex.assert_shape`) — only 3 call sites, all in that one file. It's a standard DeepMind JAX testing/assertion library (latest: `0.1.91`).

### Verification

After pushing the fork update and updating the pinned hash:
1. Redeploy the pi0.5 Modal app
2. Trigger a test training — the `ModuleNotFoundError: No module named 'chex'` should be gone

### Note

The `PackageNotFoundError: No package metadata was found for 'gibbonbot'` warning in the logs is separate and non-fatal. It comes from `.add_local_python_source("gibbonbot")` which copies source files without creating `.dist-info` metadata. It only matters if something calls `importlib.metadata.version('gibbonbot')`.
