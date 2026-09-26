# TITANIUM-6

**A seven-layer flow-based intrusion detection and response pipeline, evaluated under chronological, twin-free, and cross-dataset protocols on CSE-CIC-IDS2018.**

This repository contains the code behind the paper *"TITANIUM-6: Evaluating a Seven-Layer Flow-Based Intrusion Detection and Response Pipeline under Chronological, Twin-Free, and Cross-Dataset Protocols"* (under review). Every number in the paper is written by these scripts into per-layer manifests and can be checked against them.

> Status: manuscript under review. Code release **R8.0**.

---

## Why this project exists

Flow-based detectors routinely report near-perfect scores on random splits of benchmark captures. Our earlier system, [TITANIUM-4](https://github.com/EngrMuhammadYounas/TITANIUM-4-IDS), scored F1 0.99987 that way and then labelled nothing on a live capture. TITANIUM-6 rebuilds the pipeline on the full timestamped CSE-CIC-IDS2018 release and measures how much of the score survives when temporal, duplicate, and dataset shortcuts are removed one at a time.

| Protocol | Layer-2 F1 |
|---|---|
| Stratified random split | 0.981 |
| Chronological split (within each class) | 0.804 (60-min block 95% CI 0.491–0.904) |
| Chronological, test rows without an exact training twin | 0.672 |
| CIC-IDS2017, no retraining | 0.319 |

Other findings:
- **Thresholds do not transfer.** A threshold holding a 1% false-positive budget on validation gave 9.33% on test.
- **False positives are concentrated.** Two capture days hold 89.3% of all test false positives.
- **The OOD score fails as a novelty filter.** The family classifier's out-of-distribution score ranks zero-day flows *below* known attacks (AUROC 0.431).
- **Gated automation is rare on benign traffic.** A Wilson-bound gate limits automated actions on benign flows to 21 of 421,379.

## Architecture

| Layer | Role | Implementation |
|---|---|---|
| 1 | Online transform | Cleaning rules, 10 derived features, standardisation; identical code in training and serving |
| 2 | Binary gate | LightGBM + Platt calibration, Youden threshold (τ₂ = 0.0793) |
| 3 | Family attribution | 12-class LightGBM (11 families + Normal reject), energy-score OOD gate |
| 4 | Novelty screening | Autoencoder on Normal traffic (Isolation Forest failed validation and is disabled) |
| 5 | Response | Playbook with MITRE ATT&CK IDs, Wilson-gated automation, SHA-256 hash-chained audit log |
| 6 | Explanation | Exact per-class TreeSHAP, counterfactual search in raw feature units |
| 7 | Service | FastAPI REST API: scoped keys, rate limit, request-size limit |

The name counts the six analysis layers; Layer 7 serves them.

## Repository layout

```
00_run_config.py              # choose a profile; writes t6_active_config.json (run first)
01_layer1_preprocessing.py    # cleaning audit, clock fix, per-class chronological split, twin flags
02_layer2_binary_gate.py      # feature selection, model comparison, calibration, thresholds, robustness
03_layer3_family_classifier.py
04_layer4_zero_day.py
05_layer5_soar.py
06_layer6_xai.py
07_layer7_api.py              # starts the REST service (writes endpoint.json)
08_summary.py                 # collects all manifests into results_summary_<profile>.json
09_cross_dataset.py           # zero-shot test on CIC-IDS2017, per-family feature shift
10_fp_diagnostic.py           # false positives by day and hour
11_api_benchmark.py           # end-to-end latency through the API (run as a separate process)
verify_r8.py                  # checks that every documented fix is present and all files compile
RUN_TITANIUM6_R8.ipynb        # notebook that runs the scripts in order
```

## Data

| Dataset | Source | Used for |
|---|---|---|
| CSE-CIC-IDS2018 (10 daily CSV files) | [UNB CIC](https://www.unb.ca/cic/datasets/ids-2018.html); Kaggle copy used: [solarmainframe/ids-intrusion-csv](https://www.kaggle.com/datasets/solarmainframe/ids-intrusion-csv) | training and evaluation |
| CIC-IDS2017, Parquet release without metadata | [Kaggle: dhoogla/cicids2017](https://www.kaggle.com/datasets/dhoogla/cicids2017) | cross-dataset test |

The datasets are not redistributed here. The scripts expect the Kaggle input layout (`/kaggle/input/...`); elsewhere, pass the CSV folder with `python 01_layer1_preprocessing.py --source <folder>` and adjust the CIC-IDS2017 path list near the top of `09_cross_dataset.py`.

## Requirements

Python 3.12 and the versions used for the reported run:

```
numpy==2.0.2  pandas==2.3.3  pyarrow==24.0.0  scipy==1.16.3  scikit-learn==1.6.1
lightgbm==4.6.0  xgboost==3.2.0  torch==2.10.0  shap==0.51.0  fastapi==0.136.1  matplotlib==3.10.0
```

```bash
pip install -r requirements.txt
```

The reported run used a Kaggle CPU session (Intel Xeon 2.00 GHz, 4 logical cores, 31.3 GB RAM).

## Running

Run the scripts in order, each in the same session:

```bash
python 00_run_config.py      # profile 'main' by default
python 01_layer1_preprocessing.py
python 02_layer2_binary_gate.py
python 03_layer3_family_classifier.py
python 04_layer4_zero_day.py
python 05_layer5_soar.py
python 06_layer6_xai.py
python 09_cross_dataset.py
python 10_fp_diagnostic.py
python 08_summary.py
python verify_r8.py
```

For the API latency benchmark, start the service in the notebook (`07_layer7_api.py`), then run the client **as a separate process** so the two do not share an interpreter:

```bash
python 11_api_benchmark.py
```

Re-running `07` stops the previous server and reuses port 8000. If a port is still taken, the live address is written to `l7_R8/endpoint.json`, which `11` reads.

## Profiles

Set `PROFILE` in `00_run_config.py`. Each profile writes to its own output folders, so results never overwrite each other.

| Profile | What changes | Reported in the paper |
|---|---|---|
| `main` | per-class chronological split, semantic invalid-value policy | yes |
| `global_split` | one global time cut for all classes | not run |
| `invalid_median` | all invalid values set to the training median | not run |
| `invalid_zero` | all invalid values set to zero (earlier behaviour) | not run |
| `embargo_60s` | 60-second embargo around partition boundaries | not run |
| `no_shortcut` | retrain without the TCP window and segment features | not run |

## Outputs and reproducibility

Each layer writes a `manifest_*.json` that records its configuration, random seed (42), library versions, input-file hashes, and every reported value. Layer 1 also exports `label_review_worklist.csv` and `raw_sample.csv` for manual label review.

Everything reported comes from a single run with a fixed seed. Autoencoder training is not guaranteed to be bit-for-bit identical across hardware, so a re-run can differ slightly in Layer-4 values.

## Limitations

- The split is chronological **within each class**, not across classes. A single global cut would move 40.04% of rows on average.
- The profiles marked "not run" above were implemented but not evaluated. A manual label audit was not carried out.
- The CIC-IDS2017 release used has no timestamps, so its confidence intervals are flow-level.
- Layer 5 actions are simulated; no traffic is blocked.
- API latency was measured with client and server in one process and is an upper bound.

## Citation

If you use this code, please cite the paper (details will be added on publication):

```bibtex
@article{younas2026titanium6,
  title   = {{TITANIUM-6}: Evaluating a Seven-Layer Flow-Based Intrusion Detection and Response
             Pipeline under Chronological, Twin-Free, and Cross-Dataset Protocols},
  author  = {Younas, Muhammad and Farooq, Muhammad and Amin, Ruhul},
  journal = {IEEE Access},
  year    = {2026},
  note    = {Under review}
}
```

Earlier work:

```bibtex
@article{younas2026titanium4,
  title   = {{TITANIUM-4}: A Low-Latency Four-Layer Intrusion Detection Pipeline and the Limits of
             Benchmark-Trained Thresholds under Domain Shift},
  author  = {Younas, Muhammad and Sadeeda and Ain, Hoor Ul and Shahzad, Huzaifa and Habib, Wasim and
             Siddiqui, Salman Ilahi and Haq, Ihsan Ul and Farooq, Muhammad},
  journal = {International Journal of Innovations in Science and Technology},
  volume  = {8}, number = {4}, pages = {1559--1582}, year = {2026}
}
```

## License

[Choose a license before publishing, e.g. MIT or Apache-2.0, and add a `LICENSE` file.]

## Contact

Muhammad Younas, University of Engineering and Technology, Peshawar: 22pwele5980@uetpeshawar.edu.pk
