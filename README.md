# AI-Powered GHDB Attack Surface Intelligence Platform

An end-to-end research pipeline for **search-engine reconnaissance, ML-based exposure triage, and risk-prioritised attack-surface analysis** — executed entirely against a synthetic target the notebook generates at runtime.

[![Open In Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/USERNAME/REPO/blob/main/ghdb_attack_surface_intelligence.ipynb)
![Python](https://img.shields.io/badge/python-3.10%2B-blue)
![Runtime](https://img.shields.io/badge/runtime-CPU--only-green)
![No API keys](https://img.shields.io/badge/API%20keys-not%20required-green)
![Deterministic](https://img.shields.io/badge/seed-42%20(deterministic)-blueviolet)

> Replace `USERNAME/REPO` in the Colab badge above with your GitHub path after pushing.

---

## Safety and scope — read this first

**All reconnaissance runs against a synthetic environment created by the notebook itself.** No third-party system is contacted, scanned, or queried. In the default configuration (`SEARCH_MODE = "offline"`) the notebook makes **zero outbound network requests**.

- "Acme Research Technologies" is fictional. Its `*.acme.local` hosts are non-resolvable (RFC 6762 reserved namespace).
- Every credential-shaped string is a self-labelling placeholder — `DEMO_NOT_A_REAL_KEY`, `FAKE_PASSWORD_ONLY`, `SYNTHETIC_TOKEN_DO_NOT_USE`.
- The optional live-search backend is a **guarded stub**. It fails closed without a user-supplied provider, API key, and explicit written-authorisation domain allow-list, and blocks any query outside that list. It deliberately implements no rate-limit evasion, user-agent rotation, proxying, CAPTCHA handling, or scraping fallback.

This project implements **reconnaissance and detection only**. It contains no exploitation, credential harvesting, authentication bypass, or payload generation, and must not be extended to add them. Passive reconnaissance against systems you don't own may still be unlawful depending on jurisdiction — get a signed scope document first.

---

## Why this exists

The hard part of operationalising GHDB dorking isn't writing dorks. It's everything downstream:

| Problem | Approach here |
|---|---|
| The public GHDB has thousands of queries — which apply to *this* target? | Technology-aware and TF-IDF-ranked query selection, measured against a static baseline |
| A broad sweep returns mostly noise — what's a real exposure? | TF-IDF + linear classifier over search-visible fields, benchmarked against a strong keyword baseline |
| "A config file is exposed" isn't a severity | Seven-factor weighted geometric risk model, fully decomposable, normalised 0–100 |
| Published dorking work is anecdotal | Complete ground truth ⇒ recall, false-negative rate, and coverage are computed, not asserted |

Most dork tooling reports what it *found*. Because the environment is synthesised here, this reports what it **missed** — which is the number that actually matters.

---

## Results

All figures below are from an actual Colab execution (seed 42). Nothing is hard-coded; re-running reproduces them exactly.

### Detection vs. ground truth

| Metric | Value |
|---|---|
| Indexed documents / ground-truth exposures | 1,600 / 1,152 |
| Queries executed (technology-aware) | 54 of 68 |
| Unique findings | 817 |
| **Precision** | **0.9229** |
| **Recall (detection rate)** | **0.6545** |
| F1 | 0.7659 |
| False-negative rate | 0.3455 |
| Exposure-class coverage | 100% |

### Classification — held-out test split (n = 240)

| Model | Accuracy | Macro-F1 |
|---|---|---|
| Rule-based baseline | 0.7000 | 0.7109 |
| **TF-IDF + Logistic Regression** | **0.8875** | **0.8863** |

- 5-fold CV macro-F1 on train: **0.8750 ± 0.0229** — consistent with test, no overfitting
- High-severity detection: **ROC-AUC 0.9945**, **PR-AUC 0.9842**
- Head-to-head: on the 51 documents where exactly one model is right, ML wins **48–3**

### Experiments

| # | Method | Queries | Findings | Precision | Recall | F1 | High-risk |
|---|---|---|---|---|---|---|---|
| 1 | Static GHDB selection | 68 | 863 | 0.8749 | 0.6554 | 0.7494 | 279 |
| 2 | Technology-aware selection | 54 | 817 | 0.9229 | 0.6545 | 0.7659 | 279 |
| 3 | **Tech-aware + ML filter** | 54 | 759 | **0.9934** | 0.6545 | **0.7891** | 279 |
| 4 | Intelligent (TF-IDF) + ML filter | 26 | 385 | 0.9896 | 0.3307 | 0.4958 | 133 |

The headline result: **technology-aware selection dropped 14 off-stack queries with zero recall cost** (0.6554 → 0.6545), and the ML filter pushed precision from 0.875 to **0.993 while leaving recall completely untouched** — it removed 58 findings, all of them false positives.

Experiment 4 is the honest counterweight: halving the query budget halves recall. These configurations aren't strictly ordered, and the table is meant to expose the trade-off rather than crown a winner.

### Feature ablation (validation macro-F1)

| Configuration | Macro-F1 | Marginal |
|---|---|---|
| URL only | 0.7269 | — |
| URL + title | 0.7982 | +0.0713 |
| URL + title + snippet | 0.7831 | **−0.0151** |
| **All metadata** | **0.9036** | **+0.1205** |

---

## Three findings worth calling out

**1. The first build scored a perfect 1.000 macro-F1 — and that was a failure.**

A synthetic corpus where every backup file has `backup` in its filename yields a clean diagonal confusion matrix, a flat ablation, and an empty error-analysis section. The metrics are real in the sense of being computed, and completely uninformative.

The fix was a **structural camouflage** mechanism: 34% of documents keep their true label but borrow their path, title, and/or snippet from a semantically confusable partner class (`backup_artifact` ↔ `directory_listing`, `admin_interface` ↔ `legacy_application`, `benign_content` ↔ `api_documentation`), and are often relocated to the partner's host. The constraint that keeps it rigorous: **at most two of three search-visible fields are ever borrowed**, so residual true-class signal always survives somewhere. The task stays learnable — but only by a model that integrates evidence *across* fields, which is exactly where a first-match keyword rule loses.

**2. The ablation is non-monotonic, and that's a real result.**

Adding snippet features *reduced* validation macro-F1 (0.7982 → 0.7831) before metadata recovered it to 0.9036. The snippet is the field most often camouflaged, so its high-dimensional bigrams carry the largest share of misleading evidence. It earns its place only once file type, technology, and keyword flags give the model anchors to discount a misleading snippet. This was left in and explained rather than tuned until the curve looked tidy.

**3. Error analysis makes a falsifiable prediction, then tests it.**

If camouflage is what drives the residual error, mistakes should concentrate on camouflaged documents. They do:

| Measure | Value |
|---|---|
| Camouflage base rate (all test docs) | 0.400 |
| Camouflage rate among ML **errors** | 0.963 |
| Camouflage rate among ML **successes** | 0.329 |
| **Lift** | **2.41×** |

Model confidence is also usable as a triage signal — 0.936 mean when correct vs. 0.743 when wrong. In the operational binary view (high-severity detection), the split is 55 TP / 11 FP / **2 FN** / 172 TN: the model very rarely misses the expensive cases.

---

## Pipeline

```
Controlled Synthetic Target
  → Asset Discovery              passive host extraction from the index
  → Technology Fingerprinting    signature matching, confidence-weighted
  → GHDB Query Selection         static │ tech-aware │ TF-IDF ranked │ optional LLM
  → Search Backend               deterministic offline index │ guarded live stub
  → Finding Extraction           normalise, dedupe, track provenance, group
  → Finding Classification       rule baseline vs. TF-IDF + linear model
  → Risk Scoring                 7-factor weighted geometric mean, 0–100
  → Attack-Surface Graph         NetworkX, typed nodes and edges
  → Evaluation                   ground truth, ablation, error analysis
  → Security Report              self-contained HTML + CSV/JSON exports
  → Interactive Dashboard        Gradio
```

### Components

| Component | Responsibility |
|---|---|
| `TargetEnvironment` | Generates the synthetic corpus with ground-truth labels and camouflage |
| `AssetDiscovery` | Recovers the host inventory from indexed content |
| `TechnologyFingerprinter` | Passive technology detection with per-host confidence |
| `GHDBQueryEngine` | Four query-selection strategies over a 68-entry catalog |
| `SearchBackend` (ABC) | `OfflineSearchBackend` (default) / `AuthorizedLiveSearchBackend` (guarded stub) |
| `FindingExtractor` | Deduplication, provenance tracking, grouping |
| `RuleBasedClassifier` | Ordered keyword baseline — a deliberately strong comparator |
| `FindingClassifier` | TF-IDF + Logistic Regression / Linear SVM, selected on validation |
| `RiskScorer` | Transparent, decomposable multi-factor risk model |
| `AttackSurfaceGraph` | Typed graph + centrality metrics and per-asset roll-ups |
| `FindingAnalyst` | LLM narrative when configured; deterministic templates otherwise |
| `EvaluationEngine` | Precision, recall, FN/FP rates, coverage vs. ground truth |
| `ReportGenerator` | Self-contained HTML report and data exports |

### Dork operator grammar

The offline engine implements the operator subset real GHDB dorks rely on: `site:`, `filetype:`, `ext:`, `inurl:`, `intitle:`, `intext:`, `"quoted phrases"`, `-negation`, and `A OR B`. Ranking is a transparent BM25-lite (term frequency with a title boost), ties broken on `doc_id` — so **result ordering is fully deterministic**.

---

## Risk model

Severity is a property of an artefact **in context**, not of the artefact alone. Seven factors, each normalised to (0, 1]:

| Factor | Weight |
|---|---|
| Exposure severity | 0.28 |
| Asset criticality | 0.22 |
| Detection confidence | 0.16 |
| Information sensitivity | 0.13 |
| Exploitability | 0.11 |
| Public discoverability | 0.06 |
| Authentication status | 0.04 |

```
risk = 100 × Π (factorᵢ ^ weightᵢ)      where Σ weightᵢ = 1
```

**Geometric, not arithmetic** — a weighted sum lets one large factor mask a near-zero one, so a benign marketing page on a critical host would still accumulate a middling score. A product collapses when any necessary condition collapses, which is closer to how risk actually behaves.

Bands: Informational 0–24 · Low 25–49 · Medium 50–74 · High 75–89 · Critical 90–100. Observed distribution across 817 findings: 58 / 11 / 508 / 237 / 3.

Every factor value is retained on the finding, so any score can be decomposed and argued with.

---

## Quickstart

### Google Colab (recommended)

1. **File → Upload notebook** → select `ghdb_attack_surface_intelligence.ipynb`
2. **Runtime → Run all**

No API keys, no GPU, no manual downloads. Runs on a standard CPU runtime in well under a minute.

### Local

```bash
git clone https://github.com/USERNAME/REPO.git
cd REPO
pip install pandas numpy scikit-learn matplotlib networkx joblib requests beautifulsoup4 gradio
jupyter notebook ghdb_attack_surface_intelligence.ipynb
```

`gradio` is optional — without it, Section 27 reports the dashboard was skipped and everything else is unaffected.

---

## Configuration

Every switch lives in one `CONFIG` dictionary near the top. **The defaults are the supported path.**

```python
CONFIG = {
    "USE_GOOGLE_DRIVE": False,
    "USE_LLM": False,
    "SEARCH_MODE": "offline",        # "offline" | "authorized_live"
    "RANDOM_SEED": 42,
    "OUTPUT_DIR": "/content/ghdb_project",
    "CORPUS_SIZE": 1600,
    "DECOY_RATE": 0.16,              # cross-class vocabulary noise
    "AMBIGUITY_RATE": 0.34,          # structural camouflage
    "RESULTS_PER_QUERY": 30,
    "LAUNCH_DASHBOARD": False,
    ...
}
```

Things worth trying: raise `AMBIGUITY_RATE` to make classification harder, raise `RESULTS_PER_QUERY` to trade precision for recall, set `LAUNCH_DASHBOARD = True` for the Gradio UI, or change `RANDOM_SEED` to confirm the *relative* results hold while absolute values shift.

### Optional LLM

Provider-agnostic (`AnthropicProvider`, `OpenAIProvider`). When enabled it produces finding explanations, remediation guidance, confidence assessments, and an analyst summary. It is **never** asked to generate exploits, retrieve credentials, or suggest evasion. With `USE_LLM = False`, deterministic templates fill the same interface — no downstream section can tell the difference.

---

## Outputs

Written to `OUTPUT_DIR` (default `/content/ghdb_project/`):

```
data/       corpus.csv
models/     finding_classifier.joblib
results/    findings.csv · findings.json · risk_summary.csv · query_metrics.csv
            experiments.csv · ablation.csv · ml_metrics.json · ground_truth.json
reports/    security_assessment.html
figures/    01_corpus_composition · 02_technology_fingerprint · 03_confusion_matrices
            04_roc_pr_curves · 05_risk_overview · 06_attack_surface_graph
            07_query_effectiveness · 08_experiment_comparison · 09_ablation_study
cache/
```

The HTML report is self-contained — no external CSS, fonts, or scripts — with Executive Summary, Scope, Methodology, Attack Surface, Findings, Risk Summary, Query Effectiveness, ML Performance, Limitations, and Recommendations. It leads with an explicit synthetic-environment disclosure.

---

## Reproducibility

Seed 42 propagates to Python, NumPy, and scikit-learn. Corpus synthesis uses a dedicated RNG so it's independent of incidental global RNG use. The search backend is deterministic by construction.

This notebook was executed twice in a container and once in Google Colab. **All three runs produced identical metrics** — same corpus, same 817 findings, same 0.8863 test macro-F1, same 27 misclassifications.

Leakage controls: the split is at document level *before* any fitting; TF-IDF vocabularies and IDF weights are learned inside the `Pipeline` so CV refits per fold; ground-truth environment metadata (`sensitivity`, `auth_required`, `discoverability`) is **excluded** from the feature matrix; the test split is opened exactly once, after model selection is locked.

---

## Limitations

Stated plainly, because it's what makes the rest usable.

- **The environment is synthetic**, and that bounds every number. Template-generated text has regularities real web content doesn't. What transfers is the *relative* comparison — ML vs. rules, tech-aware vs. static — since all configurations face the identical corpus. Read 0.886 as "beat a strong keyword baseline by ~0.18 macro-F1 on a task with engineered ambiguity", not as a real-world forecast.
- **The offline backend is not a search engine.** Real indexes are partial, temporally unstable, and ranked by hundreds of signals. Determinism buys reproducibility at the cost of realism, and recall here is optimistic.
- **The error floor is a configuration constant.** Errors concentrate on camouflaged documents by design, so the residual error rate is a property of `AMBIGUITY_RATE`, not an estimate of anything external.
- **Small per-class support.** The rarest class contributes ~12 documents per held-out split; differences of a few points are noise. The CV standard deviation is the honest stability indicator.
- **Risk factors are partly unobservable in practice.** Sensitivity and exploitability are known here; a real deployment falls back to coarser class-level priors.
- **Findings are not validated.** The pipeline stops at retrieval and classification by design. The reported false-positive rate is a *classification* FP rate, not a *confirmed-finding* FP rate.
- **68 hand-written queries** model the GHDB; they are not the GHDB.

Full detail in Section 29 of the notebook.

---

## Future work

Ordered by expected value per unit of effort — full detail in Section 30.

1. **Validate against a real, owned estate.** The highest-value next step; the gap between those numbers and these is the actual measure of external validity.
2. **Learn the risk weights** by ordinal regression on analyst-triaged findings instead of asserting them.
3. **Calibrate confidence** (isotonic/Platt) so a threshold-based auto-triage policy becomes defensible.
4. **Temporal modelling** — exposure duration, regression detection, MTTR as a programme KPI.
5. **Cross-encoder reranking** applied only to low-confidence cases, preserving the CPU-only requirement.
6. **Active learning** on the ML/rule disagreement set — the highest-information examples for human labelling.
7. **Adversarial evaluation** — model a defender using noindex, cache removal, and path randomisation, and measure recall degradation.
8. **Beyond web search** — code repos, container registries, CT logs, and paste sites share this pipeline's shape.

---

## Skills demonstrated

Offensive-security reconnaissance methodology · GHDB/dorking · attack-surface discovery · controlled target design with ground truth · information extraction · ML classification with leakage controls · experiment design and ablation · error analysis · risk modelling · graph-based security analysis · quantitative evaluation · reproducible research · automated reporting · optional LLM integration · interactive security visualisation

---

## License

MIT — see `LICENSE`.

**Educational and research use.** The techniques demonstrated here are legitimate defensive and authorised-assessment practice. Applying them to systems you do not own or lack written authorisation to assess may be unlawful. You are responsible for your own use.
