# Excess Hospital Readmissions: What Explains the Variation CMS Can't?

An analysis of Medicare's Hospital Readmissions Reduction Program (HRRP), asking whether a
hospital's *structural* characteristics still predict excess readmissions after CMS has already
divided out patient case mix — and finding that one of the strongest apparent effects is an
artifact of CMS's own estimator rather than a fact about hospitals.

**[`hospital_readmission_analysis.ipynb`](hospital_readmission_analysis.ipynb)** — full analysis, executed with outputs.

---

## The question

CMS publishes an **Excess Readmission Ratio** (ERR) for every eligible hospital and condition. It is
already risk-adjusted: age, comorbidities and clinical severity are conditioned out. An ERR above 1.0
means a hospital readmitted more patients than CMS predicted *for patients like theirs*.

So "which patients get readmitted" is the wrong question — CMS has answered it. The open question is:

> After patient case mix is accounted for, do ownership, size, rating and geography still predict
> excess readmissions?

If no, HRRP measures care quality, as designed. If yes, it partly measures what kind of hospital you
are — the core of the long-running safety-net critique.

## Data

Two public CMS Provider Data Catalog files, resolved through the metastore API at runtime (CMS
re-versions download paths every refresh, so pinned URLs rot):

| Dataset | Catalog ID | Grain | Rows |
| --- | --- | --- | --- |
| Hospital Readmissions Reduction Program, FY2026 | `9n3s-kdb3` | hospital × condition | 18,330 |
| Hospital General Information | `xubh-q36u` | hospital | 5,419 |

Measurement window 2021-07-01 to 2024-06-30. Six conditions: heart attack, heart failure, pneumonia,
COPD, hip/knee replacement, CABG. After dropping rows where CMS suppresses the ratio and joining
hospital characteristics: **11,681 records across 2,818 hospitals**, 48.1% of which exceeded their
predicted readmission rate.

## Method

One model — logistic regression — because the question is inferential. The coefficients *are* the
result, so a single well-specified model with honest standard errors says more here than a
leaderboard of classifiers. The choices that actually change the answer in this dataset are not
architectural:

- **Cluster-robust standard errors** on `Facility ID`. A hospital contributes up to six rows, so
  uncertainty reflects 2,818 independent hospitals, not 11,681 pseudo-independent records.
- **Grouped cross-validation** by hospital, never by row, so a hospital's own average can't leak
  across the split. (Measured, not assumed — it turned out to change almost nothing here, and the
  notebook says so.)
- **Informative missingness.** Suppressed discharge counts mean *low volume*, not *unknown*. Dropping
  those rows would silently delete the small hospitals the safety-net critique is about.

### Performance

| Metric | Value | Note |
| --- | --- | --- |
| Accuracy @ 0.50 | 62.9% | vs 51.9% majority-class baseline |
| Balanced accuracy | 62.7% | near-identical — no imbalance artifact |
| AUC | 0.679 | grouped by hospital |
| PR-AUC | 0.663 | base rate 48.1% |
| Brier | 0.225 | well calibrated |
| F1 @ 0.50 | 0.603 | |

Accuracy is reported because it is the first thing most readers ask for, but it is the least
informative number here — it collapses a calibrated probability onto one arbitrary threshold and
hides the precision/recall trade-off. Accuracy varies only between 59.7% and 63.0% across the entire
usable threshold range, while precision moves from 59% to 73%.

### Why isn't accuracy higher?

62.9% invites the obvious objection: shouldn't a good model clear 90%? On this problem, no — a high
score is a symptom, not an achievement. Three checks establish where the ceiling sits (notebook §5.4):

- **Target reconstruction.** The source file carries `Predicted Readmission Rate` and
  `Expected Readmission Rate`, and the excess ratio *is* their quotient — maximum absolute
  discrepancy 0.000065. Those two columns alone score **99.99%**. On this dataset a high accuracy
  usually means the outcome was reconstructed rather than predicted.
- **The outcome is a residual by construction.** CMS has already removed everything systematically
  predictable about the patients. What remains is designed to be noise; a structural model explaining
  95% of it would be evidence that CMS's risk adjustment had failed, not that the model was good.
- **The empirical ceiling.** Predicting each record from *the same hospital's measured performance on
  its other five conditions* — the most informative non-leaking feature available, and strictly more
  direct evidence than any structural attribute — reaches only **59.9% accuracy, AUC 0.636**. That is
  *worse* than the structural model's 62.9% / 0.679. The limit belongs to the problem, not the
  estimator, and no change of algorithm can move it.

A quarter of records (25.3%) sit within ±0.02 of the 1.0 threshold against a measure whose own
standard deviation is 0.082 — hospitals that are indistinguishable but receive opposite labels.

## Findings

**1. Structure predicts excess readmission, but explains little of it.** Grouped-CV AUC 0.679,
accuracy 62.9% against a 51.9% majority-class baseline, McFadden pseudo-R² 0.078, and the best
available flag is only 1.5× better than random. The signal is real and precisely estimated; the great
majority of variation is not explained by hospital structure. On the program's own terms, that is
mostly reassuring.

**2. The apparent small-hospital advantage is an artifact of CMS's estimator.** Hospitals with
suppressed discharge counts exceed prediction least often (36% vs a 48% base; adjusted OR 0.36
[0.33, 0.40]) — but their ERR distribution is compressed to half or two-thirds the spread of larger
hospitals *in every one of the six conditions*. CMS's hierarchical model shrinks low-volume hospitals
toward a national average sitting just below 1.0, which mechanically parks them on the favourable
side of the threshold. Read as a quality result, the coefficient is simply wrong.

**3. Among hospitals large enough to be measured on their own data, smaller is worse.** Set the
shrinkage group aside and the gradient is clean: the lowest reported-volume decile exceeds prediction
82% of the time, the highest around 46%, mean ERR falling from 1.038 to 0.992. Adjusted OR 0.78
[0.75, 0.82] per SD of log volume. This is the direction the safety-net critique predicts.

**4. The condition you are judged on matters more than who you are.** The two largest coefficients are
condition dummies, not hospital attributes — hip/knee OR 1.94 [1.66, 2.26], CABG OR 1.57 [1.32, 1.86],
both against heart failure. The six risk-adjustment models are not calibrated to a common scale.

**5. For-profit ownership shows a small but consistent association.** OR 1.15 [1.02, 1.29], p = 0.023.
Present and consistent in sign across checks; far too small to carry a policy argument alone.

**6. Star rating moves with readmissions, as it should.** OR 0.64 per SD (p ≈ 10⁻⁷⁸), monotone from
73% excess among 1-star hospitals to 31% among 5-star. A coherence check rather than independent
evidence — readmission measures feed into the star rating.

### One trap worth naming

Pooled across conditions, the suppressed-volume group's ERR standard deviation (0.087) is *higher*
than the reported group's (0.078), appearing to refute finding 2 outright. It doesn't: hip/knee has
the widest spread of any condition (SD 0.22) and 83% of its rows are suppressed, so it dominates the
pooled group and drags its variance up. With suppression rates ranging from 12% to 83% across
conditions, the pooled comparison compares different mixtures, not different hospitals. Every
conclusion here rests on the within-condition comparison.

## Limitations

- **Hospital-level, not patient-level.** Nothing here supports a claim about individual patient risk.
- **Cross-sectional and non-causal.** One window overlapping COVID recovery. An odds ratio on
  ownership describes who exceeds prediction, not what would happen if ownership changed.
- **No social risk adjustment.** The public file has no dual-eligible share, DSH status, or area
  deprivation index. Finding 3 establishes that a volume gradient exists, not that payer mix drives it.
- **Shrinkage evidence is inferential.** §4.1 shows a variance-compression pattern consistent with
  hierarchical shrinkage; CMS's model isn't reproducible from this public file, so the mechanism is
  strongly evidenced rather than directly verified.
- **Star rating is endogenous** to the outcome being modelled.
- **Binary outcome discards magnitude.** A hospital at 1.001 and one at 1.63 are identical to the
  logit. The continuous sensitivity check agrees on direction.

## Reproducing

```bash
python3 -m venv .venv && source .venv/bin/activate
pip install pandas numpy scikit-learn matplotlib statsmodels jupyter
jupyter nbconvert --to notebook --execute --inplace hospital_readmission_analysis.ipynb
```

The notebook downloads both CMS files on first run and caches them in `data/`. Figures are written to
`figures/`, and every number quoted above is emitted to `results/findings.json` by the final cell —
the write-up is generated from the fit, not typed by hand.

## Layout

```
hospital_readmission_analysis.ipynb   # the analysis
project_page.html                     # self-contained write-up, portable to a portfolio site
data/                                 # CMS files, downloaded on first run (gitignored)
figures/                              # generated plots
results/findings.json                 # machine-readable results
```
