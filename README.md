# autotrain — autonomous-loop protocol

A protocol for an LLM agent to autonomously run **surgical experiments** against a defined goal, until a stopping condition fires.

Originally written for ML hyperparameter search (minimize `val_bpb` overnight). After running it on a separate game-development project (108 iterations on a Godot RPG demo), the disciplines that survived **both** domains were distilled into a single universal base: `program_base.md`.

The pattern in one sentence: per project, **one `program.md` + one `results.tsv`**. Each downstream project copies `program_base.md`, fills in the `{SLOT}`s, and runs the loop.

## Files

### Canonical base (start here)

| File | Purpose |
|---|---|
| **`program_base.md`** / `program_base_zh.md` | **Universal base.** Distilled from two real runs. Copy → fill `{SLOT}`s → run. |
| `progress.png` | Sample chart from an autotrain run: 83 experiments, 15 kept, baseline 0.998 → 0.977 val_bpb. |
| `CLAUDE.md` | Orientation for Claude Code instances opened in this directory. |

### Historical (kept for diff / lineage)

| File | Era |
|---|---|
| `program.md` / `program_zh.md` | v0 — original ML-only protocol. |
| `program_v1.md` / `program_v1_zh.md` | v1 — adds Hypothesis discipline, seed re-check, periodic reflection. |
| `program_v3.md` / `program_v3_zh.md` | v3 — adds Karpathy LLM-coding guidelines (Surgical diffs, Tripwire, Open Questions, soft cap). |
| `skills/karpathy-guidelines/SKILL.md` | Source skill that informed v3. **Intentionally not loaded inside the loop** — its #1 principle ("ask if uncertain") would freeze a never-stop loop. |

The historical chain is `program.md → v1 → v3 → program_base.md`. New projects should use `program_base.md`; v0/v1/v3 are kept only so the design lineage stays auditable.

## What `program_base.md` is

A single file with two halves:

**Universal core** (copy verbatim into your project's `program.md`)
- Hypothesis block per commit (4 lines: Hypothesis / Assumptions / Prediction / Tripwire)
- Surgical diffs — every line traces to the hypothesis; cleanup goes in separate commits
- Simplicity bias with a 50-line soft cap, measured with `git diff --stat -w`
- Open Questions discipline — write doubts to `journal.md`, never ask the human
- Journal as living memory (Open Questions + Reflections)
- The loop skeleton

**Project slots** (you must fill these)

`{MISSION}`, `{SCOPE}`, `{HARD_CONSTRAINTS}`, `{METRIC}`+`{DECISION_RULE}`, `{VALIDATION}`, `{ITERATION_BUDGET}`, `{NOISY}`+`{SMALL_WIN_THRESHOLD}`, `{STOPPING_CONDITIONS}`, `{REFLECTION_CADENCE}`, `{EXPLORATORY_EVERY}`+`{CONSOLIDATION_EVERY}`, `{IDEA_SOURCES}`.

Each slot has worked examples from both case studies (autotrain ML and pikachu-godot game-dev) so a new project can pattern-match its way to a filled `program.md`.

## How to use

1. Read `program_base.md` (or `program_base_zh.md`).
2. Copy it into your project as `program.md`.
3. Fill every `{SLOT}` — most have two reference examples to model from.
4. Delete the "How to use this base" header and case-study examples once filled.
5. Start the loop. Each iteration re-reads `program.md` as the single source of truth, appends to `results.tsv`, and updates `journal.md` per the cadence you set.

The loop never asks the human. It only stops when one of the `{STOPPING_CONDITIONS}` you defined fires.

## What was learned that fed back into the base

The pikachu-godot run produced eight universal lessons that are now in `program_base.md`. The most load-bearing:

- **100% keep rate is a smell, not a victory.** It means the agent is only pulling pre-vetted safe items. The base now mandates an `EXPLORATORY_EVERY` cadence to force lower-acceptance-bar iterations.
- **Monotonically growing is also a smell.** No deletion-as-win iteration ever fired in 108 game iterations. The base now mandates a `CONSOLIDATION_EVERY` cadence targeting negative LOC delta.
- **README rot is real.** The pikachu-godot README claimed 14 features were "not in scope" that had already shipped. The base recommends a hard constraint requiring README sync in any feature commit.
- **Direct user feedback is the highest-ROI input.** A single user note in pikachu surfaced four hidden bugs across three iterations. The base now recommends a `feedback_inbox.md` channel as the highest-priority idea source.
- **Use `git diff --stat -w` for the size cap.** Indent-shift refactors balloon raw stats with whitespace-only changes.
