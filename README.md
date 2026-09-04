# Module 1 Lab Package: Data Ingestion, Hidden Leakage, Reproducible Pipelines

Three interactive labs on a synthetic SOC environment (`acme-soc`) with a private week-2 gate.
Runtime-agnostic: a step/probe harness runs from the command line today and exposes a JSON manifest for a Fable-style
platform to wire against.

## 1. Setup

Requirements: Python 3.11 or 3.12, 16 GB RAM, about 1 GB disk for generated data.

```
pip install -r requirements.txt
python prepare.py                 # generates both weeks (about 90 s), builds the Lab 2 tables (about 40 s), writes the manifest
```

`prepare.py --scale 0.2` builds a 20% environment for authoring and smoke tests. Thresholds in `fable/thresholds.json`
were calibrated at full scale and do not apply to reduced scales.

The generator is seeded (`--seed 20260901`). Every learner sees identical traps and the answer key is deterministic.

## 2. Layout

```
common.py               constants, helpers, paths
generate_data.py        seeded synthetic environment (week 1 public, week 2 private)
prepare.py              one-shot build: data, caches, manifest
run_lab.py              CLI runner: executes a learner script, applies probes, prints feedback
gate.py                 private week-2 grader (Lab 2 spec gate, Lab 3 artefact gate)
pipeline_src/ingest.py  reference ingestion (the Lab 1 answer key in code form; graders import it)
pipeline_src/enrich.py  the "quick baseline" enrichment; two functions leak by construction; learners may read it in Lab 2
labs/lab1_ingestion.py  learner scaffold, Lab 1
labs/lab2_leakage.py    learner scaffold, Lab 2
labs/lab2_tools.py      model builder and investigation tools for Lab 2
labs/lab3_pipeline.py   learner scaffold, Lab 3
solutions/              reference solutions that pass every probe
fable/probes.py         one probe per step, all feedback text in MESSAGES
fable/manifest.py       emits fable/fable_manifest.json (step graph, decision points, state contract, branches)
fable/thresholds.json   week-2 pass thresholds
data/w1/                learner-visible week 1
data/private/w2/        week 2, grader only. Do not distribute.
data/_cache/            grader tables
```

Files under `data/w1/` that learners must not open: `truth.json`, `_reference_flows.pkl`, `_flowid_map.pkl`. They exist
so the Lab 1 probes can grade. If your platform cannot hide files inside a learner-visible directory, move them and
change `W1` handling in `common.py` and `fable/probes.py`.

## 3. Running a lab from the command line

Learners edit the scaffold in `labs/` and check each step:

```
python run_lab.py --lab 1 --step 2 --script labs/lab1_ingestion.py
python run_lab.py --lab 2 --all  --script labs/lab2_leakage.py       # every step in order, stops at first STOP
python run_lab.py --lab 3 --step 4 --script labs/lab3_pipeline.py
python run_lab.py --lab 1 --all  --solution                          # reference solution, for instructors
```

The learner script must define a dict named `state`. Probes inspect `state`, not the code, so a learner may take any
route that leaves the right objects behind. The per-lab contract:

| Lab | Step | Required keys in `state` |
| --- | --- | --- |
| 1 | 1 | `zeek_a`, `zeek_b`, `flows`, `windows` (DataFrames) |
| 1 | 2 | `flows` cleaned: stripped names, one `Fwd Header Length`, numeric rate columns, `zero_duration`, same row count |
| 1 | 3 | `ts_utc` tz-aware UTC on all four DataFrames |
| 1 | 4 | `table` with the `FLOW_SCHEMA` columns from `common.py` |
| 2 | 1, 2 | `spec`, `df` |
| 2 | 3 | `decisions`: every pool column mapped to `{"action": keep/drop/transform, "reason": ...}` |
| 2 | 4 | `spec` (final) |
| 2 | 5 | `spec`, `threshold`, `ppv_estimate` |
| 3 | 1 | `pipe` (sklearn Pipeline) |
| 3 | 2 | `split`, `train_idx`, `test_idx`, `raw`, `y`, `pipe` fitted on train rows |
| 3 | 3 | `checks`: dict of booleans, at least `train_before_test`, `no_identical_rows_across_split`, `no_session_overlap`, `scaler_fit_on_train_only` |
| 3 | 4 | `artefact_dir` containing `feature_pipeline.joblib` and `schema.json` |

Probe output looks like:

```
Lab 1 Step 4 (0.1s)
[STOP] (l1s4_beacon_killed) You dropped the ephemeral source port as noise and then deduplicated. You destroyed the beacon: 1 of 360 ...
  metrics: {"beacon_rows": 1}
```

`STOP` means the step is not accepted; the branch id in parentheses matches `fable_manifest.json`.

## 4. Wiring to Fable (or any interactive runtime)

`fable/fable_manifest.json` is the integration surface. For each lab it lists steps, the decision prompt and options to
render, the `state` keys the learner must produce, the probe function name, and the branch ids. `messages` maps every
branch id to its feedback text. The runtime needs to:

1. Hold a persistent Python kernel with the package on `sys.path` and a `state` dict.
2. After the learner submits a step, call `fable.probes.PROBES[(lab, step)](state)` and render `Result.branch` and `Result.message`.
3. Keep `data/private/` unreadable by the learner; only `gate.py` reads it.

Decision options in the manifest are descriptive, not enforced. A learner who picks "replace with 0" in the UI but
writes NaN in code passes; the probe looks at the DataFrame. If your runtime wants to enforce the option, gate the code
cell on the choice.

The probes are the graders. If you change the generator, rerun `prepare.py`, then rerun both reference solutions with
`--all --solution` and recalibrate `fable/thresholds.json`.

## 5. Measured numbers, full scale, seed 20260901

These replace the design targets in the specification. PR-AUC throughout.

| Measurement | Value |
| --- | --- |
| Week 1 unique flows / attack flows / base rate | 282,100 / 1,740 / 1 in 162 |
| Week 2 unique flows / attack flows / base rate | 300,510 / 150 / 1 in 2,003 |
| Lab 2, all features, week-1 random CV | 1.00 |
| Lab 2, honest spec, week-1 random CV | 0.54 |
| Lab 2, all features, week 2 | 0.077 |
| Lab 2, only ticket_id and analyst_note removed, week 2 | 0.18 |
| Lab 2, importance-pruned (sensor, port, cmdline kept), week 2 | 0.037 |
| Lab 2, honest spec, week 2 | 0.27 |
| Lab 3, random split, week-1 test estimate | 0.50 |
| Lab 3, time split, week-1 test estimate | 0.17 |
| Lab 3, time split with 10-minute embargo, week-1 test estimate | 0.16 |
| Lab 3, any split, week-2 reality | 0.27 |
| Label-shuffle PR-AUC, honest spec | 0.006 |
| Full-data target encoding on `src_port` with random labels | 0.35 |
| Gate thresholds (Lab 2 / Lab 3) | 0.20 / 0.18 |

Two points the instructor must own. First, the honest per-flow model is weak on week 2 (0.27) because a different red
team on different ports with a Windows source is hard to see in per-flow features. That is the truthful result, and the
gap between 0.54 (estimate) and 0.27 (reality) is the Module 1 lesson. Second, the three Lab 3 splits produce the same
week-2 number. The split changes what you report, not how the model fares. Say that in class; learners will otherwise
reduce it to "time splits are better."

## 6. Lab-by-lab notes

**Lab 1 (self-paced pre-work, about 30 minutes).** Traps in order of discovery: `#fields` header, `Infinity` parsed to
`inf`, the corrupt EVTX chunk in `WS-040-05`, the doubled column, rows dropped are reconnaissance-shaped, sensor-B at
+02:00, 12-hour clock with no AM/PM, three Windows hosts on local time (`WS-030-02`, `WS-030-05`, `SRV-020-09`),
11,996 replay duplicates, 117,974 cross-sensor twins, and the 360-flow beacon that dies if the ephemeral source port is
dropped before dedup. Order matters: twins are only visible after timezone normalisation.

**Lab 2 (in class, 35 minutes).** The pool has seven leaks and one misleading legitimate feature. Any one of
`host_alert_count_24h`, `orig_ttl`, `ticket_id` or `analyst_note` is sufficient for a perfect week-1 score, so removing
them one at a time does not move the number until the last one goes. That is deliberate: leaks are redundant in real
tables. `source_sensor` sits at importance rank 5 to 9 and is caught by `label_rate` or ablation, not by ranking.
`cmdline_window` is the one most SOC engineers will recognise from their own environment. The full-data target-encoding
probe demonstrates on `src_port` because the pool's categoricals are too low-cardinality to show the leak; the message
explains that the construction is the leak, not the column.

**Lab 3 (homework, about 25 minutes).** Failure states are all live: `handle_unknown='error'` dies on week 2's `quic`;
cleaning left outside the artefact dies with a `KeyError`; positional column selection silently mis-scores on the
reordered week-2 export; a scaler fitted on full data is caught by `n_samples_seen_`; an unseeded estimator fails the
two-process hash comparison.

## 7. Known deviations from the specification

- Windows logs ship as python-evtx style XML record dumps (`*.evtx.xml`), not binary `.evtx`. Producing valid binary EVTX
  was not feasible here. The corrupt-chunk, missing-field and local-time defects are preserved. If you want real EVTX,
  a Windows host with `wevtutil` can re-export the XML; the parsers in `ingest.py` then need a python-evtx front end.
- The export CSV carries `Conn State`, `History`, `Fwd TTL` and `Payload Prefix` columns a stock CICFlowMeter would not.
  Acme's exporter is "CICFlowMeter-derived with extensions"; say so in the scenario text.
- The 5-tuple dedup trap became a "drop the ephemeral source port, then dedup" trap, because a 5-tuple dedup does not
  touch a beacon whose source port changes per connection.
- The label-shuffle test does not catch full-data target encoding computed before shuffling. The probe demonstrates the
  leak by recomputing on shuffled labels against a high-cardinality column.

## 8. Cleaning up between cohorts

`rm -rf data outputs && python prepare.py` regenerates everything. Change `--seed` to rotate the environment; then
recalibrate thresholds. `outputs/` holds learner artefacts written by the Lab 3 reference solution.
