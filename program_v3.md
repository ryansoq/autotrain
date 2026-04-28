# autoresearch (v3)

This is an experiment to have the LLM do its own research.

v3 builds on v1. v1 added three disciplines (per-commit hypothesis+prediction, seed re-check for small wins, periodic reflection). v3 absorbs the four Karpathy LLM-coding guidelines — adapted so they do **not** stop the loop or break the "never ask the human" invariant. The four additions are: (a) extended hypothesis block with `Assumptions:` and `Tripwire:` lines, (b) a new **Surgical experimental diffs** discipline, (c) a soft cap in the simplicity criterion, (d) an *Open questions* section in `journal.md`.

## Setup

To set up a new experiment, work with the user to:

1. **Agree on a run tag**: propose a tag based on today's date (e.g. `mar5`). The branch `autoresearch/<tag>` must not already exist — this is a fresh run.
2. **Create the branch**: `git checkout -b autoresearch/<tag>` from current master.
3. **Read the in-scope files**: The repo is small. Read these files for full context:
   - `README.md` — repository context.
   - `prepare.py` — fixed constants, data prep, tokenizer, dataloader, evaluation. Do not modify.
   - `train.py` — the file you modify. Model architecture, optimizer, training loop.
4. **Verify data exists**: Check that `~/.cache/autoresearch/` contains data shards and a tokenizer. If not, tell the human to run `uv run prepare.py`.
5. **Initialize `results.tsv`**: Create with just the header row. Untracked.
6. **Initialize `journal.md`** with this template (untracked):

   ```markdown
   # Journal

   ## Open questions
   - (empty)

   ## Reflections
   - (empty)
   ```

7. **Confirm and go**: Confirm setup looks good.

Once you get confirmation, kick off the experimentation. **This is the last time you will ask the human anything.**

## Experimentation

Each experiment runs on a single GPU. The training script runs for a **fixed time budget of 5 minutes** (wall clock training time, excluding startup/compilation). You launch it simply as: `uv run train.py`.

**What you CAN do:**
- Modify `train.py` — this is the only file you edit. Everything is fair game: model architecture, optimizer, hyperparameters, training loop, batch size, model size, etc.

**What you CANNOT do:**
- Modify `prepare.py`. It is read-only. It contains the fixed evaluation, data loading, tokenizer, and training constants (time budget, sequence length, etc).
- Install new packages or add dependencies. You can only use what's already in `pyproject.toml`.
- Modify the evaluation harness. The `evaluate_bpb` function in `prepare.py` is the ground truth metric.

**The goal is simple: get the lowest val_bpb.** Since the time budget is fixed, you don't need to worry about training time — it's always 5 minutes.

**VRAM** is a soft constraint. Some increase is acceptable for meaningful val_bpb gains, but it should not blow up dramatically.

**Simplicity criterion**: All else being equal, simpler is better. A small improvement that adds ugly complexity is not worth it. Conversely, removing something and getting equal or better results is a great outcome — that's a simplification win. **Soft cap (new in v3): if your experimental diff exceeds 50 added lines, defend the size in the commit body** — large diffs are an interpretation hazard and usually a sign of overengineering. Ask yourself: "would a senior engineer call this overcomplicated?" If yes, simplify before committing.

**The first run**: Your very first run should always be to establish the baseline, so you will run the training script as-is.

## Hypothesis discipline (extended in v3)

Before every commit, the commit message must contain four short lines after the subject:

```
Hypothesis: <one sentence — the mechanism by which this should reduce val_bpb>
Assumptions: <one line — what about the current code I'm relying on; if this is wrong, the experiment is uninterpretable>
Prediction: <expected direction and magnitude, e.g. "↓ 0.001–0.003", "↓ but VRAM +10%", "exploratory">
Tripwire: <a result that means the experiment broke (not just that the idea was wrong); investigate before discarding if this fires>
```

Example:

```
Lower AdamW beta2 from 0.95 to 0.9

Hypothesis: reducing momentum staleness should help convergence within the
            5-minute budget where step count is small.
Assumptions: current beta1=0.9 and LR schedule are reasonable; the bottleneck
             is in fact momentum staleness, not capacity or LR.
Prediction: ↓ 0.001–0.003.
Tripwire: if val_bpb > 1.05 or VRAM jumps >20%, something else broke — read
          the diff again before discarding.
```

Why each line matters:
- **Hypothesis** turns a hack into an experiment. Failures become informative.
- **Assumptions** surfaces what you are *not* testing — when an assumption turns out wrong, the result is uninterpretable, not just negative.
- **Prediction** is your prior; comparing prediction vs reality calibrates judgment over many runs.
- **Tripwire** distinguishes "idea was wrong" from "experiment didn't run as intended". When the tripwire fires, do a brief sanity check before logging discard.

If you genuinely cannot articulate a mechanism, write `Prediction: exploratory` — but flag exploratory commits as such so the journal can count them.

## Surgical experimental diffs (new in v3)

**Every changed line in an experimental commit must trace back to the current hypothesis.** Incidental changes — formatting tweaks, comment edits, variable renames, dead-code cleanup, drive-by refactors — must NOT be bundled into the experimental commit.

Why: when a run gets `discard`, a clean diff lets you tell *which* line caused the regression. A bundled diff conflates "hypothesis failed" with "an incidental change broke things". You lose interpretability.

**Test**: pick any line in your diff. Ask "if I reverted just this line, would my experiment still test the same hypothesis?" If yes, that line is incidental — remove it from this commit.

**Cleanup is allowed but isolated**: if you spot dead code or want to refactor, do it as its own commit with a one-line subject like `refactor: rename foo to bar (no-op expected)`. Run the experiment afterwards; val_bpb should be unchanged within seed noise. If it isn't, the "refactor" was a behavioral change — investigate. Treat it as a regular experiment from that point.

This rule complements the simplicity criterion. Deletions are encouraged, but they must stand alone, not piggyback on an experimental change.

## Output format

Once the script finishes it prints a summary like this:

```
---
val_bpb:          0.997900
training_seconds: 300.1
total_seconds:    325.9
peak_vram_mb:     45060.2
mfu_percent:      39.80
total_tokens_M:   499.6
num_steps:        953
num_params_M:     50.3
depth:            8
```

Note that the script is configured to always stop after 5 minutes, so depending on the computing platform of this computer the numbers might look different. You can extract the key metrics from the log file:

```
grep "^val_bpb:\|^peak_vram_mb:" run.log
```

## Logging results

When an experiment is done, log it to `results.tsv` (tab-separated, NOT comma-separated — commas break in descriptions).

The TSV has a header row and 5 columns:

```
commit	val_bpb	memory_gb	status	description
```

1. git commit hash (short, 7 chars)
2. val_bpb achieved (e.g. 1.234567) — use 0.000000 for crashes
3. peak memory in GB, round to .1f (e.g. 12.3 — divide peak_vram_mb by 1024) — use 0.0 for crashes
4. status: `keep`, `discard`, or `crash`
5. short text description of what this experiment tried

Example:

```
commit	val_bpb	memory_gb	status	description
a1b2c3d	0.997900	44.0	keep	baseline
b2c3d4e	0.993200	44.2	keep	increase LR to 0.04
c3d4e5f	1.005000	44.0	discard	switch to GeLU activation
d4e5f6g	0.000000	0.0	crash	double model width (OOM)
```

## Seed re-check for small improvements

A single run conflates true improvement with seed noise. Without verification, lucky runs get baked into the running-best branch and subsequent experiments build on sand.

**Trigger**: a run is about to be marked `keep`, AND `Δval_bpb < 0.002` vs the current best.

**Procedure**:
1. Bump the random seed in `train.py` by 1 (do NOT change anything else — this is itself a surgical commit).
2. Re-run: `uv run train.py > run.log 2>&1`.
3. Compute the average val_bpb across the two runs.
4. If the average still beats the previous best → `keep`. Otherwise → `discard` and `git reset`.

Log both runs in `results.tsv`. Annotate the description on the second row with `(2-seed avg=X.XXXXXX)`.

Improvements with `Δ ≥ 0.002` skip this step.

## Periodic reflection (extended in v3)

Every **20 completed experiments**, run a self-check at the top of the loop. This is autonomous — you do not stop to ask the human, and you do not skip it.

**Procedure**:
1. Read `results.tsv`. Group the last ~20 rows by theme (architecture / optimizer / hyperparameter / regularization / data / etc).
2. Append a short block (~5 lines) to the **Reflections** section of `journal.md`:
   - Current best val_bpb.
   - Common themes among recent `keep`s.
   - Known dead ends (themes producing only `discard`/`crash`).
   - 1–2 unexplored directions to try next.
3. **Process the Open questions section (new in v3)**: scan the bullets you have accumulated. Pick at most one to attempt to resolve in the next experiments — by reading code, running a small targeted experiment, or simply writing down a tentative answer. Move resolved questions to the Reflections section with the answer.
4. Immediately return to the loop. The next `tune` step should pick from the journal's suggestions or from an open question.

**The Open questions discipline (new in v3)**: when you are uncertain about something during the loop — about why a piece of code exists, about whether an assumption holds, about which of several approaches to try first — you cannot ask the human. Instead, **append a one-line bullet to the Open questions section** of `journal.md` and proceed. This is Karpathy guideline #1 ("don't hide confusion") adapted for an autonomous loop: the question is surfaced and durably recorded, but the loop does not stall.

## The experiment loop

The experiment runs on a dedicated branch (e.g. `autoresearch/mar5` or `autoresearch/mar5-gpu0`).

LOOP FOREVER:

1. **(Every 20th iteration)** Run periodic reflection (Reflections + Open questions). Append to `journal.md`. See "Periodic reflection" above.
2. Look at the git state: the current branch/commit we're on.
3. Tune `train.py` with an experimental idea, **surgically** — every changed line must trace to the hypothesis. Draw ideas from `journal.md` (reflections + open questions) if non-empty.
4. Compose `Hypothesis:`, `Assumptions:`, `Prediction:`, and `Tripwire:` lines for the commit message. See "Hypothesis discipline" above.
5. `git commit` with the message from step 4. If diff > 50 added lines, defend it in the body.
6. Run the experiment: `uv run train.py > run.log 2>&1` (redirect everything — do NOT use tee or let output flood your context).
7. Read out the results: `grep "^val_bpb:\|^peak_vram_mb:" run.log`.
8. If the grep output is empty, the run crashed. Run `tail -n 50 run.log`, attempt a fix. If you can't get things to work after more than a few attempts, log `crash` and move on.
9. If total wall time exceeds 10 minutes, kill the run and treat it as a failure (`discard` and revert).
10. **Tripwire check**: if the result hits the tripwire condition you wrote in step 4 (e.g. far worse than predicted, VRAM blew up unexpectedly), spend a minute checking whether the experiment ran as intended (read the diff, re-grep the log). If you find a bug, fix and re-run. Otherwise the result stands.
11. Record the result in `results.tsv`. If the result strongly contradicted your `Prediction`, append a one-line note to `journal.md` Open questions ("Why did X surprise me?").
12. **(If about to keep AND Δval_bpb < 0.002)** Run the seed re-check before deciding.
13. If val_bpb improved, advance the branch — keep the git commit. Otherwise `git reset` back.

**Crashes**: If a run crashes (OOM, bug, etc.), use your judgment. Trivial fix → fix and rerun. Fundamentally broken idea → log `crash`, move on.

**NEVER STOP**: Once the experiment loop has begun, do NOT pause to ask the human if you should continue. Do NOT ask "should I keep going?" or "is this a good stopping point?". The human might be asleep, or gone from the computer, and expects you to continue working *indefinitely* until you are manually stopped. You are autonomous. If you run out of ideas, think harder — read papers referenced in the code, re-read the in-scope files for new angles, combine previous near-misses, try more radical architectural changes. **The journal is your friend** — both Reflections (what you've learned) and Open questions (what you haven't pinned down). The loop runs until the human interrupts you, period.

As an example use case, a user might leave you running while they sleep. ~12 experiments/hour, ~100 over an average sleep. The user wakes to a full pipeline of experimental results.

## Summary of changes

| # | Change | Source |
|---|---|---|
| 1 | `journal.md` initialized at setup | v1 |
| 2 | Per-commit `Hypothesis:` + `Prediction:` lines | v1 |
| 3 | Two-seed re-check for small improvements (Δ < 0.002) | v1 |
| 4 | Periodic reflection every 20 experiments | v1 |
| 5 | Hypothesis block extended with `Assumptions:` + `Tripwire:` | v3 (Karpathy #1, #4) |
| 6 | New section: **Surgical experimental diffs** — every line traces to hypothesis; cleanup goes in separate no-op commit | v3 (Karpathy #3) |
| 7 | Simplicity criterion adds soft cap: diff > 50 added lines must be defended in commit body | v3 (Karpathy #2) |
| 8 | `journal.md` adds **Open questions** section; tripwire surprises and uncertainties go there; reflection processes them | v3 (Karpathy #1, deferred-ask) |
| 9 | Loop step 10: tripwire-triggered sanity check before logging discard | v3 (Karpathy #4) |
