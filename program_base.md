# program.md — universal base

This file is the universal base for any autonomous-loop project: an LLM agent that runs surgical experiments / improvements indefinitely against a defined goal, until a stopping condition fires.

It is distilled from two real runs:

- **autotrain** (ML hyperparameter search) — minimize `val_bpb`, fixed 5-minute training budget per iteration, never stop until human interrupts. ~80+ experiments, ~18% keep rate.
- **pikachu-godot** (game-dev content build) — improve a Godot RPG demo across multi-axis quality rubric, no time budget, stop when backlog converges. 108 iterations, 100% keep rate.

The disciplines that survived both runs are universal. Everything else is a slot you fill per project.

## How to use this base

1. **Copy** this file into your project as `program.md`.
2. **Keep "Universal core" verbatim** — these are the disciplines.
3. **Fill every `{SLOT}`** in "Project slots" with project-specific content.
4. **Delete this "How to use" header** and the case-study examples once filled.
5. Start the loop. Each iteration re-reads `program.md` as the single source of truth.

The only paired file is `results.tsv` (untracked, append-only log). Optional adjuncts: `journal.md` (always recommended), `backlog.md` (recommended for finite-content tasks), `feedback_inbox.md` (recommended if a human is around to nudge the loop).

---

## Universal core (copy verbatim into your project's program.md)

### Hypothesis block — per commit

Every commit message must contain four lines after the subject:

```
Hypothesis: <one sentence — the mechanism by which this advances the goal>
Assumptions: <one line — repo state I'm relying on; if wrong, result is uninterpretable>
Prediction: <expected effect: direction, magnitude, or qualitative description>
Tripwire: <a result that means the experiment broke, not the idea — investigate before discarding>
```

If you genuinely cannot articulate a mechanism, prefix `Prediction:` with `exploratory:` and treat the result with extra scepticism.

### Surgical diffs

Every changed line in an experimental commit must trace to the current Hypothesis. Incidental changes — formatting, renames, drive-by refactors, dead-code cleanup — must NOT be bundled with experiments. They muddy interpretation when a discard happens.

Test: pick any line in the diff. Ask "if I reverted just this line, would this commit still test the same hypothesis?" If yes, the line is incidental — remove it from this commit.

Cleanup is allowed, but in its own commit with subject `refactor: ... (no behavior change expected)`. After such a commit, run validation; the metric should be unchanged within noise.

When measuring diff size for the soft cap below, use `git diff --stat -w` — indent-shift refactors balloon raw stats with whitespace-only changes.

### Simplicity bias with a soft cap

All else equal, simpler wins. A small improvement that adds ugly complexity is not worth it. Removing code while holding the metric is always a keep.

**Soft cap**: if your experimental diff exceeds 50 added non-whitespace lines, defend the size in the commit body.

### Open Questions discipline

When uncertain during the loop ("why does this code exist?", "is this assumption right?", "should X be Y?") and you cannot resolve cheaply: append a one-line bullet to `journal.md` Open Questions and continue. **Do not stop the loop. Do not ask the human.** The next reflection cycle processes these.

### Journal as living memory

`journal.md` has two sections, populated by the loop:

```markdown
# Journal
## Open Questions
- (uncertainties accumulated during the loop; one-liners)

## Reflections
- (periodic syntheses; one block per reflection cycle)
```

Reflections fire every `{REFLECTION_CADENCE}` keeps. They summarize the recent batch (what landed, what didn't, common themes, dead ends, 1–2 next directions) and process the Open Questions list.

### Never ask the human

Once the loop has begun, do not pause to ask "should I continue?" The human may be asleep. Surface doubts in Open Questions and proceed. The loop ends only when a `{STOPPING_CONDITIONS}` check fires.

### The loop

```
LOOP UNTIL any of {STOPPING_CONDITIONS} is true:

  0. If iteration_count % {REFLECTION_CADENCE} == 0 and is_keep_iter:
       run reflection — update journal.md Reflections, process Open Questions.

  0a. If feedback_inbox.md exists and is non-empty:
       prioritize its contents over backlog/reflection-generated ideas.

  1. Orient: git status, read journal.md, read last 5 rows of results.tsv.

  2. Pick next item per {IDEA_SOURCES}; cadence overrides:
       - every {EXPLORATORY_EVERY} iters: force a non-backlog, non-safe item
         (expect higher discard probability; combats 100%-keep complacency)
       - every {CONSOLIDATION_EVERY} keeps: force an iteration whose goal is
         negative LOC delta (refactor, dedupe, delete; metric must hold)

  3. Write Hypothesis / Assumptions / Prediction / Tripwire BEFORE coding.

  4. Implement surgically. Check `git diff --stat -w` is reasonable.

  5. Run {VALIDATION}.

  6. Tripwire check: if the result hits the tripwire, sanity-check whether
     the experiment ran as intended (re-read diff, re-grep log) before
     logging discard.

  7. If {NOISY} and improvement < {SMALL_WIN_THRESHOLD}: seed re-check
     (one extra run with a different seed; require average to still beat
     baseline before keeping). Otherwise skip this step.

  8. Decide keep/discard per {DECISION_RULE}.

  9. Commit (only on keep) with the four-line block in the body.
     If diff > 50 added non-whitespace lines, defend in body.

 10. Append a row to results.tsv (every attempt, including discard/crash).

 11. Update journal.md only if the result was surprising or revealed
     something worth remembering. Do not journal every iteration.

 12. Update backlog.md (strike done item, add follow-ups discovered).

 13. Check {STOPPING_CONDITIONS}; if any true, write a final journal entry
     and stop.
```

---

## Project slots — you must fill these

### {MISSION}

One paragraph. What does success look like for this project?

- *autotrain*: "Get the lowest `val_bpb` on the fixed evaluation harness. Each training run is capped at 5 minutes wall clock. The loop runs indefinitely; the curve in `progress.png` is the deliverable."
- *pikachu-godot*: "Make `examples/pikachu-godot/` into a more complete game — gameplay systems, content, mechanics — one small surgical change at a time. Treat as a long-running content-build run, not a polish pass."

### {SCOPE}

In-scope (may modify) vs out-of-scope (must not modify).

- *autotrain*: in = [`train.py`]; out = [`prepare.py`, eval harness, dependencies, time budget].
- *pikachu-godot*: in = [`scenes/`, `scripts/`, `tools/make_assets.py`, project README, journal/results/program]; out = [`skills/`, `src/`, top-level repo docs, `CLAUDE.md`].

### {HARD_CONSTRAINTS}

Inviolable rules. List them explicitly.

- *autotrain*: no new packages beyond `pyproject.toml`; do not modify the eval harness; project must run without crashing within the time budget.
- *pikachu-godot*: no new deps beyond `Pillow`+`numpy`; no `image_gen` (use Pillow placeholders); no audio files (`AudioStreamGenerator` only); SKILL replacement points kept clean; project must boot via `godot4 --headless --import`.

**Common addition (recommended)**: if your project has a user-facing README listing features or limitations, add: *"Any commit that ships a user-visible feature must update the README's Limitations section in the same commit."* This prevents README rot — pikachu-godot's README still listed 14 false "not in scope" claims by iter15 because the loop never updated it.

### {METRIC} and {DECISION_RULE}

Pick one of three shapes:

**Scalar metric** (e.g. autotrain's `val_bpb`):
- Direction: lower-is-better or higher-is-better.
- Decision: keep iff metric strictly improves vs current best.
- Set `{NOISY}` and `{SMALL_WIN_THRESHOLD}` below.

**Multi-axis rubric** (e.g. pikachu-godot's depth/content/polish/code/skill-ready/bug):
- List your axes.
- Priority order (e.g. depth > content > code > polish): when in doubt, prefer the higher-priority axis.
- Decision: keep iff at least one axis improves and none regresses.
- Recommended hard rule: no more than `N` (default 3) consecutive iterations on the same axis — combats axis drift (especially "polish trap").

**Pass-fail** (validation succeeds or it doesn't):
- Decision: keep iff `{VALIDATION}` passes and the change addresses a tracked goal.
- Use this for refactor/cleanup-only loops; rare in practice.

### {VALIDATION}

A deterministic go/no-go check, runnable from the command line. Must not require human judgment to interpret.

- *autotrain*: `uv run train.py > run.log 2>&1` then `grep "^val_bpb:\|^peak_vram_mb:" run.log` (empty grep ⇒ crash; `tail -n 50 run.log` for traceback).
- *pikachu-godot*: `godot4 --headless --import` + boot each scene for ~4s with `--quit-after`, grep stderr for `error|invalid|fail` (excluding harmless "ObjectDB instances leaked").

### {ITERATION_BUDGET}

- *autotrain*: fixed 5-minute training (enforced by script); total wall-clock kill at 10 minutes.
- *pikachu-godot*: best-effort (no time cap; iteration completes when validation passes).

If your domain has a natural per-iteration time, fix it. If not, leave best-effort but **state a kill threshold** so runaway iterations get aborted.

### {NOISY} and {SMALL_WIN_THRESHOLD}

Is the metric stochastic per iteration?

- *autotrain*: noisy=true, threshold=0.002. Below threshold, do a one-extra-run seed re-check and require the 2-run average to still beat baseline.
- *pikachu-godot*: noisy=false (game changes are deterministic) → seed re-check disabled.

### {STOPPING_CONDITIONS}

- *autotrain*: never stop (human interrupt only).
- *pikachu-godot*: stop when **any** is true:
  - Backlog has fewer than 3 unticked items AND no new ideas after a full reflection.
  - Two consecutive iterations crash and cannot be fixed within the iteration.
  - Hard validation failure on baseline after revert (environment broken).

For finite-content domains, set real stopping conditions. For open-ended search, "never stop" is correct.

### {REFLECTION_CADENCE}

Default: every **5 keeps** (pikachu-godot's choice; lessons compound there). autotrain's original "every 20" was too sparse — backport 5 unless your iterations are extremely cheap.

### {EXPLORATORY_EVERY} and {CONSOLIDATION_EVERY}

Optional but earned through experience.

- `{EXPLORATORY_EVERY}` (default 10): every Nth iteration, force an idea **not** on the backlog. Lower acceptance bar. Combats the comfort zone — pikachu-godot ran 108 iterations with 100% keep rate, which is too low a rate of failed exploration.
- `{CONSOLIDATION_EVERY}` (default 15): every Nth keep, force an iteration whose goal is **negative LOC delta** (delete, dedupe, extract). Metric must hold. Combats the "monotonically growing" tendency — pikachu-godot grew `Battle.gd` to 50KB and `Overworld.gd` to 41KB without any deletion-as-win iteration.

### {IDEA_SOURCES} (priority order)

Default order:

1. `feedback_inbox.md` (if present and non-empty) — highest ROI, validated by pikachu-godot iter47–49.
2. `backlog.md` — explicit ranked task pool. Required for finite-content tasks.
3. Reflection-generated ideas (in `journal.md` Reflections).
4. Exploratory (per `{EXPLORATORY_EVERY}` cadence).

For open-ended search domains (autotrain), backlog may be omitted; reflection + exploratory carry the load.

---

## results.tsv

Tab-separated. Untracked (`.gitignore` it). Append-only — every attempt logs a row, including `discard` and `crash`.

Column shape adapts to `{METRIC}`:

- **scalar**: `commit  metric  memory_or_cost  status  description`
- **multi-axis**: `commit  axis  status  summary`
- **pass-fail**: `commit  status  summary`

Common columns:
- `commit`: short hash (7 chars). Use `-` for discard/crash with no commit.
- `status`: `keep | discard | crash`.
- The trailing free-text field uses no tabs.

Crashes log as `0` for any numeric column.

---

## Recommended adjuncts

- `journal.md` — always create; the Open Questions + Reflections sections are referenced throughout the loop.
- `backlog.md` — recommended for finite-content domains. Plain markdown; ranked tiers; strikethrough done items rather than deleting (history is informative).
- `feedback_inbox.md` — recommended whenever a human is around. Append-only file the human writes to between sessions; loop reads it at iter start. Highest-ROI feedback channel.

---

## Lessons from the case studies (universal)

These came out of the two runs and are worth re-applying anywhere:

1. **Frameworks pay compound interest.** A ~50-LOC investment in a reusable framework (e.g. pikachu's `DialogueBox`, the quest framework, the held-item axes) typically enables 5–10 follow-up iterations of ~30 LOC each. Recognize when an iteration is laying foundation versus extending a foundation, and prefer the former when it unlocks future surface.

2. **"Done" is wrong about 80% of the time.** When the agent declares an axis tapped out, force one more reflection scan — pikachu's "depth tapped out" judgments at iter50, iter65, iter90 each turned out wrong (multiple canonical mechanics still missing). Fresh review with structured prompts beats gut "we're done".

3. **Indent-shift refactors fool diff stats.** Use `git diff --stat -w` for the size cap. A 130/119 diff that's actually 11 lines of new logic should not trip the 50-line defense rule.

4. **Self-damage paths need catch-all victory triggers.** Generalisation: when adding a new way for state to change at end-of-turn (recoil, confusion, poison, etc.), make sure the existing termination checks see the new path. Pure additive features still need integration sweeps.

5. **Cross-scene framework duplication is acceptable up to 2 scenes.** Refactor to a shared autoload only when a 3rd consumer arrives. Premature factor-out is a common over-engineering trap.

6. **The single highest-ROI input is direct user feedback.** One user note ("attack screen feels stuck") in pikachu surfaced four hidden bugs and correctness issues. Build a `feedback_inbox.md` channel before you need it.

7. **README rot is a real risk.** A loop that edits code without a hard rule to update the README will leave the README claiming the project is missing features that already shipped. Make README sync a hard constraint when the project has user-facing docs.

8. **100% keep rate is a smell, not a victory.** It usually means the agent is only pulling pre-vetted safe items. Force exploratory iterations even at the cost of accepting more discards.
