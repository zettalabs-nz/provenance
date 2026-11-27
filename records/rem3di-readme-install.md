# Rem3Di: README install instructions from a checkout with uv sync

- Upstream: https://github.com/molsuit/Rem3Di (Apache-2.0)
- Contribution: issue https://github.com/molsuit/Rem3Di/issues/6, pull request https://github.com/molsuit/Rem3Di/pull/7; status as of 2026-11-27: open
- Kind: non-trivial

## The problem

The README's Setup section says `pip install remedi`, which gives the 0.1.1 release from July, before the workspace split. That release has no `remedi.embedding`, and `remedi-data` and `remedi-eval` are not on PyPI, so the documented extras cannot be installed.

## Reproduction

Against PyPI and upstream commit f42d32c:

- `pip install remedi` installs 0.1.1, which has no `remedi.embedding`.
- `pip install "remedi-eval[descriptor-analysis]"` fails: the package is not on PyPI.
- `uv build --package remedi` produces a wheel whose metadata lists a bare `mace-torch` requirement.

## The change

README only (diff: https://github.com/molsuit/Rem3Di/pull/7/files). The Setup section now installs with `uv sync` from a checkout, as CI does, with one command per setup, since `uv sync` removes packages the command does not name. A line states which platforms the lock covers: Linux x86_64; aarch64 and Windows without `cuda`; not macOS. Documenting the path that works today is the smallest correct fix until a new release reaches PyPI.

## Verification

- `uv sync --locked --all-packages --extra descriptor-analysis --extra telemetry [--extra cuda] --dry-run` at f42d32c, for x86_64 and aarch64 manylinux_2_28, Windows and macOS. `cuda` resolves on Linux x86_64 only; macOS does not resolve, as the README now states.
- A build script checks that the README patch applies to the pinned commit.

## Dead ends

A container image built from `uv.lock` was prepared and held back from the pull request; the repository has no container setup, and the issue offers it in one line. It was built from f42d32c (240 s, 8.1 GB) and tested on an RTX 3090 (driver 595.71.05, CUDA 13.2):

- without a GPU, the CI selection (`-m "not gpu and not slow"`): 955 passed, 25 skipped, 13 deselected
- with the GPU, `-m gpu`: 6 passed, 2 skipped (both need MACE model files that are not public)

Earlier runs the same day failed 2 CPU tests and skipped 3 GPU tests until the image kept the git metadata, ran as a non-root user and included gcc for torch.compile.

## How it was made

Made with Claude Code (Anthropic), under the direction of the person who posted it. A separate review pass, run independently of the build, tried to break the change and the texts before posting.

## What this record leaves out

This record summarises how the change was made and checked; internal working notes and drafts are not published.

---
Licence: CC BY 4.0 (see LICENSE.md). Code and diffs quoted here keep their original licences. How this work is done: ../HOW.md
