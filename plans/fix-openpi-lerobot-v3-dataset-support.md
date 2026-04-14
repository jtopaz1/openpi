## Plan: Upgrade lerobot to support v3.0 dataset format

**TL;DR**: Training fails with `ForwardCompatibilityError` when loading datasets in v3.0 format (e.g. `AgentAppStore/Test_Data_Ex`) because the pinned lerobot (commit `0878c68`, CODEBASE_VERSION v2.1) only supports up to v2.1. Fix by upgrading lerobot to v0.4.4 (CODEBASE_VERSION v3.0) and adapting the tasks API change.

### Error

```
lerobot.datasets.backward_compatibility.ForwardCompatibilityError:
The dataset you requested (AgentAppStore/Test_Data_Ex) is only available in 3.0 format.
As we cannot ensure forward compatibility with it, please update your current version of lerobot.
```

### Root cause

- openpi pinned lerobot at commit `0878c68` which is v2.1 (`CODEBASE_VERSION = "v2.1"`)
- The dataset `AgentAppStore/Test_Data_Ex` was created with lerobot v3.0 format
- lerobot v2.1 raises `ForwardCompatibilityError` when it encounters a dataset version higher than its own

### Version selection rationale

| lerobot version | CODEBASE_VERSION | huggingface-hub | Compatible with transformers==4.53.2? |
|---|---|---|---|
| 0.3.4 (old pin) | v2.1 | any | Yes — but can't read v3.0 datasets |
| 0.4.0–0.4.4 | v3.0 | `<0.36.0,>=0.34.2` | **Yes** |
| 0.5.0–0.5.1 | v3.0 | `>=1.0.0,<2.0.0` | **No** — transformers==4.53.2 requires `huggingface-hub<1.0` |

**v0.4.4** was chosen because it supports v3.0 datasets while remaining compatible with the existing `transformers==4.53.2` pin.

### Changes

#### 1. `pyproject.toml`

- **lerobot rev**: `0878c68` → `8fff0fde` (v0.4.4 tag)
- **numpy**: Widened from `<2.0.0` to `<3.0.0` — required because lerobot v0.4.x depends on `rerun-sdk` which requires `numpy>=2` on Python ≥3.13
- **override-dependencies**: Added `numpy>=1.22.4,<3.0.0` to resolve the conflict between `rerun-sdk` (needs `numpy>=2`) and `tensorflow-cpu==2.15.0` in the `rlds` dependency group (needs `numpy<2`)

#### 2. `packages/openpi-client/pyproject.toml`

- **numpy**: Widened from `<2.0.0` to `<3.0.0` to match the main project

#### 3. `src/openpi/training/data_loader.py`

- **tasks API change**: In lerobot v0.4.x, `LeRobotDatasetMetadata.tasks` changed from `dict[int, str]` to `pd.DataFrame` (index=task name, column `task_index`=int). Added conversion before passing to `PromptFromLeRobotTask`:
  ```python
  tasks = {int(row.task_index): task for task, row in dataset_meta.tasks.iterrows()}
  ```

#### 4. `uv.lock`

- Regenerated to reflect all dependency changes

### Import path note

The import `lerobot.datasets.lerobot_dataset` used in `data_loader.py` is the correct path for both the old pin (v2.1) and v0.4.4. No import path change was needed for core code.

Example scripts under `examples/` use the old `lerobot.common.datasets.*` paths which no longer exist, but these are standalone scripts outside the training pipeline and were not changed.

### Verification

1. `uv lock` resolves successfully
2. `uv run python -c "import lerobot.datasets.lerobot_dataset as lds; assert lds.CODEBASE_VERSION == 'v3.0'"` passes
3. `uv run python -m pytest src/openpi/transforms_test.py::test_extract_prompt_from_task` passes
4. Tasks DataFrame→dict conversion produces correct output verified with inline test
