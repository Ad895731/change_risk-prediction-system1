# Data Plan, Feature Engineering & Leakage Prevention (ApacheJIT)

## 1. Dataset Strategy & Data Honesty

### A. The Core Reality
In enterprise IT operations, real production Change Request failure logs (e.g., internal ServiceNow tables) are confidential proprietary assets containing infrastructure topology, server hostnames, and security incident post-mortems. 

Instead of relying on synthetic toy data or naive build-break logs (e.g., Travis CI compiler errors), this project is anchored on **ApacheJIT**: a large-scale, peer-reviewed benchmark dataset for **Just-In-Time (JIT) Defect Prediction** across 14 major Apache Software Foundation ecosystems.

### B. Public Proxy & Retraining Architecture
- **Public Proxy Role**: ApacheJIT serves as an empirical, peer-reviewed public proxy for software change risk. Both the project README and resume bullets explicitly state this fact.
- **Enterprise Retraining Design**: The end-to-end preprocessing, training, calibration, and inference pipeline is intentionally designed to be retrained on an enterprise's own internal change and incident history.
- **Strict Rule on Synthetic Data**:
  > **Synthetic data is permitted ONLY for API and UI testing (e.g. mocking change requests in Pytest or Flutter widgets). Synthetic data is NEVER used for reported ML metrics or evaluation benchmarks.**

### C. Dataset Provenance & Scale
- **Publication**: IEEE/ACM International Conference on Mining Software Repositories (**MSR 2022 Data Showcase**) by Hossein Keshavarz & Meiyappan Nagappan:
  - **Paper**: *"ApacheJIT: A Large Dataset for Just-In-Time Defect Prediction"* ([arXiv:2203.00101](https://arxiv.org/abs/2203.00101))
  - **Zenodo Archive**: [DOI: 10.5281/zenodo.5907001](https://doi.org/10.5281/zenodo.5907001)
- **Dataset Scale**:
  - **Total Commits**: $106,674$ software changes across 14 Apache projects (Cassandra, Camel, Flink, Hadoop, HBase, Hive, Ignite, Kafka, Lucene, Spark, etc.).
  - **Bug-Inducing Commits (Failures)**: $28,239$ ($\approx 26.47\%$).
  - **Clean Commits (Successes)**: $78,435$ ($\approx 73.53\%$).

---

## 2. Ground Truth Label Definition & SZZ Methodology

- **Target Variable**: `buggy` $\in \{0, 1\}$ (or `False`/`True`).
- **Ground Truth Identification (The SZZ Algorithm)**:
  - ApacheJIT links Jira issue tracking systems to Git commit histories using the established SZZ (Śliwerski, Zimmermann, Zeller) algorithm.
  - When an issue is logged as a bug fix in Jira, SZZ uses `git blame` and syntactic diff analysis to identify the exact historical commit that introduced the defect.
  - **Class 1 (`buggy = 1`)**: Bug-inducing change that caused a downstream production defect or bug ticket.
  - **Class 0 (`buggy = 0`)**: Clean change that introduced no downstream bug tickets.

---

## 3. Strict Submission-Time Leakage Prevention

To guarantee **zero data leakage**, every feature used for prediction MUST be strictly available at the exact moment the requester submits the change form or opens the Pull Request.

### A. Permitted Submission-Time Features
All 14 Kamei metrics are computable *before* deployment/merge:
1. **Diffusion**: `NS` (Subsystems), `ND` (Directories), `NF` (Files), `Entropy` (Change sprawl across files).
2. **Size / Churn**: `LA` (Lines Added), `LD` (Lines Deleted), `LT` (Lines of Code in touched files prior to change).
3. **Purpose**: `FIX` (Whether the commit is flagged as a bug fix).
4. **History**: `NDEV` (Number of past developers on touched files), `AGE` (Days since last change to touched files), `NUC` (Number of unique changes to touched files).
5. **Developer Experience**: `EXP` (Author cumulative historical changes prior to this change), `REXP` (Recent developer experience), `SEXP` (Author experience in this specific subsystem).

### B. Strictly Banned Post-Submission / Post-Merge Features
The following fields exist in IT databases but **must NEVER enter the feature matrix**:
- `actual_duration_hours`: Only known after deployment finishes.
- `post_deployment_incidents_count`: Target label proxy!
- `approver_decision_time` or `approval_status`: Occurs post-submission.
- `merge_commit_sha` or `post_merge_ci_status`: Generated during or after merge.
- `execution_error_logs`: Known only during deployment.

---

## 4. Time-Based Split: Rationale & Methodology

A fundamental flaw in naive ML projects is using random $K$-fold cross-validation or random `train_test_split` on temporal change data. Random splits allow a model to train on changes from 2018 to predict changes from 2010 (lookahead leakage).

ApacheJIT provides a **strict, chronologically partitioned time-based split**:
1. **Training Partition (`apachejit_train.csv`)**:
   - Commits from the first 14 years (**2003 – 2016**).
   - Pre-balanced subset designed to prevent models from collapsing into majority-class triviality during initial gradient descent / tree splitting.
2. **Testing Partition (`apachejit_test_large.csv`)**:
   - Commits from the final 3 years (**2017 – 2019**).
   - Unbalanced, natural real-world distribution ($N \approx 25,000+$ commits).
   - Evaluates whether a model trained strictly on historical organizational knowledge can predict future deployment failures.

---

## 5. Class Imbalance, Baselines & Evaluation Rigor

### A. The Positive Rate Context (~26% vs. 5–15%)
- In **ApacheJIT**, the positive class (`buggy = 1`) represents **$\approx 26.47\%$** of changes.
- In **Enterprise IT Operations** (e.g., traditional ITIL infrastructure changes), change failure rates typically hover between **$5\%$ and $15\%$**.
- **Mentor Guideline for Academic & Interview Honesty**:
  State clearly in the README and resume:
  > *"In our empirical proxy dataset (ApacheJIT), the defect rate is $\sim 26\%$, whereas enterprise IT change failure rates typically range from $5-15\%$. We evaluate our models on the natural $26\%$ test distribution and also report sensitivity tests at a downsampled $10\%$ rate."*

### B. The PR-AUC Random Baseline Rule
In **Precision-Recall analysis (PR-AUC / Average Precision)**:
$$\text{Baseline}_{\text{PR-AUC}} = \frac{P}{P + N} = \text{Positive Prevalence}$$

- On a test set with **$26\%$ positive rate**, the **random baseline is $0.26$** (not $0.50$!).
- If your trained Random Forest achieves a PR-AUC of **$0.58$**, you must report:
  - **PR-AUC**: $0.58$
  - **Baseline (Positive Rate)**: $0.26$
  - **Relative Lift**: $\frac{0.58}{0.26} = 2.23\times$ improvement over random chance.
- Reporting PR-AUC without its baseline is scientifically meaningless; reporting both demonstrates mastery of imbalanced evaluation.

### C. Secondary Evaluation Metrics
1. **Recall at 80% Precision**:
   If an IT Change Advisory Board mandates that 80% of flagged high-risk changes must be genuine failures (to prevent CAB alert fatigue), what percentage of all defects can the model intercept?
2. **Probability Calibration & Brier Score (Optional / If Time Permits)**:
   Measures mean squared error of predicted probabilities: $BS = \frac{1}{N}\sum (\hat{p}_i - y_i)^2$. Target: $BS < 0.12$.
3. **Confusion Matrix at Justified Decision Thresholds**:
   - **Low Risk**: $\hat{p} < 0.20$ (Eligible for automated / standard approval)
   - **Medium Risk**: $0.20 \le \hat{p} < 0.45$ (Standard peer review)
   - **High Risk**: $\hat{p} \ge 0.45$ (Mandatory CAB review & rollback plan verification)

