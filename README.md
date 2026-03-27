# DNA Melting Temperature Prediction in Mixed Salt Buffers
**Ayman Hussein — Data Science Bootcamp, The Erdős Institute, Spring 2026**

---

## Overview

The **melting temperature (Tm)** of a DNA duplex is the temperature at which half the duplexes are open. Above this temperature, the duplex is thermally unstable and denatures. Accurately predicting Tm for short DNA strands (<100 base pairs) in buffers containing both **monovalent** (e.g., Na⁺, K⁺) and **divalent** (e.g., Mg²⁺) cations is critical for many molecular biology applications.

A key example is **Polymerase Chain Reaction (PCR)**, where optimal amplification requires an annealing temperature 1–2 °C away from the Tm of the target duplex. Since typical PCR buffers contain both monovalent and divalent cations, accurate Tm prediction directly impacts PCR assay design.

---

## Goals

1. **High accuracy (low MAE):** Predict Tm in mixed-salt conditions using a small dataset (~516 samples, 12 sequences).
2. **Single global model:** Develop one simple, interpretable model that works across all salt conditions — in contrast to prior approaches that rely on a decision tree of multiple formulas.

> *It is indeed a challenge to achieve all three of accuracy, generality, and interpretability — but let's see how far we can get!*

---

## Background & Literature

### Primary Reference Model (Owczarzy et al., 2008 — *Owc-2008*)
- Developed empirical correction formulas for Tm prediction in Mg²⁺-dominant (Eq. 16) and mixed-salt buffers (Eqs. 16 + 18–20), using linear regression on 680+ melting temperatures.
- Also uses a monovalent-only formula (Eq. 4) from a prior study (*Owc-2004*).
- A **decision tree** (Fig. 9) selects among the three formulas based on the ratio of ion concentrations.
- Reference Tm at 1 M Na⁺ is computed from nearest-neighbor thermodynamic parameters (*SantaLucia, 1998*).
- Achieved **MAE ≈ 1 °C** on an independent test set.

**Limitations of this approach:**
- Multiple formulas with a concentration-ratio-based decision tree (not physically well-motivated)
- Some parameters are themselves salt-dependent with complex functional forms
- Requires a separate model (nearest-neighbor) to get the 1 M Na⁺ reference Tm
- Highly nonlinear dependencies on both salt concentrations

### Alternative Approach (Huguet et al., 2010 & 2017 — *Unz-2010/2017*)
- Used single-molecule force-rupture experiments on DNA hairpins.
- Proposed **linear (in log[Salt]) salt dependencies** per nearest-neighbor dinucleotide parameter rather than a global salt correction.
- Showed prediction power comparable to thermal melting experiments.
- However, a unified global formula covering all salt conditions was not fully established.

---

## Datasets

### Training / Validation Set
- **Source:** Table 4 of *Owc-2008*
- **Size:** 516 melting temperatures for **12 unique sequences**
- **Lengths:** 15, 20, 25, and 30 bp (3 sequences per length)
- **GC content:** ~0.3, ~0.5, or ~0.7 per length group
- **Salt conditions:** Various combinations of monovalent (0–1 M) and divalent (0–125 mM) concentrations

### Test Set
- **Source:** Table S2 of *Owc-2008* (independent validation set)
- **Filtered to:** 39 data points (sequences melted at 2 µM duplex concentration with a 1 M Na⁺ reference Tm recorded)
- Note: the test set includes sequences of 40 bp and 60 bp — **outside the training length range** — which should be considered when evaluating generalization.

---

## Features Engineering

All feature sets include:
- `length_bp` — duplex length in base pairs
- `log_monovalent_M` — log([monovalent]) in mol/L
- `log_Mg2+_M` — log([Mg²⁺]) in mol/L

Three sequence-representation strategies were compared, each also tested with and without **salt interaction features** (`log[Mon] × feature` and `log[Mg²⁺] × feature`):

| Feature Set | # Features | Sequence Representation |
|---|---|---|
| **Typical** | 4 / 8 (with salt) | GC content |
| **Naive** | 6 / 14 (with salt) | Fraction of each nucleotide (A, T, C, G) |
| **Modern** | 11 / 29 (with salt) | Fraction of 10 dinucleotide steps (AA/TT, AT/TA, TA/AT, CA/GT, GT/CA, CT/GA, GA/CT, CG/GC, GC/CG, GG/CC) — motivated by *Unz-2010/2017* |

---

## Models

Three model families were compared:

| Model | Notes |
|---|---|
| **Linear Regression (LR)** | Interpretable baseline |
| **Generalized Additive Model (GAM)** | Captures nonlinear effects while remaining interpretable |
| **Decision Tree Regressor (DT)** | More flexible but prone to overfitting |

**Cross-validation strategy:** Leave-one-sequence-out (12-fold), ensuring evaluation on held-out sequences rather than held-out salt conditions.

---

## Results

### Cross-Validation (Validation MAE, °C)

| Feature Set | LR | GAM | DT |
|---|---|---|---|
| Typical (4) | 2.60 ± 0.77 | 2.11 ± 0.56 | 5.95 ± 1.58 |
| Typical + salt (8) | 2.58 ± 0.78 | **1.39 ± 0.24** | 6.84 ± 1.65 |
| Naive (6) | 3.20 ± 1.03 | 1.81 ± 0.53 | 8.42 ± 6.57 |
| Naive + salt (14) | 3.19 ± 1.05 | 1.71 ± 0.64 | 7.59 ± 5.73 |
| Modern (11) | 5.67 ± 2.67 | 5.43 ± 2.88 | 9.10 ± 8.47 |
| Modern + salt (29) | 5.67 ± 2.68 | 7.91 ± 3.97 | 8.29 ± 7.28 |

✅ **Best model: GAM with Typical features + salt interactions (8 features)**

### Test Performance (MAE, °C)

| Dataset | Our Model (GAM) | Owc-2008 |
|---|---|---|
| Full training set (516 pts) | 1.17 | 0.58 |
| Full test set (39 pts) | 6.25 | 0.61 |
| Filtered test set — 15–30 bp only (20 pts) | 1.42 | 0.38 |

The elevated error on the full test set is largely driven by 40 bp and 60 bp sequences outside the training length range. After filtering to the training length range, the MAE drops to ~1.4 °C, consistent with validation performance.

---

## Conclusions

While our model does not outperform *Owc-2008*, it offers several advantages:

1. **Minimal feature set** — captures the most relevant physical dependencies (length, GC content, log[salt]) in agreement with literature.
2. **Trained on ~500 data points** — less than half the data used by *Owc-2008* (>1000 total).
3. **Single unified model** — no decision tree of formulas, no separate reference model required.
4. **Competitive performance** — similar to or better than several other prior models in Table 5 of *Owc-2008*, while covering all salt ranges in one formula.

---

## Future Directions

- **More data:** Add length-diverse sequences to improve generalization beyond 15–30 bp.
- **Modern feature set revisited:** Dinucleotide-step features may unlock a simpler linear model given sufficient data.
- **Dataset splitting strategies:** Explore whether 4-length-category structure can be leveraged further without overfitting.
- **Salt–salt interactions:** Physically motivated modeling of monovalent–divalent competition (e.g., via MD simulations).
- **Hybrid models:** Combine linear (interpretable) and nonlinear contributions, potentially using prior models as a scaffold.

---

## References

1. **SantaLucia-1998:** J. SantaLucia, *Proc. Natl. Acad. Sci. U.S.A.* 95(4), 1460–1465. https://doi.org/10.1073/pnas.95.4.1460
2. **Owc-2004:** R. Owczarzy et al., *Biochemistry* 43(12), 3537–3554. https://doi.org/10.1021/bi034621
3. **Owc-2008:** R. Owczarzy et al., *Biochemistry* 47(19), 5336–5353. https://doi.org/10.1021/bi702363u
4. **Unz-2010:** J.M. Huguet et al., *Proc. Natl. Acad. Sci. U.S.A.* 107(35), 15431–15436. https://doi.org/10.1073/pnas.1001454107
5. **Unz-2017:** J.M. Huguet et al., *Nucleic Acids Research* 45(22), 12921–12931. https://doi.org/10.1093/nar/gkx1161
6. **Volo-2018:** A. Vologodskii & M.D. Frank-Kamenetskii, *Physics of Life Reviews* 25, 1–21. https://doi.org/10.1016/j.plrev.2017.11.012

Credit:
Claude made this using my summary PDF.

