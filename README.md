# autotrain — the autoresearch protocol

A protocol for an LLM agent to autonomously run ML training experiments overnight.

This repo is **not** training code. It is a spec: the playbook the agent reads before it starts, plus the disciplines it follows during the loop. The training code itself (`train.py`, `prepare.py`, `README.md`) lives in a separate repo. The agent operates on a fresh branch `autoresearch/<tag>` of that repo.

The loop in one sentence: tune `train.py`, run a fixed 5-minute experiment, decide `keep` or `revert` against a single metric (`val_bpb`), repeat indefinitely until the human interrupts.

## Files

| File | Purpose |
|---|---|
| `program.md` / `program_zh.md` | Original protocol (English / Traditional Chinese). |
| `program_v1.md` / `program_v1_zh.md` | v1: per-commit Hypothesis + Prediction, two-seed re-check for small wins (Δ < 0.002), periodic reflection every 20 experiments. |
| `program_v3.md` / `program_v3_zh.md` | **Current.** v1 + the four Karpathy LLM-coding guidelines, adapted so they never stop the loop. |
| `CLAUDE.md` | Orientation for Claude Code instances opened in this directory. |
| `skills/karpathy-guidelines/SKILL.md` | Source skill that informed v3. Kept for reference; **intentionally not loaded inside the experiment loop** — its #1 principle ("ask if uncertain") would freeze a never-stop loop. |
| `progress.png` | Sample chart from a prior run — 83 experiments, 15 kept improvements, baseline 0.998 → 0.977 val_bpb. |

## Which version to use

Default to `program_v3.md`. v1 and the original are kept as diff anchors, not active instructions.

## What v1 and v3 add over the original

| Concern | Original | v1 | v3 |
|---|---|---|---|
| Auditability — why was each run tried? | implicit in commit subject | explicit `Hypothesis:` + `Prediction:` lines | + `Assumptions:` (what's load-bearing) |
| Noise resistance — is a small win real? | not addressed | two-seed re-check when Δ < 0.002 | unchanged |
| Drift control — is the search still going somewhere? | not addressed | reflection every 20 experiments → `journal.md` | + `journal.md` *Open questions* section, deferred-ask |
| Interpretability — when a run fails, was it the idea or an incidental change? | not addressed | not addressed | **Surgical experimental diffs** rule + `Tripwire:` |
| Simplicity bias | qualitative criterion | qualitative criterion | + soft cap: > 50 added lines must be defended in commit body |

All additions preserve the never-stop invariant: the agent surfaces doubts in writing, but never pauses to ask the human.

## How to use

1. Read `program_v3.md` (or `program_v3_zh.md`).
2. Follow Setup → Experimentation → Loop. The agent runs autonomously until manually stopped.

The first run is always the unmodified baseline. Everything after that is fair game on `train.py`, subject to the disciplines in the spec.
