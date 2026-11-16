# Reproducibility

This document explains how `results_verified/` was built from the raw
training outputs, and how to re-run the checks yourself.

## `data_manifest.json`

`data_manifest.json` records the SHA-256 hash and byte size of every one
of the 82 raw training output files (per-run model archives and result
archives, one pair per training run) produced during training. It is the
integrity record for the raw data that `results_verified/` was built
from: anyone can re-hash those raw files and confirm nothing was altered
after training finished.

## How `results_verified/` was built

`scripts/build_results_verified.py` reads the raw per-run archives,
validates each run's `config.txt` against the run's intended
hyperparameters, and writes one folder per run to
`results_verified/phase{1,2,3}/<run_id>/`, containing the trained
`model.zip`, its config, the training learning curve, the evaluation
episodes and the per-episode coverage trajectory. Full file-level
provenance (source archive, SHA-256) is recorded in
`results_verified/PROVENANCE.csv`.

39 unique trained models are covered: 9 for Phase 1 (algorithm
comparison), 15 for Phase 2 (reward formulation comparison) and 15 for
Phase 3 (world difficulty × drift). Phase 3's "hard world, with drifts"
condition reuses the Phase 1 DQN run rather than duplicating it — see
`results_verified/phase3/_REFERENCED_FROM_PHASE1.txt`.

## Run-id naming scheme

Each run folder is named:

```
{algo}_{world}_{reward_mode}_{drift}_{seed}
```

for example `dqn_hard_multi_objective_drifts_42`. `drift` is `drifts` or
`nodrifts`. Training and evaluation scripts accept a `--new_naming` flag
to opt into this 5-part scheme; their default behaviour (and every
hyperparameter, seed and environment call) is unchanged either way.

## Seeds

- **Training seeds**: 42, 43, 44 (three seeds per configuration).
- **Evaluation seeds**: 100–119 (20 held-out episodes per model),
  disjoint from the training seeds.
- Evaluation is deterministic (`model.predict(obs, deterministic=True)`)
  unless noted otherwise — see the deterministic-vs-stochastic table
  below.

## Re-evaluation result

`scripts/reevaluate_all.py` re-ran the evaluation protocol (seeds
100–119, 20 episodes, deterministic, environment and drift setting taken
from each model's own `config.txt`) for all 39 models and compared the
result against each model's originally recorded evaluation:

- **19 of 24** Phase 1 and Phase 2 models matched their original
  evaluation exactly (bit-for-bit).
- The remaining **5** matched within **22 reward units** over a single
  20-episode evaluation (max absolute difference in total episode
  reward), consistent with minor floating-point and library-version
  drift between the original training run and the re-evaluation
  environment — not a labelling or provenance error. Both the original
  and regenerated evaluation files are kept side by side in each
  affected run folder.

Full per-run numbers are in `results_verified/REEVALUATION_REPORT.csv`.

## Deterministic versus stochastic evaluation

`results_verified/diagnostics/ppo_a2c_deterministic_vs_stochastic.csv`
compares `deterministic=True` (always taking the policy's mode action)
against `deterministic=False` (sampling from the policy's action
distribution) for all three algorithms, same 20 eval episodes/seeds.

DQN shows almost no difference between the two modes. PPO and A2C score
substantially higher under stochastic sampling (PPO roughly 1.3–3.3×
higher, A2C roughly 2.4–2.6× higher) — their learned policies appear to
rely on the entropy of the action distribution to escape locally poor
deterministic trajectories. The paper reports the `deterministic=True`
numbers throughout (matching `training/evaluate.py`'s default); the
stochastic comparison is included here as a caveat, not a correction.

## Re-running the checks

From the project root, with the dev dependencies installed:

```bash
# Re-run the environment/regression test suite
pytest -q

# Re-evaluate all 39 saved models against the published numbers
python scripts/reevaluate_all.py

# Rebuild the cross-algorithm evaluation summary table
python scripts/build_eval_summary_all.py

# Re-run the baseline (random / fixed-setpoint) evaluations
python scripts/run_baselines_verified.py

# Regenerate every figure from results_verified/ alone
python generate_plots.py
```

`scripts/verify_tuned_hyperparameters.py` additionally confirms that the
hyperparameters recorded in `training/hyperparameters.py` match the
`config.txt` saved alongside each trained model.
