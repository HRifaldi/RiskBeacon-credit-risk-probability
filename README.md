# RiskBeacon - Automated Credit Risk Scoring Pipeline with Predictive Analytics and Real-Time Monitoring

> **Minimizing losses from defaults while maintaining credit access for eligible members.**

---

## Table of Contents

- [About the Project](#about-the-project)
- [Business Background](#business-background)
- [Goals and Objectives](#goals-and-objectives)
- [Dataset](#dataset)
- [System Architecture](#system-architecture)
- [Pipeline Overview](#pipeline-overview)
- [Exploratory Data Analysis](#exploratory-data-analysis)
- [Modeling](#modeling)
- [Business Impact Simulation](#business-impact-simulation)
- [Credit Decision Logic](#credit-decision-logic)
- [Project Structure](#project-structure)
- [Technology Stack](#technology-stack)
- [Team](#team)
- [References](#references)

---

## About the Project

**RiskBeacon** is a credit risk assessment system designed to evaluate and monitor member default probability before credit decisions are made.

RiskBeacon transforms historical borrower data into **objective, data-driven credit decisions** through an integrated analytics and modeling pipeline based on the **5C of Credit** framework: Character, Capacity, Capital, Collateral, and Conditions.

The system is designed for both **banking institutions** and **credit cooperatives**, bridging traditional trust-based credit assessment with modern predictive analytics.

### System Components

| Component | Description |
|---|---|
| **Data Pipeline & Quality Monitoring** | Ingestion, transformation, and validation of historical borrower data; handles missing values, outliers, and class imbalance |
| **Exploratory Data Analysis** | Business insights from borrower behavior mapped to the 5C framework |
| **Predictive Modeling** | LightGBM classification model to estimate individual Probability of Default (PD) |
| **Model Explainability** | SHAP-based feature attribution to explain each borrower risk score |
| **Operational Risk Monitoring** | Risk segmentation into three tiers (Low, Medium, High) based on PD score distribution |

---

## Business Background

As financial institutions, from banks to credit cooperatives, continue to expand, subjective credit assessment remains a critical operational risk. Overreliance on social reputation and informal judgment can lead to:

- Higher Non-Performing Loans (NPL)
- Inconsistent and bias-prone credit decisions
- Preventable financial losses from defaults

In the context of **credit cooperatives**, the `Character` dimension in 5C credit analysis is still often based on social reputation rather than measurable data. This creates a blind spot that RiskBeacon is specifically built to address.

> *"Before RiskBeacon, an actual default rate of 6.7% in 150,000 borrower records was not detected at the individual level. After applying the predictive model to a different set of 100,000 borrower records, RiskBeacon identified 28.5% of applicants as default-indicated, revealing hidden risk that previously passed without proper monitoring."*

---

## Goals and Objectives

1. Build an end-to-end automated pipeline from raw Excel files to credit risk scoring
2. Represent the **5C of Credit** framework through feature design and model architecture
3. Generate a **Probability of Default (PD) score** for each member
4. Segment members into **Low / Medium / High Risk** tiers
5. Provide risk-based credit limit recommendations
6. Minimize NPL while preserving credit access for eligible members

---

## Dataset

| Property | Details |
|---|---|
| Source | [Give Me Some Credit - Kaggle](https://www.kaggle.com/datasets/brycecf/give-me-some-credit-dataset) |
| Origin | Credit Fusion Competition, Kaggle 2011 |
| Training rows | 150,000 |
| Features | 11 columns |
| Target variable | `SeriousDlqin2yrs` (1 = default, 0 = non-default) |
| Class imbalance | ~6.7% default rate |

### Feature Mapping to 5C of Credit

| Feature | 5C Dimension | Business Meaning |
|---|---|---|
| `SeriousDlqin2yrs` | - | Target: default within 2 years |
| `RevolvingUtilizationOfUnsecuredLines` | Capacity | Percentage of credit card limit already used |
| `Age` | Character | Member age; proxy for credit history length |
| `NumberOfTime30-59DaysPastDueNotWorse` | Character | Early warning signal: 30-59 day delinquency |
| `DebtRatio` | Capacity | Total monthly installments divided by monthly income |
| `MonthlyIncome` | Capacity | Core repayment ability indicator |
| `NumberOfOpenCreditLinesAndLoans` | Capital | Total active credit obligations |
| `NumberOfTimes90DaysLate` | Character | Strongest signal: 90+ day delinquency history |
| `NumberRealEstateLoansOrLines` | Collateral | Property-backed credit, generally lowers risk |
| `NumberOfTime60-89DaysPastDueNotWorse` | Character | Mid-level delinquency signal |
| `NumberOfDependents` | Capital | Financial burden from family dependents |

---

## System Architecture

```text
Credit Cooperative Excel Data
           |
           v
      Upload to Storage
           |
           v
+----------------------------------+
|        Apache Airflow DAG        |
|  +----------------------------+  |
|  | Extract                    |  |
|  | Schema Validation          |  |
|  | Transform                  |  |
|  | Load to Database           |  |
|  +----------------------------+  |
+----------------------------------+
           |
           v
   Exploratory Data Analysis
    (WoE + IV Feature Selection)
           |
           v
+----------------------------------+
|         LightGBM Model           |
|      Train/Test Split 80:20      |
|      Hyperparameter Tuning       |
| Eval: AUC, Recall, KS-Statistic, |
|             SHAP                 |
+----------------------------------+
           |
           v
    PD Score per Member (0.0-1.0)
           |
      +----+---------+----------+
      |              |          |
      v              v          v
   Low Risk      Medium Risk  High Risk
  High Limit    Medium Limit   Rejected
```

---

## Pipeline Overview

### ETL - Apache Airflow

| Stage | Process |
|---|---|
| **Extract** | Read Excel files and parse into DataFrame |
| **Validate** | Check schema, data types, and duplication; send error notifications to admin if invalid |
| **Transform** | Missing value imputation, outlier handling, encoding, feature engineering |
| **Load** | Store clean data into database / data lake |

### Analytics and Modeling

| Stage | Process |
|---|---|
| **EDA** | Distribution analysis, correlation heatmap, class imbalance checks |
| **Feature Selection** | Weight of Evidence (WoE) + Information Value (IV) |
| **Modeling** | LightGBM with imbalance handling (SMOTE / class weight) |
| **Evaluation** | ROC-AUC, Recall, KS-Statistic, SHAP explainability |

---

## Exploratory Data Analysis

Key EDA findings:

- **Class imbalance**: 93.3% non-default vs 6.7% default, requiring SMOTE or class weighting
- **Top predictors** (based on IV): `NumberOfTimes90DaysLate`, `RevolvingUtilizationOfUnsecuredLines`, `NumberOfTime30-59DaysPastDueNotWorse`
- **DebtRatio** has significant outliers requiring IQR-based capping
- **MonthlyIncome** has around 20% missing values, imputed using median by age group

---

## Modeling

### Model: LightGBM

LightGBM was selected for its strong performance on imbalanced tabular data, built-in missing value handling, and fast training speed suitable for production pipeline deployment.

### Evaluation Metrics

| Metric | Role in Credit Context |
|---|---|
| **ROC-AUC** | Overall model discrimination strength |
| **Recall** | Priority metric, minimizing undetected defaults (False Negative = real financial loss) |
| **KS-Statistic** | Banking industry standard, maximum separation between default and non-default distributions |
| **SHAP** | Explainability support for credit officers and regulators |

> **Why prioritize Recall over Precision?**
> In credit risk, False Negatives (predicting safe borrowers who actually default) directly erode institutional capital. False Positives (rejecting actually safe borrowers) are missed opportunities: painful but less existential. Therefore, Recall is the primary optimization target.

### KS-Statistic Interpretation

| KS Value | Model Quality |
|---|---|
| < 0.20 | Very weak; rebuild recommended |
| 0.20-0.40 | Fairly adequate |
| 0.40-0.60 | Good; production-ready |
| 0.60-0.75 | Very good |
| > 0.75 | Extremely high; validate for overfitting risk |

### Achieved Metrics

- KS-Statistic: `0.58`
- ROC-AUC: `0.92`
- Recall: `0.74`

---

## Business Impact Simulation

| | Before (Baseline) | After (Model Applied) |
|---|---|---|
| Population | 150,000 borrower records | 100,000 borrower records |
| Default rate | 6.7% (actual) | 28.5% (predicted) |
| Interpretation | Real distribution in training data | Model reveals hidden risk in new population |

> *Before: actual default distribution from training data (`n=150,000`). After: model predictions on a new, unseen borrower population (`n=100,000`). The higher predicted default rate reflects the model's ability to detect risk patterns that manual assessment can miss.*

**Estimated financial impact**: From a population of 80,000 borrowers, RiskBeacon flags around 22,800 high-risk applicants. Assuming an average loan value of IDR 5,000,000, proactive rejection could prevent an estimated **IDR 114 billion** in problematic credit exposure.

---

## Credit Decision Logic

PD score thresholds are determined from the optimal KS-Statistic cut-off and PD score percentile distribution in training data.

| Risk Segment | PD Score | Credit Decision | Limit |
|---|---|---|---|
| `Low Risk` | < 0.43 | Approved | High limit with standard interest |
| `Medium Risk` | 0.43-0.53 | Approved with review | Moderate limit with manual officer review |
| `High Risk` | >= 0.53 | Rejected | No credit; refer to financial literacy support |

> Note: Thresholds are calibrated based on portfolio default rate (~6.7%) and optimal KS-Statistic separation point. These are not fixed constants; recalibration is recommended for new borrower populations.

---

## Project Structure

```text
|-- dags/                              # Apache Airflow DAG files
|   `-- etl_pipeline.py
|-- notebooks/
|   |-- 01_eda.ipynb                   # Exploratory Data Analysis
|   |-- 02_feature_engineering.ipynb   # WoE, IV, feature engineering
|   |-- 03_modeling.ipynb              # LightGBM training and evaluation
|   `-- 04_shap_explainability.ipynb   # SHAP explainability
|-- src/
|   |-- preprocessing.py
|   |-- feature_selection.py           # WoE + IV
|   `-- model.py
|-- data/
|   |-- raw/                           # cs-training.csv
|   `-- processed/
|-- outputs/
|   |-- model/                         # Saved LightGBM model
|   `-- reports/                       # Evaluation metrics, SHAP plots
|-- requirements.txt
`-- README.md
```

---

## Technology Stack

| Layer | Technology |
|---|---|
| **Orchestration** | Apache Airflow |
| **Data Processing** | Python, Pandas, NumPy |
| **Feature Engineering** | Scikit-learn, custom Weight of Evidence |
| **Modeling** | LightGBM |
| **Explainability** | SHAP |
| **Imbalance Handling** | Imbalanced-learn (SMOTE) |
| **Evaluation** | Scikit-learn metrics, KS-Statistic |
| **Storage** | PostgreSQL / CSV |
| **Visualization** | Matplotlib, Seaborn |
| **Version Control** | Git, GitHub |

---

## Team

**FTDS Batch 037 - Hacktiv8 | Group 001**

| Name | Role |
|---|---|
| Austin Silitonga | Project Lead - Business Understanding |
| Hernanda Rifaldi | Data Engineer - ETL Pipeline and Airflow |
| Kesyia Patty | Data Analyst - EDA and Business Insights |
| M. Nabil | Data Scientist - Feature Engineering and Modeling |
| Rezha Aulia | Data Scientist - SHAP and KS-Statistic |

---

## References

- Give Me Some Credit Dataset - [Kaggle](https://www.kaggle.com/datasets/brycecf/give-me-some-credit-dataset)
- Credit Fusion and Will Cukierski. *Give Me Some Credit*. Kaggle Competition, 2011
- Basel II/III IRB Approach - Bank for International Settlements
- OJK Regulation No.40/POJK.03/2019 - Commercial Bank Asset Quality
- Siddiqi, N. (2006). *Credit Risk Scorecards*. Wiley
- SHAP: Lundberg and Lee (2017). *A Unified Approach to Interpreting Model Predictions*

---

<p align="center">
  Built with one goal: making credit fairer, more objective, and data-driven.<br>
  <strong>RiskBeacon</strong> · FTDS Batch 037 · Hacktiv8 · 2025
</p>
