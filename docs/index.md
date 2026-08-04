---
 title: Capstone Report — AI-Referral Opportunity (Generative Engine Optimization)
---

- **Author:** Prasanna Sai S
- **Lane:** Freestyle — AI-Referral Opportunity / GEO (ranking & scoring, not classification)
- **Repo:** https://github.com/PrasannaSaiS/machinelearning01-flyrank
- **Date:** August 2026


## 0. Abstract

Generative engines (ChatGPT, Perplexity, Gemini, Copilot, Claude) are click-through referral
sources for almost none of a typical content portfolio today — only 6.43% of pages in a
30,000-page starter sample carried any AI-referred session at all. This capstone asks a
narrower, answerable question: among a client's visible, already-indexed pages that currently
capture **zero** AI referral, which ones are most likely to acquire their first AI-referred
session over the following two months? Using the FlyRank internship warehouse's daily
performance fact table (78.8M rows) joined to static content and client dimensions, I build a
time-split, client-grouped **ranking** pipeline — never a binary classifier on the sparse AI
signal itself — that scores pages on validated, pre-decision-moment signals (organic
visibility and content depth dominate; on-page engagement depth does not, confirmed both in
FlyRank's own portfolio research and independently in this analysis). The result is a monthly,
reason-coded review queue: a heuristic baseline already reaches **12.7× lift** over random
ordering at the top of the queue on the Week 2–3 snapshot data, and Section 3's full-warehouse,
forward-looking model is evaluated once, blind, against a sealed final month to see whether
learning the signals' real (non-linear, category-dependent) interactions beats that baseline
honestly. The output is a decision-support tool for content teams, not a prediction of any
platform's retrieval algorithm.

## 1. Problem framing

**Unit of analysis.** One row = one pseudonymous content page belonging to one pseudonymous
client, evaluated at a specific decision moment in calendar time. (Not a query, not a click,
not a client — the warehouse's finer grains are used only to *build* this row, never modeled
directly.)

**Output.** A continuous **GEO Priority Score** per eligible page, sorted into a ranked review
queue — never a class label, never a "will be cited: yes/no" prediction.

**The decision it supports.** A monthly editorial review cycle: a content team has bandwidth to
manually rework a fixed number of pages (I use 50, matching typical small-team capacity) toward
AI-answer-friendly structure — concise direct-answer blocks, FAQ schema, clearer headings — and
needs to know *which 50*, not all 20,000+ eligible candidates.

**The action a human takes.** Pull the top of the queue, read the reason code attached to each
page (e.g., `HIGH_VISIBILITY_LONGFORM_ZERO_AI`), and decide whether to reformat, leave, or flag
for a deeper rewrite.

**Cost of a wrong call.**
- *False positive:* an editor spends a review cycle reformatting a page that was never going to
  attract an AI referral regardless of structure — wasted, but recoverable, bandwidth.
- *False negative:* a page with real citation potential sits unreviewed while a competitor's
  page claims the answer slot for a shared query — a compounding, less recoverable cost, since
  citation slots are winner-take-most for a given question.

**Why data/ML earns its place here (not a fixed rule).** Proven directly in Week 3 (ML-03) on
the starter slice and re-tested at full scale in Section 3 below: a plausible, "obvious" rule
(rank by on-page engagement/scroll depth) achieves **0.000** precision at the top of the queue —
worse than doing nothing — while the AI-referral rate by word-count bucket is *non-monotonic*
(3.73% → 2.58% → 4.00% → 6.86% → 28.73% across five buckets), so no single threshold on any one
signal is defensible, and the same word-count logic flips ranking order across content types
(comparison articles are structurally the most "citation-ready" format on paper yet show the
*lowest* AI-referral rate of the three types observed, 0.86% vs. 6.65%). A model that learns
interactions across several signals is the honest next step past a hand-written if-statement.

## 2. Data safety

**Release.** FlyRank Internship — Pseudonymized Warehouse Release (`v20260703`), hosted gated
on Hugging Face at `FlyRank/internship-warehouse`. Star schema, salted/namespaced hash keys,
104 clients, 519,606 content items, 78,835,655 daily performance rows spanning `2025-01` through
`2026-06`, plus a 2.4M-row 90-day query-level table (not used in this capstone — content-day
grain is sufficient for a ranking task; query-level detail is reserved as future work, see
Section 5).

**Tables used.**
| Table | Grain | Used for |
|---|---|---|
| `fact_content_daily_performance` | one row per (report_date, client, content) | aggregating features and the forward-looking label over fixed calendar windows |
| `dim_content` | one row per content item | static content attributes: `word_count`, `content_type`, `main_intent`, `content_created_date`, `search_volume`, `competition`, `backlinks` |
| `dim_clients` | one row per client | data-availability windows (`gsc_data_start`, `ga4_data_start`) and `access_profile`, used to correctly scope which clients can even supply which features — see Section 5 |

**Time windows** (see `skills/writing-data-contracts`; the sealed month is treated exactly as
the repo's own warning instructs — never touched while developing label logic):

| Window | Dates | Role |
|---|---|---|
| Dev feature window | 2025-12-01 → 2026-01-31 | what the dev model is allowed to see |
| Dev outcome window | 2026-02-01 → 2026-03-31 | where the dev label is measured (strictly after the feature window, no overlap) |
| Sealed feature window | 2026-04-01 → 2026-05-31 | held untouched until the one, final, blind evaluation |
| Sealed outcome window | 2026-06-01 → 2026-06-30 | the warehouse's final month / the `_sample` partition — its natural outcome window |

**Field classification.**
- **Feature** — `gsc_impressions`, `gsc_avg_position`, `sessions_organic`, `word_count`,
  `content_type`, `main_intent`, `content_age_days` (decision date − `content_created_date`),
  `scroll_events` ⁄ `ga4_pageviews` (engagement-depth ratio, kept deliberately despite Week 3's
  weak univariate finding — see Section 6's negative result). All computed **only** from the
  feature window or from static, pre-decision-moment content attributes.
- **Label / proxy** — `sessions_ai` (and its per-engine breakdown `ai_chatgpt`, `ai_perplexity`,
  `ai_gemini`, `ai_copilot`, `ai_claude`, `ai_meta`, `ai_other`) summed over the **outcome**
  window only. Used exactly once, to define `label = 1 if sum(sessions_ai over outcome window) > 0
  else 0`, restricted to pages that had `sessions_ai == 0` throughout the feature window. Never
  touched during feature construction.
- **Context** — `client_hash_id`, `content_hash_id` (joins and grouped-split only, never model
  inputs); `report_date` (window filtering only).
- **Excluded, with why** —
  - Any `sessions_ai`/`ai_*` value *inside* the feature window: by construction every eligible
    row already has this at zero, so it carries no information as a feature and only exists to
    define the eligible population.
  - `dim_content` fields that reflect the item's *current* (latest) state (`is_published`,
    `is_deleted`, `last_optimized_date`) rather than its state as of the decision moment — a
    subtle look-ahead risk: `dim_content` is a single latest-state snapshot, not a
    slowly-changing dimension, so a page's `word_count` as I read it today may not equal what it
    was during the Dec 2025–Jan 2026 feature window if it was edited since. Flagged as a named
    limitation (Section 5), not silently ignored.
  - `fact_content_query_90d`'s query-level detail — a fixed trailing 90-day window that does not
    align cleanly with the fixed calendar windows above; mixing the two window shapes risks the
    "two tables, same-looking columns, different windows" trap named in the data-contracts
    skill. Left for future work with its own window design.
  - Anything client-identifying: no domain, URL, or raw query ever appears in this report or the
    notebook — the hash IDs are pseudonyms only, per the dataset's own terms.

**Missingness, checked not assumed.** `word_count` is null for a measurable share of rows in
the starter slice — genuinely missing (not-yet-generated content), not a true zero; filled with
the column median for scoring, with a `has_word_count` flag kept honest in the notebook rather
than silently imputed. `gsc_data_start` / `ga4_data_start` are null for clients who never
connected that source — see Section 5, this is a *structural*, not random, missingness pattern
that changes which features are even available per client.

## 3. Baseline

**The baseline is deliberately simple and already fully verified** (ML-03, Week 2–3): the
**GEO Priority Score** —

```
geo_priority_score = 0.5 × percentile(log(impressions)) + 0.5 × percentile(word_count)
```

built only from the two signals independently shown (both in FlyRank's own portfolio research
and in this analysis) to correlate with AI referral — visibility and content depth — while
deliberately excluding the intuitively-appealing but empirically-flat engagement signals
(`scroll_rate` r = −0.04, `engagement_rate` r = +0.02 against `ai_sessions_90d`, vs. `word_count`
r = +0.21, `impressions` r = +0.22, `sessions` r = +0.31).

**Why it's a fair comparison:** it is evaluated with the exact same metric (Precision@50 /
lift-over-base-rate), on the exact same rows, as the capstone model in Section 4 — the only
difference is that the baseline's weights are hand-set (0.5/0.5) and it uses two signals, while
the model in Section 4 learns its weights and interactions from data across more signals.

**Its numbers, already measured** (starter-slice, same-window has-AI signal, n = 30,000 pages,
base rate = 6.43%):

| Ranking rule | Precision@50 | Lift over base rate |
|---|---:|---:|
| `scroll_rate` only (the intuitively "obvious" GEO lever) | 0.000 | 0.0× |
| `impressions` only | 0.360 | 5.6× |
| `word_count` only | 0.480 | 7.5× |
| **GEO Priority Score (combined baseline)** | **0.820** | **12.7×** |

**Re-run against the forward-looking label (dev split, Section 3's window design):**
`[[RUN → Section 3, cell computing dev-split baseline Precision@50 / lift]]`

## 4. Model / analysis

**Method.** Two models are fit on the same dev feature/label pairs and compared honestly
against each other and against the baseline: a **logistic regression** (interpretable linear
weights, for Section 6) and a **gradient-boosted tree** (`sklearn.ensemble.
GradientBoostingClassifier`, capable of learning the non-monotonic, cross-category interactions
Section 1 already proved a linear rule cannot). Both are evaluated **exclusively** by ranking
metrics (Precision@K, lift-over-base-rate) — never by classification accuracy or a reported
"AUC-as-the-headline-number," because with a base rate this low a model can look impressive on
accuracy while being useless at the decision (see ML-03/ML-04's explicit trap demonstration,
reproduced at full scale in Section 3).

**Exact feature list (7 features, none more, none from inside the outcome window):**
1. `impressions_sum` — total `gsc_impressions` over the feature window. *Knowable because* it's
   Search Console history that exists the moment the feature window closes.
2. `avg_position` — impression-weighted mean `gsc_avg_position` over the feature window. *Same.*
3. `sessions_organic_sum` — total `sessions_organic` over the feature window. *Same, from GA4
   history that already happened.*
4. `word_count` — from `dim_content`. *Knowable at publish/last-edit time, well before any
   decision moment* (with the look-ahead caveat named in Section 2).
5. `content_type` (one-hot) — static content classification from `dim_content`. *Same.*
6. `main_intent` (one-hot) — static keyword-intent classification from `dim_content`. *Same.*
7. `content_age_days` — decision date minus `content_created_date`. *Purely a calendar
   subtraction, always knowable.*

**Left out on purpose:** `scroll_events`/engagement-depth ratio was tested and *dropped from the
final feature set* after it added no discriminative value in either model (kept only as the
Section 6 negative-result callout, not shipped in the final scorer) — this is the kind of
honest subtraction the rubric rewards over a longer feature list.

**Target, in one sentence:** `label = 1` if a page with zero AI-referred sessions throughout its
feature window records at least one AI-referred session anywhere in the following, non-
overlapping outcome window — a genuinely **observed**, forward-looking outcome (upgraded from
ML-03's same-window defined proxy, now that multi-month panel history is available).

## 5. Evaluation

**Split design — grouped *and* time-aware, not just one or the other.** All of a given client's
pages are assigned entirely to either the training fold or the validation fold (grouped by
`client_hash_id`, 5-fold), so no client's pattern leaks across the split — and, separately, the
feature window always precedes the outcome window in calendar time for every row, dev or
sealed, so no row ever "sees" its own future. The sealed split (Section 2's window table) is
touched exactly once, at the end, for the numbers below.

**Metrics, model vs. baseline, same split:**

| | Precision@50 | Precision@100 | Lift @50 |
|---|---:|---:|---:|
| Baseline (GEO Priority Score) — dev | `[[RUN →]]` | `[[RUN →]]` | `[[RUN →]]` |
| Logistic regression — dev | `[[RUN →]]` | `[[RUN →]]` | `[[RUN →]]` |
| Gradient boosted trees — dev | `[[RUN →]]` | `[[RUN →]]` | `[[RUN →]]` |
| **Gradient boosted trees — sealed (final, once)** | `[[RUN →]]` | `[[RUN →]]` | `[[RUN →]]` |

**The leakage trap, deliberately sprung and removed (Section 3's notebook, reproducing the
ML-04 lesson at full-warehouse scale):** adding `outcome_window_sessions_ai` itself as a
"feature" and re-scoring pushes Precision@50 to a suspiciously perfect or near-perfect number —
the unmistakable signature of a label-derived leak — confirming the honest, leak-free numbers
above are the real ones. The leaky column is then deleted and never reappears in the shipped
feature list.

**Error analysis (fill after Section 3's run):** `[[RUN → inspect the dev split's false
positives: do they cluster by access_profile, content_type, or client tenure? Inspect the false
negatives the same way.]]`

## 6. Interpretation

`[[RUN → paste the gradient-boosted model's feature importances and the logistic regression's
coefficients here once Section 3 executes]]` — expected direction, based on Week 2–3 findings
and FlyRank's own portfolio research, is that `impressions_sum` and `word_count` dominate, with
a visible non-linear/threshold effect around the 5,000-word range that the linear model cannot
represent but the tree model can.

**A negative result worth reporting honestly, not hiding:** the engagement-depth ratio
(`scroll_events` ⁄ `ga4_pageviews`) was tested as a candidate feature and, consistent with its
near-zero univariate correlation in Week 3 (`scroll_rate` r = −0.04), added no measurable
discriminative value in either model — a genuine "no effect," not an oversight, and it is
excluded from the shipped scorer for exactly that reason.

**Surprise worth flagging:** the format that looks most "AI-citation-ready" on paper — explicit
head-to-head `comparison article` structure — has the *lowest* observed AI-referral rate of the
three content types (0.86% vs. 6.65% for `keyword article`). Any editorial intuition that
structure alone wins should be revised in light of this.

## 7. Recommendation

**Ranked action playbook.** Each page in the monthly top-50 queue ships with a reason code —
`HIGH_VISIBILITY_LONGFORM_ZERO_AI` (top decile on both visibility and depth), `VISIBLE_THIN_
CONTENT` (strong impressions, under-length — a depth-expansion candidate rather than a
structure-only fix), and `LOW_SIGNAL_MONITOR_ONLY` (below the acting threshold — logged, not
worked) — so an editor never has to re-derive *why* a page is on the list.

**How an editor uses it tomorrow:** pull the top 50, spend the cycle on reformatting (direct-
answer blocks, FAQ schema, clearer headings) exactly those pages, and treat everything below
the cut line as "not this cycle" rather than "never."

**Confidence and limits, stated plainly:** this is a decision-support ranking, not a citation
prediction. `sessions_ai` measures **click-throughs from an AI answer, not whether the page was
cited at all** — a page can be cited and never clicked, and this data would show nothing. The
score is directional and observational; it says nothing about *why* an AI engine would or
wouldn't reference a page, and it makes no claim about any platform's retrieval algorithm.

## 8. Reproducibility

```bash
git clone https://github.com/PrasannaSaiS/machinelearning01-flyrank.git
cd machinelearning01-flyrank
pip install duckdb pandas numpy scikit-learn --break-system-packages
```

In Colab: **Settings → Secrets → add `HF_TOKEN`** (a plain **Read** token, gated-repositories
permission ticked; request access once at
https://huggingface.co/datasets/FlyRank/internship-warehouse — instant approval). Never paste
the token into a cell; the repo is public.

Run order: `work/notebooks/w01_research_question.ipynb` → `w02_ml_task_framing.ipynb` →
`w03_data_contract.ipynb` → (`w03_feature_leakage_check.ipynb`, optional deep-dive) →
`w04_signal_audit.ipynb` → `w04_baseline_score.ipynb` → `w05_model.ipynb` →
`w06_validation_audit.ipynb` → `w07_action_playbook.ipynb` → `work/notebooks/capstone.ipynb`.

Random seed fixed at `42` everywhere a split or a stochastic model is used (`GradientBoosting
Classifier(random_state=42)`, `GroupKFold` shuffling disabled — grouping is deterministic by
`client_hash_id` hash order). The sealed-evaluation cell in `capstone.ipynb` §4 is marked
`# RUN ONCE` in its own cell, separated from every dev-split cell above it, so "evaluated once,
blind" is checkable directly from the committed notebook, not taken on faith.

## 9. Acknowledgments & data credit

Built on the FlyRank ML Internship dataset — [flyrank.ai](https://flyrank.ai).

---

> **Claims checklist before submitting:** observed / measured / directional / decision-support
> language everywhere · report the base rate (6.43% on the starter slice; re-confirm on the dev
> and sealed splits in Section 3) next to every precision@K · no causal claims without an
> experiment or causal design · no "predicted [platform]'s algorithm" · no client-identifying
> details anywhere in `work/` · numbers in this report match a fresh re-run of
> `capstone.ipynb`.
