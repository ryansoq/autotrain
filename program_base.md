---
# Edit every value below before starting the loop.
# Validate with: grep -nE '\{FILL' program.md   (must return nothing).

mission: "{FILL: one sentence — what does success look like for this project?}"

scope:
  in:  ["{FILL: paths the agent may modify}"]
  out: ["{FILL: paths the agent must not modify}"]

hard_constraints:
  - "{FILL: rule 1}"
  - "{FILL: rule 2}"

metric:
  # Pick one branch: scalar | multi_axis | pass_fail. Delete the unused branches.
  type: scalar

  # --- if scalar ---
  name: "{FILL: metric name, e.g. val_bpb}"
  direction: lower_is_better        # or higher_is_better
  noisy: false                      # if true, set small_win_threshold
  small_win_threshold: 0.0

  # --- if multi_axis ---
  # axes: [depth, content, polish, code]
  # priority: [depth, content, code, polish]   # ranked, highest first
  # consecutive_same_axis_max: 3

  # --- if pass_fail ---
  # (no extra fields; validation passing is the decision)

validation:
  command: |
    {FILL: shell command(s) — deterministic go/no-go check}
  on_failure: fix_or_revert

iteration_budget:
  type: best_effort                 # or fixed_time
  time_minutes: 0                   # only used if type: fixed_time
  kill_minutes: 10                  # hard kill regardless of type

stopping_conditions:
  never_stop: false                 # set true for open-ended search (autotrain style)
  conditions:
    - "{FILL: e.g. backlog thin AND no new ideas after full reflection}"
    - "{FILL: e.g. two consecutive iterations crash unfixed}"

cadence:
  reflection_every_n_keeps: 5
  exploratory_every_n_iters: 10
  consolidation_every_n_keeps: 15

idea_sources:                       # tried in order; missing/empty sources skip
  - feedback_inbox
  - backlog
  - reflection_generated
  - exploratory
---

# program.md — universal base

This file is the universal base for any autonomous-loop project: an LLM agent runs surgical experiments / improvements indefinitely against a defined goal, until a stopping condition fires.

It is distilled from two real runs: **autotrain** (ML hyperparameter search; minimize `val_bpb`; never-stop) and **pikachu-godot** (Godot RPG content build; multi-axis rubric; finite stopping). The disciplines that survived both runs are the body of this file. The frontmatter at the top is what each project fills in.

## How to use this base

1. **Copy** this file into your project as `program.md`.
2. **Edit the YAML frontmatter** at the top: replace every `{FILL: ...}` placeholder.
3. **Pick one `metric.type`** (`scalar` | `multi_axis` | `pass_fail`). Delete the unused branches.
4. **Decide `stopping_conditions.never_stop`**: `true` for open-ended search, `false` with explicit `conditions` for finite tasks.
5. **Pre-flight slot-leak check**: run `grep -nE '\{FILL' program.md`. It must return nothing.
6. *(Optional after filling)* delete this "How to use" section and the "Reference examples" section below.
7. Start the loop. Each iteration re-reads `program.md` as the single source of truth.

The only paired file is `results.tsv` (untracked, append-only). Optional adjuncts: `journal.md` (always), `backlog.md` (for finite tasks), `feedback_inbox.md` (when a human is around), `STOP` (kill-switch — see below).

---

## Universal core (do not edit)

### Hypothesis block — per commit

Every commit message must contain four lines after the subject:

```
Hypothesis: <one sentence — the mechanism by which this advances the goal>
Assumptions: <one line — repo state I'm relying on; if wrong, result is uninterpretable>
Prediction: <expected effect: direction, magnitude, or qualitative description>
Tripwire: <a result that means the experiment broke, not the idea>
```

If you cannot articulate a mechanism, prefix `Prediction:` with `exploratory:` and treat the result with extra scepticism.

### Surgical diffs

Every changed line in an experimental commit must trace to the current Hypothesis. Incidental changes — formatting, renames, drive-by refactors, dead-code cleanup — must NOT be bundled with experiments.

Test: pick any line in the diff. Ask "if I reverted just this line, would this commit still test the same hypothesis?" If yes, the line is incidental — remove it.

Cleanup is allowed in its own commit with subject `refactor: ... (no behavior change expected)`. After such a commit, run validation; the metric should be unchanged within noise.

Measure diff size with `git diff --stat -w` — indent-shift refactors balloon raw stats.

### Simplicity bias with a soft cap

All else equal, simpler wins. Removing code while holding the metric is always a keep. **Soft cap**: if the experimental diff exceeds 50 added non-whitespace lines, defend the size in the commit body.

### Open Questions discipline

When uncertain during the loop and you cannot resolve cheaply: append a one-line bullet to `journal.md` Open Questions and continue. **Do not stop the loop. Do not ask the human.** The next reflection cycle processes these.

### Journal as living memory

`journal.md` has two sections:

```markdown
# Journal
## Open Questions
- (uncertainties accumulated during the loop; one-liners)

## Reflections
- (periodic syntheses; one block per reflection cycle)
```

Reflections fire every `cadence.reflection_every_n_keeps` keeps. They summarize the recent batch (what landed, what didn't, themes, dead ends, 1–2 next directions) and process the Open Questions list.

### Never ask the human, but always honor `STOP`

Once the loop has begun, do not pause to ask "should I continue?" Surface doubts in Open Questions. The loop ends only when one of three things happens:

1. A `stopping_conditions.conditions` predicate fires.
2. A file named `STOP` exists at the project root (deterministic kill-switch — humans use this to stop the loop without editing `program.md`).
3. `iteration_budget.kill_minutes` exceeded for the current iteration (treated as crash).

### The loop

```
LOOP:

  0. Pre-iteration checks (always run):
       - If STOP file exists → write final journal entry "loop terminated by STOP", exit.
       - Confirm `grep -nE '\{FILL' program.md` is empty (slot-leak guard).
       - If keeps_count > 0 and keeps_count % cadence.reflection_every_n_keeps == 0
         and last_reflection_at_keeps != keeps_count:
           run reflection — update journal.md Reflections, process Open Questions.

  1. Orient: git status; read journal.md; read last 5 rows of results.tsv.

  2. Read feedback_inbox.md if present and non-empty (highest-priority idea source).

  3. Pick next item per idea_sources priority order; cadence overrides:
       - every cadence.exploratory_every_n_iters iters: force a non-backlog,
         lower-acceptance-bar item (combats 100%-keep complacency).
       - every cadence.consolidation_every_n_keeps keeps: force an iteration
         whose goal is negative LOC delta (refactor, dedupe, delete; metric
         must hold).

  4. Write Hypothesis / Assumptions / Prediction / Tripwire BEFORE coding.

  5. Implement surgically. Check `git diff --stat -w` is reasonable.

  6. Run validation.command.

  7. Tripwire check: if the result hits the tripwire, sanity-check whether the
     experiment ran as intended (re-read diff, re-grep log) before logging discard.

  8. If metric.noisy and improvement < metric.small_win_threshold:
     seed re-check (extra run with different seed; require average to beat baseline).
     Otherwise skip.

  9. Decide keep/discard per metric.type:
       - scalar: keep iff metric strictly improves vs current best.
       - multi_axis: keep iff at least one axis improves and none regresses;
         honor consecutive_same_axis_max.
       - pass_fail: keep iff validation.command passes.

 10. Commit (only on keep) with the four-line block in the body.
     If diff > 50 added non-whitespace lines, defend in body.

 11. Append a row to results.tsv (every attempt, including discard/crash).

 12. Update journal.md only if surprising or revealing. Do not journal every iter.

 13. Update backlog.md (strike done item, add follow-ups discovered).

 14. Check stopping_conditions.conditions; any true → final journal entry, exit.
```

---

## Reference examples — how each case study filled the frontmatter

### autotrain (ML hyperparameter search)

```yaml
mission: "Get the lowest val_bpb on the fixed evaluation harness; each training run capped at 5 minutes; the loop runs indefinitely."

scope:
  in:  [train.py]
  out: [prepare.py, evaluation harness, dependencies, time budget]

hard_constraints:
  - "no new packages beyond pyproject.toml"
  - "do not modify the eval harness"
  - "project must run without crashing within the time budget"

metric:
  type: scalar
  name: val_bpb
  direction: lower_is_better
  noisy: true
  small_win_threshold: 0.002

validation:
  command: |
    uv run train.py > run.log 2>&1
    grep "^val_bpb:\|^peak_vram_mb:" run.log
  on_failure: fix_or_revert

iteration_budget:
  type: fixed_time
  time_minutes: 5
  kill_minutes: 10

stopping_conditions:
  never_stop: true
  conditions: []

cadence:
  reflection_every_n_keeps: 5
  exploratory_every_n_iters: 10
  consolidation_every_n_keeps: 15

idea_sources:
  - feedback_inbox
  - reflection_generated
  - exploratory
```

### pikachu-godot (game-dev content build)

```yaml
mission: "Make examples/pikachu-godot/ into a more complete game — gameplay systems, content, mechanics — one small surgical change at a time."

scope:
  in:  [scenes/, scripts/, tools/make_assets.py, README.md, journal.md, results.tsv, program.md]
  out: [skills/, src/, top-level repo docs, CLAUDE.md]

hard_constraints:
  - "no new dependencies beyond Pillow + numpy"
  - "no image_gen — use Pillow placeholder shapes"
  - "no audio files — synthesized AudioStreamGenerator only"
  - "SKILL replacement points must remain swappable"
  - "project must boot via godot4 --headless --import"
  - "any commit shipping a user-visible feature must update README's Limitations section in the same commit"

metric:
  type: multi_axis
  axes: [depth, content, code, skill_ready, polish, bug]
  priority: [depth, content, code, skill_ready, polish, bug]
  consecutive_same_axis_max: 3

validation:
  command: |
    cd examples/pikachu-godot
    rm -rf .godot
    godot4 --headless --import 2>&1 | grep -iE '^ERROR' | grep -v leaked
    timeout 6 godot4 --headless --quit-after 240 res://scenes/TitleScreen.tscn 2>&1 | grep -iE 'error|invalid|fail' | grep -v leaked
    timeout 6 godot4 --headless --quit-after 240 res://scenes/Overworld.tscn   2>&1 | grep -iE 'error|invalid|fail' | grep -v leaked
    timeout 6 godot4 --headless --quit-after 240 res://scenes/Battle.tscn      2>&1 | grep -iE 'error|invalid|fail' | grep -v leaked
  on_failure: fix_or_revert

iteration_budget:
  type: best_effort
  kill_minutes: 10

stopping_conditions:
  never_stop: false
  conditions:
    - "backlog has fewer than 3 unticked items AND a full reflection produces no new directions"
    - "two consecutive iterations crash and cannot be fixed"
    - "hard validation failure on baseline after revert (environment broken)"

cadence:
  reflection_every_n_keeps: 5
  exploratory_every_n_iters: 10
  consolidation_every_n_keeps: 15

idea_sources:
  - feedback_inbox
  - backlog
  - reflection_generated
  - exploratory
```

---

## results.tsv — shape derives from `metric.type`

Tab-separated, untracked, append-only. Every attempt logs a row, including `discard` and `crash`.

| `metric.type` | columns |
|---|---|
| `scalar` | `commit  metric  memory_or_cost  status  description` |
| `multi_axis` | `commit  axis  status  summary` |
| `pass_fail` | `commit  status  summary` |

Common: `commit` is short hash (7 chars), or `-` if no commit. `status ∈ {keep, discard, crash}`. Trailing free-text uses no tabs. Crashes use `0` for any numeric column.

---

## Recommended adjuncts

- `journal.md` — always create; Open Questions + Reflections are referenced throughout the loop.
- `backlog.md` — recommended for finite-content domains (i.e. `stopping_conditions.never_stop = false`). Plain markdown, ranked tiers, strikethrough done items.
- `feedback_inbox.md` — recommended whenever a human is around. Append-only file the human writes to between sessions; loop reads at iter start.
- `STOP` — the kill-switch. Empty file at project root; agent only checks for existence.

---

## Lessons from the case studies (universal)

1. **Frameworks pay compound interest.** A ~50-LOC investment in a reusable framework typically enables 5–10 follow-up iterations of ~30 LOC each.
2. **"Done" is wrong about 80% of the time.** When the agent declares an axis tapped out, force one more reflection scan with structured prompts.
3. **Indent-shift refactors fool diff stats.** Use `git diff --stat -w` for the size cap.
4. **State-mutation paths need catch-all decision triggers.** When adding a new way for state to change at end-of-step (recoil, retry, side-effect), make sure existing termination checks see the new path.
5. **Cross-scene framework duplication is acceptable up to 2 instances.** Refactor to a shared module only when a 3rd consumer arrives.
6. **The single highest-ROI input is direct user feedback.** One user note can surface multiple hidden bugs across iterations. Build the `feedback_inbox.md` channel before you need it.
7. **README rot is a real risk.** A loop that edits code without a hard rule to update the README will leave the README claiming missing features that already shipped.
8. **100% keep rate is a smell, not a victory.** It usually means the agent is only pulling pre-vetted safe items. The `cadence.exploratory_every_n_iters` is the antidote — accept higher discard rate as the price.
