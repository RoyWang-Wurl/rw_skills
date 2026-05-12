---
name: auto-hparams-tune
description: Agent-guided hyperparameter tuning loop. Recursively runs trials, diagnoses underperformance, proposes new params one knob at a time, and converges on the best tune. Use when the user explicitly asks to "auto-tune", "search hyperparameters", "auto hparams tune", or "run a hyperparameter tuning loop". Disk-anchored so it survives context compaction.
---

# auto-hparams-tune — Agent-guided hyperparameter tuning

A `run → diagnose → propose → run …` loop that converges on a strong hyperparameter
set without human intervention after kickoff. The user can interrupt at any
point with **STOP**.

## When to invoke

Only when the user explicitly asks. Do not auto-trigger from ordinary
parameter sweeps the user is hand-driving.

## Disk-anchored design (READ BEFORE EVERY TRIAL)

The agent must NOT rely on conversation memory for trial state. Three files on
disk are the source of truth:

- `kickoff.json` — user-provided inputs locked at kickoff (default params,
  trial runner, target metric, max trials, etc.)
- `auto_tune_log.jsonl` — append-only trial log; one JSON object per line
- This `SKILL.md` — the rules

**Hard rule**: at the top of every iteration of the main loop, re-read this
SKILL.md, the last 100 lines of `auto_tune_log.jsonl`, and `kickoff.json`.
Decisions for the next trial must be derivable from these three files alone.
Context compaction is expected; the loop must survive it.

## Required user inputs (gather at kickoff; ask if any are missing)

1. **Default param set** — the immutable baseline. Saved to `kickoff.json`.
   Treat as read-only forever.
2. **Trial runner** — what to invoke for one trial. Could be:
   - a python script + a function returning a metric, or
   - a shell command whose output is parseable, or
   - a self-contained recipe the user describes.
   If unclear, ask the user.
3. **Target metric** — a single scalar (state higher-better OR lower-better).
   If the user gives a tuple, ask which scalar to optimize.
4. **Training set / data** — provided conceptually; confirm if ambiguous.
5. **Max trials** — user-specified. No default; ask if missing.

## Defaults (mention at kickoff; let user override)

- **Seeds per trial**: 3 (`random_seed=0,1,2`).
- **Marginal-gain threshold**: 0.1% of current best metric value. If the
  best-so-far hasn't improved by more than this in `max(5, ceil(0.1 ×
  max_trials))` consecutive completed trials, terminate.
- **Recent-trial window for "propose"**: last 100 trials in the log.
- **Max proposals per diagnosis cycle**: 3, **one knob per proposal**.
- **Top-N at conclusion**: 20.
- **Failure handling**: retry once on exception/timeout; if it fails again,
  log status `failed` and continue.
- **Localized-area type for diagnosis**: ask the user once on the first
  diagnosis cycle (examples: time-window, feature dimension, per-arm,
  per-seed, per-(arm, hour)). Save their answer to `kickoff.json` and reuse.

## Kickoff procedure

1. Confirm all five required inputs back to the user.
2. Mention the defaults and that they can override.
3. Save `kickoff.json` next to the trial log.
4. State explicitly:

   > "I'll run the default config first as trial #0. After each trial I'll
   > brief you with: trial duration, queue size, best-so-far, and ETA to
   > exhaust the budget. **Type STOP at any time to conclude with current
   > results.** For runs longer than ~30 trials, I recommend wrapping this
   > with `/loop` so each invocation is a clean session that resumes from
   > the log on disk."

5. Initialize the trial queue with one entry: the default config, marked
   `proposing_trial_id=null, rationale="baseline"`.

## The loop

### Top-of-loop self-rehydration (every iteration)

1. Re-read this `SKILL.md`.
2. Read `kickoff.json`.
3. Read `tail -100` of `auto_tune_log.jsonl` (parse each line as JSON).
4. Reconstruct trial queue and best-so-far from the log + any pending
   queue file (or in-memory if same session).

### Run

- Pop the next trial from the queue.
- Execute it for each of the configured seeds.
- Capture per-seed metric, aggregate (mean), wall-clock duration, exit
  status.
- **Append a log entry to `auto_tune_log.jsonl` immediately, before any
  further processing.** This is the single point of state persistence.

### Diagnose (the "check" step)

For the just-finished trial, produce two layers of cause analysis:

1. **Direct cause** — the *what*. The localized area where performance is
   bad (using the localization type confirmed at kickoff). Examples:
   - "Hours 24–48 show 8 pp lower pct_opt than other windows."
   - "ARM_6's predicted RPM is 6 RPM below true RPM."
   - "Seed=2 dropped to 35% while 0/1 stayed at 55%+."
2. **Algorithmic cause** — the *why*. The mechanism behind the symptom.
   - Don't say "RMSE is high" — say "GP posterior reverts to prior because
     length_recency is shorter than time-since-last-observation."
   - Don't say "lift is low" — say "TS argmax pins on baseline because σ=0
     fallback eliminates exploration on starved arms."

Append both causes into the log entry's `diagnosis` field.

### Propose

- Read the most recent 100 log entries (already loaded from rehydration).
- For each algorithmic cause identified, propose at most ONE parameter
  change that would plausibly resolve it. Cap total proposals at 3 per
  cycle.
- **One knob at a time** for clean attribution. Do NOT stack multi-knob
  changes in a single proposal. If a multi-knob hypothesis seems
  necessary, schedule it as a follow-up after single-knob trials finish.
- **Dedup**: skip any proposal whose param-set already exists in the log.
- **Inferring valid ranges**: judge from context (default value's
  magnitude, naming, related knobs). Never propose negative values for
  knobs that are clearly non-negative; never propose values that violate
  obvious algorithmic constraints (e.g., `max_active_non_baseline ≥ n_arms`).
  If you've never tuned this knob before and you're unsure of the range,
  start with a 0.5× and 2× perturbation around the current value.
- Each new queue entry carries: full param dict, the knob that changed,
  the proposing trial id, and the rationale (linking to the diagnosis it
  addresses).
- Persist the queue update to disk so it survives mid-loop interruption.

### Brief the user (after every trial)

Print a short status line. Format:

```
Trial K/N done · duration Ds · queue M · best-so-far X% (seeds [...]) ·
ETA to budget: ~Tmin · type STOP to conclude
```

If running >30 trials, also remind on every 5th trial: "long run — consider
wrapping with /loop for clean sessions."

### Termination check

Exit the loop and go to *Conclude* when ANY of these is true:

- Max trials reached (`completed_count >= max_trials`).
- Best-so-far improved by less than 0.1% of current best over the last
  `max(5, ceil(0.1 × max_trials))` consecutive completed trials.
- Queue is empty AND the propose step generated no new candidates.
- Wall-clock cap exceeded (if user set one).
- User typed **STOP** (case-insensitive substring match, anywhere in their
  most recent message).

### Conclude

Output to the conversation AND save to `auto_tune_summary.md` next to the log:

1. **Top-N table** (default N=20) of trials by metric, with each tune's
   diff vs default and the per-seed values.
2. **Knob importance summary** — which parameters showed clear directional
   effects, which didn't, which combinations were dead ends. Rough
   ranking by "magnitude of mean-metric-change-when-this-knob-varied".
3. **Open questions / next steps** — what wasn't explored, what to try
   if the budget were extended, any user-attention items (e.g., tune
   transferred poorly to held-out, suggestive of overfit).
4. **Path** to `auto_tune_log.jsonl` for the user to inspect.

If concluded due to STOP: note in the summary "loop user-interrupted at
trial K of N".

## Trial log format (`auto_tune_log.jsonl`)

Append-only JSON Lines (one trial per line). Each entry MUST be a single
line so partial writes don't corrupt the file.

```json
{
  "trial_id": 0,
  "ts_started": "2026-05-07T18:30:00",
  "ts_ended": "2026-05-07T18:30:42",
  "duration_s": 42.3,
  "status": "completed",
  "proposing_trial_id": null,
  "rationale": "baseline",
  "knob_changed": null,
  "params": { "...": "full param dict" },
  "params_diff_from_default": {},
  "metrics": {
    "per_seed": {"0": 65.28, "1": 69.20, "2": 62.55},
    "aggregate": 65.68,
    "std": 3.34
  },
  "diagnosis": {
    "direct_cause": "ARM_6 mean predicted RPM is 6 RPM below true; gate locked it out hours 50–120.",
    "algorithmic_cause": "Bernoulli SE fallback returns 0 when N=0 → TS draws fixed at μ → arm never re-explored.",
    "localized_area": "ARM_6, hours 50–120"
  }
}
```

## Hard rules

- **Default params are immutable.** Every trial passes a `deepcopy`.
- **Every executed trial is logged before the next one runs.**
- **Diagnosis must include a *mechanism*** (the algorithmic cause), not
  just a symptom.
- **No duplicate trials.**
- **Bad outcomes are not skipped** — they're logged and used as evidence.
- **Self-rehydrate from disk at the top of every loop iteration.** Do not
  trust conversation memory for trial state.

## STOP behavior

When the user types **STOP** (case-insensitive substring, anywhere in their
message), the agent must:

1. Wait for the currently-running trial to finish (or kill cleanly if safe).
2. Skip propose / next-trial.
3. Jump straight to Conclude.
4. Note in the conclusion: "loop user-interrupted at trial K of N".

## Long runs (>30 trials) — `/loop` wrapper

If the expected run will be long enough to risk context compaction:

1. After kickoff, suggest the user wrap with `/loop`:
   `/loop /auto-hparams-tune resume`
2. In "resume" mode (next user invocation), the agent:
   - Reads `kickoff.json` to recover all original config.
   - Reads the existing `auto_tune_log.jsonl` to recover trial history.
   - Continues from the next un-run queued trial OR generates new
     proposals from the most recent diagnoses.
   - Runs a small batch (default 5 trials), then exits cleanly so the
     `/loop` driver can re-fire with a fresh context.
3. If the user runs the skill standalone (no `/loop`), the agent does
   the whole loop in one session. Re-invoking later still resumes from
   disk.
