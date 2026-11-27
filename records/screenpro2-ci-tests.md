# ScreenPro2: run the tests in CI

- Upstream: https://github.com/ArcInstitute/ScreenPro2 (MIT)
- Contribution: issue https://github.com/ArcInstitute/ScreenPro2/issues/114, pull request https://github.com/ArcInstitute/ScreenPro2/pull/115; status as of 2026-11-27: pull request merged on 2026-11-26, issue open
- Kind: non-trivial

## The problem

Release 0.7.1 declares `requires-python <3.12`, so pip on Python 3.12 or 3.13 skips it and installs 0.7.0, which has no dependency metadata; `import screenpro` then fails. The test step in CI had been commented out, so packaging breakage of this kind reached users without a red build.

## Reproduction

macOS arm64, upstream commit 74754d9:

- Python 3.12.13: `python -m pip install ScreenPro2` resolves to 0.7.0, and `import screenpro` fails with `ModuleNotFoundError: No module named 'pandas'`.
- Python 3.12 with `uv pip install ScreenPro2`: uv ignores the upper bound, picks 0.7.1 and fails building pandas 1.5.3 from source.
- Python 3.11.15, clean checkout: `uv pip install . pytest`, then `pytest tests -q`: 4 passed.

## The change

Two steps added to the existing Python 3.11 job, before the conda step: `pip install . pytest` and `pytest tests` (diff: https://github.com/ArcInstitute/ScreenPro2/pull/115/files). Since #113, a plain `pip install .` installs everything the tests need, and the four tests use demo data already in the repository, so re-enabling them needs no new environment or job.

## Verification

- In a `python:3.11-slim` container from a shallow clone, both steps passed: 4 tests, five runs in a row, adding about a minute.
- Clean installs on Python 3.9, 3.10 and 3.11 containers each passed 4 of 4 tests.
- The pull request was approved and merged on 2026-11-26.

## Dead ends

- Yanking 0.7.0 was considered and dropped: every earlier release also lacks dependency metadata, so pip would fall back to 0.6.1.
- The Python cap itself is left for a separate change. The follow-up comment on the issue proposes dropping the upper bound before 0.7.2 is tagged, so Python 3.12 users get a clear build error at install time; proper support for newer Python needs pandas 2. The maintainer invited a further pull request on the version bounds.

## How it was made

Made with Claude Code (Anthropic), under the direction of the person who posted it. A separate review pass, run independently of the build, tried to break the change and the texts before posting.

## What this record leaves out

This record summarises how the change was made and checked; internal working notes and drafts are not published.

---
Licence: CC BY 4.0 (see LICENSE.md). Code and diffs quoted here keep their original licences. How this work is done: ../HOW.md
