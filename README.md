# DEGO Group Project: Team TXC5
## Credit Application Governance Analysis | NovaCred

> **Course:** Data Ecosystems and Governance in Organizations (2606) | Nova SBE
> **Dataset:** `raw_credit_applications.json`, 502 raw credit application records

---

## Team Members & Roles

| Student ID | Name | Role | Contribution | % |
|---|---|---|---|---|
| 73373 | Dariusch Jose Hassunizadeh | Data Engineer | Data loading, cleaning, and quality assessment (`01_data_quality.ipynb`) | 25% |
| 70197 | Luca Isaak | Product Lead | Overall project coordination, repository structure, presentation design, and video editing | 25% |
| 73719 | Patrick Bansbach | Data Scientist | Bias detection, fairness metrics, and proxy discrimination analysis (`02_bias_analysis.ipynb`) | 25% |
| 56730 | Diana Sousa | Governance Officer | Privacy analysis, GDPR mapping, and governance recommendations (`03-privacy-governance.ipynb`) | 25% |

---

## Repository Structure

```
dego-project-team-TXC5/
├── README.md
├── data/
│   ├── raw_credit_applications.json        # Original dataset (502 records)
│   └── processed/
│       ├── cleaned_credit_applications.csv         # After dedup + remediation
│       └── privacy_safe_credit_applications.csv    # After pseudonymization + generalization
├── notebooks/
│   ├── 01_data_quality.ipynb
│   ├── 02_bias_analysis.ipynb
│   └── 03-privacy-governance.ipynb
├── src/
│   └── __init__.py
├── reports/                                # Exported visualizations
└── presentation/                           # Video link / file
```

---

## Executive Summary

NovaCred's credit application dataset contains **intentional data quality issues, statistically significant gender bias, and serious privacy risks**. Our audit identified all major issues across four dimensions. Key headline numbers:

- **DIR = 0.767**: female applicants approved at 76.7% the rate of males, violating the four-fifths rule (threshold: 0.80)
- **p = 0.000694**: the gender-approval association is statistically significant (χ² = 11.51)
- **97.4% of records are k=1**: almost every individual is uniquely identifiable via quasi-identifiers alone
- **6 data quality issue categories** identified and remediated across 500 records

---

## 1. Data Quality Findings

**Notebook:** `01_data_quality.ipynb`

### Dataset Overview
- Raw records loaded: **502**
- Records after deduplication: **500** (2 dropped)
- Columns after flattening nested fields: **35**

### Issue Summary

| Dimension | Issue | Count | % of 500 |
|---|---|---|---|
| Uniqueness | Duplicate `_id` records (app_001, app_042) | 4 rows → 2 removed | 0.4% |
| Uniqueness | Duplicate SSN values (2 SSNs, flagged) | 4 records flagged | 0.8% |
| Completeness | `notes` missing | 500 | 100% |
| Completeness | `loan_purpose` missing | 450 | 90.0% |
| Completeness | `processing_timestamp` missing | 438 | 87.6% |
| Completeness | `decision.rejection_reason` missing (approved cases) | 292 | 58.4% |
| Completeness | `decision.approved_amount` / `decision.interest_rate` missing (rejected cases) | 208 | 41.6% |
| Completeness | `applicant_info.email` (hidden, blank strings) | 7 | 1.4% |
| Completeness | `financials.annual_income` missing | 5 | 1.0% |
| Completeness | `applicant_info.ssn` missing | 4 | 0.8% |
| Completeness | `applicant_info.ip_address` missing | 4 | 0.8% |
| Completeness | `applicant_info.date_of_birth` (hidden, blank strings) | 4 | 0.8% |
| Completeness | `applicant_info.gender` (hidden, blank strings) | 2 | 0.4% |
| Consistency | `financials.annual_income` stored as `object` (should be numeric) | all records | N/A |
| Consistency | `applicant_info.gender` encoded inconsistently: `Male`, `M`, `Female`, `F`, empty | mixed | N/A |
| Validity | Negative `financials.credit_history_months` (min = −10) | 2 records | 0.4% |
| Validity | Negative `financials.savings_balance` (min = −5,000) | 1 record | 0.2% |
| Validity | `financials.debt_to_income` > 1.0 (max = 1.85) | detected | N/A |
| Validity | Malformed email addresses (format check) | 11 records | 2.2% |

> **Note on accuracy:** Direct accuracy verification against a ground truth is not possible with this dataset. Plausibility checks (range bounds, format rules) serve as accuracy proxies.

### Remediation Actions Applied

| Action | Records Affected |
|---|---|
| Hidden missing tokens (`""`, `"na"`, `"-"` etc.) normalized to `NaN` | pipeline-wide |
| `financials.annual_salary` backfilled into `financials.annual_income` (then dropped) | 5 records |
| `applicant_info.date_of_birth` parsed to datetime | pipeline-wide |
| Gender labels standardized (`M`→`Male`, `F`→`Female`) | multiple; 2 set to `Unknown` |
| `loan_purpose` missing values filled with `"Unknown"` | 450 records |
| Malformed emails set to `NaN` | 4 records |
| Negative `savings_balance` set to `NaN` | 1 record |
| Negative `credit_history_months` set to `NaN` | 2 records |
| Duplicate SSNs flagged via `flag_duplicate_ssn` column (not dropped) | 4 records |

**Output:** `data/processed/cleaned_credit_applications.csv` (500 records × 35 columns)

---

## 2. Bias Detection & Fairness Findings

**Notebook:** `02_bias_analysis.ipynb`

### Dataset for Bias Analysis
- Records used: **498** (2 excluded: `Unknown` gender)
- Female applicants: **251** | Male applicants: **247**
- Overall approval rate: **58.4%**

### Disparate Impact Ratio (DIR): Four-Fifths Rule

| Group | Approval Rate | Approved | Total |
|---|---|---|---|
| Female | 50.6% | 127 | 251 |
| Male | 66.0% | 163 | 247 |

**DIR = 50.6% / 66.0% = 0.767** → **BELOW the 0.80 threshold → Disparate impact detected**

### Statistical Significance

| Test | Statistic | Result |
|---|---|---|
| Chi-squared test | χ² = 11.51, df = 1, **p = 0.000694** | Statistically significant (α = 0.05) |
| Cramér's V (effect size) | 0.152 | Small-to-moderate |
| Demographic Parity Difference | 0.154 | UNFAIR (\|DPD\| > 0.10 threshold) |
| Demographic Parity Ratio | 0.767 | UNFAIR (< 0.80 threshold) |

### Primary Driver: Algorithm Risk Score

The `algorithm_risk_score` rejection reason accounts for the majority of denials in both groups, but affects females disproportionately:

| Rejection Reason | Female | Male |
|---|---|---|
| algorithm_risk_score | 100 (39.8% of all females) | 69 (27.9% of all males) |
| insufficient_credit_history | 15 | 8 |
| high_dti_ratio | 8 | 4 |
| low_income | 1 | 3 |

### Bias Persists Across Income Brackets

Female approval rates are lower in every income bracket. The disparity is not explained by income differences alone.

### Financial Profiles Are Comparable

No statistically significant difference between male and female applicants on any financial indicator:

| Feature | Female Mean | Male Mean | p-value |
|---|---|---|---|
| Annual income | 83,770 | 81,296 | 0.326 |
| Credit history (months) | 51.4 | 49.8 | 0.555 |
| Debt-to-income ratio | 0.24 | 0.25 | 0.717 |
| Savings balance | 29,552 | 29,660 | 0.943 |

→ Female applicants are being rejected at higher rates **despite comparable financial profiles**.

### Proxy Discrimination

Features flagged as **HIGH proxy risk** (correlated with both gender and approval outcome):

| Feature | Corr with Gender | Corr with Approval |
|---|---|---|
| `spending_Shopping` | +0.086 | +0.075 |
| `spending_Insurance` | +0.052 | +0.073 |

These spending categories differ systematically by gender and correlate with approval, potentially encoding gender indirectly in the decision algorithm.

---

## 3. Privacy & Governance Findings

**Notebook:** `03-privacy-governance.ipynb`

### PII Inventory

**Direct identifiers** (can identify an individual alone):

| Field | Risk |
|---|---|
| `applicant_info.full_name` | High |
| `applicant_info.email` | High |
| `applicant_info.ssn` | High |
| `applicant_info.ip_address` | High |
| `applicant_info.date_of_birth` | High |

**Quasi-identifiers** (identifying when combined):
- `applicant_info.gender`, `applicant_info.zip_code`, `financials.annual_income`

### Re-identification Risk (k-Anonymity)

| State | Unique QI Combinations | Share with k=1 |
|---|---|---|
| Before generalization | 493 / 500 (ratio: 0.986) | **97.4%** |
| After generalization (ZIP prefix + age bands) | 135 / 500 | **25.4%** |

→ Almost every individual is uniquely identifiable via gender + ZIP + income alone, even without direct identifiers.

### Privacy-Preserving Transformations Applied

1. **Pseudonymization:** salted SHA-256 hashing applied to `email` and `ssn`; a stable `applicant_pseudo_id` derived from hashed SSN. Raw identifiers (`full_name`, `email`, `ssn`, `ip_address`) removed from the analytical dataset.
2. **Quasi-identifier generalization:** ZIP code truncated to 3-digit prefix; exact date of birth replaced with age band (`<25`, `25–34`, `35–44`, `45–54`, `55–64`, `65+`).

> **Important:** Pseudonymization is not anonymization. The dataset remains personal data under GDPR if a re-linking key exists. Appropriate access controls must apply.

**Output:** `data/processed/privacy_safe_credit_applications.csv` (500 records × 34 columns)

### GDPR Mapping

| GDPR Principle | Article | Finding / Gap |
|---|---|---|
| Lawful basis | Art. 6 | No explicit consent or lawful-basis record in dataset |
| Data minimization | Art. 5(1)(b) | Direct identifiers (name, SSN, IP, email) present and not required for credit modelling |
| Storage limitation | Art. 5(1)(e) | No data retention policy defined; no deletion mechanism present |
| Integrity & confidentiality | Art. 5(1)(f) | No encryption or access control evident in raw dataset |
| Right to erasure | Art. 17 | No mechanism to locate and delete individual records on request |
| Privacy by design | Art. 25 | Identifiers collected without documented necessity; no pseudonymization at ingestion |

### EU AI Act Classification

Credit scoring systems fall under **Annex III (High-Risk AI Systems)**. NovaCred must ensure:
- Bias testing before deployment (Art. 10)
- Human oversight mechanisms (Art. 14)
- Transparency and explainability of decisions (Art. 13)
- Post-market monitoring (Art. 61)

---

## 4. Governance Recommendations

1. **Audit and retrain the risk scoring algorithm.** The `algorithm_risk_score` is the primary driver of the gender approval gap (100 female vs. 69 male rejections). All input features must be reviewed for direct or indirect gender encoding. Spending categories (`Shopping`, `Insurance`, `Groceries`) are flagged as potential proxy variables and should be removed or de-biased.

2. **Apply fairness constraints.** Implement demographic parity or equalized odds constraints during model training (e.g., Fairlearn `ThresholdOptimizer`). Deploy automated fairness monitoring (DIR, DPD, χ²) on a rolling basis with threshold alerts.

3. **Pseudonymize at ingestion.** Direct identifiers should never reach analytical pipelines in raw form. Implement pseudonymization at the point of data collection, with keys stored separately under strict access controls.

4. **Define and enforce a data retention policy.** Establish a maximum retention period for application-level data (e.g., aligned with regulatory audit requirements) and implement automated deletion or anonymization for records past that period.

5. **Implement an audit trail.** Every automated credit decision should be logged with a timestamp, the model version, input feature values (pseudonymized), and the outcome. This is required for GDPR Art. 22 compliance (automated individual decision-making) and EU AI Act Art. 12 (record-keeping).

6. **Establish role-based access control.** Raw PII (name, SSN, email, IP) must be accessible only to authorized personnel on a need-to-know basis. Analysts and data scientists should work exclusively with pseudonymized datasets.

7. **Add consent and lawful-basis tracking.** Each application record must be linked to a documented lawful basis under GDPR Art. 6 (e.g., contract necessity or legitimate interest). A consent mechanism is required if legitimate interest is not established.

---

## 5. Key Metrics at a Glance

| Metric | Value |
|---|---|
| Raw records | 502 |
| Clean records | 500 |
| Duplicate IDs removed | 2 |
| Disparate Impact Ratio (gender) | **0.767** (threshold: 0.80) |
| Chi-squared p-value | **0.000694** |
| Demographic Parity Difference | **0.154** |
| Female approval rate | **50.6%** |
| Male approval rate | **66.0%** |
| Records uniquely re-identifiable (k=1) | **97.4%** |
| Direct PII fields in dataset | **5** |
| Records with negative credit history | 2 |
| Records with negative savings balance | 1 |
| Malformed emails | 11 |

---

## 6. Video Presentation

> Link / file: *(https://youtu.be/4G6JUpEPzmM)*

---

*DEGO 2606 | Nova SBE | Team TXC5*
