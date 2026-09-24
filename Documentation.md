# Mitra CareLoop — Technical Documentation

## Data-Driven Follow-Up Assurance for Rural Teleconsultations

---

## 1. Executive Summary & Introduction

**Mitra CareLoop** (referred to interchangeably as **Mitra** or **Kestrel CareLoop**) is an enterprise-grade, data-driven follow-up assurance and care-continuity platform designed to solve the chronic care dropout crisis in rural teleconsultations across Telangana.

Under the Ayushman Arogya Mandir (AAM) program, Community Health Officers (CHOs) connect rural patients diagnosed with non-communicable diseases (NCDs)—primarily Hypertension and Type 2 Diabetes—to specialist physicians at sub-district and district hospitals using the eSanjeevani teleconsultation network. However, existing health informatics systems record a teleconsultation as "Completed" the instant the virtual video/audio call disconnects. 

In practice, a completed consultation represents merely the first milestone of care:
$$\text{Teleconsultation} \longrightarrow \text{Prescription} \longrightarrow \text{Medicine Dispensing} \longrightarrow \text{Diagnostic Testing} \longrightarrow \text{30-Day Follow-Up Review} \longrightarrow \text{Care Closure}$$

Due to systemic data fragmentation across teleconsultation logs, NCD screening databases, pharmacy registers, diagnostic laboratories, and Accredited Social Health Activist (ASHA) diaries, healthcare administrators lack any unified visibility into post-consultation adherence. A substantial proportion of rural patients drop out immediately: failing to collect essential anti-hypertensive or anti-diabetic medications, missing crucial lab tests (HbA1c, Serum Creatinine), and failing to return for mandatory 30-day reviews.

Mitra CareLoop shifts the operational paradigm from:
> **Passively counting completed teleconsultations**

to:
> **Actively tracking, predicting, and assuring completed closed-loop clinical care.**

The platform follows a rigorous five-stage operational framework:
$$\mathbf{Connect} \longrightarrow \mathbf{Predict} \longrightarrow \mathbf{Explain} \longrightarrow \mathbf{Act} \longrightarrow \mathbf{Close}$$

---

## 2. Problem Definition & Rural Operational Realities

### 2.1 The Data Fragmentation Crisis
In Telangana's rural primary healthcare network, patient touchpoints are recorded across disparate, unlinked IT and manual systems:
1. **eSanjeevani Logs:** Captures remote doctor-patient encounters, clinical diagnoses, and broad advice flags (`medicine_advised`, `test_advised`, `review_advised`).
2. **NCD Screening App (CPHC):** Captures community blood pressure, random blood sugar (RBS), BMI, and baseline non-communicable disease classifications.
3. **e-Aushadhi / Pharmacy Registers:** Records drug dispensing, prescribed proxy quantities, dispensed quantities, partial fills, and facility-level stockout flags.
4. **Diagnostic Laboratory Systems:** Tracks test orders, specimen collection dates, test completion statuses, and biochemical abnormalities.
5. **Encounter / Visit Registers:** Logs physical visits to Primary Health Centres (PHCs) or Health & Wellness Centres (HWCs/AAMs).
6. **ASHA Offline Field Registers & Outreach Logs:** Records home visits, village health day reminders, phone calls, and patient contact barriers.
7. **Supply Chain & Stock Registers:** Monthly snapshots of facility drug stocks, opening balances, receipts, and recorded stockout durations.
8. **Geographic & Infrastructure Registries:** Village-level population, road accessibility, and cellular connectivity indices.

Because these databases use distinct, system-specific local identifiers (e.g., `tele_source_patient_id`, `ncd_source_patient_id`, `rx_source_patient_id`, `pharm_source_patient_id`, `lab_source_patient_id`, `visit_source_patient_id`, `outreach_source_patient_id`), patient journeys cannot be tracked with standard relational database foreign keys.

### 2.2 Operational Challenges in Rural Care Continuity
* **The "Completed" Illusion:** A teleconsultation marked as complete in eSanjeevani masks post-consultation failure. Patients who fail to collect drugs or attend reviews are never identified until acute cardiovascular or diabetic complications emerge.
* **Invisible Root Causes:** When a patient drops out, the system cannot distinguish between a *behavioral barrier* (lack of symptoms, health illiteracy, migration), an *accessibility barrier* (remote hamlet, unpaved roads, lack of transport to PHC), or a *systemic supply barrier* (facility-level stockout of Amlodipine or Metformin).
* **Health Worker Overburden & Cognitive Fatigue:** ASHAs and CHOs cannot comb through disparate registers or make hundreds of manual verification calls. Without strict prioritization, task lists become unmanageable and are ignored.
* **Severe Physical Constraints:** Rural Telangana exhibits patchy 2G/3G connectivity, frequent power outages, and field reliance on low-cost entry-level Android smartphones with constrained RAM and storage.

---

## 3. System Objectives & Mandatory Constraints

### 3.1 Core Objectives
1. **Deterministic & Probabilistic Data Linkage:** Reconcile fragmented source systems to canonical patient identities in `patient_360_reference.csv` without crosswalk tables, measuring record linkage completeness using the **Continuity-of-Care Index (CCI-7)**.
2. **Proactive LFU Risk Prediction:** Predict Loss to Follow-Up (LTFU) probability $P(\text{LTFU}) \in [0, 1]$ immediately upon teleconsultation completion ($T_{\text{pred}}$) before downstream failures materialize.
3. **Dropout Stage & Barrier Attribution:** Pinpoint the exact expected failure stage (`Medicine not collected`, `Test not completed`, `Review not attended`) and isolate the primary operational barrier (e.g., medicine stockout, geographic distance, communication failure).
4. **Actionable Operational Dispatch:** Translate risk and barrier scores into a capped, prioritized action queue using the **Kestrel-7 Escalation Ladder**, dispatching tasks to the **ASHA "NCD-Mitra" Offline Android App**, **Automated Telugu IVR voice engine**, or **CHO Clinical Portal**.
5. **Closed-Loop Verification:** Provide 1-click verification logging and track patients through to validated care closure.

### 3.2 Mandatory Constraints & Compliance
* **Offline-First Resilience:** The frontline ASHA interface operates fully offline on low-end hardware using local encrypted storage (SQLCipher AES-256), syncing delta payloads when connectivity is restored.
* **Zero-Leakage Temporal Boundary:** Prediction occurs strictly at the completion of teleconsultation. Future dispensing records, future lab results, and future follow-up attendance are strictly quarantined during training and inference.
* **Strict Anti-Manipulation & Clinical Guardrails:** AI predictions are strictly bounded by monotonicity constraints, physical clinical realities, and deterministic decision trees to prevent hallucinations or queue gaming.
* **National & State Regulatory Compliance:** Full compliance with the **Ayushman Bharat Digital Mission (ABDM)** consent architecture, the Digital Personal Data Protection (DPDP) Act 2023, and the **Telangana Rural Data Stewardship Charter (TRDSC-7)**.

---

## 4. End-to-End System Architecture

The following diagram illustrates the complete, integrated pipeline from raw disparate files to field intervention and care closure:

```text
┌────────────────────────────────────────────────────────────────────────────────────────┐
│                              DISPARATE DATA SOURCES (13 CSVs)                          │
│  teleconsultations | ncd_screening | prescriptions | medicine_dispensing | lab_tests   │
│  followup_visits | visit_history | outreach_actions | medicine_stock_status            │
│  facility_reference | geography_reference | patient_360_reference | episode_outcomes   │
└───────────────────────────────────────────┬────────────────────────────────────────────┘
                                            │
                                            ▼
┌────────────────────────────────────────────────────────────────────────────────────────┐
│                       STAGE 1: NORMALIZATION & ENTITY RESOLUTION                       │
│  • Cleaning: Whitespace, Case, +91 Prefix, Date standardization                        │
│  • Primary Match: ABHA ID / Canonical Match                                            │
│  • Secondary Fallback: Double Metaphone(Name) + Sex + Age ±2yr + Village + Masked Mobile│
│  • Output: submission_template_linkage.csv                                             │
└───────────────────────────────────────────┬────────────────────────────────────────────┘
                                            │
                                            ▼
┌────────────────────────────────────────────────────────────────────────────────────────┐
│                       STAGE 2: EPISODE RECONSTRUCTION & CARE GRAPH                     │
│  • Anchor: teleconsult_id mapped to episode_id (from episode_outcomes.csv)             │
│  • Attach: Prescriptions, Dispensing, Diagnostics, Historical Visits, Facility, Geo    │
│  • Audit: Continuity-of-Care Index (CCI-7) evaluated across 7 core care edges          │
└───────────────────────────────────────────┬────────────────────────────────────────────┘
                                            │
                                            ▼
┌────────────────────────────────────────────────────────────────────────────────────────┐
│                   STAGE 3: TEMPORAL FEATURE STORE & ANTI-LEAKAGE BOUNDARY              │
│  • Prediction Cutoff: Exactly at consult_date / consult completion (T_pred)            │
│  • Strict Quarantine: Bar future dispensing, future lab results, future follow-up dates │
│  • Engineered Vectors: Demographics, Comorbidities, Pill Burden, Geo Distance, Stockout│
└───────────────────────────────────────────┬────────────────────────────────────────────┘
                                            │
                                            ▼
┌────────────────────────────────────────────────────────────────────────────────────────┐
│                   STAGE 4: AI ANALYTICS ENGINE & ACCURACY GUARDRAILS                   │
│  • Model: Tabular XGBoost Binary Classifier + Multi-class Stage Classifier             │
│  • Calibration: Platt Scaling / Isotonic Regression (Calibrated P(LTFU))               │
│  • Guardrail 1: Enforced Feature Monotonicity (Distance ↑, Burden ↑, Stockout ↑ => P ↑)│
│  • Guardrail 2: Deterministic Clinical Sanity Checks (Suppresses invalid advice paths) │
│  • Guardrail 3: Demographic Parity & Calibration across Vulnerability Segments         │
│  • Output: submission_template_episode_predictions.csv                                 │
└───────────────────────────────────────────┬────────────────────────────────────────────┘
                                            │
                                            ▼
┌────────────────────────────────────────────────────────────────────────────────────────┐
│                   STAGE 5: ROOT-CAUSE BARRIER ENGINE & DECISION TREES                  │
│  • Decision Tree 1: Priority Tier Mapping (Tiers 0–4: Low, Medium, High, Urgent, Crit) │
│  • Decision Tree 2: Dropout Stage Detection (Meds vs Test vs Review)                   │
│  • Decision Tree 3: Root-Cause Attribution (Stockout vs Geographic vs Contact vs Clinic)│
│  • Decision Tree 4: Operational Cadre Dispatch (ASHA vs CHO vs Telugu IVR)             │
│  • Guardrail 4: Anti-Flooding Capacity Throttle (Capped Top 5-10 daily tasks per ASHA) │
│  • Output: submission_template_action_queue.csv                                        │
└───────────────────────────────────────────┬────────────────────────────────────────────┘
                                            │
                                            ▼
┌────────────────────────────────────────────────────────────────────────────────────────┐
│                      STAGE 6: FRONTLINE OPERATIONAL INTERVENTION                       │
│  ┌──────────────────────────┐ ┌──────────────────────────┐ ┌─────────────────────────┐ │
│  │ ASHA "NCD-Mitra" App     │ │ Automated Telugu IVR     │ │ CHO Web Dashboard       │ │
│  │ • Top-5 Daily Task Cards │ │ • Local dialect calls    │ │ • Facility Stock Alerts │ │
│  │ • Offline SQLite (AES256)│ │ • T-3 and T-0 auto-dials │ │ • Clinical Escalations  │ │
│  │ • 1-Click Action Logging │ │ • ₹0.15/call efficiency  │ │ • High-Risk Review Queue│ │
│  └─────────────┬────────────┘ └─────────────┬────────────┘ └────────────┬────────────┘ │
└────────────────┼────────────────────────────┼───────────────────────────┼──────────────┘
                 └────────────────────────────┼───────────────────────────┘
                                              ▼
┌────────────────────────────────────────────────────────────────────────────────────────┐
│                        STAGE 7: CLOSED-LOOP CARE CLOSURE PROTOCOL                      │
│  • Delta Synchronization over MQTT/HTTPS when connected                                │
│  • Longitudinal Care Timeline Updated                                                  │
│  • Care Closed Status: Teleconsult ✓ + Meds Dispensed ✓ + Lab Done ✓ + Review Done ✓   │
│  • SHA-256 Cryptographic Audit Trail for Tamper-Proof Governance                       │
└────────────────────────────────────────────────────────────────────────────────────────┘
```

---

## 5. Comprehensive Dataset Blueprint & Utilization Guide

The dataset provided for rural teleconsultation follow-up assurance (`Infinum_2026_Candidate_Dataset_Pack`) represents a synthetic cohort of **5,000 unique patients** and **5,516 teleconsultation episodes** across Telangana.

### 5.1 Dataset Inventory & Cross-Table Relationship Matrix

The table below details every file, its exact schema, join mechanics, and role in the pipeline:

| Dataset File | Primary Key(s) | Foreign Key(s) & Join Target | Deliberate Source Noise | Pipeline Utilization Role |
| :--- | :--- | :--- | :--- | :--- |
| `patient_360_reference.csv` | `patient_id` | Canonical Master Table | Masked mobile (`XXXXXX5988`), clean canonical names | **Ground Truth Entity Master:** The reference entity table representing canonical patient identities. Target for all source linkages. |
| `teleconsultations.csv` | `teleconsult_id` | `facility_id` $\to$ `facility_reference.facility_id` | `tele_source_patient_id` (unlinked), typos in names/villages, noisy phones | **Episode Anchor:** Defines the consultation encounter date, clinical advice flags (`medicine_advised`, `test_advised`, `review_advised`), and connectivity. |
| `episode_outcomes.csv` | `episode_id` | `teleconsult_id` $\to$ `teleconsultations.teleconsult_id` | None (Evaluation labels blank for competition evaluation) | **Ground Truth Training & Evaluation Target:** Maps `episode_id` to `teleconsult_id`. Contains `lost_to_followup_label` (0/1) and `dropout_stage_label`. |
| `ncd_screening.csv` | `ncd_record_id` | `ncd_source_patient_id` $\to$ `patient_360_reference` | `ncd_source_patient_id` (unlinked), noisy names, ages, phones | **Pre-Consultation Baseline:** Captures baseline SBP, DBP, glucose, BMI, and `control_status` prior to the consultation. |
| `prescriptions.csv` | `prescription_id`, `prescription_line_id` | `teleconsult_id` $\to$ `teleconsultations.teleconsult_id` | `rx_source_patient_id` (unlinked), noisy patient name/phone | **Clinical Treatment Regimen:** Contains prescribed drug names, classes, dosages, routes, and `days_prescribed` (used to compute Pill Burden). |
| `medicine_dispensing.csv` | `dispense_id` | `prescription_id` $\to$ `prescriptions.prescription_id` | `pharm_source_patient_id` (unlinked), noisy phone/village | **Post-Consultation Verification:** Records actual drug dispensing, partial fills, and pharmacy-level stockout flags. Quarantined during prediction. |
| `medicine_stock_status.csv` | `stock_record_id` | `facility_id` $\to$ `facility_reference.facility_id` | None (Structured monthly snapshots) | **Systemic Supply Context:** Monthly opening stock, receipts, dispensations, and `stockout_days` by medicine and facility. |
| `lab_tests.csv` | `lab_record_id` | `lab_source_patient_id` $\to$ `patient_360_reference` | `lab_source_patient_id` (unlinked), missing result dates | **Diagnostic Milestone:** Tracks ordered tests (CBC, Creatinine, HbA1c, ECG, Lipid Profile), test completion status, and abnormal/critical flags. |
| `followup_visits.csv` | `visit_id` | `episode_id` $\to$ `episode_outcomes.episode_id` | `visit_source_patient_id` (unlinked), noisy names | **Follow-Up Encounter Register:** Records scheduled 30-day follow-up attendance, `clinical_status` (Improved, Stable, Needs escalation), and referrals. |
| `visit_history.csv` | `visit_id` | `visit_source_patient_id` $\to$ `patient_360_reference` | `visit_source_patient_id` (unlinked), noisy dates/names | **Behavioral Care History:** Past visit history prior to the current episode used to compute historical missed visits and compliance rate. |
| `outreach_actions.csv` | `outreach_id` | `episode_id` $\to$ `episode_outcomes.episode_id` | `outreach_source_patient_id` (unlinked), noisy phones | **Outreach Interaction Log:** Past and ongoing ASHA/CHO outreach attempts (`action_method`, `contact_outcome`, `cadre`, `followup_reason`). |
| `facility_reference.csv` | `facility_id` | Keyed by `facility_id` | None (Clean facility master) | **Facility Capacity Context:** Facility type (PHC/HWC), coordinates, `network_context` (Offline-prone), CHO count, and linked ASHA count. |
| `geography_reference.csv` | `village_id` | `village` $\to$ Patient/Facility tables | Minor spelling variations in village names | **Geographic Accessibility:** Village population, `road_access` (Paved, Mostly paved, Mixed, Poor), and `mobile_connectivity` (Good, Moderate, Weak, Very weak). |

---

## 6. Data Normalization & Preprocessing Hygiene

Before running entity resolution or feature engineering, all raw textual, numeric, and categorical attributes undergo systematic sanitization:

1. **Text & Name Normalization:**
   * Remove honorifics, titles, and special characters: `re.sub(r'[^a-zA-Z\s]', '', name)`.
   * Standardize to uppercase: `"Mohan Basha "` $\to$ `"MOHAN BASHA"`.
   * Whitespace squashing: Replace multiple spaces, tabs, or trailing whitespace with a single space.
2. **Phone Number Sanitization:**
   * Strip non-digit characters (`+`, `-`, spaces, brackets).
   * Remove national prefixes (`91`, `0`): `re.sub(r'^(\+91|91|0)', '', phone)`.
   * Standardize to 10-digit format: If length $\neq 10$, flag as invalid or missing.
   * Derive `last_4_digits` to match against `masked_mobile` (`XXXXXX5988` $\to$ `5988`).
3. **Geographic String Normalization:**
   * Strip trailing designators (`Thanda`, `Guda`, `Colony`, `Village`).
   * Apply phonetic dictionary mapping to resolve known Telugu dialect spelling variations (e.g., `Jadcherla` vs `Jedcharla`, `Bodajanampet` vs `Boda Janampeta`).
4. **Date/Time Parsing:**
   * Coerce all timestamps into ISO 8601 (`YYYY-MM-DD`).
   * Compute relative temporal differences in integer days:
     $$\Delta t_{\text{screen\_to\_consult}} = \text{consult\_date} - \text{screening\_date}$$
5. **Categorical Standardization:**
   * Binary flags: Map `{'Yes', 'Y', 1, 'True', 'true'}` $\to 1$; `{'No', 'N', 0, 'False', 'false'}` $\to 0$.
   * Sex/Gender: Map `{'Male', 'M', 'm'}` $\to \text{'M'}$; `{'Female', 'F', 'f'}` $\to \text{'F'}$.

---

## 7. Entity Resolution & Canonical Patient Linking

Because the source operational databases lack a common patient identifier, Mitra CareLoop executes a high-precision, two-stage entity resolution pipeline to link every source record to its canonical `patient_id` in `patient_360_reference.csv`.

```text
Source Record (eSanjeevani, NCD, Dispensing, Lab, Visits, Outreach)
                           │
                           ▼
            Does ABHA ID match directly?
                 ├── Yes ──► Direct Canonical Assignment (Confidence = 1.00)
                 │
                 └── No ───► Deterministic Phone + Name Fallback
                                  ├── Match? ──► Link Assigned (Confidence = 0.95)
                                  │
                                  └── No ────► Multi-Condition Fuzzy Scoring Engine:
                                               1. Metaphone/Soundex Phonetic Name Match
                                               2. Gender Exact Match
                                               3. Age within ±2 Years
                                               4. Village Name Jaro-Winkler Similarity ≥ 0.85
                                               5. Phone Last 4 Digits == Masked Mobile
                                                    │
                                                    ├── Composite Score ≥ Threshold (0.80)?
                                                    │        ├── Yes ──► Link Assigned (Confidence = Composite Score)
                                                    │        └── No ───► Cluster as New Unlinked Entity
```

### 7.1 Deterministic & Phonetic Reconciliation Algorithm
For records missing an ABHA ID:
1. **Phonetic Encoding:** Generate primary and secondary phonetic keys using the **Double Metaphone** algorithm on `patient_name`. This neutralizes Anglicized spelling differences of Telugu names (e.g., `Lakshmi` vs `Laxmi`, `Mallaiah` vs `Malliah`).
2. **Deterministic Composite Rule:**
   A match is established if and only if:
   $$\text{DoubleMetaphone}(\text{Source\_Name}) == \text{DoubleMetaphone}(\text{Canonical\_Name})$$
   $$\land \quad \text{Source\_Gender} == \text{Canonical\_Gender}$$
   $$\land \quad |\text{Source\_Age} - \text{Canonical\_Age}| \le 2$$
   $$\land \quad (\text{Source\_Village} == \text{Canonical\_Village} \lor \text{JaroWinkler}(\text{Source\_Village}, \text{Canonical\_Village}) \ge 0.85)$$
   $$\land \quad (\text{Right}(\text{Source\_Mobile}, 4) == \text{Right}(\text{Canonical\_Masked\_Mobile}, 4))$$

### 7.2 Linkage Confidence Calculation
For probabilistic matches, the confidence score $C \in [0, 1]$ is computed as:
$$C = 0.35 \cdot \text{JW}(\text{Name}) + 0.30 \cdot \mathbf{1}_{\{\text{Phone4}\}} + 0.15 \cdot \text{JW}(\text{Village}) + 0.10 \cdot \mathbf{1}_{\{\Delta \text{Age} \le 2\}} + 0.10 \cdot \mathbf{1}_{\{\text{Gender}\}}$$
Where $\text{JW}$ represents the Jaro-Winkler string similarity and $\mathbf{1}_{\{\cdot\}}$ is an indicator function.

### 7.3 Linkage Output Format
Matches are compiled directly into the required format for `submission_template_linkage.csv`:
```csv
source_system,source_patient_id,predicted_patient_id,confidence
teleconsultations,TEP0000001,P000001,0.985
ncd_screening,NCP0000001,P000001,0.990
prescriptions,RXP0000001,P000001,1.000
medicine_dispensing,PHP0000001,P000001,0.975
lab_tests,LAP0000001,P000001,0.950
visits,VIP0000001,P000001,0.980
outreach_actions,OUP0000001,P000001,0.965
```

---

## 8. Episode Reconstruction & Care Continuum Graph

A teleconsultation is not an isolated event; it represents the initiation of an **Episode of Care**. The reconstruction engine anchors on the teleconsultation encounter and constructs a directed care continuum graph:

```text
[NCD Screening Baseline]
          │
          ▼ (Δt: Screening to Consult)
[Teleconsultation (consult_date)] ◄─── ANCHOR (episode_id)
     │            │             │
     ▼            ▼             ▼
[Prescription]  [Lab Order]   [Review Advice]
     │            │             │
     ▼ (Δt: 0-3d) ▼ (Δt: 0-7d)  ▼ (Δt: review_due_days)
[Dispensing]    [Lab Sample]  [Scheduled Review]
                  │             │
                  ▼ (Δt: 1-2d)  ▼
                [Lab Result]  [Follow-Up Visit Outcome]
```

### 8.1 Temporal Episode Association Logic
1. **Prescriptions & Dispensing:** All records in `prescriptions.csv` sharing `teleconsult_id` are bound to the episode. Dispensing records in `medicine_dispensing.csv` matching `prescription_id` within $t_{\text{consult}} \le t_{\text{dispense}} \le t_{\text{consult}} + 14\text{ days}$ are linked.
2. **Diagnostic Tests:** Lab orders in `lab_tests.csv` matching the resolved `patient_id` with $\text{order\_date} \in [t_{\text{consult}} - 1, t_{\text{consult}} + 3\text{ days}]$ are attached.
3. **Follow-Up Reviews:** Encounters in `followup_visits.csv` carrying the matching `episode_id` or occurring within $t_{\text{consult}} \le t_{\text{visit}} \le t_{\text{consult}} + \text{review\_due\_days} + 15\text{ days}$ are bound.

---

## 9. Continuity-of-Care Index (CCI-7)

The **Continuity-of-Care Index (CCI-7)** is a rigorous, quantitative metric that evaluates the data integrity and connectivity of a patient's reconstructed care graph across seven vital systemic transitions:

$$\text{CCI-7} = \frac{1}{7} \sum_{i=1}^{7} w_i \cdot e_i, \quad e_i \in \{0.0, 0.5, 1.0\}$$

Where each edge $e_i$ evaluates a specific operational continuum link:

| Edge ($e_i$) | Transition Evaluated | Verified Data Linkage Source | Evaluation Rule |
| :---: | :--- | :--- | :--- |
| **$e_1$** | Patient $\to$ Teleconsultation | `patient_360_reference` $\leftrightarrow$ `teleconsultations` | $1.0$ if canonical linkage established; $0.5$ if unconfirmed fuzzy match; $0.0$ if unlinked. |
| **$e_2$** | Teleconsultation $\to$ NCD Context | `teleconsultations` $\leftrightarrow$ `ncd_screening` | $1.0$ if pre-consult NCD baseline linked within 180 days; $0.0$ otherwise. |
| **$e_3$** | Teleconsultation $\to$ Prescription | `teleconsultations` $\leftrightarrow$ `prescriptions` | $1.0$ if `medicine_advised == 'Yes'` has matching Rx; $1.0$ if `medicine_advised == 'No'`; $0.0$ if advised but missing Rx. |
| **$e_4$** | Prescription $\to$ Pharmacy Dispensing | `prescriptions` $\leftrightarrow$ `medicine_dispensing` | $1.0$ if all prescribed items dispensed; $0.5$ if partially dispensed; $0.0$ if uncollected/out of stock. |
| **$e_5$** | Test Advised $\to$ Lab Completion | `teleconsultations` $\leftrightarrow$ `lab_tests` | $1.0$ if `test_advised == 'No'` or test completed with results; $0.5$ if sample taken; $0.0$ if ordered-not-completed. |
| **$e_6$** | Review Advised $\to$ Follow-Up Visit | `teleconsultations` $\leftrightarrow$ `followup_visits` | $1.0$ if `review_advised == 'No'` or visit attended; $0.0$ if review advised but visit absent. |
| **$e_7$** | Care Gap $\to$ Outreach Action | `episode_outcomes` $\leftrightarrow$ `outreach_actions` | $1.0$ if outreach recorded upon care gap detection; $1.0$ if no care gap occurred; $0.0$ if care gap uncontacted. |

> [!NOTE]
> **Operational Purpose:** The CCI-7 score functions as a systemic data health indicator. A drop in district-wide CCI-7 immediately alerts state health directors to administrative bottlenecks (e.g., pharmacy register sync failures).

---

## 10. Prediction-Time Boundary & Anti-Leakage Protocol

To ensure absolute real-world clinical validity and prevent target leakage, Mitra CareLoop establishes an uncompromising temporal quarantine at the prediction cutoff point:

$$\mathbf{T_{\text{pred}}} = \text{Timestamp of Teleconsultation Completion}$$

```text
PRE-CONSULTATION & CONSULTATION TIME (T ≤ T_pred) │ POST-CONSULTATION FUTURE (T > T_pred)
====================================================│===================================================
✓ PERMITTED FEATURES:                              │ ✗ STRICTLY QUARANTINED (LEAKAGE):
• Patient Demographics (Age, Gender, Language)      │ • Future Medicine Dispense Dates & Quantities
• Geography (Distance to facility, Road access)     │ • Dispense Status (Dispensed / Partial / Stockout)
• Pre-consult NCD Screening (SBP, DBP, Glucose, BMI)│ • Diagnostic Test Completion Status & Results
• Historical Visits & Missed Appointments           │ • Future Lab Specimen Collection Dates
• Historical Outreach & Past Contact Success Rates  │ • Follow-up Visit Encounter Dates & Clinical Status
• Current Teleconsultation Details (Complaint, Mode)│ • Future ASHA/CHO Outreach Actions & Notes
• Current Prescriptions (Drug class, Pill burden)   │ • Ground Truth Labels (lost_to_followup_label)
• Facility Characteristics & Past Month Drug Stock  │ • Ground Truth Dropout Stage (dropout_stage_label)
```

Any model utilizing features timestamped $T > T_{\text{pred}}$ suffers from look-ahead bias and is rejected by the Mitra deployment validator.

---

## 11. Feature Engineering & Schema Crosswalk

Every episode is converted into an engineered feature vector using only data available at $T_{\text{pred}}$:

### 11.1 Engineered Feature Groups

1. **Demographic & Vulnerability Indicators:**
   * `age`: Patient age in years.
   * `gender_encoded`: Binary indicator ($1 = \text{M}, 0 = \text{F}$).
   * `vulnerability_encoded`: One-hot encoded vulnerability (`Elderly living alone`, `Low digital access`, `Mobility limitation`, `Seasonal migrant`, `General`).
   * `preferred_language_telugu`: Binary indicator ($1 = \text{Telugu}, 0 = \text{Other}$).
2. **Geographic & Physical Accessibility:**
   * `distance_to_facility_km`: Road-independent distance proxy from patient village to AAM/PHC.
   * `road_access_quality`: Ordinal encoding (`Poor` = 0, `Mixed` = 1, `Mostly paved` = 2, `Paved` = 3).
   * `mobile_connectivity_score`: Ordinal encoding (`Very weak` = 0, `Weak` = 1, `Moderate` = 2, `Good` = 3).
3. **Clinical Severity & Chronic Burden:**
   * `baseline_sbp`, `baseline_dbp`: Blood pressure readings from latest pre-consult NCD screening.
   * `baseline_glucose`: Random blood sugar level in mg/dL.
   * `bmi`: Body Mass Index.
   * `comorbidity_score`: Derived integer: $0 = \text{None}, 1 = \text{HTN or DM}, 2 = \text{HTN + DM}$.
   * `uncontrolled_ncd_flag`: Binary indicator if SBP $\ge 140$, DBP $\ge 90$, or Glucose $\ge 200$.
4. **Prescription Complexity & Pill Burden:**
   * `prescribed_drug_count`: Total number of active pharmaceutical lines prescribed.
   * `max_days_prescribed`: Maximum duration in days across prescribed medications.
   * `cv_metabolic_drug_prescribed`: Binary indicator for high-risk chronic drugs (Amlodipine, Metformin).
5. **Behavioral Compliance History:**
   * `past_visit_count`: Total historical encounters in `visit_history.csv`.
   * `past_missed_visit_count`: Count of past encounters where status was recorded as non-compliant/missed.
   * `past_compliance_ratio`: $\frac{\text{Completed Visits}}{\text{Total Scheduled Visits}}$.
   * `days_since_last_visit`: Recency of prior healthcare interaction.
   * `past_outreach_failure_rate`: Ratio of past outreach attempts with outcome `Phone switched off` or `Patient unavailable`.
6. **Supply Chain & Facility Capacity:**
   * `facility_network_offline`: Binary indicator if `network_context == 'Offline-prone'`.
   * `cho_patient_ratio`: Facility patient load proxy ($\frac{\text{Active Episodes}}{\text{CHO Count}}$).
   * `prescribed_drug_facility_stockout`: Binary flag: 1 if any drug prescribed to the patient suffered a stockout (`stockout_flag_month == 1`) in the facility's latest snapshot.

---

## 12. AI Model Integration Architecture

The AI model integration in Mitra CareLoop is architected as an asynchronous, modular inference service that bridges central clinical analytics with distributed field execution:

```text
Central Database (PostgreSQL) ──► Inference Pipeline (FastAPI / Celery) ──► ML Model Engine (XGBoost)
                                                                                  │
                                                                                  ▼
                                                                           Calibrated P(LTFU)
                                                                                  │
                                                                                  ▼
Frontline Deployment Layer ◄── Sync Service (MQTT / REST) ◄── Deterministic Decision Matrix
 ├── ASHA "NCD-Mitra" Offline App (SQLite AES-256)
 ├── Automated Telugu IVR Telephony Gateway
 └── CHO Clinical Operations Dashboard
```

### 12.1 End-to-End Pipeline Execution
1. **Trigger:** The completion of a teleconsultation in eSanjeevani triggers a webhook to the Mitra Inference Service.
2. **Feature Extraction:** The feature pipeline extracts patient history, current prescriptions, facility drug stock, and geographic attributes.
3. **Primary Scoring:** The calibrated tabular XGBoost classifier evaluates the feature vector and outputs the predicted loss-to-follow-up probability:
   $$P(\text{LTFU}) \in [0.00, 1.00]$$
4. **Secondary Multi-Class Stage Prediction:** An auxiliary multi-class XGBoost classifier predicts the conditional probability distribution over the three potential failure stages:
   $$P(\text{Medicine not collected}), \quad P(\text{Test not completed}), \quad P(\text{Review not attended})$$
5. **Output Schema Generation:** The model outputs populate `submission_template_episode_predictions.csv`:
   * `episode_id`
   * `risk_probability`: Calibrated $P(\text{LTFU})$ rounded to 4 decimal places.
   * `predicted_lost_to_followup`: Binary prediction ($1$ if $P \ge 0.50$, else $0$).
   * `predicted_dropout_stage`: Most probable failure stage.
   * `priority_tier`: Tier 0 through Tier 4.

---

## 13. Model Training Accuracy, Guardrails & Anti-Manipulation Framework

In rural public health systems, deploying unconstrained machine learning models poses severe risks. An unconstrained black-box model can generate hallucinated recommendations, allocate scarce ASHA time to low-risk patients, or be gamed by administrators attempting to artificially suppress dropout statistics. 

Mitra CareLoop implements a multi-tiered **Guardrail and Constraint Framework** that guarantees clinical accuracy, physical feasibility, and tamper resistance:

```text
┌────────────────────────────────────────────────────────────────────────────────────────┐
│                   MITRA 6-TIER MODEL ACCURACY & GUARDRAIL ARCHITECTURE                 │
├────────────────────────────────────────────────────────────────────────────────────────┤
│ 1. TEMPORAL BOUNDARY GUARDRAIL (Zero Look-Ahead Bias)                                  │
│    • Strict cutoff at T_pred; automated schema audit bars future timestamps.           │
├────────────────────────────────────────────────────────────────────────────────────────┤
│ 2. CLINICAL & PHYSICAL MONOTONICITY CONSTRAINTS                                        │
│    • Enforced feature split monotonicity: ∂P/∂(Distance) ≥ 0, ∂P/∂(PillBurden) ≥ 0.    │
├────────────────────────────────────────────────────────────────────────────────────────┤
│ 3. DETERMINISTIC CLINICAL SANITY GUARDRAILS                                            │
│    • Hard business rules: Cannot assign stage if doctor advised against it.            │
├────────────────────────────────────────────────────────────────────────────────────────┤
│ 4. STATISTICAL CALIBRATION & DEMOGRAPHIC FAIRNESS GUARDRAILS                           │
│    • Isotonic probability calibration + equalized odds across vulnerability groups.    │
├────────────────────────────────────────────────────────────────────────────────────────┤
│ 5. CAPACITY THROTTLING & ANTI-FLOODING CONTROLS                                        │
│    • Capped Top-5 daily tasks per ASHA village to prevent cognitive fatigue & gaming.  │
├────────────────────────────────────────────────────────────────────────────────────────┤
│ 6. CRYPTOGRAPHIC AUDIT TRAILS & HUMAN-IN-THE-LOOP OVERRIDES                            │
│    • SHA-256 state hashing + mandatory worker reason codes for overrides.              │
└────────────────────────────────────────────────────────────────────────────────────────┘
```

### 13.1 Guardrail 1: Enforced Feature Monotonicity Constraints
Real-world clinical physics dictate that certain risk relationships must be monotonic. For example, a patient living 35 km away cannot be deemed lower risk than an identical patient living 2 km away, all else being equal. 

In XGBoost, monotonicity is mathematically enforced during tree construction using the `monotone_constraints` hyperparameter:

| Feature | Enforced Constraint Direction | Mathematical Rationale |
| :--- | :---: | :--- |
| `distance_to_facility_km` | $+1$ (Non-decreasing) | Increased travel distance cannot decrease dropout probability. |
| `prescribed_drug_count` | $+1$ (Non-decreasing) | Higher pill burden increases regimen complexity and non-adherence. |
| `past_missed_visit_count`| $+1$ (Non-decreasing) | Demonstrated past non-adherence is an established risk factor. |
| `prescribed_drug_facility_stockout`| $+1$ (Non-decreasing) | Medicine stockouts physically prevent drug collection. |
| `road_access_quality` | $-1$ (Non-increasing) | Better paved roads improve physical access, reducing dropout risk. |
| `past_compliance_ratio` | $-1$ (Non-increasing) | Consistent past clinic attendance is protective. |

```python
# Monotonicity Configuration in XGBoost
monotone_constraints = {
    'distance_to_facility_km': 1,
    'prescribed_drug_count': 1,
    'past_missed_visit_count': 1,
    'prescribed_drug_facility_stockout': 1,
    'road_access_quality': -1,
    'past_compliance_ratio': -1
}
model = xgb.XGBClassifier(
    monotone_constraints=monotone_constraints,
    n_estimators=300,
    max_depth=5,
    learning_rate=0.03,
    eval_metric='logloss'
)
```

### 13.2 Guardrail 2: Deterministic Clinical Sanity Guardrails
To prevent model hallucinations from generating nonsensical field tasks, all machine learning outputs are post-processed through hard deterministic clinical rules:
* **Rule A (Medicine Sanity):** If `teleconsultations.medicine_advised == 'No'`, the predicted dropout stage can **never** be `Medicine not collected`, and no medicine delivery task may be dispatched.
* **Rule B (Diagnostic Sanity):** If `teleconsultations.test_advised == 'No'`, the predicted dropout stage can **never** be `Test not completed`.
* **Rule C (Review Sanity):** If `teleconsultations.review_advised == 'No'`, the system suppresses review reminders.
* **Rule D (Supply vs Patient Barrier Sanity):** If a prescribed drug suffered an active facility stockout (`stockout_flag_month == 1`), the system is barred from flagging the patient for "Patient Refusal" or "Care Neglect". The barrier is forced to `Medicine Stockout`, routing the task to the CHO for inventory reorder rather than sending an ASHA to admonish the patient.

### 13.3 Guardrail 3: Statistical Probability Calibration & Fairness
Raw boosting tree outputs are notoriously uncalibrated. A predicted probability of $0.70$ must reflect an empirical $70\%$ failure rate in validation cohorts.
* **Calibration Method:** Mitra applies **Isotonic Regression** (or Platt Scaling) on out-of-fold predictions to ensure minimal Brier score and reliable risk probabilities.
* **Demographic Parity & Fairness Auditing:** The model is evaluated across vulnerability groups (`Elderly living alone`, `Seasonal migrant`, `Low digital access`) and gender. The decision thresholds are calibrated to prevent disparate impact and ensure equitable outreach distribution across isolated hamlets.

### 13.4 Guardrail 4: Capacity Throttling & Anti-Flooding Controls
If an AI model generates 40 "High Priority" tasks in a village where a single ASHA can only perform 5 visits per day, the system breaks down. Health workers experience cognitive exhaustion, leading to task neglect or fraudulent batch-completion ("gaming" the app).
* **Workload Cap:** The ASHA priority worklist strictly caps active daily home visit assignments to the **Top 5 High-Risk Patients per village**.
* **Dynamic Spillover:** Overflow high-risk cases that exceed ASHA capacity are automatically re-routed to the automated Telugu IVR voice engine or scheduled for the next Village Health, Sanitation and Nutrition Day (VHSND).

### 13.5 Guardrail 5: Cryptographic Auditability & Human-in-the-Loop Overrides
To ensure data integrity, legal compliance, and human agency:
* **State Hashing:** Every model inference event computes a SHA-256 hash of the input feature vector, model version, and generated recommendation:
  $$\text{Hash} = \text{SHA-256}(\text{episode\_id} \,\|\, \text{features} \,\|\, \text{model\_version} \,\|\, \text{timestamp})$$
* **Worker Override Protocol:** Frontline health workers possess absolute authority to override an AI recommendation (e.g., marking a patient as "Migrated for seasonal harvest" or "Meds purchased from private pharmacy"). All overrides require a single-tap mandatory reason code, which is logged to an immutable audit ledger to monitor model drift without permitting ad-hoc tampering.

---

## 14. Operational Decision Mapping & Decision Trees

Mitra CareLoop converts continuous model probabilities, stage likelihoods, and contextual facility signals into concrete operational actions via five interconnected decision trees.

### 14.1 Decision Tree 1: Risk Probability to Priority Tier Mapping
Maps calibrated risk probability $P(\text{LTFU})$ and acute clinical signals to the operational Priority Tier (Tiers 0–4):

```text
                                  [Calibrated P(LTFU)]
                                           │
                    ┌──────────────────────┴──────────────────────┐
                    │                                             │
             P(LTFU) ≥ 0.70                                  P(LTFU) < 0.70
                    │                                             │
         ┌──────────┴──────────┐                       ┌──────────┴──────────┐
         │                     │                       │                     │
   Comorbidity = 2       Comorbidity < 2         0.35 ≤ P < 0.70         P < 0.35
   or SBP ≥ 160          & Stable Vitals               │                     │
         │                     │                       │                     │
         ▼                     ▼                       ▼                     ▼
     [TIER 4]              [TIER 3]                [TIER 2]              [TIER 1]
    (CRITICAL)             (HIGH)                  (MEDIUM)               (LOW)
  Immediate Action    48h Home Visit            Telugu IVR Calls       Standard SMS
```

* **Tier 4 (Critical Risk, $P \ge 0.70$ + Severe Clinical Comorbidity):** Patient has both Hypertension and Diabetes or critical vitals (SBP $\ge 160$, RBS $\ge 250$). Requires joint CHO clinical review and same-day ASHA outreach.
* **Tier 3 (High Risk, $P \ge 0.70$):** High dropout probability with stable baseline vitals. Queued for an in-person ASHA home visit within 48 hours.
* **Tier 2 (Medium Risk, $0.35 \le P < 0.70$):** Moderate risk. Queued for automated Telugu IVR voice calls at $T-3$ days and $T-0$ days.
* **Tier 1 (Low Risk, $0.15 \le P < 0.35$):** Low risk. Sent automated Telugu SMS reminders.
* **Tier 0 (Monitor / Care Completed, $P < 0.15$):** Minimal risk. Monitored through standard passive registers.

---

### 14.2 Decision Tree 2: Dropout Stage Detection
Determines the primary care juncture where care disruption is expected to occur:

```text
                             [Teleconsultation Advice Flags]
                                           │
         ┌─────────────────────────────────┼─────────────────────────────────┐
         │                                 │                                 │
 medicine_advised == 'Yes'         test_advised == 'Yes'          review_advised == 'Yes'
         │                                 │                                 │
   Dispensed in                    Lab Sample Collected            Review Completed
   Pharmacy within 3 Days?          within 7 Days?                  within Due Days + 7?
    ├── Yes ──► (Stage Cleared)     ├── Yes ──► (Stage Cleared)     ├── Yes ──► (Stage Cleared)
    └── No ───►                     └── No ───►                     └── No ───►
         │                                 │                                 │
         ▼                                 ▼                                 ▼
[Medicine not collected]          [Test not completed]            [Review not attended]
```

* **Stage 1: `Medicine not collected`:** Triggered when prescribed medications are not dispensed within 3 days of consultation. Represents $58.9\%$ of empirical dropouts.
* **Stage 2: `Test not completed`:** Triggered when diagnostic tests are advised but no sample is logged within 7 days. Represents $13.8\%$ of empirical dropouts.
* **Stage 3: `Review not attended`:** Triggered when review visit is advised but patient does not attend within $\text{review\_due\_days} + 7\text{ days}$. Represents $27.3\%$ of empirical dropouts.
* **Stage 4: `Completed care journey`:** Assigned when all advised milestones are successfully verified.

---

### 14.3 Decision Tree 3: Operational Barrier Attribution
Isolates the root-cause barrier preventing care completion to ensure appropriate intervention assignment:

```text
                                [Predicted Dropout Stage]
                                           │
         ┌─────────────────────────────────┼─────────────────────────────────┐
         │                                 │                                 │
[Medicine not collected]          [Test not completed]            [Review not attended]
         │                                 │                                 │
   Is Drug Out of Stock              Is Lab Type External           Distance > 15 km or
   at Facility?                      or Distance > 15 km?           Road Access == 'Poor'?
   ├── Yes ──► [Stockout Barrier]   ├── Yes ──► [Access Barrier]   ├── Yes ──► [Transport Barrier]
   └── No                           └── No                         └── No
         │                                 │                                 │
   Distance > 10 km?                Test Status ==                 Phone Switched Off
   ├── Yes ──► [Distance Barrier]   'Ordered-not-completed'?       or Unreachable?
   └── No  ──► [Adherence Barrier]  ├── Yes ──► [Diagnostic Gap]  ├── Yes ──► [Contact Barrier]
                                    └── No  ──► [Adherence Gap]   └── No  ──► [Behavioral Gap]
```

1. **Systemic Supply Barrier (`Medicine Stockout`):** Facility drug stock status indicates 0 units or `stockout_flag_month == 1`. (Action directed to supply chain/CHO).
2. **Geographic Accessibility Barrier (`Distance / Transport`):** Distance to AAM/PHC $> 15\text{ km}$ or village `road_access == 'Poor'`. (Action directed to village-level delivery).
3. **Communication / Contact Barrier (`Unreachable / Phone Issue`):** Phone switched off, missing phone number, or invalid format. (Action directed to ASHA physical visit).
4. **Diagnostic Capacity Barrier (`Lab Unavailable`):** Facility lacks reagents or equipment for HbA1c/Creatinine, requiring travel to district hub. (Action directed to sample collection camp).
5. **Behavioral Adherence Barrier (`Asymptomatic Non-compliance`):** Patient feels well and discontinues medication. (Action directed to vernacular counselling).

---

### 14.4 Decision Tree 4: Operational Cadre Dispatch & Action Queue
Integrates Priority Tier, Dropout Stage, and Barrier into an assigned cadre, concrete action, and urgency level:

```text
                      [Priority Tier + Dropout Stage + Barrier]
                                         │
       ┌─────────────────────────────────┼─────────────────────────────────┐
       │                                 │                                 │
[Tier 3/4 + Meds/Review Gap]      [Tier 1/2 + Any Stage]          [Any Tier + Stockout]
       │                                 │                                 │
   Is ASHA Daily Capacity          Is Mobile Connectivity          Is Facility CHO
   Available (Count < 5)?          'Good' or 'Moderate'?           Available?
   ├── Yes ──► [ASHA Home Visit]   ├── Yes ──► [Telugu IVR Call]   └── Yes ──► [CHO Reorder Task]
   └── No                          └── No
       │                                 │                                 │
       ▼                                 ▼                                 ▼
   [Re-route to VHSND                [Offline SMS /                    [CHO Inventory
    or CHO Outreach]                  Village Notice]                   Reorder Action]
```

The resulting action records directly populate `submission_template_action_queue.csv`:

| Episode ID | Predicted Patient ID | Priority | Reason | Recommended Action | Assigned Cadre |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `E0000004` | `P000004` | `High` | High risk of medicine non-collection due to remote distance (24.5 km) | Conduct home visit, deliver 30-day drug blister pack, verify adherence | `ASHA` |
| `E0000012` | `P000012` | `Critical` | Uncontrolled HTN+DM with stockout of Metformin at local PHC | Arrange inter-facility medicine transfer and expedite CHO review | `CHO` |
| `E0000019` | `P000019` | `Medium` | Scheduled 30-day follow-up review due in 3 days | Dispatch automated pre-recorded Telugu IVR reminder call | `IVR` |
| `E0000027` | `P000027` | `High` | Diagnostic test (Serum Creatinine) ordered but sample not collected | Mobilize patient for upcoming PHC diagnostic collection day | `ASHA` |
| `E0000035` | `P000035` | `Low` | Routine follow-up review with high historical compliance | Send automated SMS confirmation in Telugu | `SMS` |

---

## 15. The Kestrel-7 Intervention Escalation Framework

Mitra CareLoop incorporates the **Kestrel-7 Escalation Framework**, establishing seven progressive, standardized tiers of intervention intensity to ensure that patients are managed at the lowest sufficient operational cost before escalating to scarce clinical resources:

```text
    ▲
[K7]│ Clinical & Specialist Escalation (District Hospital Specialist Review)
[K6]│ CHO Clinical Intervention (PHC Consultation, Regimen Modification)
[K5]│ Operational Resolution (Inter-facility drug transfer, reagent restocking)
[K4]│ Community In-Person Follow-Up (ASHA Home Visit, drug delivery)
[K3]│ Direct Worker Communication (ASHA / ANM Phone Call)
[K2]│ Automated Localized Outreach (Pre-recorded Telugu IVR / SMS nudges)
[K1]│ Passive Continuum Monitoring (Routine register audits)
────┴────────────────────────────────────────────────────────────────────────
```

* **K1 — Passive Continuum Monitoring:** Patients with $P(\text{LTFU}) < 0.15$ and fully linked records are monitored passively via digital health records without active worker outreach.
* **K2 — Automated Localized Outreach:** Automated IVR telephone calls in local dialect (Telugu) delivered at $T-3$ days and $T-0$ days. Cost $\approx \text{₹}0.15$ per call.
* **K3 — Direct Healthcare Worker Communication:** For patients failing IVR outreach or with moderate risk, the system prompts the village ASHA to make a direct mobile call.
* **K4 — Community In-Person Follow-Up:** For high-risk patients ($P \ge 0.70$) or patients with invalid contact information, the ASHA conducts a home visit equipped with the offline "NCD-Mitra" task card.
* **K5 — Systemic Operational Resolution:** When the barrier is facility-side (drug stockout or broken lab equipment), the task bypasses the ASHA and alerts the CHO to execute a stock transfer from the district warehouse.
* **K6 — CHO Clinical Intervention:** For patients exhibiting deteriorating clinical markers (e.g., SBP $> 160\text{ mmHg}$) or reporting drug side effects, the CHO schedules an urgent physical evaluation at the AAM.
* **K7 — Clinical & Specialist Escalation:** For critical acute conditions (suspected hypertensive crisis, diabetic ketoacidosis, acute renal deterioration), the case triggers a fast-track referral to the district hospital specialist.

---

## 16. Healthcare Worker Interfaces & Field Operations

### 16.1 The ASHA "NCD-Mitra" Offline Android App
ASHAs are the bedrock of rural health delivery in Telangana. To ensure maximum usability, the **NCD-Mitra** mobile interface is designed with radical simplicity:
* **The "Top 5" Daily Worklist:** The application completely eliminates complex multi-tab navigation and tabular spreadsheets. Upon opening, the ASHA sees a single screen displaying the **Top 5 High-Priority Patients** in her village for that day.
* **Color-Coded Visual Cards:** Each patient card displays the patient's name, photo/house landmark, village hamlet, prescribed medicine name, and a color-coded priority tag (Red = Urgent, Amber = Follow-up Due).
* **1-Click Outcome Logging:** The ASHA logs actions with a single tap:
  * `[Meds Delivered]`
  * `[Patient Reminded & Agreed]`
  * `[Patient Refused / Reluctant]`
  * `[Patient Migrated / Away]`
  * `[Wrong Phone / Unreachable]`
* **Local Language (Telugu) Interface:** All clinical explanations are translated into conversational Telugu (e.g., *"బీపీ మాత్రలు అయిపోయాయి - వెంటనే అందించండి"*).

### 16.2 Automated Pre-Recorded Telugu IVR Voice Engine
For medium-risk patients, relying on manual phone calls from ASHAs is inefficient. Mitra integrates an automated Interactive Voice Response (IVR) engine:
* **Pre-Recorded Voice Prompts:** Calls feature clear, pre-recorded audio prompts recorded by native Telugu speakers rather than robotic text-to-speech engines.
* **Interactive Confirmation:** Patients are asked to press `1` to confirm they will attend the AAM or press `2` if they need medicine delivered to their home.
* **Cost Efficiency:** At ₹0.15 per successful call, the IVR system reduces health-worker calling burden by $>75\%$ while achieving $>65\%$ contact rates.

### 16.3 The CHO Clinical & Administrative Portal
Community Health Officers at Ayushman Arogya Mandirs access a browser-based dashboard providing:
* **Facility Drug Stock Alert Map:** Highlights predicted stockouts of Amlodipine, Telmisartan, and Metformin 14 days before depletion.
* **Care-Continuity Waterfall:** Visualizes patient dropouts across the 5 stages of care for their catchment area.
* **Clinical Review Queue:** Flags patients returning for follow-up who require treatment titration or district specialist re-consultation.

---

## 17. Offline-First Architecture & Data Synchronization

Given the severe cellular connectivity constraints in rural Telangana, Mitra CareLoop utilizes an **Offline-First Synchronization Architecture**:

```text
[Cloud Server / PostgreSQL]
          │
          ▼ HTTPS / REST (When online)
[Delta Sync Engine] ◄── Compression & JSON Patch
          │
          ▼ Encrypted Storage (AES-256)
[Mobile Device / Local SQLite (SQLCipher)]
          │
          ▼ Offline Read/Write
[ASHA "NCD-Mitra" UI]
```

1. **Local SQLite with SQLCipher:** The mobile application stores all village-level master records, priority queues, and cached care journeys in a local SQLite database encrypted with SQLCipher (AES-256).
2. **Delta Synchronization:** When 2G/3G connectivity is detected, the app transmits only incremental JSON delta patches containing new outcome logs rather than full table dumps.
3. **Conflict Resolution:** In the event of conflicting updates between the server and the mobile client, the system applies a **Deterministic Vector Clock / Timestamp Rule** where confirmed clinical dispensing always supersedes unconfirmed predictive states.

---

## 18. Security, Privacy & ABDM Compliance

Healthcare data handling in Mitra adheres to the highest national and international standards:

### 18.1 ABDM Consent Architecture
Mitra integrates natively with the **Ayushman Bharat Digital Mission (ABDM)**:
* **ABHA Creation & Linking:** Patients are linked using their 14-digit ABHA (Ayushman Bharat Health Account) number.
* **Consent Manager Integration:** Data sharing between teleconsultation, pharmacy, and diagnostic systems is governed by explicit digital consent artifacts processed through ABDM's Consent Manager APIs.
* **Paper-Backed Fallback Consent:** For patients lacking digital literacy or smartphones, ASHAs record physical, thumbprint-verified consent forms, which are digitized and linked to the patient's record.

### 18.2 Telangana Rural Data Stewardship Charter (TRDSC-7)
In accordance with the TRDSC-7 framework:
* **Principle of Data Minimization:** ASHAs and IVR systems receive only the minimum necessary data required to execute their specific task (e.g., ASHA sees patient name, address, and medicine name, but cannot access full clinical diagnostic notes).
* **De-identification in AI Analytics:** Central machine learning training and inference pipelines operate exclusively on anonymized, pseudonymous UUIDs. Direct identifiers (names, full phone numbers, Aadhaar) are stored in a physically segregated, access-restricted key vault.
* **Right to Revoke & Erasure:** Patients retain the absolute legal right to withdraw consent from automated IVR calling or community follow-up tracking at any time.

---

## 19. Evaluation Framework & Submission Artifacts

Mitra CareLoop evaluates performance across three dimensions: Machine Learning Rigor, Operational Feasibility, and Economic Impact.

### 19.1 Machine Learning Evaluation Metrics
* **Discrimination:** Evaluated using Area Under the ROC Curve (AUROC) and Area Under the Precision-Recall Curve (AUPRC). Given the class imbalance ($48.1\%$ LTFU rate), AUPRC is the primary discrimination benchmark (Target: $\text{AUPRC} \ge 0.82$).
* **Probability Calibration:** Evaluated using Brier Score and calibration curves to verify that risk tiers reflect empirical dropout rates.
* **Recall@Worker Capacity:** Measures the percentage of true LTFU patients captured within the top $K$ cases assigned to an ASHA:
  $$\text{Recall@Capacity} = \frac{\text{True Positives in Top } K}{\text{Total True Positives in Village}}$$

### 19.2 Operational & Economic Metrics
* **Care-Continuity Closure Rate:** Target $>30\%$ reduction in loss to follow-up, shifting closed care journeys from $51.9\%$ to $>70\%$.
* **ASHA Action Rate:** Percentage of assigned high-risk patients visited within 48 hours (Target: $>85\%$).
* **Cost Per Retained Patient (CPRP):** Evaluates total operational spend (IVR calls + ASHA incentives + software infrastructure) per chronic patient successfully retained in care:
  $$\text{CPRP} = \frac{\text{Total Intervention Spend}}{\text{Retained Chronic Patients}} \approx \text{₹}42.50 \text{ per patient/year}$$
  Comparing favorably to the estimated $\text{₹}3,500 - \text{₹}12,000$ required to manage uncontrolled diabetic or hypertensive crises in secondary hospitals.

### 19.3 Competition Submission Templates Guide
Mitra CareLoop generates the three required submission files exactly to specification:

1. **`submission_template_linkage.csv`:**
   ```csv
   source_system,source_patient_id,predicted_patient_id,confidence
   teleconsultations,TEP0000001,P000001,0.985
   ```
2. **`submission_template_episode_predictions.csv`:**
   ```csv
   episode_id,risk_probability,predicted_lost_to_followup,predicted_dropout_stage,priority_tier
   E0000001,0.1245,0,Completed care journey,Tier 0
   E0000002,0.7832,1,Medicine not collected,Tier 3
   ```
3. **`submission_template_action_queue.csv`:**
   ```csv
   episode_id,predicted_patient_id,priority,reason,recommended_action,assigned_cadre
   E0000002,P000002,High,Prescribed Amlodipine not collected within 3 days,Conduct home visit and deliver medication,ASHA
   ```

---

## 20. System Summary & Strategic Impact

Mitra CareLoop bridges the critical implementation gap in rural telemedicine. By combining robust record linkage, leak-free machine learning, strict clinical guardrails, and radically simplified mobile workflows, the platform ensures that teleconsultations in rural Telangana deliver genuine clinical continuity.

$$\text{Fragmented Data} \xrightarrow[\text{CCI-7}]{\text{Linkage}} \text{Unified Care Graph} \xrightarrow[\text{Guardrails}]{\text{XGBoost}} \text{Calibrated Risk} \xrightarrow[\text{Kestrel-7}]{\text{Decision Trees}} \text{Targeted Action} \longrightarrow \mathbf{Closed\text{-}Loop\ Care}$$
