# FluidFrame: do not write a checkpoint in train() when save=False

- Upstream: https://github.com/sm7610/fluidframe (MIT)
- Contribution: issue https://github.com/sm7610/fluidframe/issues/1, pull request https://github.com/sm7610/fluidframe/pull/2; status as of 2026-11-27: open
- Kind: non-trivial

## The problem

`train()` called `save_checkpoint` in its logging branch on the last episode, so it wrote `checkpoints/policy_<episode>.npy` even with `save=False`. As a result `tests/test_train.py` fails with `FileNotFoundError` unless a `checkpoints/` folder already exists. Separately, `tests/test_environment.py` cannot be collected because `environments/utils.py` is missing from the repository; the issue reports that gap.

## Reproduction

Environment: conda-forge, Python 3.13, dedalus 3.0.5, on macOS arm64, built from an `environment.yml`; upstream at commit c77f55b.

- `pytest tests/test_train.py` on the unpatched commit: fails with `FileNotFoundError`.
- `pytest tests/test_environment.py`: stops at collection with `ModuleNotFoundError: No module named 'environments.utils'` (from `environments/stratified_shear.py`, line 15).

## The change

One line removed from `train.py` (diff: https://github.com/sm7610/fluidframe/pull/2/files): the `save_checkpoint` call in the last-episode logging branch. With `save=True`, the save branch above it already writes the last-episode checkpoint, so `main.py` writes the same files as before. Removing the stray call is the smallest change that makes `save=False` mean what it says.

## Verification

- Without the change: `tests/test_train.py` fails.
- With the change: `pytest tests --ignore=tests/test_environment.py` gives 6 passed (macOS arm64, Python 3.13, dedalus 3.0.5).
- A build script checks all three states from the pinned commit: the unpatched failure, the patched pass, and that `tests/test_environment.py` still stops on `environments.utils`.

## Dead ends

- An alternative fix keeps the save and has `save_checkpoint` create `checkpoints/`. It was set aside for the one-line removal and offered in the pull request as an option.
- An `environment.yml` (conda-forge, `matplotlib-base` without a GUI stack) and a GitHub Actions workflow running the tests were prepared and held back. With `environments/utils.py` missing, CI would stay red; the issue offers them once that file lands.

## How it was made

Made with Claude Code (Anthropic), under the direction of the person who posted it. A separate review pass, run independently of the build, tried to break the change and the texts before posting.

## What this record leaves out

This record summarises how the change was made and checked; internal working notes and drafts are not published.

---
Licence: CC BY 4.0 (see LICENSE.md). Code and diffs quoted here keep their original licences. How this work is done: ../HOW.md
