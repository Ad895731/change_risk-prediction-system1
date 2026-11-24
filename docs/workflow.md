# 28-Day Workflow, Git Strategy & Metric Logging Plan (ApacheJIT)

## 1. Daily Schedule & Allocation (2–3 Hours/Day)

This plan is structured for a 3rd-year Mathematics & Computing student balancing academic coursework and placement prep. Total expected investment: ~65 hours across 4 weeks.

```
Week 1: ApacheJIT Ingestion, Kamei Features, Time-Split Train/Test, Baseline & RF (Phase 1)
Week 2: FastAPI Backend, Supabase PostgreSQL, RBAC, Policy & Flutter Web (Phase 3-4 screens) (Phase 2)
Week 3: LangChain RAG, pgvector Embeddings, 15-Query Golden Set & Abstention (Phase 3)
Week 4: End-to-End Testing, Hardening, Local-First then Cloud Deployment & Interview Defense
```

---

## 2. Day-by-Day Implementation Roadmap

### Week 1: Phase 1 — Classical ML & Explainability (ApacheJIT)

| Day | Allocated Time | Focus Area | Specific Tasks & Milestone | DoD Checkpoint |
| :--- | :--- | :--- | :--- | :--- |
| **Day 1** | 2.5 hrs | Data Ingestion & EDA | Code `data/apachejit/download_data.py`. Download Zenodo archive (DOI: 10.5281/zenodo.5907001). Write `eda.py` to inspect column counts, missing values, and verify the 26.47% positive rate. | `apachejit_train.csv` (2003–2016) and `apachejit_test_large.csv` (2017–2019) loaded and validated. |
| **Day 2** | 2.5 hrs | Feature Pipeline | Write `features.py`. Implement `ColumnTransformer` with `Log1p` for skewed churn (`LA`, `LD`, `LT`, `AGE`) and `RobustScaler` for diffusion/experience metrics. Fit strictly on train split. | Preprocessor transforms train and test partitions statelessly with zero lookahead leakage. |
| **Day 3** | 2.5 hrs | Baseline Model | Train Logistic Regression with `class_weight='balanced'`. Compute gradient descent convergence, log loss, and odds ratios ($\exp(\beta)$). | Baseline PR-AUC and Confusion Matrix logged against 0.26 baseline. |
| **Day 4** | 3.0 hrs | Random Forest Challenger | Train `RandomForestClassifier` (`n_estimators=150`, `max_depth=10`, `min_samples_leaf=8`). Evaluate out-of-time generalization on 2017–2019 test split. | Challenger beats baseline on PR-AUC by $\ge 0.12$. |
| **Day 5** | 2.5 hrs | Probability Calibration | Implement `calibrate.py` using `CalibratedClassifierCV(method='sigmoid')` (if time allows). Plot reliability diagrams before vs after. | Brier score improves from $> 0.16$ to $< 0.12$; probabilities reflect empirical failure frequencies. |
| **Day 6** | 2.5 hrs | Explainability (SHAP) | Initialize `shap.TreeExplainer` on Random Forest (or LR coefficients). Write `get_top_risk_factors()` returning top 3 Kamei drivers (e.g. Entropy, SEXP, LA). | Given a test commit, function returns top 3 factors with signed contributions in $< 30\text{ ms}$. |
| **Day 7** | 2.5 hrs | Serialization & Tests | Export `.joblib` artifacts. Write unit tests in `pytest` verifying latency, probability bounds ($[0, 1]$), and SHAP schemas. | Phase 1 tests pass in CI; artifacts saved in `ml_pipeline/artifacts/`. |

---

### Week 2: Phase 2 — FastAPI, PostgreSQL, RBAC & Flutter Web (3-4 Screens)

| Day | Allocated Time | Focus Area | Specific Tasks & Milestone | DoD Checkpoint |
| :--- | :--- | :--- | :--- | :--- |
| **Day 8** | 2.5 hrs | Database & Models | Setup local PostgreSQL / Supabase. Write SQLAlchemy models (`User`, `ChangeRequest` with Kamei metrics, `RiskPrediction`, `Approval`, `Outcome`, `AuditLog`). | Alembic migration applies cleanly. |
| **Day 9** | 2.5 hrs | Security & RBAC | Implement bcrypt password hashing and JWT auth. Create FastAPI dependency checkers for `requester`, `approver`, `admin`. | Unit tests confirm unauthorized routes return 401/403. |
| **Day 10** | 3.0 hrs | Policy Engine & ML Loader | Code `policy_rules.py` (blackout windows, rollback requirement). Load ML model on FastAPI startup; build scoring endpoint. | `POST /changes` runs ML inference and validates policies in $< 150\text{ ms}$. |
| **Day 11** | 2.5 hrs | GitHub PR Ingestion & Approvals | Code GitHub API PR parser (`/changes/parse-pr`) and approval decision endpoints. Wire automatic audit logging on every database mutation. | Given a PR URL, API extracts `la`, `ld`, `nf`, and `entropy`. Approvals update state cleanly. |
| **Day 12** | 3.0 hrs | Flutter Web Foundation | Initialize Flutter Web project. Setup HTTP client, JWT token storage, and Screen 1: Auth (Login/Register). | User can log in from browser; JWT stored in session. |
| **Day 13** | 3.0 hrs | Flutter Core Screens | Build Screen 2: Change Submission Form (Manual entry + PR URL fetcher) and Screen 3: Approver Dashboard (Risk Badge & SHAP Cards). | Requester submits change; Approver sees it ranked by risk. |
| **Day 14** | 2.5 hrs | Local Verification & Screen 4 | Build Screen 4: Admin Outcome & Audit View. Verify complete flow locally. Deploy on Render + Supabase only after local is 100% verified. | Local end-to-end flow verified, then cloud deployed over HTTPS. |

---

### Week 3: Phase 3 — LangChain RAG, pgvector & Golden Set Evaluation

| Day | Allocated Time | Focus Area | Specific Tasks & Milestone | DoD Checkpoint |
| :--- | :--- | :--- | :--- | :--- |
| **Day 15** | 2.5 hrs | Runbook Corpus | Author 12 realistic markdown SOPs and incident post-mortems in `data/runbooks/` (Database, K8s, Ingress Gateway, Networking). | Corpus formatted with clear operational headers. |
| **Day 16** | 2.5 hrs | Ingestion & Chunking | Setup LangChain `DirectoryLoader` and `RecursiveCharacterTextSplitter(chunk_size=500, overlap=50)`. Inspect chunk statistics. | Chunks preserve context and headers cleanly. |
| **Day 17** | 2.5 hrs | pgvector Integration | Enable `vector` extension in Supabase PostgreSQL. Configure LangChain `PGVector` with Google Gemini embeddings (768-dim). | Runbooks embedded and stored in Supabase `runbook_embeddings`. |
| **Day 18** | 3.0 hrs | Prompt & Structured Parser | Build LCEL chain with Gemini 1.5 Flash (behind one clean LLM wrapper). Enforce strict citation rules and Pydantic output schema. | Output adheres 100% to JSON schema with recommendations and sources. |
| **Day 19** | 2.5 hrs | Abstention & Golden Set | Implement abstention logic (return `NO_EVIDENCE` when similarity $< 0.65$). Build a **15-query golden evaluation set** (`data/eval/golden_queries.json`). | Golden set measures Hit@3 and Abstention accuracy via evaluation script. |
| **Day 20** | 2.5 hrs | Backend & Flutter Display | Expose `POST /api/v1/changes/{id}/rag-advice`. Build Runbook Advisory card in Flutter Web detail view to display rollback steps & citations. | Advice and citations appear cleanly in Flutter UI. |
| **Day 21** | 2.0 hrs | Fallback & Edge Cases | Verify negative test queries trigger abstention without hallucinating commands. | Zero-evidence query gracefully returns `NO_EVIDENCE` without errors. |

---

### Week 4: Verification, Hardening, Polish & Interview Defense

| Day | Allocated Time | Focus Area | Specific Tasks & Milestone | DoD Checkpoint |
| :--- | :--- | :--- | :--- | :--- |
| **Day 22** | 2.5 hrs | Backend Test Suite | Write comprehensive Pytest test suite covering full lifecycle, edge cases, RBAC violations, and model inference. | Backend test coverage reaches $> 85\%$. |
| **Day 23** | 2.5 hrs | Performance & Free Tier | Profile Render memory consumption ($< 150\text{MB}$ of 512MB limit). Configure SQLAlchemy connection pool (max 5 connections). | Zero memory leak warnings; connection pooling verified. |
| **Day 24** | 2.0 hrs | Evaluation Report | Run `evaluate.py` to produce final PR-AUC vs baseline, Confusion Matrix, and Golden Set Hit@3 artifacts. | Evaluation metrics cleanly generated and saved. |
| **Day 25** | 2.5 hrs | Documentation & README | Finalize root `README.md` with folded phases, learning map, and honest resume bullet templates. | README reads like a production-grade enterprise project. |
| **Day 26** | 2.0 hrs | UI Polish | Refine Flutter UI styling: color-coded risk badges, loading spinners, and error snackbars. | UI feels crisp and responsive on desktop/web browser. |
| **Day 27** | 2.0 hrs | Demo Video & Walkthrough | Record a crisp 3-minute video walkthrough demonstrating submission, risk score, approval, and RAG. | Video link ready for portfolio and resume. |
| **Day 28** | 2.5 hrs | Interview Defense Prep | Rehearse technical defense questions (ApacheJIT vs synthetic data, time-based split vs random split, PR-AUC baseline = positive rate). | Confident and articulate on every architectural decision. |
