<!-- README updated with interview context, PDF content, findings and recommendations. -->
# DolarApp KYC Case — Analysis & Findings (Alexis Carrillo)

This repository contains the analysis performed as part of the Credit Analyst take-home exercise provided by DolarApp. The goal was to investigate a measurable drop in the KYC funnel pass rate and produce actionable recommendations to recover conversion.

Interview / Task context (email)
--------------------------------------------------
Hi Alexis!

We are excited to move forward with your interview process for the Credit Analyst. We would like you to showcase your analytical and problem-solving skills through a real exercise which we have previously solved at DolarApp.

Task summary:
- Analyze KYC data (provided: `KYC_Details`, `KYC_Summary`) to identify causes for a decrease in KYC funnel pass rate (people who start and finish KYC).
- Produce a written report (PDF) with findings and recommendations and submit it with your analysis files.

Key dataset paths used in the notebooks
- `KYC_details.csv` — fine-grained per-check fields (image checks, usability, liveness, extraction, similarity, etc.).
- `KYC_summary.csv` — per-user summary records including the final `decision_type` and timestamps.

Executive summary (concise)
--------------------------------------------------
- Date range analyzed: 2023-07-25 → 2023-09-22.
- Baseline pass rate in July / early August: ~88%.
- Observed drop: to ~80% in late August — an ~8 percentage-point reduction equivalent to thousands of rejected users daily.
- Root cause (confirmed): a regression in the Mexico document classification pipeline producing an "Unknown" document type that triggers an "Impossible State" bug (Usability OK → PRECONDITION_NOT_FULFILLED), accounting for the majority of the pass-rate decline.
- Actionable recommendation: immediate rollback or hotfix for the Mexico document classifier plus monitoring and UX improvements to address secondary quality issues.

Detailed findings (from the analysis )
--------------------------------------------------
1) Problem framing
- Pass rate is defined as percent of records with `decision_type` in {PASSED, APPROVED}.
- The drop is a structural regression (not random noise): timing and scale implicate a systemic failure.

2) Methodology
- Merged `KYC_summary` and `KYC_details` on `user_reference`.
- Cleaned anomalies where records show PASSED/APPROVED but lack watchlist screening results.
- Created binary flags for failure modes via heuristics on free-text columns (fraud, quality, tech failures, mismatches).
- Derived grouped features (`combo_*`) for combinatorial analysis of country + document type + age.
- Built a baseline XGBoost classifier with target encoding + count encoding to detect non-linear signals and confirm feature importance (SHAP analysis).

3) Pattern identification
- `combo_country_type` is the top predictor of failure — specific country-document combos show extreme failure rates.
- A standout cohort: `MEX_UNKOWN` (Mexico issuing country with detected document type 'UNKOWN') — shows a near-zero pass rate in the window analyzed. This cohort's traffic rose from ~4% to ~10% of traffic in September.

4) The "Impossible State" bug (confirmed root cause)
- Definition: records with `usability_decision_details == 'OK'` but downstream `image_checks_decision_details == 'PRECONDITION_NOT_FULFILLED'` (an impossible workflow transition).
- Evidence: ~2,198 users in the 'Unknown Doc' cohort experienced this paradoxical flow.
- Correlation: the prevalence of this bug tracks closely (near-perfect inverse correlation, ~-0.64) with the overall pass rate decline. As the bug surged, pass rate dropped in lockstep.
- Additional correlation signals (within the MEX+UNK cohort):
  - `is_mex_unk_unssuported_document`: -0.59
  - `is_mex_unk_fraud`: -0.18
  - `is_mex_unk_quality_fail`: 0.04
  - `is_mex_unk_mismatch`: 0.39
- Interpretation: user error / fraud signals show low or inconsistent correlation with the pass-rate decline; the dominant signal is the technical classification/regression producing Unknown documents and the impossible-state transitions.

5) Modeling and diagnostics
- Model: XGBoost classifier trained on binary flags + encoded categorical features.
- Train/test split: 70/30.
- High model fidelity: ~98% accuracy in reproducing pass/fail labels (classification report captured in the notebook). This indicates the failure modes are systematic and well-captured by the engineered features.
- SHAP analysis highlights `combo_country_type` and the MEX+UNK cohort features as primary drivers.

Recommendations (actionable)
--------------------------------------------------
P0 — Immediate technical remediation (Engineering)
- Roll back the Mexico document-classifier change introduced in late August, or deploy a hotfix implementing fallback logic for documents classified as `UNKOWN`.
- Specific fallback: if `data_type` is `UNKOWN` but `usability_decision_details == 'OK'`, bypass the failing classifier path and route to a robust fallback (e.g., alternative classifier, manual review queue, or lenient acceptance threshold) until the root bug is fixed.

P1 — Observability & Alerting (Operations)
- Implement alerting for the "Impossible State" condition (Usability: OK + image_checks: PRECONDITION_NOT_FULFILLED). Suggested threshold: trigger a P1 alert if the condition exceeds 1% of daily traffic.
- Add daily dashboards tracking `is_mex_unk` prevalence, weekly pass-rate, and the impossible-state rate.

P2 — UX & Product (Medium-term)
- Improve pre-scan guidance and UI to reduce submission of unsupported or partial documents (visual archetypes, required framing). This addresses secondary quality issues.
- Add client-side quick checks (e.g., document boundary detection) to reduce bad captures.

Operational next steps / playbook
- Hotfix / rollback (Engineering) — target immediate deployment (hours).
- Post-fix: validate recovery by monitoring the pass rate and the impossible-state metric for 72 hours.
- Implement alerting and dashboards (1–2 days).
- Product UX improvements and client-side checks (weeks).

Appendix — Data treatment & technical notes
--------------------------------------------------
- Merge: `KYC_summary` + `KYC_details` on `user_reference`.
- Anomaly filter: drop records where `decision_type` in {PASSED, APPROVED} but `watchlist_screening_decision` is null — these appear to be noise.
- Failure heuristics: lists of keywords for fraud, image quality, and technical/extraction failures were used to create binary features (`is_confirmed_fraud`, `is_quality_fail`, `is_tech_data_fail`, etc.). Update keywords if partner vendors change text labels.
- Cohorts: created via `combo_*` features to find high-risk country+document combinations.

Model performance snapshot (as reported in the notebooks)
- Model: XGBoost
- Accuracy: 0.98 (15,597 samples used in the reported metrics)
- Precision/Recall/F1 (class 0/1 reported in notebook) — see `notebooks/01_EDA_and_Cleaning.ipynb` for full matrix and SHAP plots.

Reproducibility & how to run
--------------------------------------------------
1. Create a Python environment and install dependencies listed in `requirements.txt`.
2. Provide the two CSVs (`KYC_details.csv`, `KYC_summary.csv`) and update the notebook path if needed.
3. Open the notebooks in Jupyter or Colab and run them sequentially. The notebooks contain explanatory markdown and code cells performing the analysis described here.
