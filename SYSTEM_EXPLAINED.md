# Mitra CareLoop — System Architecture, AI Integration & Decision Mapping Explained

> **Comprehensive Plain-English Guide to the Mitra Follow-Up Assurance Platform for Rural Teleconsultations**

---

## 1. Executive Overview & The Core Real-World Problem

### 1.1 The Illusion of "Completed Care"
In rural Telangana, the primary healthcare backbone is the **Ayushman Arogya Mandir (AAM)** (formerly Sub-Health Centres / Health & Wellness Centres). Community Health Officers (CHOs) connect rural patients suffering from chronic non-communicable diseases (NCDs)—predominantly **Hypertension** and **Type 2 Diabetes**—to specialist physicians at district hospitals using the **eSanjeevani** teleconsultation platform.

When a 15-minute video or audio call concludes, eSanjeevani marks the encounter as **"Completed"**. 

However, in clinical reality:
$$\mathbf{A\ Completed\ Teleconsultation\ Is\ NOT\ Completed\ Care.}$$

A consultation is only the very first milestone of a chronic care journey:
```text
[Teleconsultation Encounter]
           │
           ▼
[Doctor Issues Prescription & Lab Orders]
           │
           ▼
[Patient Must Collect Medicines at Pharmacy]
           │
           ▼
[Patient Must Provide Diagnostic Lab Samples]
           │
           ▼
[Patient Must Return for 30-Day Clinical Review]
           │
           ▼
[Care Closure & Health Stabilization]
```

### 1.2 The Failure Reality in the Field
A large portion of rural patients drop out of care immediately after the virtual call:
* They do not collect their blood pressure or diabetes medicines.
* They never complete recommended diagnostic blood tests (like HbA1c or Serum Creatinine).
* They fail to return for their scheduled 30-day review.

The healthcare administration has **zero visibility** into this post-consultation breakdown. Data documenting these downstream steps sits scattered across isolated, disconnected computer systems and paper registers:
* Teleconsultation logs inside eSanjeevani.
* NCD screening records in the national NCD portal.
* Pharmacy drug inventory and dispensing registers (e-Aushadhi).
* Diagnostic laboratory logbooks.
* Physical clinic visit registers.
* ASHA paper diaries and field logs.

These systems use completely different patient IDs and contain spelling mistakes, phonetically spelled names, and formatting variations. They are **never linked together**.

### 1.3 What Mitra Does
**Mitra CareLoop** connects these fragmented puzzle pieces into a single unified patient timeline, predicts who is at risk of dropping out before it happens, figures out the root-cause reason for the dropout, and delivers simple, prioritized action tasks to frontline health workers (ASHAs and CHOs) and patients via automated local-dialect voice calls.

---

## 2. Walkthrough: The Patient Journey Example

To understand how Mitra works in practice, follow the care journey of **Mohan Basha**, a 47-year-old farmer living in the rural village of Chandur in Nalgonda district:

```text
Day 0: The Doctor's Call
Mohan visits his village AAM. The CHO connects him to a specialist doctor over eSanjeevani.
Doctor's diagnosis: Hypertension + Diabetes.
Doctor advises:
  • Medicine Advised = YES (Cetirizine, Amlodipine, Metformin)
  • Lab Test Advised = YES (Lipid Profile, Serum Creatinine)
  • Review Advised   = YES (Return in 30 days)

The call disconnects. In legacy systems, Mohan's file is closed as "Success".
-----------------------------------------------------------------------------------------
Day 0 (Instant): Mitra Steps In
1. Linkage Engine: Connects Mohan's eSanjeevani ID (TEP0000001) to his canonical profile (P000001)
   using his ABHA ID and phonetic Double Metaphone matching.
2. Timeline Builder: Constructs an active care episode (E0000001) with 3 open milestones:
   [Medicine Collection], [Lab Testing], [30-Day Review].
3. AI Predictor (XGBoost): Evaluates Mohan's features:
   • Distance to clinic: 29.7 km (High)
   • Pill burden: 3 daily drugs (High complexity)
   • Past history: 2 missed visits last year
   • Local clinic stock: Metformin opening stock is critically low
   -> AI Risk Score: P(LTFU) = 0.78 (High Risk / Tier 3).
4. Guardrails & Sanity Check: Monotonicity rules confirm risk is genuine; clinical sanity
   confirms drugs and tests were genuinely ordered by the doctor.
5. Root-Cause Barrier Engine: Detects two major failure points:
   • Dropout Stage: Medicine not collected.
   • Root-Cause Barrier: Long travel distance (29.7 km) + impending pharmacy stockout.
6. Action Dispatch:
   • Task to Local Clinic CHO: Expedite Metformin stock transfer from district warehouse.
   • Task to Village ASHA: Mohan is placed on her "NCD-Mitra" mobile app Top-5 worklist.
-----------------------------------------------------------------------------------------
Day 2: Action in the Village
ASHA Sunitha opens her phone. She sees Mohan's card: "Deliver 30-day BP & Sugar pack".
She visits Mohan's house during her village round, hands over the medicines, and explains how
to take them. She taps [Meds Delivered] on her app (100% offline).
-----------------------------------------------------------------------------------------
Day 27: Automated Local Nudge
Mitra's Automated Telugu IVR system dials Mohan's phone:
"Namaskaram Mohan garu! Your health checkup at Chandur clinic is due in 3 days. Press 1 to confirm."
Mohan presses 1.
-----------------------------------------------------------------------------------------
Day 30: Care Completed & Closed
Mohan attends his review at the clinic. His blood pressure is stable.
Mitra updates his longitudinal timeline:
  Teleconsult ✓  -->  Medicines Dispensed ✓  -->  Review Attended ✓
Status: CARE CLOSED. Mohan was saved from dropping out.
```

---

## 3. The 13 Datasets Explained: Roles, Schemas & Connections

The synthetic dataset provided (`Infinum_2026_Candidate_Dataset_Pack`) models 5,000 synthetic patients and 5,516 teleconsultation episodes. Below is a detailed explanation of each file and how it connects into the system:

```text
                               ┌─────────────────────────┐
                               │ patient_360_reference  │
                               │   (Canonical Master)    │
                               └────────────┬────────────┘
                                            │ (Linked via Entity Resolution)
                 ┌──────────────────────────┼──────────────────────────┐
                 │                          │                          │
                 ▼                          ▼                          ▼
     ┌──────────────────────┐   ┌──────────────────────┐   ┌──────────────────────┐
     │  teleconsultations   │   │    ncd_screening     │   │    visit_history     │
     │  (Episode Anchor)    │   │ (Pre-Consult Vitals) │   │ (Historical Visits)  │
     └──────────┬───────────┘   └──────────────────────┘   └──────────────────────┘
                │
                ├──────────────────────────────────────────────────────┐
                │ (Anchor: teleconsult_id -> episode_id)               │
                ▼                                                      ▼
     ┌──────────────────────┐                              ┌──────────────────────┐
     │    prescriptions     │                              │   episode_outcomes   │
     │ (Prescribed Regimen) │                              │ (Ground-Truth Labels)│
     └──────────┬───────────┘                              └──────────┬───────────┘
                │                                                      │
                ▼                                                      ▼
     ┌──────────────────────┐                              ┌──────────────────────┐
     │ medicine_dispensing  │                              │   followup_visits    │
     │ (Pharmacy Records)   │                              │ (30-Day Review Logs) │
     └──────────────────────┘                              └──────────────────────┘
                ▲                                                      ▲
                │ (Facility Context)                                   │
     ┌──────────┴───────────┐                              ┌──────────┴───────────┐
     │ medicine_stock_status│                              │   outreach_actions   │
     │  (Drug Inventory)    │                              │ (ASHA/CHO Call Logs) │
     └──────────┬───────────┘                              └──────────────────────┘
                │
                ▼
     ┌──────────────────────┐   ┌──────────────────────┐   ┌──────────────────────┐
     │  facility_reference  │   │ geography_reference  │   │      lab_tests       │
     │ (PHC/HWC Staff & Net)│   │ (Roads & Cell Signal)│   │ (Diagnostics Orders) │
     └──────────────────────┘   └──────────────────────┘   └──────────────────────┘
```

### Detailed Dataset Dictionary & Connection Blueprint

1. **`patient_360_reference.csv` (The Canonical Entity Master):**
   * **Contains:** `patient_id` (e.g. `P000001`), clean canonical name, gender, date of birth, age, masked mobile (`XXXXXX5988`), village, block, district, language, known NCD status, vulnerability group (`Elderly living alone`, `Seasonal migrant`, `Low digital access`, etc.).
   * **Role:** The single source of truth for patient identities. All other unlinked source records must be mapped to this file.
2. **`teleconsultations.csv` (The Episode Anchor):**
   * **Contains:** `teleconsult_id` (e.g. `TC0000001`), unlinked `tele_source_patient_id`, patient name, age, phone, village, `consult_date`, facility ID, chief complaint, diagnosis group, consult mode, `medicine_advised` (Yes/No), `test_advised` (Yes/No), `review_advised` (Yes/No), `review_due_days`, `distance_to_facility_km`, `connectivity_quality`.
   * **Role:** Serves as the anchor for every care episode. Captures the exact orders given by the doctor.
3. **`episode_outcomes.csv` (The Ground Truth Training & Test Labels):**
   * **Contains:** `episode_id` (e.g. `E0000001`), `teleconsult_id`, `consult_date`, `cohort` (`DEVELOPMENT` vs `EVALUATION`), `lost_to_followup_label` (1 = Dropped out, 0 = Completed care), `dropout_stage_label` (`Medicine not collected`, `Test not completed`, `Review not attended`, `Completed care journey`).
   * **Role:** Maps `episode_id` to `teleconsult_id`. Used to train and validate machine learning models on the DEVELOPMENT cohort (4,132 episodes) and evaluate on the EVALUATION cohort (1,384 episodes).
4. **`ncd_screening.csv` (Pre-Consultation Clinical Baseline):**
   * **Contains:** `ncd_record_id`, unlinked `ncd_source_patient_id`, screening date, systolic BP, diastolic BP, random glucose, BMI, NCD status, control status.
   * **Role:** Provides clinical severity context recorded *before* the teleconsultation took place.
5. **`prescriptions.csv` (Prescribed Treatment Regimen):**
   * **Contains:** `prescription_id`, `prescription_line_id`, `teleconsult_id`, medicine name, class (e.g., Antihypertensive, Biguanide), frequency, `days_prescribed`, route.
   * **Role:** Linked to `teleconsultations` via `teleconsult_id`. Used to calculate the patient's **Pill Burden** (number of daily tablets and regimen complexity).
6. **`medicine_dispensing.csv` (Pharmacy Dispensing Records):**
   * **Contains:** `dispense_id`, `prescription_id`, medicine name, prescribed quantity proxy, dispensed quantity, `partial_fill` (0/1), `stockout_flag` (0/1), `dispense_status` (`Dispensed`, `Partially dispensed`, `Not dispensed due to stock`).
   * **Role:** Verifies whether the patient actually collected their medicine. **Note:** Quarantined at prediction time to prevent temporal target leakage.
7. **`medicine_stock_status.csv` (Systemic Supply Context):**
   * **Contains:** `stock_record_id`, `facility_id`, snapshot month, medicine name, opening stock, received quantity, dispensed quantity, closing stock, `stockout_flag_month`, `stockout_days`, `stock_status` (`Adequate`, `Low stock`, `Reorder raised`).
   * **Role:** Identifies clinic-side inventory shortages, allowing the system to distinguish between a patient refusing medicine vs the clinic running out of stock.
8. **`lab_tests.csv` (Diagnostic Testing Records):**
   * **Contains:** `lab_record_id`, unlinked `lab_source_patient_id`, order date, test name (CBC, Creatinine, HbA1c, ECG, Lipid Profile), sample date, result date, result flag (`Normal`, `Abnormal`, `Critical`), test status (`Available`, `Ordered-not-completed`), lab type (`Facility lab`, `Hub lab`, `External lab`).
   * **Role:** Tracks whether ordered laboratory tests were completed.
9. **`followup_visits.csv` (Follow-Up Encounter Register):**
   * **Contains:** `visit_id`, unlinked `visit_source_patient_id`, visit date, visit type, `episode_id`, clinical status (`Improved`, `Stable`, `Needs escalation`), referral flag.
   * **Role:** Confirms whether the patient attended their scheduled 30-day review visit.
10. **`visit_history.csv` (Behavioral Care History):**
    * **Contains:** Prior clinic encounters occurring before the current teleconsultation episode.
    * **Role:** Used to compute historical compliance ratios and count past missed visits.
11. **`outreach_actions.csv` (Outreach Interaction Log):**
    * **Contains:** `outreach_id`, `episode_id`, action date, action method (`Phone call`, `Home visit`, `Village health day reminder`, `WhatsApp/SMS`), contact outcome (`Reached; promised follow-up`, `Phone switched off`, `Patient unavailable`), cadre (ASHA/CHO), followup reason.
    * **Role:** Documents past outreach attempts and identifies unreachable phone numbers.
12. **`facility_reference.csv` (Facility Staff & Infrastructure):**
    * **Contains:** `facility_id`, facility name, type (PHC/HWC), coordinates, `network_context` (`Offline-prone`), CHO count, linked ASHA count.
    * **Role:** Provides facility capacity and cellular infrastructure context.
13. **`geography_reference.csv` (Geographic Accessibility):**
    * **Contains:** `village_id`, village name, population, `road_access` (`Paved`, `Mostly paved`, `Mixed`, `Poor`), `mobile_connectivity` (`Good`, `Moderate`, `Weak`, `Very weak`).
    * **Role:** Establishes physical and digital access barriers for every village.

---

## 4. Component 1: Data Linkage & Entity Resolution Engine

### 4.1 The Challenge
Every operational system uses its own local patient ID. For instance, Mohan Basha has:
* `TEP0000001` in teleconsultations
* `NCP0000001` in NCD screening
* `RXP0000001` in prescriptions
* `PHP0000001` in pharmacy dispensing
* `LAP0000001` in lab tests

Moreover, names are misspelled across registers (`Mohan Basha` vs `M. Basha`), phone numbers have missing digits, and ages vary by 1–2 years due to self-reporting differences.

### 4.2 The Linkage Decision Tree
Mitra resolves these records to the canonical `patient_360_reference.csv` using a hierarchical matching algorithm:

```text
                          [Raw Record from Any Source]
                                       │
                                       ▼
                     Does record have a valid ABHA ID?
                                       │
                      ┌────────────────┴────────────────┐
                      │ YES                             │ NO
                      ▼                                 ▼
             [Direct ABHA Link]              Does Normalized Full Name
             Confidence = 1.000             AND 10-Digit Phone Match?
                                                        │
                                       ┌────────────────┴────────────────┐
                                       │ YES                             │ NO
                                       ▼                                 ▼
                              [Deterministic Match]            Calculate Composite
                               Confidence = 0.950                 Fuzzy Score
```

### 4.3 The Multi-Condition Fuzzy Matcher
If direct matching fails, the engine applies a deterministic multi-condition rule:
1. **Double Metaphone Phonetic Encoding:** Converts names to phonetic sound codes. `"Laxmi"` and `"Lakshmi"` generate the exact same phonetic key (`LKSM`).
2. **Gender Exact Match:** Source gender must equal canonical gender.
3. **Age Tolerance Window:** $|\text{Source Age} - \text{Canonical Age}| \le 2\text{ years}$.
4. **Village Jaro-Winkler Similarity:** Spelling similarity between village strings must be $\ge 0.85$.
5. **Masked Mobile Match:** The last 4 digits of the phone number must match the canonical masked mobile (`XXXXXX5988` $\to$ `5988`).

Composite confidence formula:
$$\text{Confidence} = 0.35 \cdot \text{NameSim} + 0.30 \cdot \mathbf{1}_{\{\text{Phone4}\}} + 0.15 \cdot \text{VillageSim} + 0.10 \cdot \mathbf{1}_{\{\Delta\text{Age} \le 2\}} + 0.10 \cdot \mathbf{1}_{\{\text{Gender}\}}$$

Matches with $\text{Confidence} \ge 0.80$ are accepted and output to `submission_template_linkage.csv`.

---

## 5. Component 2: Episode Reconstruction & The CCI-7 Metric

Once records are linked to a single patient, Mitra anchors on `teleconsult_id` and builds the patient's care timeline.

### 5.1 The Continuity-of-Care Index (CCI-7)
To ensure the healthcare system maintains complete records across the patient's care journey, Mitra calculates the **Continuity-of-Care Index (CCI-7)**. It measures whether data was successfully recorded across **seven critical care transitions**:

| Edge | Transition Checked | What Success Looks Like |
| :---: | :--- | :--- |
| **$e_1$** | Patient $\to$ Teleconsultation | Source record successfully linked to canonical patient. |
| **$e_2$** | Teleconsultation $\to$ NCD Context | Baseline blood pressure and glucose records exist within 180 days. |
| **$e_3$** | Teleconsultation $\to$ Prescription | Prescription records exist for all advised medicines. |
| **$e_4$** | Prescription $\to$ Pharmacy Dispensing | Dispensing record confirms medicines were handed out. |
| **$e_5$** | Test Advised $\to$ Lab Result | Diagnostic lab system logs sample collection and results. |
| **$e_6$** | Review Advised $\to$ Follow-Up Visit | Visit register confirms patient attended 30-day review. |
| **$e_7$** | Care Gap $\to$ Outreach Action | If a gap occurred, outreach register logs an ASHA or IVR contact. |

Calculation:
$$\text{CCI-7} = \frac{1}{7} \sum_{i=1}^{7} e_i, \quad e_i \in \{0.0, 0.5, 1.0\}$$

**Why this matters:** If a district's average CCI-7 drops from $0.90$ to $0.65$, it immediately proves that local pharmacy or lab operators have stopped entering data, allowing supervisors to fix data bottlenecks before patients drop out.

---

## 6. Component 3: AI Analytics Engine (XGBoost)

### 6.1 Proactive Prediction Point ($T_{\text{pred}}$)
Legacy healthcare dashboards only flag dropouts **after** 30 days have passed and the patient is already lost. 

Mitra predicts dropout risk **immediately upon teleconsultation completion ($T_{\text{pred}}$)**. This gives health workers a 30-day proactive window to intervene before the patient drops out.

### 6.2 Feature Vector Inputs (Known at $T_{\text{pred}}$)
1. **Demographic & Vulnerability:** Age, gender, vulnerability category (`Elderly living alone`, `Seasonal migrant`, `Low digital access`), language.
2. **Clinical Severity:** Baseline SBP/DBP, random blood sugar, BMI, comorbidity status (Hypertension + Diabetes).
3. **Pill Burden & Prescription Complexity:** Total prescribed drugs, maximum days prescribed, high-risk cardiovascular drugs (Amlodipine, Metformin).
4. **Geographic Accessibility:** Distance from village to clinic in km, road access condition (Paved vs Poor), cellular signal strength.
5. **Historical Patient Behavior:** Past clinic visits, past missed appointments, past compliance ratio, historical outreach contact failures.
6. **Supply Chain Context:** Whether the local clinic had an active drug stockout in the preceding month.

### 6.3 Dual Model Architecture
1. **Primary Model (Binary XGBoost):** Predicts calibrated dropout probability $P(\text{LTFU}) \in [0.00, 1.00]$.
2. **Secondary Model (Multi-Class XGBoost):** Predicts the conditional probability distribution over the three potential failure stages:
   * $P(\text{Medicine not collected})$
   * $P(\text{Test not completed})$
   * $P(\text{Review not attended})$

Outputs are written to `submission_template_episode_predictions.csv`.

---

## 7. Component 4: Accuracy Guardrails & Anti-Manipulation Suite

Deploying AI in rural public health requires strict mathematical and operational constraints to prevent hallucinations, gaming, and data manipulation:

```text
┌────────────────────────────────────────────────────────────────────────┐
│                   MITRA 6-TIER ACCURACY & GUARDRAIL SUITE              │
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
│ 4. STATISTICAL CALIBRATION & DEMOGRAPHIC FAIRNESS                      │
│    • Isotonic calibration + equalized opportunity across hamlets.      │
├────────────────────────────────────────────────────────────────────────┤
│ 5. CAPACITY THROTTLING & ANTI-FLOODING CONTROLS                        │
│    • Capped Top-5 daily tasks per ASHA village. Prevents gaming.       │
├────────────────────────────────────────────────────────────────────────┤
│ 6. CRYPTOGRAPHIC AUDIT TRAILS & HUMAN OVERRIDES                        │
│    • SHA-256 state hashing + mandatory worker reason codes.            │
└────────────────────────────────────────────────────────────────────────┘
```

### Detailed Guardrail Explanations
1. **Strict Temporal Anti-Leakage Boundary:**
   * The model is strictly barred from seeing future data. Post-consultation dispensing records, future lab completion dates, subsequent clinic visits, and future outreach notes are placed in an isolated quarantine.
2. **Enforced Monotonicity Constraints:**
   * In unconstrained tree models, noisy data can create nonsensical rules (e.g., claiming a patient living 30 km away is safer than one living 2 km away).
   * Mitra hard-codes monotonic directionality in XGBoost:
     * Travel distance $\uparrow \implies$ Risk $\not\downarrow$ (Non-decreasing)
     * Pill burden $\uparrow \implies$ Risk $\not\downarrow$ (Non-decreasing)
     * Past missed appointments $\uparrow \implies$ Risk $\not\downarrow$ (Non-decreasing)
     * Road quality $\uparrow \implies$ Risk $\not\uparrow$ (Non-increasing)
3. **Deterministic Clinical Sanity Bounds:**
   * If `medicine_advised == 'No'`, the system physically forbids the AI from predicting `Medicine not collected` or ordering drug deliveries.
   * If `test_advised == 'No'`, diagnostic reminders are blocked.
   * If a clinic suffers an active drug stockout (`stockout_flag_month == 1`), the system forbids blaming the patient; the task is routed to the CHO for supply reordering.
4. **Statistical Probability Calibration (Isotonic Regression):**
   * A raw tree score of $0.70$ is mathematically calibrated so that exactly 70 out of 100 validation patients with that score empirically drop out.
5. **Capacity Throttling (Top-5 Daily Tasks per Village):**
   * Flooding an ASHA with 30 tasks guarantees task abandonment or fake batch-logging. Mitra strictly caps daily home visits to the **Top 5 High-Risk Patients per village**.
6. **Cryptographic SHA-256 Audit Hashing & Human Overrides:**
   * Every prediction logs a SHA-256 digital fingerprint:
     $$\text{Hash} = \text{SHA-256}(\text{episode\_id} \,\|\, \text{features} \,\|\, \text{model\_version} \,\|\, \text{timestamp})$$
   * Frontline health workers can override AI decisions (e.g., marking "Patient migrated to city for harvest") with a mandatory single-tap reason code. Overrides are tracked for model drift without allowing arbitrary data editing.

---

## 8. Component 5: Complete Operational Decision Trees

Mitra maps raw data and model predictions into real-world actions using five explicit decision trees:

### Decision Tree 1: Entity Resolution & Canonical Linkage
```text
                      [Raw Operational Record]
                                 │
                 Is valid 14-digit ABHA ID present?
                    ├── YES ──► Canonical Match (Confidence = 1.00)
                    └── NO  ──► Does Phone & Full Name match Canonical?
                                   ├── YES ──► Canonical Match (Confidence = 0.95)
                                   └── NO  ──► Check Double Metaphone(Name)
                                               + Gender Match
                                               + Age within ±2 Years
                                               + Village Similarity ≥ 0.85
                                               + Masked Phone 4 Digits Match
                                                    ├── YES ──► Fuzzy Match (Confidence ≥ 0.80)
                                                    └── NO  ──► Unlinked Record (Quarantined)
```

### Decision Tree 2: Risk Probability to Priority Tier Mapping
```text
                          [Calibrated P(LTFU)]
                                   │
               ┌───────────────────┴───────────────────┐
               │                                       │
        P(LTFU) ≥ 0.70                           P(LTFU) < 0.70
               │                                       │
     ┌─────────┴─────────┐                   ┌─────────┴─────────┐
     │                   │                   │                   │
Comorbidity == 2   Comorbidity < 2     0.35 ≤ P < 0.70         P < 0.35
or SBP ≥ 160       & Stable Vitals           │                   │
     │                   │                   │                   │
     ▼                   ▼                   ▼                   ▼
 [TIER 4]            [TIER 3]            [TIER 2]            [TIER 1]
(CRITICAL)            (HIGH)             (MEDIUM)             (LOW)
Same-Day CHO      48-Hour ASHA        Automated Telugu    Automated Telugu
+ ASHA Action      Home Visit            IVR Calls          SMS Nudge
```

### Decision Tree 3: Dropout Stage Attribution
```text
                     [Doctor's Advice Flags at T_pred]
                                     │
     ┌───────────────────────────────┼───────────────────────────────┐
     │                               │                               │
medicine_advised == 'Yes'   test_advised == 'Yes'          review_advised == 'Yes'
     │                               │                               │
Was medicine dispensed       Was lab sample collected        Did patient attend review
within 3 days of consult?    within 7 days of consult?       within Due Days + 7 days?
     ├── YES ──► Cleared             ├── YES ──► Cleared             ├── YES ──► Cleared
     └── NO                          └── NO                          └── NO
          │                               │                               │
          ▼                               ▼                               ▼
[Medicine not collected]         [Test not completed]            [Review not attended]
(58.9% of Dropouts)              (13.8% of Dropouts)             (27.3% of Dropouts)
```

### Decision Tree 4: Root-Cause Barrier Identification
```text
                          [Predicted Dropout Stage]
                                     │
     ┌───────────────────────────────┼───────────────────────────────┐
     │                               │                               │
[Medicine not collected]    [Test not completed]            [Review not attended]
     │                               │                               │
Is clinic drug stockout      Is lab external or              Is patient's mobile phone
flagged in current month?    distance to clinic > 15 km?     switched off or invalid?
  ├── YES ──► [STOCKOUT]          ├── YES ──► [DIAGNOSTIC          ├── YES ──► [COMMUNICATION
  └── NO                          │            ACCESS]             └── NO       BARRIER]
       │                          └── NO                                │
Distance > 10 km?                      │                       Distance > 15 km or
  ├── YES ──► [GEOGRAPHIC              ▼                       road access == 'Poor'?
  │            ACCESS]           Test status is                  ├── YES ──► [PHYSICAL
  └── NO  ──► [BEHAVIORAL        'Ordered-not-completed'?        │            TRANSPORT]
               ADHERENCE]          ├── YES ──► [LAB SUPPLY]      └── NO  ──► [BEHAVIORAL
                                   └── NO  ──► [ADHERENCE]                    ADHERENCE]
```

### Decision Tree 5: Operational Cadre Dispatch & Action Queue
```text
                [Priority Tier + Dropout Stage + Root-Cause Barrier]
                                         │
     ┌───────────────────────────────────┼───────────────────────────────────┐
     │                                   │                                   │
[Tier 3/4 + Meds/Review Gap]     [Any Tier + Stockout Barrier]       [Tier 1/2 + Review Gap]
     │                                   │                                   │
Is ASHA Village Daily Task         Assign task to Facility CHO:        Mobile connectivity
Count < 5?                         "Initiate inter-facility            is Good or Moderate?
  ├── YES ──► ASHA Home Visit      drug transfer and notify             ├── YES ──► Automated
  └── NO      Worklist Card         district warehouse"                 │           Telugu IVR
       │                                                                └── NO  ──► Vernacular
       ▼                                                                            SMS / VHSND
Re-route overflow cases to                                                          Notice
Automated Telugu IVR or VHSND
```

Outputs populate `submission_template_action_queue.csv`:
```csv
episode_id,predicted_patient_id,priority,reason,recommended_action,assigned_cadre
E0000004,P000004,High,Prescribed Amlodipine not collected within 3 days,Conduct home visit and deliver medication blister pack,ASHA
E0000012,P000012,Critical,Uncontrolled HTN+DM with Metformin stockout at PHC,Arrange inter-facility medicine transfer and expedite CHO review,CHO
E0000019,P000019,Medium,Scheduled 30-day review due in 3 days,Dispatch automated pre-recorded Telugu IVR reminder call,IVR
```

---

## 9. Component 6: Frontline Operational Delivery Interfaces

### 9.1 The ASHA "NCD-Mitra" Offline Android App
* **Radical Simplicity:** ASHAs are community health workers with limited digital literacy. The app features **zero complex dashboards**.
* **Top-5 Daily Cards:** Each morning, the app presents a clean stack of **5 cards** representing the 5 highest-risk patients in her village.
* **100% Offline Capability:** Operates entirely from an embedded SQLite database encrypted with SQLCipher (AES-256). No cell connectivity is required during village rounds.
* **1-Click Outcome Logging:** Action buttons: `[Meds Delivered]`, `[Patient Reminded]`, `[Refused]`, `[Migrated]`.
* **Delta Synchronization:** When 2G/3G connectivity is reached, the app transmits lightweight JSON patches.

### 9.2 Automated Pre-Recorded Telugu IVR Voice Engine
* **High Efficiency at Scale:** For medium-risk patients (Tier 2), sending an ASHA is an inefficient use of scarce time.
* **Culturally Resonant Voice:** Calls are pre-recorded by native Telugu speakers using authentic regional idioms, achieving $>65\%$ pick-up rates.
* **Touch-Tone Verification:** Prompts: *"Press 1 to confirm clinic review visit; Press 2 to request medicine delivery assistance."*
* **Low Cost:** At $\approx\text{₹}0.15$ per call, thousands of patients are nudged for minimal public expenditure.

### 9.3 The CHO Clinical & Supply Dashboard
* **14-Day Predictive Drug Stockout Map:** Monitors monthly balances in `medicine_stock_status.csv` and warns CHOs two weeks before Amlodipine or Metformin runs out.
* **Clinical Review Queue:** Surfaces high-risk patients returning for review who require regimen adjustments or specialist teleconsultation escalation (Kestrel-7 Level K7).

---

## 10. Component 7: Closed-Loop Care Closure Verification

Mitra ensures care is closed through verifiable evidence rather than assumptions:

```text
[Episode Initiated at Teleconsultation]
                   │
                   ▼
[Milestone 1: Medicine Dispensing Verified in Pharmacy Register]  ──► [YES]
                   │
                   ▼
[Milestone 2: Diagnostic Sample & Result Verified in Lab System]  ──► [YES]
                   │
                   ▼
[Milestone 3: 30-Day Follow-Up Attendance Verified in Visit Log]  ──► [YES]
                   │
                   ▼
       ═════════════════════════
       STATUS: CARE JOURNEY CLOSED
       ═════════════════════════
```

If any milestone is unverified after the due window, the episode automatically triggers the **Kestrel-7 Escalation Ladder** until resolved or formally closed by the CHO.

---

## 11. Quick Reference Architecture Matrix

| Dimension | Implementation Details | Target Submission File |
| :--- | :--- | :--- |
| **Data Linkage Engine** | ABHA ID First + Double Metaphone Phonetic Name Matching + Gender + Age $\pm 2$ + Village Jaro-Winkler $\ge 0.85$ + Masked Mobile | `submission_template_linkage.csv` |
| **Data Continuity Benchmark** | Continuity-of-Care Index (CCI-7) evaluating 7 core transitions | District Health KPI |
| **Risk Prediction Model** | Tabular XGBoost with Isotonic Probability Calibration & Enforced Monotonicity Constraints | `submission_template_episode_predictions.csv` |
| **Prediction Target Variables** | Binary `lost_to_followup_label` ($0/1$), Multi-class `dropout_stage_label`, Priority Tiers ($0 - 4$) | `submission_template_episode_predictions.csv` |
| **Frontline Mobile Interface** | ASHA "NCD-Mitra" Offline Android App (Embedded SQLite / SQLCipher AES-256, Top-5 Daily Task Cards, 1-Click Logging) | Frontline Field App |
| **Automated Nudge Channel** | Pre-recorded Vernacular Telugu IVR Voice Telephony (Calls at $T-3$ & $T-0$ days, ₹0.15/call) | Automated Telephony |
| **Operational Intervention Queue** | Action Queue mapped via Kestrel-7 Ladder (K1 Monitor to K7 Specialist Referral) with ASHA Workload Throttling | `submission_template_action_queue.csv` |
| **Regulatory & Privacy Standard** | ABDM Consent Manager + TRDSC-7 Charter + De-identified Patient UUIDs + SHA-256 Audit Hashes | Compliance Ledger |
