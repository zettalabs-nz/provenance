# ScreenPro2: relax the dependency version bounds

- Upstream: https://github.com/ArcInstitute/ScreenPro2 (MIT)
- Contribution: issue https://github.com/ArcInstitute/ScreenPro2/issues/114, pull request https://github.com/ArcInstitute/ScreenPro2/pull/116
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
| pydeseq2 | `<0.5` | `>=0.5.2` | 0.4 breaks with anndata 0.12; Python 3.12+ gets the latest |

`runDESeq` now passes `design="~" + " + ".join(factors)` in place of the deprecated `design_factors=design`, where `factors` is the given list or a one-item list for a single factor; this is how pydeseq2 itself built the formula from `design_factors`. Old and new calls gave bit-identical results on every allowed pydeseq2. After an automated review on the pull request pointed out that a list of factors would have broken the first version (`f"~{design}"`), the list case was added and checked against the old call on pydeseq2 0.5.2 and 0.7.0, for a single factor and for `['batch', 'condition']`: identical results.

## Verification

- A Linux x86_64 host: fresh `python:3.10-slim` to `python:3.13-slim` containers without Rust, plain `pip install`; the old stack from upstream main on 3.11.
- Resolved, with pydeseq2 uncapped: pandas 2.3.3, numpy 2.2.6 to 2.5.5, pydeseq2 0.5.2 (3.10), 0.5.4 (3.11) and 0.7.0 (3.12, 3.13); old stack pandas 1.5.3, numpy 1.26.4, pydeseq2 0.4.12. The earlier capped run resolved pydeseq2 0.5.2 to 0.6.0.
- `pytest tests`: 4 passed on all five stacks.
- 38 output tables per stack (counts, phenotype scores, DESeq on clean and hard data) against the old stack. With the `<0.7` cap: 0 differences (counts 0, scores 3.2e-15, DESeq 3.8e-12). Uncapped: the same on 3.10 and 3.11; on 3.12 and 3.13 (pydeseq2 0.7.0) 35 tables match and the three hard-data DESeq tables move by up to 0.8 relative.
- `runDESeq` new formula against the old `design_factors` call, full results (baseMean, log2FoldChange, lfcSE, stat, pvalue, padj) on three synthetic sets, single factor and `['batch', 'condition']`, on pydeseq2 0.5.2, 0.5.4 and 0.7.0: bit-identical.
- A macOS arm64 laptop with Rust hidden: Python 3.12 installs from wheels and imports; Python 3.10 stops building biobear, which has no macOS cp310 wheel (0.7.1 has the same gap).

## Dead ends

- pydeseq2 was first left open above 0.5.2, then capped at `<0.7` because 0.7.0 changes p-values and Cook's filtering (on hard data DESeq drifted up to 0.8 relative from 0.5.x). The maintainer asked to keep the latest pydeseq2 for ongoing benchmarking, so the cap was removed; Python 3.10 and 3.11 stay on 0.5.x because pydeseq2 0.6 and later require 3.12.
- The first proposal reported Python 3.10 working on macOS; that install built biobear with a local Rust toolchain, so the claim was withdrawn.

## Known gaps

- biobear has no wheels for macOS 3.10, Windows 3.13 or Python 3.14, so those installs need Rust.
- `environment.yml` keeps the old pins.
- CI tests Python 3.11 only.

## Real-data check

At the maintainer's request on the pull request, `calculateDrugScreenDESeq` was run on three public decitabine CRISPRi screens (MOLM13, HL60, SKM1; `mergedcountstable.txt` from GilbertLabUCSF/Decitabine-treatment, about 100k guides each after `filterLowCounts`, 2 replicates per arm), on macOS arm64 and in Linux x86_64 containers.

- The new formula against the old `design_factors` call on the same pydeseq2: identical on all 72 result tables (3 screens, 3 contrasts, Python 3.10 to 3.13, both platforms), and for a two-factor `['replicate', 'condition']` design with string labels.
- main's pinned install (pydeseq2 0.4.12, pandas 1.5.3) against the change (pydeseq2 0.5.4, pandas 2.3.3), Python 3.11: log2FoldChange within 0.002 on every guide; lfcSE or p-value moves by more than 0.001 on at most 6 guides per screen; padj < 0.05 sets differ by at most 3 guides.
- pydeseq2 0.5.4 against 0.7.0: at most 3 guides per screen change; padj < 0.05 sets identical. Cook's filtering needs 3 or more replicates, so its 0.7 change is not exercised by this data.
- The version change alters one behaviour: pydeseq2 0.4.12 casts design factors to strings, 0.5 and later fit an integer column as numeric. On a synthetic set, a 2-replicate integer `replicate` gives the same condition effect either way; with 3 replicates the condition LFC moves by up to 0.012.

## How it was made

Made with Claude Code (Anthropic), under the direction of the person who posted it. A separate review pass, run independently of the build, tried to break the change and the texts before posting.

## What this record leaves out

This record summarises how the change was made and checked; internal working notes and drafts are not published.

---
Licence: CC BY 4.0 (see LICENSE.md). Code and diffs quoted here keep their original licences. How this work is done: ../HOW.md
