# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this directory is

This is **not** a code repository. It holds the protocol for running autonomous LLM-driven research:

- `program.md` — the canonical spec. **Read it in full before doing anything.** It defines the run setup, the experiment loop, the results format, and the rules of engagement.
- `progress.png` — a sample chart from a prior run (83 experiments, 15 kept improvements, baseline 0.998 → 0.977 val_bpb). Useful only as a reference for what a healthy run looks like.

The actual training code (`prepare.py`, `train.py`, `README.md`) lives in a separate repo. You operate on a fresh branch `autoresearch/<tag>` of that repo, not here.

## Operating rules that must not be forgotten

These are the points from `program.md` most commonly missed — re-read the source for full nuance.

- **The loop never asks for permission to continue.** Once setup is confirmed, do not pause to ask "should I keep going?" The user expects you to run indefinitely until manually stopped (the canonical use case is leaving you running overnight).
- **Only `train.py` is editable.** `prepare.py` is read-only; do not modify the eval harness, data pipeline, tokenizer, or the fixed time budget. Do not add dependencies beyond `pyproject.toml`.
- **Run command is fixed:** `uv run train.py > run.log 2>&1`. Always redirect — never `tee`, never let stdout into your context. Read results with `grep "^val_bpb:\|^peak_vram_mb:" run.log`. Empty grep ⇒ crash ⇒ `tail -n 50 run.log`.
- **5-minute training budget is enforced by the script.** If wall clock exceeds ~10 minutes total, kill and treat as failure.
- **Decision rule:** lower val_bpb ⇒ keep the commit and advance. Equal or worse ⇒ `git reset` back. Rewinding past a kept commit should be extremely rare.
- **`results.tsv` is tab-separated, 5 columns (`commit val_bpb memory_gb status description`), and stays untracked.** Crashes log as `0.000000 / 0.0 / crash`.
- **Simplicity tiebreaker.** Tiny gains that add ugly complexity → discard. Equal or better results from deleting code → always keep. VRAM is a soft constraint; modest growth for real gains is fine, blow-ups are not.
- **Crashes:** trivial fixes (typo, missing import) → fix and rerun. Fundamentally broken idea → log `crash`, move on.
