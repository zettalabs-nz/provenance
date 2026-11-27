# ScreenPro2: relax the dependency version bounds

- Upstream: https://github.com/ArcInstitute/ScreenPro2 (MIT)
- Contribution: issue https://github.com/ArcInstitute/ScreenPro2/issues/114, pull request: opened after this record
- Kind: non-trivial

## The problem

`pyproject.toml` caps Python below 3.12 and pins pandas, numpy, matplotlib, pyarrow and pydeseq2 to old ranges, so pip on Python 3.12 or newer installs 0.7.0, which has no dependencies. The issue thread asked which bounds are needed.

## Reproduction

From upstream main at 2d07441 (the 0.7.2 bump). The change was then rebased onto 2347611 (0.7.3, released with the same Python cap), which touched only the version number and documentation files:

- pandas 1.5.3 has no cp312 or cp313 wheels; the Python cap follows from the pandas pin.
- With `pydeseq2<0.5` kept and anndata 0.12 or newer installed, `DeseqDataSet.deseq2()` fails with `IndexError: too many indices for array`.
- With pandas unbounded, pip picks pandas 3.1.1 on 3.12 and 3.13, and `runPhenoScore(..., collapse_var=...)` raises `KeyError: 'target'`.

## The change

| Bound | Old | New | Reason |
|---|---|---|---|
| python | `>=3.9,<3.12` | `>=3.10` | Cap followed pandas; 3.10 is the pydeseq2 0.5.2 floor |
| pandas | `<2.0` | `<3` | pandas 3 breaks `collapse_var` |
| numpy | `<2.0` | `*` | No removed names used; equal results |
| matplotlib | `<3.7` | `*` | Old scanpy workaround |
| pyarrow | `<26` | `*` | Implied by the Python cap |
| pydeseq2 | `<0.5` | `>=0.5.2,<0.7` | 0.4 breaks with anndata 0.12; 0.7 changes results |

`runDESeq` now passes `design=f"~{design}"` in place of the deprecated `design_factors=design`, the formula pydeseq2 builds internally; old and new calls gave bit-identical results on every allowed pydeseq2.

## Verification

- A Linux x86_64 host: fresh `python:3.10-slim` to `python:3.13-slim` containers without Rust, plain `pip install`; the old stack from upstream main on 3.11.
- Resolved: pandas 2.3.3, numpy 2.2.6 to 2.5.5, pydeseq2 0.5.2 to 0.6.0; old stack pandas 1.5.3, numpy 1.26.4, pydeseq2 0.4.12.
- `pytest tests`: 4 passed on all five stacks.
- 38 output tables per stack (counts, phenotype scores, DESeq on clean and hard data) against the old stack: 0 differences. Maximum relative differences: counts 0, scores 3.2e-15, DESeq 3.8e-12.
- A macOS arm64 laptop with Rust hidden: Python 3.12 installs from wheels and imports; Python 3.10 stops building biobear, which has no macOS cp310 wheel (0.7.1 has the same gap).

## Dead ends

- pydeseq2 was first left open above 0.5.2. Version 0.7.0 changes p-values and Cook's filtering, and on hard data DESeq drifted up to 0.8 relative, so it is capped at `<0.7`.
- The first proposal reported Python 3.10 working on macOS; that install built biobear with a local Rust toolchain, so the claim was withdrawn.

## Known gaps

- biobear has no wheels for macOS 3.10, Windows 3.13 or Python 3.14, so those installs need Rust.
- `environment.yml` keeps the old pins.
- CI tests Python 3.11 only.

## How it was made

Made with Claude Code (Anthropic), under the direction of the person who posted it. A separate review pass, run independently of the build, tried to break the change and the texts before posting.

## What this record leaves out

This record summarises how the change was made and checked; internal working notes and drafts are not published.

---
Licence: CC BY 4.0 (see LICENSE.md). Code and diffs quoted here keep their original licences. How this work is done: ../HOW.md
