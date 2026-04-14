# Plan: Fix openpi task format compatibility

## TL;DR
Fix `create_torch_dataset()` in openpi's `data_loader.py` to handle both old-format (task as column) and v3-format (task as index) `tasks.parquet` files. Currently only v3 is handled, causing `AttributeError: 'int' object has no attribute 'item'` when training pi0.5 on old-format datasets.

## Root Cause
In `data_loader.py` line 151, the task dict comprehension assumes task strings are the DataFrame index (v3 format). When the DataFrame has a numeric index (old format), the dict maps ints to ints instead of ints to strings. This int "prompt" eventually reaches `TokenizePrompt` which calls `.item()` on it.

## Steps

### Phase 1: Fix task format detection (1 file) — DONE

1. Extracted a `parse_tasks(tasks: pd.DataFrame) -> dict[int, str]` helper in `src/openpi/training/data_loader.py` that detects the DataFrame format:
   - If `"task"` is a column (old format): `{int(row["task_index"]): row["task"] for _, row in tasks.iterrows()}`
   - Otherwise (v3 format, task strings as index): `{int(row.task_index): task for task, row in tasks.iterrows()}`
   - `create_torch_dataset()` now calls `parse_tasks(dataset_meta.tasks)` instead of inlining the comprehension.

### Phase 2: Add test coverage (1 file) — DONE

2. Existing `test_extract_prompt_from_task()` in `src/openpi/transforms_test.py` already tests the transform itself. No change needed.

3. Added two unit tests in `src/openpi/training/data_loader_test.py`:
   - `test_parse_tasks_v3_format`: v3 DataFrame with task strings as index
   - `test_parse_tasks_old_format`: old DataFrame with `task` column and numeric index
   - Both assert the result is `{0: "pick up cup", 1: "put down cup"}`

## Relevant files
- `src/openpi/training/data_loader.py` — lines 148-152 in `create_torch_dataset()`: the only place with the v3-index assumption
- `src/openpi/gibbonbot/compute_norm_stats.py` — line 30: indirectly affected (calls `create_torch_dataset`), no changes needed
- `src/openpi/transforms.py` — line 310: `PromptFromLeRobotTask` class, no changes needed (consumes dict correctly)
- `src/openpi/transforms_test.py` — line 109: existing transform test, no changes needed
- `src/openpi/training/data_loader_test.py` — add test for task dict construction

## Verification — DONE
1. Ran `test_extract_prompt_from_task` — passed
2. Ran `test_parse_tasks_v3_format` and `test_parse_tasks_old_format` — both passed
3. Ran `test_torch_data_loader` — passed (no regressions)

## Decisions
- Fix goes in openpi (not gibbonbot), since openpi is the consumer and should be robust to both formats
- No changes to `PromptFromLeRobotTask` transform itself — it already correctly expects `dict[int, str]`
- `compute_norm_stats.py` benefits automatically since it delegates to `create_torch_dataset()`
