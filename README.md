# Adversarial Robustness Testing Framework for ML-Based Intrusion Detection

**Status:** Independent ongoing research / applied project. Not yet submitted for peer review.

**Author:** Sabit Md Asad ([sabitpe97.com](https://sabitpe97.com) · [GitHub](https://github.com/Sabit400) · [ResearchGate](https://researchgate.net/profile/Sabit-Md-Asad-2))

## Overview

This project builds a PyTorch-based intrusion detection classifier, attacks it
with two standard adversarial methods (FGSM and PGD), quantifies how much the
attack degrades detection accuracy, then applies adversarial training as a
defense and re-measures robustness. The goal is to make the practical case for
adversarial defense in ML-based security tooling concrete and reproducible,
rather than asserted: every number in the results section below comes from
code in this repository, runnable end-to-end.

This work is a hands-on extension of the adversarial machine learning defense
theme addressed in prior published research, built as an independent,
runnable framework rather than a replication of any specific study.

## Data note

The dataset (`data/generate_data.py`) is **synthetically generated**, modeled
on NSL-KDD-style connection features (duration, byte counts, connection
counts, error rates, protocol/service/flag categories). Real intrusion
datasets (NSL-KDD, CIC-IDS2017) require external download access not
available in this project's development environment. The generator
deliberately introduces label noise and feature overlap between classes so
the resulting classification task is non-trivial — an early version of this
dataset separated perfectly (100% baseline accuracy), which would have made
the adversarial attack experiments meaningless, since there would be no
real decision boundary to perturb. This is stated plainly so results are
read correctly: they demonstrate attack/defense methodology and dynamics,
not real-world detection performance on live network traffic.

## Methodology

![Methodology workflow](diagrams/methodology_workflow.svg)

1. **Dataset** — 30,000 synthetic connection records, binary labeled
   (Normal / Attack), with realistic class overlap.
2. **Baseline model** — a 3-hidden-layer feed-forward PyTorch network,
   trained on clean data only.
3. **Attack generation** — FGSM (single-step) and PGD (10-step, L-infinity
   norm), both implemented directly with PyTorch autograd rather than via
   an external adversarial-ML library, so the mechanics are transparent.
4. **Baseline robustness evaluation** — accuracy measured across an epsilon
   sweep (0 to 0.3) under both attacks.
5. **Adversarial training (defense)** — a second model trained the same way,
   except every training batch is replaced with a PGD-perturbed version
   (epsilon = 0.1, 7 steps) generated against the model's current weights
   before the loss is computed (Madry et al., 2018 style defense).
6. **Hardened model re-evaluation** — the same attacks, same epsilon sweep,
   against the hardened model.
7. **Comparison** — baseline vs. hardened robustness curves, and the
   clean-accuracy cost of the defense.

## Algorithms and tools

| Category | Tools / Algorithms |
|---|---|
| Model | 3-layer feed-forward neural network (PyTorch) |
| Attacks | FGSM (Goodfellow et al., 2015), PGD (Madry et al., 2018) — both L-infinity, implemented from scratch |
| Defense | Adversarial training (PGD-based, on-the-fly per batch) |
| Serving | FastAPI, Pydantic, Uvicorn |
| Core stack | Python, PyTorch, scikit-learn, pandas, matplotlib |

## Results

### Clean accuracy (no attack)

| Model | Clean test accuracy |
|---|---|
| Baseline (undefended) | 94.44% |
| Adversarially trained (hardened) | 94.42% |

The defense cost essentially nothing in clean accuracy (−0.02 pp) — a
favorable outcome, since adversarial training often trades clean accuracy
for robustness.

### Accuracy under attack (PGD, 10 steps)

| Epsilon | Baseline | Hardened | Improvement |
|---|---|---|---|
| 0.00 | 94.44% | 94.42% | — |
| 0.05 | 93.51% | 94.19% | +0.68 pp |
| 0.10 | 87.76% | 92.13% | +4.38 pp |
| 0.15 | 74.38% | 86.46% | +12.08 pp |
| 0.20 | 61.09% | 82.99% | +21.90 pp |
| 0.30 | 49.72% | 75.14% | +25.42 pp |

At the largest tested perturbation budget, adversarial training recovers
**25.4 percentage points** of accuracy that the undefended model loses to a
PGD attack. Full data: [`results/comparison_summary.json`](results/comparison_summary.json).

![Robustness comparison](results/comparison_robustness.png)

PGD is consistently the stronger attack (as expected — it is a multi-step,
iterative refinement of FGSM), and the gap between the two attack curves
widens with epsilon on the baseline model. See
[`results/baseline_robustness_curve.png`](results/baseline_robustness_curve.png)
and [`results/hardened_robustness_curve.png`](results/hardened_robustness_curve.png)
for the individual curves.

### Single-example illustration

At epsilon = 0.1, on a borderline connection record correctly classified as
"Attack" by both models with high confidence:

- **Baseline model**, under PGD attack: flips to "Normal" (61.3% confidence) — a missed detection.
- **Hardened model**, under the same attack: holds "Attack" (86.9% confidence) — correctly robust.

This exact scenario is reproducible via the prototype API (see below).

## Repository structure

```
adversarial-ids-robustness/
├── data/
│   ├── generate_data.py       # synthetic dataset generator
│   └── intrusion_traffic.csv  # generated dataset (30,000 rows)
├── src/
│   ├── preprocessing.py       # encoding, MinMax scaling, train/test split
│   ├── model.py                # PyTorch model architecture
│   ├── train_baseline.py       # baseline (undefended) training
│   ├── attacks.py              # FGSM and PGD implementations
│   ├── evaluate_attacks.py     # robustness sweep across epsilon values
│   ├── adversarial_train.py    # adversarial training defense
│   └── compare_models.py       # baseline vs hardened comparison + plots
├── diagrams/
│   └── methodology_workflow.svg
├── prototype/
│   └── app.py                  # FastAPI side-by-side comparison service
├── results/                    # generated metrics, plots, serialized models
├── requirements.txt
└── README.md
```

## Running it yourself

```bash
pip install -r requirements.txt

# 1. Generate the synthetic dataset
python data/generate_data.py

# 2. Train the baseline (undefended) model
python src/train_baseline.py

# 3. Evaluate baseline robustness under FGSM/PGD
python src/evaluate_attacks.py

# 4. Train the adversarially-hardened model
python src/adversarial_train.py

# 5. Evaluate hardened model robustness (edit evaluate_attacks.py call,
#    or run inline as shown in compare_models.py)

# 6. Generate the comparison plot and summary table
python src/compare_models.py

# 7. Run the prototype comparison API
cd prototype && uvicorn app:app --reload --port 8001
# POST to http://localhost:8001/compare — see app.py for the request schema
```

## Known limitations

- **Synthetic data.** As noted above, results characterize the attack/defense
  methodology, not real-network detection performance.
- **Feature-space realism.** Perturbations are applied and clipped in the
  normalized [0, 1] feature space. This is standard practice for tabular
  adversarial ML research, but does not guarantee every perturbed feature
  vector corresponds to a physically realizable network connection (e.g., a
  fractional byte count or an internally inconsistent flag/protocol
  combination). A production-grade version would need a feasibility
  projection step to enforce valid connection semantics.
- **White-box assumption.** Both FGSM and PGD as implemented here assume
  full access to model gradients (white-box attack). Black-box or
  transfer-attack robustness is not evaluated in this version.

## Relationship to prior published work

This project is motivated by, and methodologically related to, prior
peer-reviewed research on adversarial machine learning defense in
cybersecurity contexts. It is an independent, hands-on framework — built to
demonstrate applied, reproducible capability in adversarial training and
robustness evaluation — rather than a replication of any specific published
study. All code, data generation, and results in this repository are
original work produced for this project.


