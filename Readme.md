# QBER-Aware Quantum-Safe RTGS Orchestration

A reproducible Python implementation of an adaptive cryptographic orchestration framework for RTGS-oriented quantum-safe communications.

The implementation dynamically selects between **PQC-only operation** and **QKD-assisted hybrid operation** using observed QBER, smoothed QBER, usable secret-key availability, key-buffer occupancy, controller state, and recovery dwell.

The final controller uses an **Immediate-Exit Hysteresis** policy: safety-critical conditions trigger immediate fallback to PQC-only operation, while re-entry into hybrid operation requires sustained evidence of channel recovery.

## Overview

The implementation evaluates a two-mode quantum-safe controller:

* **PQC_ONLY** — mandatory quantum-safe fallback based on ML-KEM-768 when QKD-derived material is unavailable or unsuitable.
* **HYBRID** — combines ML-KEM-derived and available QKD-derived key material through a key-derivation function.

RSA-2048 is retained only as a **legacy software-latency benchmark** and is never selected as an operational fallback.

## Controller Inputs

At decision time (t), the controller uses:

* (Q_{\mathrm{obs},t}) — current observed/raw QBER
* (\bar{Q}_t) — EMA-smoothed QBER
* (R_t) — usable secret-key-rate/key-availability proxy
* (B_t) — QKD key-buffer occupancy
* (M_{t-1}) — controller mode at the preceding decision
* (d_t) — recovery-dwell counter

## Immediate-Exit Logic

When operating in `HYBRID`, the controller immediately switches to `PQC_ONLY` if any of the following conditions occurs:

```text
observed QBER >= 0.10
OR usable secret-key rate <= 0
OR key-buffer occupancy < minimum required level
```

Emergency exit uses the raw observation rather than the EMA-smoothed value so that smoothing does not delay a safety-critical fallback.

## Recovery Logic

Re-entry from `PQC_ONLY` to `HYBRID` is deliberately conservative.

Hybrid operation is restored only when the recovery conditions remain satisfied for the required dwell period:

```text
observed QBER < upper recovery threshold
AND EMA-smoothed QBER <= lower threshold
AND usable QKD key material is available
AND key buffer is sufficient
AND the conditions persist for D consecutive decisions
```

This asymmetric policy provides fast fallback while reducing mode flapping during recovery.

## Repository Structure

```text
QBER_RTGS/
├── README.md
├── LICENSE
├── requirements.txt
│
├── notebooks/
│   ├── QBER_RTGS_Implementation.ipynb
│   └── QBER_RTGS_Immediate_Exit_Evaluation.ipynb
│
├── data/
│   └── README.md
│
└── results/
    ├── tables/
    └── figures/
```

## Evaluation

The implementation compares the following controller configurations:

1. Immediate-Exit Hysteresis
2. Original EMA-Exit Hysteresis
3. Dwell-Time controller
4. Single-Threshold controller
5. Static Hybrid
6. Static PQC

The full execution uses:

```text
100 independently seeded trajectories
5,000 decisions per trajectory
matched seeds across controller comparisons
```

A smaller quick-run configuration is available for verifying the implementation.

## Attack and Degradation Scenarios

The controller is evaluated under six channel and telemetry conditions:

1. Step attack
2. Ramp attack
3. On-off attack
4. Boundary oscillation attack
5. QKD denial of service
6. Telemetry underreporting

The scenarios modify channel conditions, usable QKD key material, or reported telemetry while preserving identical seeded trajectories across controller comparisons.

## Evaluation Metrics

The generated outputs include:

* Mean software cryptographic latency
* P95 and P99 latency
* Mode switches
* Redundant switches
* Hybrid-operation fraction
* PQC-only fraction
* Unsafe-hybrid fraction
* Safety-invariant violations
* Number of runs containing violations
* Maximum consecutive violations
* Key-starvation events
* Fallback delay
* Minimum key-buffer occupancy

## Statistical Analysis

A complete trajectory is treated as the independent statistical unit.

The analysis includes:

* bootstrap 95% confidence intervals
* paired Wilcoxon signed-rank tests
* Holm correction for multiple comparisons
* paired effect sizes
* moving-block bootstrap for autocorrelated trajectories
* sensitivity analysis
* ablation analysis
* threshold-pair evaluation
* Pareto analysis

## Benchmark Dataset

The implementation uses an archived synthetic benchmark trace generated for controlled and reproducible evaluation of the QBER-aware orchestration framework.

The archived dataset contains 5,000 records including:

* transaction index
* QBER
* channel state
* cryptographic mode
* classical software latency
* quantum-safe software latency
* latency overhead

The dataset is provided with a permanent DOI for reproducibility.

**Dataset DOI:**
`https://doi.org/10.5281/zenodo.20754899`

The archived trace is a synthetic benchmark and is not presented as a measured physical QKD or production RTGS trace.

## Reproducibility

The full configuration can be enabled in the notebook using:

```python
FULL_RUN = True
```

The full evaluation executes 100 independently seeded trajectories with 5,000 decisions per trajectory.

For a faster implementation check:

```python
FULL_RUN = False
```

The random master seed and derived trajectory seeds are fixed so that controller comparisons can be reproduced.

## Checkpointing

Long executions can optionally save run-level checkpoints to Google Drive.

Completed scenario/controller/run combinations are persisted after execution and skipped automatically when a disconnected Colab runtime is restarted.

This allows the full multi-scenario evaluation to resume without repeating already completed trajectories.

## Output Files

Representative result tables include:

```text
baseline_run_records.csv
baseline_summary_ci.csv
attack_run_records.csv
attack_summary.csv
ablation_run_records.csv
ablation_summary.csv
sensitivity_run_records.csv
sensitivity_summary.csv
key_regime_run_records.csv
key_regime_summary.csv
threshold_pair_run_records.csv
threshold_scored.csv
final_six_scenario_run_records.csv
final_scenario_summary_ci.csv
final_paired_tests.csv
```

Representative figures include:

```text
qber_thresholds.png
controller_tradeoff.png
attack_unsafe_fraction.png
threshold_pareto_frontier.png
immediate_exit_true_risk_by_scenario.png
```

## Important Interpretation

The unsafe-hybrid metric is an operational policy-risk indicator. It should not be interpreted as cryptographic proof of compromise.

Similarly, the reported latency values are software cryptographic benchmarks and do not include physical QKD key generation, WAN propagation, hardware-security-module processing, or end-to-end RTGS settlement latency.

The telemetry-underreporting experiment explicitly evaluates the limitation of a controller whose input measurement can itself be biased. Immediate fallback can respond to unsafe conditions visible in telemetry, but it cannot reconstruct an unobserved true channel condition from deliberately falsified measurements.

## Requirements

The implementation uses Python 3 and common scientific-computing and cryptographic packages.

Typical dependencies include:

```text
numpy
pandas
scipy
matplotlib
seaborn
openpyxl
pycryptodome
```

Additional cryptographic dependencies used for ML-KEM benchmarking should be installed according to the corresponding notebook setup cell.

## Running in Google Colab

1. Open the required notebook in Google Colab.
2. Run the dependency-installation and import cells.
3. Set `FULL_RUN = False` for a short validation run.
4. Run all cells and verify that the pipeline completes successfully.
5. For the complete evaluation, set `FULL_RUN = True`.
6. Enable Google Drive checkpointing when executing the complete multi-scenario experiment.
7. Generated tables and figures are written to the configured output directory.

## Reproducibility Scope

The repository supports reproduction of:

* controller implementation
* simulated channel trajectories
* attack/degradation scenarios
* controller comparisons
* statistical analysis
* sensitivity analysis
* ablation analysis
* key-supply evaluation
* threshold/Pareto analysis
* Immediate-Exit safety evaluation

The implementation does not represent a production RTGS deployment or a physical QKD network.

## License

Add the selected open-source license in the repository root before release.

## Citation

If you use the archived benchmark dataset, please cite the associated Zenodo record:

```text
R Naveen Nithya Kalyan,S Vigneshwari, Adaptive Framework Dataset for Quantum-Safe
Real-Time Gross Settlement (RTGS), Zenodo, 2026.
DOI: 10.5281/zenodo.20754899
```
