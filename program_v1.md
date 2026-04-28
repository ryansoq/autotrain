# autoresearch (v1)

This is an experiment to have the LLM do its own research.

v1 adds three disciplines on top of the original protocol: a per-commit **hypothesis + prediction**, a **seed re-check for small improvements**, and a **periodic reflection** every N experiments. The intent is to convert a fast random walk into a fast, auditable, self-correcting search.

## Setup

To set up a new experiment, work with the user to:

1. **Agree on a run tag**: propose a tag based on today's date (e.g. `mar5`). The branch `autoresearch/<tag>` must not already exist — this is a fresh run.
2. **Create the branch**: `git checkout -b autoresearch/<tag>` from current master.
3. **Read the in-scope files**: The repo is small. Read these files for full context:
   - `README.md` — repository context.
   - `prepare.py` — fixed constants, data prep, tokenizer, dataloader, evaluation. Do not modify.
   - `train.py` — the file you modify. Model architecture, optimizer, training loop.
4. **Verify data exists**: Check that `~/.cache/autoresearch/` contains data shards and a tokenizer. If not, tell the human to run `uv run prepare.py`.
5. **Initialize `results.tsv`**: Create `results.tsv` with just the header row. The baseline will be recorded after the first run. Untracked.
6. **Initialize `journal.md`**: Create an empty `journal.md`. It will accumulate periodic reflections during the loop. Untracked.
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

**The goal is simple: get the lowest val_bpb.** Since the time budget is fixed, you don't need to worry about training time — it's always 5 minutes. Everything is fair game: change the architecture, the optimizer, the hyperparameters, the batch size, the model size. The only constraint is that the code runs without crashing and finishes within the time budget.

**VRAM** is a soft constraint. Some increase is acceptable for meaningful val_bpb gains, but it should not blow up dramatically.

**Simplicity criterion**: All else being equal, simpler is better. A small improvement that adds ugly complexity is not worth it. Conversely, removing something and getting equal or better results is a great outcome — that's a simplification win. A 0.001 val_bpb improvement that adds 20 lines of hacky code? Probably not worth it. A 0.001 val_bpb improvement from deleting code? Definitely keep. An improvement of ~0 but much simpler code? Keep.

**The first run**: Your very first run should always be to establish the baseline, so you will run the training script as-is.

## Hypothesis discipline (new in v1)

Before every commit, the commit message must contain two short lines after the subject:

```
Hypothesis: <one sentence — the mechanism by which this should reduce val_bpb>
Prediction: <expected direction and magnitude, e.g. "↓ 0.001–0.003", "↓ but VRAM +10%", "exploratory, no strong prior">
```

This is cheap and turns each run into a real experiment instead of a random walk. Failures become informative — you can tell "wrong idea" from "wrong execution" after the fact, and the journal can mine these.

If you genuinely cannot articulate a mechanism, write `Prediction: exploratory` — but flag exploratory commits as such so the journal can count them.

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

## Seed re-check for small improvements (new in v1)

A single run conflates true improvement with seed noise. The historical record shows many wins land in the 0.001–0.002 range — exactly the scale of seed noise. Without verification, lucky runs get baked into the running-best branch and subsequent experiments build on sand.

**Trigger**: a run is about to be marked `keep`, AND `Δval_bpb < 0.002` vs the current best.

**Procedure**:
1. Bump the random seed in `train.py` by 1 (do NOT change anything else).
2. Re-run: `uv run train.py > run.log 2>&1`.
3. Compute the average val_bpb across the two runs.
4. If the average still beats the previous best → `keep`. Otherwise → `discard` and `git reset`.

Log both runs in `results.tsv`. Annotate the description with `(2-seed avg=X.XXXXXX)` on the second row.

Improvements with `Δ ≥ 0.002` skip this step — they are large enough that seed noise is unlikely to flip the verdict, and the cost (one extra 5-minute run per small win) is not worth paying for them.

## Periodic reflection (new in v1)

Every **20 completed experiments**, run a self-check at the top of the loop. This is autonomous — you do **not** stop to ask the human, and you do **not** skip it.

**Procedure**:
1. Read `results.tsv`. Group the last ~20 rows by theme (architecture / optimizer / hyperparameter / regularization / data / etc).
2. Append a short block (~5 lines) to `journal.md` covering:
   - Current best val_bpb.
   - Common themes among the recent `keep`s.
   - Known dead ends (themes producing only `discard`/`crash`).
   - 1–2 unexplored directions to try next.
3. Immediately return to the loop. The next `tune` step should pick from the journal's suggestions.

This combats local optima — without it, the loop tends to grind on whatever direction last produced a tiny win until that vein is exhausted.

## The experiment loop

The experiment runs on a dedicated branch (e.g. `autoresearch/mar5` or `autoresearch/mar5-gpu0`).

LOOP FOREVER:

1. **(Every 20th iteration)** Run periodic reflection. Append to `journal.md`. See "Periodic reflection" above.
2. Look at the git state: the current branch/commit we're on.
3. Tune `train.py` with an experimental idea by directly hacking the code. Draw from `journal.md` if it's non-empty.
4. Compose `Hypothesis:` and `Prediction:` lines for the commit message. See "Hypothesis discipline" above.
5. `git commit` with the message from step 4.
6. Run the experiment: `uv run train.py > run.log 2>&1` (redirect everything — do NOT use tee or let output flood your context).
7. Read out the results: `grep "^val_bpb:\|^peak_vram_mb:" run.log`.
8. If the grep output is empty, the run crashed. Run `tail -n 50 run.log` to read the Python stack trace and attempt a fix. If you can't get things to work after more than a few attempts, log `crash` and move on.
9. If total wall time exceeds 10 minutes, kill the run and treat it as a failure (`discard` and revert).
10. Record the result in `results.tsv`.
11. **(If about to keep AND Δval_bpb < 0.002)** Run the seed re-check before deciding. See "Seed re-check" above.
12. If val_bpb improved (lower), advance the branch — keep the git commit. Otherwise `git reset` back to where you started.

The idea is that you are a completely autonomous researcher trying things out. If they work, keep. If they don't, discard. And you're advancing the branch so that you can iterate. If you feel stuck, you can rewind, but this should be very, very rare.

**Crashes**: If a run crashes (OOM, bug, etc.), use your judgment. Trivial fix (typo, missing import) → fix and rerun. Fundamentally broken idea → log `crash`, move on.

**NEVER STOP**: Once the experiment loop has begun (after the initial setup), do NOT pause to ask the human if you should continue. Do NOT ask "should I keep going?" or "is this a good stopping point?". The human might be asleep, or gone from the computer, and expects you to continue working *indefinitely* until you are manually stopped. You are autonomous. If you run out of ideas, think harder — read papers referenced in the code, re-read the in-scope files for new angles, combine previous near-misses, try more radical architectural changes. The journal is your friend. The loop runs until the human interrupts you, period.

As an example use case, a user might leave you running while they sleep. If each experiment takes you ~5 minutes then you can run approx 12/hour, for a total of about 100 over the duration of the average human sleep. The user then wakes up to experimental results, all completed by you while they slept!

## Summary of changes from the original protocol

| # | Change | Where |
|---|---|---|
| 1 | `journal.md` initialized at setup | Setup §6 |
| 2 | Per-commit `Hypothesis:` + `Prediction:` lines | Hypothesis discipline |
| 3 | Two-seed re-check for small improvements (Δ < 0.002) | Seed re-check + Loop §11 |
| 4 | Periodic reflection every 20 experiments | Periodic reflection + Loop §1 |

Everything else is preserved from the original.
