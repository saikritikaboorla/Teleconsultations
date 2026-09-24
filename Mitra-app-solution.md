### System At A Glance

```

DISPARATE DATA SOURCES          LINKAGE ENGINE          ANALYTICS ENGINE           ACTIONABLE OUTCOMES
┌─────────────────────────┐    ┌──────────────────┐    ┌──────────────────┐    ┌─────────────────────────────┐
│ • eSanjeevani Logs      │    │ Deterministic &  │    │ XGBoost Model    │    │ ASHA "NCD-Mitra" App        │
│ • NCD Screening Apps    │───►│ Fuzzy Matching   │───►│ Risk Probability │───►│ (Offline Top-5 Task Card)   │
│ • Pharmacy Registers    │    │ Engine           │    │ Generation       │    ├─────────────────────────────┤
│ • ASHA Offline Diaries  │    │ (ABHA Primary)   │    │ (0.00 to 1.00)   │    │ Automated Voice IVR         │
│ • Lab & Stock Registers │    │ (CCI-7 Verified) │    │ & Guardrails     │    │ (Pre-recorded Telugu Calls) │
└─────────────────────────┘    └──────────────────┘    └──────────────────┘    ├─────────────────────────────┤
                                                                               │ CHO Clinical & Stock Portal │
                                                                               └─────────────────────────────┘

```

### Key Components

#### 1. Data Linkage & Deterministic Reconciliation Engine
* **ABHA-First Linkage:** Uses the Ayushman Bharat Health Account (ABHA ID) as the primary key across eSanjeevani, NCD screening, pharmacy, and laboratory databases.   

* **Phonetic & Spatial Fallback Algorithm:** For legacy or unlinked records missing an ABHA ID, a secondary Python-based reconciliation script combines:   
  * Metaphone/Soundex Phonetic Encoding on `Patient_Name` (to account for spelling variations across registers).
  * Deterministic Multi-Condition Rule: `[Phonetic Name Match]` AND `[Gender]` AND `[Age ± 2 Years]` AND `[Village/ASHA Catchment ID]` AND `[Phone Number (Last 4 Digits)]`.

* **Master Patient Continuum Log:** Constructs a single linear event timeline per patient:

$$\text{Consultation Completed} \longrightarrow \text{Medicine Dispensed} \longrightarrow \text{Diagnostics Done} \longrightarrow \text{30-Day Follow-Up Review} \longrightarrow \text{Care Closed}$$

* **Continuity-of-Care Index (CCI-7):** Quantifies data completeness across seven core transitions: Patient $\to$ Teleconsult, Consult $\to$ NCD, Consult $\to$ Rx, Rx $\to$ Dispensing, Test $\to$ Lab, Consult $\to$ Review, and Gap $\to$ Outreach.

---

#### 2. AI & Analytics Engine (XGBoost Classifier)
* **Proactive Scoring:** Scores risk immediately after teleconsultation ($T_{\text{pred}}$) instead of waiting for a missed 30-day follow-up.   

* **Feature Inputs:**
  * **Demographic & Geographic:** Age, gender, distance to Ayushman Arogya Mandir (AAM), village remoteness, road quality.   
  * **Clinical Severity:** Baseline Systolic/Diastolic BP, RBS/HbA1c levels, total prescribed medications (pill burden), comorbidity status (Hypertension + Diabetes).   
  * **Behavioral History:** Delay (in days) between initial screening and teleconsultation, past missed appointments, past outreach outcomes.   
  * **Systemic Supply Context:** Facility drug stock status and monthly stockout flags.

* **Operational Risk Tiers:** Outputs a calibrated probability score $P(\text{LTFU}) \in [0, 1]$ categorized into:   
  * **Critical / Tier 4 ($P \ge 0.70$ + Severe Comorbidity):** Joint same-day CHO clinical review and ASHA home outreach.
  * **High Risk / Tier 3 ($P \ge 0.70$):** Flagged for an immediate ASHA home visit within 48 hours.   
  * **Medium Risk / Tier 2 ($0.35 \le P < 0.70$):** Queued for automated Telugu voice calls (IVR) at $T-3$ and $T-0$ days.   
  * **Low Risk / Tier 1 ($0.15 \le P < 0.35$):** Sent standard automated Telugu text reminders.   
  * **Monitor / Tier 0 ($P < 0.15$):** Monitored through passive routine registers.

---

#### 3. Operational Delivery & Health Worker Intervention
* **ASHA Priority Worklist App ("NCD-Mitra"):**
  * Runs an offline-first Android App backed by an embedded SQLite database.   
  * Displays a simple, color-coded daily list of the **Top 5 High-Risk Patients per village**, eliminating complex dashboards and cognitive fatigue.   
  * Features 1-Click Logging ("Meds Delivered", "Patient Reminded", "Patient Refused", "Patient Migrated").   

* **Automated Voice Calls (IVR):**
  * Sends pre-recorded native Telugu voice calls to patient mobile phones 3 days before and on the day of their follow-up date.
  * Interactive touch-tone response (`Press 1 to confirm attendance`, `Press 2 to request home delivery`).
  * Reduces manual calling burden for Community Health Officers (CHOs) and ASHAs by $>75\%$.   

* **CHO Clinical & Supply Portal:**
  * Displays 14-day predictive medicine stockout alerts to prevent facility-level supply failure.
  * Escalates complex clinical dropouts or uncontrolled vitals for specialist district review.

---

#### 4. Rural Constraints & Technical Feasibility
* **Zero-Connectivity Resilience:** The ASHA mobile app functions fully offline. Data synchronization uses lightweight delta-sync JSON payloads over MQTT/REST APIs whenever 2G/3G connectivity becomes available.   

* **Low-Tech Hardware Optimization:** Designed for basic entry-level Android smartphones with minimal storage, CPU, and battery consumption.   

---

#### 5. Privacy Safeguards, TRDSC-7 & ABDM Compliance
* **Data Security at Rest:** Local mobile SQLite databases are encrypted using SQLCipher (AES-256).   

* **Consent Architecture:** Captures digital or paper-backed explicit consent via ABDM's Consent Manager API during initial registration.   

* **Telangana Rural Data Stewardship Charter (TRDSC-7):** Enforces data minimization—ASHAs see only necessary task details, while central machine learning pipelines operate strictly on de-identified patient UUIDs.   

* **Cryptographic Tamper-Proofing:** Every prediction and recommendation logs an immutable SHA-256 state hash. Worker overrides require single-tap reason codes.

---

#### 6. Metrics & Impact Assessment
* **Clinical Effectiveness:** Target $>30\%$ reduction in Patients Lost to Follow-Up (LTFU) and increased 30-day medication pickup rates.   

* **Feasibility & Adoption:** Measured by the ASHA Action Rate (% of flagged high-risk patients visited within 48 hours, target $>85\%$).   

* **Financial Sustainability:** Evaluated using Cost Per Retained Patient (CPRP). Automated bulk IVR calls cost $\approx\text{₹}0.15$ per call, achieving an annual retention cost of $\approx\text{₹}42.50$ per patient compared to $\text{₹}3,500 - \text{₹}12,000$ for treating chronic complications at district hospitals.   

---

#### 7. Dataset-to-Decision Mapping & Operational Decision Trees

Mitra maps the 13 raw dataset files directly into deterministic decision trees to execute operational decisions:

```text
┌────────────────────────────────────────────────────────────────────────────────────────┐
│                         DATASET-TO-DECISION MAPPING PIPELINE                           │
├────────────────────────────────────────────────────────────────────────────────────────┤
│ 1. ENTITY RESOLUTION TREE:                                                             │
│    Inputs: patient_360_reference, teleconsultations, ncd_screening, prescriptions...   │
│    Logic: If ABHA ID Match => Link (1.00); Else If Phone + Name Match => Link (0.95);  │
│           Else If Metaphone(Name) + Sex + Age ±2 + Village >= 0.85 => Link (0.85);     │
│    Target Output: submission_template_linkage.csv                                      │
├────────────────────────────────────────────────────────────────────────────────────────┤
│ 2. RISK TIER DECISION TREE:                                                            │
│    Inputs: Episode features evaluated at consult_date (T_pred)                         │
│    Logic: If P(LTFU) >= 0.70 & Comorbidities/Severe BP => Tier 4 (Critical);           │
│           If P(LTFU) >= 0.70 => Tier 3 (High);                                         │
│           If 0.35 <= P(LTFU) < 0.70 => Tier 2 (Medium);                                │
│           If 0.15 <= P(LTFU) < 0.35 => Tier 1 (Low); Else => Tier 0 (Monitor)          │
├────────────────────────────────────────────────────────────────────────────────────────┤
│ 3. DROPOUT STAGE DETECTION TREE:                                                       │
│    Inputs: teleconsultations advice flags + clinical milestone trackers                │
│    Logic: If medicine_advised == 'Yes' & Not Dispensed => Medicine not collected;      │
│           If test_advised == 'Yes' & No Sample Logged => Test not completed;           │
│           If review_advised == 'Yes' & Visit Absent => Review not attended;            │
│           Else => Completed care journey                                               │
│    Target Output: submission_template_episode_predictions.csv                          │
├────────────────────────────────────────────────────────────────────────────────────────┤
│ 4. ROOT-CAUSE BARRIER ATTRIBUTION TREE:                                                │
│    Inputs: medicine_stock_status, geography_reference, outreach_actions                │
│    Logic: If Stage == 'Medicine...' & Facility Stockout Flag == 1 => Stockout Barrier; │
│           If Distance > 10 km or Road == 'Poor' => Geographic Access Barrier;          │
│           If Phone Switched Off or Inactive => Contact Barrier;                        │
│           Else => Behavioral Adherence Barrier                                         │
├────────────────────────────────────────────────────────────────────────────────────────┤
│ 5. OPERATIONAL CADRE DISPATCH TREE:                                                    │
│    Inputs: Priority Tier + Dropout Stage + Root-Cause Barrier + Worker Capacity        │
│    Logic: If Tier 3/4 + Meds/Review Gap & ASHA Daily Tasks < 5 => ASHA Home Visit;     │
│           If Any Tier + Stockout Barrier => CHO Medicine Reorder Task;                 │
│           If Tier 1/2 + Review Gap & Connectivity Good => Automated Telugu IVR Call;   │
│           If Tier 3/4 + Diagnostic Gap => ASHA VHSND Mobilization                      │
│    Target Output: submission_template_action_queue.csv                                 │
└────────────────────────────────────────────────────────────────────────────────────────┘
```

---

#### 8. AI Model Integration, Accuracy & Anti-Manipulation Guardrails

To ensure model decisions are strictly accurate, clinically sound, and immune to manipulation or gaming:

* **Strict Prediction-Time Boundary ($T_{\text{pred}}$):** The model is evaluated strictly at the moment teleconsultation completes. Features timestamped after the consultation (actual dispensing dates, subsequent test completion, future follow-up encounters, or ground-truth outcome labels) are strictly quarantined to guarantee zero look-ahead bias.
* **Enforced Monotonicity Constraints:** Monotonic risk relationships are hard-coded into XGBoost:
  $$\frac{\partial P}{\partial (\text{distance})} \ge 0, \quad \frac{\partial P}{\partial (\text{pill\_burden})} \ge 0, \quad \frac{\partial P}{\partial (\text{past\_missed\_visits})} \ge 0$$
* **Deterministic Clinical Sanity Bounds:** Hard business logic suppresses impossible AI recommendations (e.g., if a doctor did not prescribe medicines, the system can never predict `Medicine not collected` or assign medicine delivery tasks).
* **Statistical Calibration & Demographic Fairness:** Raw probabilities are calibrated via Isotonic Regression to ensure risk scores equal true empirical failure rates. Decision boundaries are audited across gender, caste, and vulnerability segments (`Elderly living alone`, `Seasonal migrant`, `Low digital access`) to prevent demographic bias.
* **Capacity Throttling:** Strict cap of the **Top 5 daily patients per village** prevents ASHA cognitive overload and eliminates perverse incentives to falsify task completion.
* **Tamper-Proof Audit Logging:** Every inference computes an immutable SHA-256 state hash. Health-worker overrides require mandatory single-tap reason codes to monitor model drift without permitting ad-hoc tampering.

---

### Summary Matrix

| Metric / Dimension | Finalized Solution Choice |
| :--- | :--- |
| **Primary Identifier** | ABHA ID + Metaphone Fuzzy Matcher (`submission_template_linkage.csv`) |
| **Machine Learning Model** | Calibrated Tabular XGBoost Classifier with Monotonicity Constraints |
| **Target Variables** | Binary `lost_to_followup_label`, Multi-class `dropout_stage_label`, Priority Tiers 0–4 |
| **Continuity Benchmark** | Continuity-of-Care Index (CCI-7) across 7 care transitions |
| **Primary Interface** | Lightweight Offline Android App (SQLite/SQLCipher AES-256) for ASHAs |
| **Patient Nudge Channel** | Pre-recorded Local Dialect (Telugu) IVR Voice Calls (₹0.15/call) |
| **Intervention Framework** | Kestrel-7 Escalation Ladder (K1 Monitor to K7 Specialist Referral) |
| **Task Queue Output** | Priority-ranked Action Worklist (`submission_template_action_queue.csv`) |
| **Data Security Standard** | ABDM Compliant + TRDSC-7 + Local AES-256 Encryption + SHA-256 Audit Trail |
