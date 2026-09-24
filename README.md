# Mitra — Closed-Loop Rural Teleconsultation Follow-Up System

Mitra is an enterprise-grade, data-driven care-continuity and follow-up assurance platform designed to eliminate Loss to Follow-Up (LTFU) after rural teleconsultations across Telangana.

The platform connects fragmented healthcare records, reconstructs each patient's longitudinal care journey, predicts dropout risk before it occurs, identifies the precise operational barrier, and converts predictive intelligence into actionable, capped tasks for Accredited Social Health Activists (ASHAs) and Community Health Officers (CHOs).

The system is designed around a single guiding principle:

> **A completed teleconsultation should not be treated as completed care.**

The platform follows every chronic-care patient across the complete care continuum:

```text
Teleconsultation
      ↓
Prescription
      ↓
Medicine Collection
      ↓
Diagnostic Testing
      ↓
30-Day Follow-up Review
      ↓
Care Closure
```

---

## 1. Problem Definition

In rural primary healthcare under the Ayushman Arogya Mandir (AAM) program, CHOs connect patients with hypertension and diabetes to specialist physicians over eSanjeevani. However, once the video or audio call ends, the system marks the encounter as "Completed" with zero visibility into post-consultation outcomes. Patients must collect prescribed medications, complete mandatory laboratory tests, and return for review within 30 days. A large share never do.

Data tracking this journey is scattered across unlinked, siloed databases:
* Canonical Patient Registry (`patient_360_reference.csv`)
* eSanjeevani Teleconsultation Logs (`teleconsultations.csv`)
* Baseline NCD Screening Records (`ncd_screening.csv`)
* Digital Prescriptions (`prescriptions.csv`)
* Pharmacy Dispensing Ledgers (`medicine_dispensing.csv`)
* Diagnostic Laboratory Information Systems (`lab_tests.csv`)
* Scheduled Follow-Up Registers (`followup_visits.csv`)
* Historical Encounter Registers (`visit_history.csv`)
* ASHA Field Outreach Logs (`outreach_actions.csv`)
* Facility Medicine Stock Ledgers (`medicine_stock_status.csv`)
* Healthcare Facility Master (`facility_reference.csv`)
* Village Geographic Master (`geography_reference.csv`)
* Care Episode Ground Truth Outcomes (`episode_outcomes.csv`)

Because these systems utilize unlinked local identifiers (`tele_source_patient_id`, `ncd_source_patient_id`, `rx_source_patient_id`, etc.) containing deliberate spelling, age, and formatting discrepancies, records are never natively linked.

Mitra resolves this fragmentation by establishing an end-to-end data linkage, predictive modeling, and closed-loop intervention pipeline.

---

## 2. Solution Overview

Mitra operates across five coordinated processing layers:

```text
┌────────────────────────────────────────────────────────────────────────┐
│                        RAW DATA SOURCES (13 CSVs)                      │
│ Teleconsultation | NCD | Pharmacy | Labs | Follow-up | Stock | Geo     │
└───────────────────────────────────┬────────────────────────────────────┘
                                    │
                                    ▼
┌────────────────────────────────────────────────────────────────────────┐
│                   DATA LINKAGE & ENTITY RESOLUTION                     │
│ ABHA-First Matching + Metaphone Phonetic & Spatial Fallback Algorithm  │
│ Output: submission_template_linkage.csv                                │
└───────────────────────────────────┬────────────────────────────────────┘
                                    │
                                    ▼
┌────────────────────────────────────────────────────────────────────────┐
│                 EPISODE RECONSTRUCTION & CARE CONTINUUM                │
│ Anchor: teleconsult_id ──► episode_id | Care Graph Construction       │
│ Measurement: Continuity-of-Care Index (CCI-7) across 7 Care Edges      │
└───────────────────────────────────┬────────────────────────────────────┘
                                    │
                                    ▼
┌────────────────────────────────────────────────────────────────────────┐
│              ANALYTICS ENGINE & ACCURACY GUARDRAILS (XGBoost)          │
│ Prediction at T_pred | Calibrated P(LTFU) | Monotonicity Constraints   │
│ Clinical Sanity Checks | Demographic Parity & Fairness Controls       │
│ Output: submission_template_episode_predictions.csv                   │
└───────────────────────────────────┬────────────────────────────────────┘
                                    │
                                    ▼
┌────────────────────────────────────────────────────────────────────────┐
│             ROOT-CAUSE BARRIER ENGINE & DECISION MAPPING               │
│ Dropout Stage Detection ──► Root-Cause Barrier ──► Kestrel-7 Ladder    │
│ Output: submission_template_action_queue.csv                           │
└───────────────────────────────────┬────────────────────────────────────┘
                                    │
                                    ▼
┌────────────────────────────────────────────────────────────────────────┐
│                         MITRA ACTION LAYER                             │
│ • ASHA "NCD-Mitra" Offline Android App (Top-5 Daily Tasks, SQLite)     │
│ • Automated Localized Telugu IVR Calls (T-3 & T-0 Auto-dials)          │
│ • CHO Clinical & Supply Portal (Drug Stockout & Escalation Alerts)     │
└────────────────────────────────────────────────────────────────────────┘
```

---

## 3. Comprehensive Dataset Blueprint & Utilization Guide

The dataset (`Infinum_2026_Candidate_Dataset_Pack`) models 5,000 synthetic patients and 5,516 teleconsultation episodes. Below is the blueprint detailing how each file is ingested, joined, and utilized:

### 3.1 Dataset Inventory & Join Crosswalk

| Dataset | Primary Key | Foreign Key / Join Condition | Noise / Challenge | Pipeline Utilization Role |
| :--- | :--- | :--- | :--- | :--- |
| `patient_360_reference.csv` | `patient_id` | Canonical Master Entity Table | Masked mobile (`XXXXXX5988`) | Target canonical entity table. Ground truth for all patient linkage. |
| `teleconsultations.csv` | `teleconsult_id` | `facility_id` $\to$ `facility_reference` | `tele_source_patient_id` (unlinked), typos | **Episode Anchor.** Identifies consult encounter, advice flags, connectivity, distance. |
| `episode_outcomes.csv` | `episode_id` | `teleconsult_id` $\to$ `teleconsultations` | Blank labels in EVALUATION cohort | Maps `episode_id` to `teleconsult_id`. Ground truth for model training and validation. |
| `ncd_screening.csv` | `ncd_record_id` | `ncd_source_patient_id` $\to$ canonical `patient_id` | `ncd_source_patient_id`, typos in name/age | Pre-consultation clinical baseline (SBP, DBP, Glucose, BMI, NCD status). |
| `prescriptions.csv` | `prescription_id`, `line_id`| `teleconsult_id` $\to$ `teleconsultations` | `rx_source_patient_id` (unlinked) | Prescribed regimen: drug names, classes, dosages, pill burden. |
| `medicine_dispensing.csv`| `dispense_id` | `prescription_id` $\to$ `prescriptions` | `pharm_source_patient_id` (unlinked) | Post-consultation drug collection verification. Quarantined at prediction time. |
| `medicine_stock_status.csv`| `stock_record_id` | `facility_id` $\to$ `facility_reference` | Clean monthly snapshots | Supply-side drug availability context (opening stock, stockout days). |
| `lab_tests.csv` | `lab_record_id` | `lab_source_patient_id` $\to$ canonical `patient_id` | `lab_source_patient_id`, missing result dates | Diagnostic orders, sample collection dates, completion statuses, abnormal flags. |
| `followup_visits.csv` | `visit_id` | `episode_id` $\to$ `episode_outcomes` | `visit_source_patient_id` (unlinked) | Scheduled follow-up visit attendance, clinical status, referrals. |
| `visit_history.csv` | `visit_id` | `visit_source_patient_id` $\to$ canonical `patient_id` | `visit_source_patient_id` | Historical clinic visits prior to consult. Used to compute past compliance. |
| `outreach_actions.csv` | `outreach_id` | `episode_id` $\to$ `episode_outcomes` | `outreach_source_patient_id` | Past and ongoing ASHA/CHO outreach attempts, contact outcomes, and methods. |
| `facility_reference.csv`| `facility_id` | Keyed by `facility_id` | Clean master | PHC/HWC context, network reliability, CHO and linked ASHA counts. |
| `geography_reference.csv`| `village_id` | `village` $\to$ Patient/Facility tables | Minor spelling variants | Village population, road access quality, mobile connectivity score. |

---

## 4. End-to-End Core Workflow

Mitra processes every teleconsultation episode through eight systematic steps:

### Step 1 — Resolve Patient Identity
Unlinked source records are resolved to `patient_360_reference.csv` using ABHA-first matching and a secondary phonetic-spatial rule:
$$\text{DoubleMetaphone}(\text{Name}) \land \text{Sex} \land |\Delta \text{Age}| \le 2 \land \text{Village Match} \land \text{Last 4 Digits Mobile}$$
Generates `submission_template_linkage.csv`.

### Step 2 — Reconstruct Care Episode & Measure Continuity
The teleconsultation serves as the central anchor (`episode_id`). Prescriptions, lab orders, baseline NCD context, and outreach history are attached. Linkage completeness is quantified using the **Continuity-of-Care Index (CCI-7)** across seven core transitions:
$$\text{CCI-7} = \frac{1}{7} \sum_{i=1}^7 e_i, \quad e_i \in \{0.0, 0.5, 1.0\}$$

### Step 3 — Construct Temporal Features
Derives features representing demographics, chronic severity, pill burden, geographic distance, historical compliance, and facility drug stock known strictly at $T_{\text{pred}}$.

### Step 4 — Predict LTFU Risk with Calibrated XGBoost
A tabular XGBoost classifier estimates:
$$P(\text{LTFU}) \in [0.00, 1.00]$$
The probability is calibrated via Isotonic Regression and converted into operational Priority Tiers (Tier 0 to Tier 4).

### Step 5 — Predict Dropout Stage
An auxiliary multi-class classifier identifies the likely stage of care failure:
* `Medicine not collected` (58.9% of empirical dropouts)
* `Test not completed` (13.8% of empirical dropouts)
* `Review not attended` (27.3% of empirical dropouts)
* `Completed care journey` (Cleared)

Generates `submission_template_episode_predictions.csv`.

### Step 6 — Identify Root-Cause Operational Barrier
Combines predicted stage with facility stock status, road access, and past contact logs:
* Systemic Stockout Barrier
* Geographic / Road Accessibility Barrier
* Communication / Inactive Phone Barrier
* Diagnostic Equipment / Lab Hub Barrier
* Behavioral / Asymptomatic Non-compliance Barrier

### Step 7 — Assign Intervention via Kestrel-7 Ladder
Translates risk, stage, and barrier into the Kestrel-7 escalation framework:
* **K1:** Passive Monitoring
* **K2:** Automated Telugu IVR Voice Call / SMS
* **K3:** Direct ASHA Mobile Call
* **K4:** In-Person ASHA Home Visit & Drug Delivery
* **K5:** Facility Operational Resolution / Stock Transfer
* **K6:** CHO Clinical Re-evaluation at AAM
* **K7:** Specialist Referral to District Hospital

### Step 8 — Dispatch to Mitra Action Queue
Generates `submission_template_action_queue.csv` containing:
`episode_id`, `predicted_patient_id`, `priority`, `reason`, `recommended_action`, `assigned_cadre`.

---

## 5. AI Model Integration Architecture

Mitra's AI model integration operates as an asynchronous, modular inference pipeline connecting central public health servers to distributed field workers:

```text
[eSanjeevani Webhook] ──► [Inference Service (FastAPI / Celery)]
                                    │
                                    ▼
                     [Feature Extraction & Store]
                     (Demographics, Rx, Stock, Geo)
                                    │
                                    ▼
                     [Calibrated XGBoost Classifier]
                     (Probability & Multi-Class Stage)
                                    │
                                    ▼
                     [Deterministic Sanity Guardrails]
                                    │
                                    ▼
                     [Decision Mapping Matrix Engine]
                                    │
         ┌──────────────────────────┼──────────────────────────┐
         │                          │                          │
         ▼                          ▼                          ▼
 [ASHA "NCD-Mitra" App]   [Automated Telugu IVR]     [CHO Web Dashboard]
 (Offline SQLite AES-256) (T-3 & T-0 Voice Calls)    (Facility Stock Alerts)
```

1. **Inference Trigger:** A completed teleconsultation in eSanjeevani triggers feature assembly in the Mitra Feature Store.
2. **Tabular XGBoost Execution:** Evaluates tabular vectors through gradient-boosted decision trees optimized for tabular healthcare data.
3. **Probability Calibration:** Uncalibrated leaf scores are passed through Isotonic Regression curves to guarantee accurate empirical risk probabilities.
4. **Field Task Dispatch:** Tasks are published via lightweight MQTT/REST APIs to the frontline interfaces.

---

## 6. Model Training Accuracy, Guardrails & Anti-Manipulation Framework

Deploying machine learning in rural public health requires strict safeguards against model hallucinations, look-ahead leakage, demographic bias, and data manipulation. Mitra implements five layers of hard guardrails:

```text
┌────────────────────────────────────────────────────────────────────────┐
│                   MITRA 5-TIER ACCURACY & GUARDRAIL SUITE              │
├────────────────────────────────────────────────────────────────────────┤
│ 1. STRICT TEMPORAL BOUNDARY (Anti-Leakage Protocol)                    │
│    • Prediction fixed at consult completion (T_pred). Zero future data.│
├────────────────────────────────────────────────────────────────────────┤
│ 2. ENFORCED FEATURE MONOTONICITY CONSTRAINTS                           │
│    • Distance ↑ => Risk ↑; Pill Burden ↑ => Risk ↑; Stockout ↑ => Risk ↑│
├────────────────────────────────────────────────────────────────────────┤
│ 3. DETERMINISTIC CLINICAL SANITY GUARDRAILS                            │
│    • Hard medical rules override impossible AI advice predictions.     │
├────────────────────────────────────────────────────────────────────────┤
│ 4. DEMOGRAPHIC FAIRNESS & STATISTICAL CALIBRATION                      │
│    • Brier score calibration + equalized opportunity across hamlets.   │
├────────────────────────────────────────────────────────────────────────┤
│ 5. CAPACITY THROTTLING & TAMPER-PROOF AUDIT TRAILS                     │
│    • Capped Top-5 daily visits per ASHA + SHA-256 state hashing.       │
└────────────────────────────────────────────────────────────────────────┘
```

### 6.1 Strict Temporal Boundary & Anti-Leakage
* **Prediction Cutoff ($T_{\text{pred}}$):** Occurs at teleconsultation completion.
* **Permitted Features:** Demographics, pre-consult NCD screening, current prescription, historical visits, facility drug stock in preceding month.
* **Quarantined Features:** Post-consultation dispensing ledgers, future lab completion records, subsequent follow-up visit registers, future outreach actions, and outcome labels.

### 6.2 Enforced Feature Monotonicity Constraints
Clinical physics require that risk does not decrease when vulnerability factors increase. Mitra mathematically enforces monotonicity in XGBoost:
$$\frac{\partial P(\text{LTFU})}{\partial (\text{distance\_km})} \ge 0, \quad \frac{\partial P(\text{LTFU})}{\partial (\text{pill\_burden})} \ge 0, \quad \frac{\partial P(\text{LTFU})}{\partial (\text{past\_missed\_visits})} \ge 0$$
$$\frac{\partial P(\text{LTFU})}{\partial (\text{road\_quality})} \le 0, \quad \frac{\partial P(\text{LTFU})}{\partial (\text{past\_compliance})} \le 0$$

### 6.3 Deterministic Clinical Sanity Guardrails
Hard business logic prevents model hallucinations from dispatching invalid field actions:
* If `medicine_advised == 'No'`, the system **cannot** output `Medicine not collected` or dispatch drug delivery tasks.
* If `test_advised == 'No'`, the system **cannot** output `Test not completed` or dispatch diagnostic reminders.
* If a facility experiences an active drug stockout (`stockout_flag_month == 1`), the system **cannot** blame patient non-compliance; the task is routed to the CHO for supply reordering.

### 6.4 Capacity Throttling & Anti-Flooding Controls
To prevent cognitive overload, task abandonment, or fraudulent batch-marking by frontline workers:
* Active ASHA daily worklists are strictly capped at the **Top 5 High-Priority Patients per village**.
* Overflow medium/high cases are automatically diverted to the automated Telugu IVR telephony engine or queued for monthly Village Health Days (VHSND).

### 6.5 Cryptographic Auditability & Human-in-the-Loop Overrides
* Every prediction logs a cryptographic **SHA-256 state hash**:
  $$\text{SHA-256}(\text{episode\_id} \,\|\, \text{features} \,\|\, \text{model\_version} \,\|\, \text{timestamp})$$
* ASHAs and CHOs can override any AI recommendation with a mandatory single-tap reason code (`Patient Migrated`, `Purchased at Private Pharmacy`). Overrides feed into drift monitoring without allowing unauthorized data tampering.

---

## 7. Operational Decision Mapping & Decision Trees

Mitra converts raw data and model predictions into actionable field decisions using five explicit decision trees:

### Decision Tree 1: Entity Resolution & Record Linkage
```text
Source Record
     │
     ▼
Is ABHA ID Present?
     ├── Yes ──► Direct Canonical Link (Confidence = 1.00)
     └── No  ──► Deterministic Name + Phone Match?
                      ├── Yes ──► Link Assigned (Confidence = 0.95)
                      └── No  ──► Double Metaphone(Name) + Sex + Age ±2 + Village Similarity ≥ 0.85
                                       ├── Match ──► Link Assigned (Confidence = 0.80 - 0.94)
                                       └── Fail  ──► Cluster as New Unlinked Entity
```

### Decision Tree 2: Risk Probability to Priority Tier Mapping
```text
Calibrated P(LTFU)
     │
     ├── P ≥ 0.70 ──► Comorbidity == 2 OR SBP ≥ 160?
     │                     ├── Yes ──► Tier 4 (Critical) ──► Same-day CHO & ASHA Action
     │                     └── No  ──► Tier 3 (High)     ──► 48-Hour ASHA Home Visit
     ├── 0.35 ≤ P < 0.70 ────────────► Tier 2 (Medium)   ──► Telugu IVR Calls (T-3, T-0)
     ├── 0.15 ≤ P < 0.35 ────────────► Tier 1 (Low)      ──► Automated Telugu SMS
     └── P < 0.15 ───────────────────► Tier 0 (Monitor)  ──► Passive Register Tracking
```

### Decision Tree 3: Dropout Stage Attribution
```text
Teleconsultation Advice Flags
     │
     ├── medicine_advised == 'Yes' ──► Dispensed within 3 days?
     │                                      ├── Yes ──► Stage Cleared
     │                                      └── No  ──► [Medicine not collected]
     ├── test_advised == 'Yes'     ──► Sample collected within 7 days?
     │                                      ├── Yes ──► Stage Cleared
     │                                      └── No  ──► [Test not completed]
     └── review_advised == 'Yes'   ──► Review attended within Due Days + 7?
                                            ├── Yes ──► Stage Cleared
                                            └── No  ──► [Review not attended]
```

### Decision Tree 4: Root-Cause Barrier Identification
```text
Predicted Dropout Stage
     │
     ├── [Medicine not collected] ──► Facility Stockout Flag == 1?
     │                                     ├── Yes ──► [Medicine Stockout Barrier]
     │                                     └── No  ──► Distance > 10 km?
     │                                                    ├── Yes ──► [Geographic Access Barrier]
     │                                                    └── No  ──► [Adherence Barrier]
     ├── [Test not completed]     ──► Lab Type == 'External' OR Distance > 15 km?
     │                                     ├── Yes ──► [Diagnostic Accessibility Barrier]
     │                                     └── No  ──► [Diagnostic Adherence Barrier]
     └── [Review not attended]    ──► Phone Switched Off / Invalid?
                                           ├── Yes ──► [Communication Barrier]
                                           └── No  ──► Distance > 15 km OR Road == 'Poor'?
                                                          ├── Yes ──► [Physical Transport Barrier]
                                                          └── No  ──► [Behavioral Adherence Barrier]
```

### Decision Tree 5: Operational Cadre Dispatch & Worklist Generation
```text
Priority Tier + Dropout Stage + Root-Cause Barrier
     │
     ├── Tier 3/4 + Meds/Review Gap + ASHA Capacity Available (< 5) ──► ASHA Home Visit Task
     ├── Any Tier + Medicine Stockout Barrier                       ──► CHO Drug Reorder Action
     ├── Tier 1/2 + Routine Review Gap + Mobile Connectivity Good   ──► Automated Telugu IVR Call
     └── Tier 3/4 + Diagnostic Gap                                  ──► ASHA VHSND Mobilization
```

---

## 8. Frontline Healthcare Worker Interfaces

### ASHA "NCD-Mitra" Offline Android App
* **Top 5 Daily Priority Task Cards:** No complex menus or data entry. Displays the 5 most critical patients in the ASHA's village each morning.
* **Offline-First SQLite (SQLCipher AES-256):** Functions fully without cell reception; syncs incremental JSON delta patches when 2G/3G is available.
* **1-Click Outcome Logging:** Single tap updates (`Meds Delivered`, `Patient Reminded`, `Patient Refused`, `Patient Migrated`).
* **Vernacular Interface:** Native conversational Telugu language display.

### Automated Telugu IVR Voice Engine
* **Pre-Recorded Native Audio:** Delivers culturally resonant, warm voice calls in authentic regional dialect at $T-3$ days and $T-0$ days.
* **Interactive IVR Touch-Tone:** Patients press `1` to confirm attendance or press `2` to request home medicine delivery.
* **Cost Efficiency:** Approximately ₹0.15 per completed call, reducing manual calling burden by $>75\%$.

### CHO Clinical Dashboard
* **14-Day Predictive Drug Stockout Map:** Alerts CHOs to impending shortages of Amlodipine and Metformin before patients arrive.
* **Care-Continuity Waterfall:** Identifies catchment-area dropouts across the 5 care milestones.

---

## 9. Technology Stack

* **Data & Analytics:** Python, Pandas, NumPy, Scikit-learn, XGBoost (with monotonic constraints)
* **Backend API & Service:** FastAPI, Celery, Pydantic, REST / MQTT
* **Storage & Encryption:** PostgreSQL (central), SQLite with SQLCipher AES-256 (offline mobile)
* **Frontline Client:** Lightweight Android Application (Java/Kotlin), pre-rendered XML cards
* **Telephony Gateway:** Cloud IVR Gateway with pre-recorded Telugu audio prompts
* **Regulatory Compliance:** ABDM Consent Architecture, DPDP Act 2023, TRDSC-7

---

## 10. Summary Matrix

| Dimension | Specification |
| :--- | :--- |
| **Primary Entity Matching** | ABHA-First + Double Metaphone Phonetic & Spatial Fallback |
| **Continuity Metric** | Continuity-of-Care Index (CCI-7) across 7 transitions |
| **Machine Learning Engine** | Calibrated Tabular XGBoost with Enforced Monotonicity |
| **Target Variables** | Binary `lost_to_followup_label`, Multi-class `dropout_stage_label`, Priority Tiers 0–4 |
| **Frontline Mobile Interface** | ASHA "NCD-Mitra" Offline Android App (Top-5 Daily Tasks, SQLite AES-256) |
| **Automated Nudge Channel** | Pre-recorded Vernacular Telugu IVR Voice Telephony (₹0.15/call) |
| **Escalation Framework** | Kestrel-7 Ladder (K1 Monitor to K7 Specialist Referral) |
| **Data Governance** | ABDM Consent Manager + TRDSC-7 + SHA-256 Audit Trail |
