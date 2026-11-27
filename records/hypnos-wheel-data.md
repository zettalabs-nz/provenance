# hypnos: ship the hypnos.data package in the wheel and sdist

- Upstream: https://github.com/joncarter1/hypnos (MIT)
- Contribution: issue https://github.com/joncarter1/hypnos/issues/5, pull request https://github.com/joncarter1/hypnos/pull/6; status as of 2026-11-27: open
- Kind: non-trivial

## The problem

After `pip install hypnos` (0.3.0), the first example in the README fails with `ModuleNotFoundError: No module named 'hypnos.data'`. Both the wheel and the sdist on PyPI lack `hypnos/data/`, which 0.1.0 still had. CI stayed green because `uv run` installs the project in editable mode and reads from `src/` directly.

## Reproduction

The upstream repository at commit 6097765 was built twice with `uv build --wheel`: once as published and once with the patch applied. Each wheel was installed into a fresh `uv venv` and imported with:

```
python -c "from hypnos.embedding import embed_edf"
```

The wheel as published failed with `ModuleNotFoundError: No module named 'hypnos.data'`; the patched wheel imported. The cause is an unanchored `data/` rule in `.gitignore`, added for the demo cache, which also matches `src/hypnos/data/`; hatchling leaves ignored files out of the build.

## The change

Two commits (diff: https://github.com/joncarter1/hypnos/pull/6/files):

- `.gitignore`: the rule becomes `/data/`, so only the top-level demo cache is ignored. The demo notebook writes there, so no new files become trackable.
- `.github/workflows/ci.yml`: a `Package` job builds the sdist and wheel and runs the tests against the installed wheel. The existing test job installs from `src/` and cannot see what the build leaves out.

A one-character rule change is the smallest fix that restores the three missing files; the CI job keeps the same gap from returning unnoticed.

## Verification

`uv build`, then `uv run --isolated --no-project --with dist/*.whl --with pytest pytest -v`:

- before the change: 6 failed (`ModuleNotFoundError: No module named 'hypnos.data'`)
- after the change: 6 passed, with `hypnos/data/` present in both the wheel and the sdist

A build script repeats the published-against-patched check from the pinned commit and stops if the published wheel starts importing, which would mean upstream has fixed it independently.

## How it was made

Made with Claude Code (Anthropic), under the direction of the person who posted it. A separate review pass, run independently of the build, tried to break the change and the texts before posting.

## What this record leaves out

This record summarises how the change was made and checked; internal working notes and drafts are not published.

---
Licence: CC BY 4.0 (see LICENSE.md). Code and diffs quoted here keep their original licences. How this work is done: ../HOW.md
