### System At A Glance

```

DISPARATE DATA SOURCES          LINKAGE ENGINE          ANALYTICS ENGINE           ACTIONABLE OUTCOMES
┌─────────────────────────┐    ┌──────────────────┐    ┌──────────────────┐    ┌─────────────────────────────┐
│ • eSanjeevani Logs      │    │ Deterministic &  │    │ XGBoost Model    │    │ ASHA "NCD-Mitra" App        │
│ • NCD Screening Apps    │───►│ Fuzzy Matching   │───►│ Risk Probability │───►│ (Offline Top-5 Task Card)   │
│ • Pharmacy Registers    │    │ Engine           │    │ Generation       │    ├─────────────────────────────┤
│ • ASHA Offline Diaries  │    │ (ABHA Primary)   │    │ (0.00 to 1.00)   │    │ Automated Voice IVR         │
└─────────────────────────┘    └──────────────────┘    └──────────────────┘    │ (Pre-recorded Telugu Calls) │
└─────────────────────────────┘

```

### Key Components

#### 1. Data Linkage & Deterministic Reconciliation Engine
* **ABHA-First Linkage:** Uses the Ayushman Bharat Health Account (ABHA ID) as the primary key across eSanjeevani, NCD screening, and pharmacy databases.   

* **Phonetic & Spatial Fallback Algorithm:** For legacy or unlinked records missing an ABHA ID, a secondary Python-based reconciliation script combines:   
  * Metaphone/Soundex Phonetic Encoding on `Patient_Name` (to account for spelling variations across registers).
  * Deterministic Multi-Condition Rule: `[Phonetic Name Match]` AND `[Gender]` AND `[Age ± 2 Years]` AND `[Village/ASHA Catchment ID]` AND `[Phone Number]`.

* **Master Patient Continuum Log:** Constructs a single linear event timeline per patient:

$$\text{Consultation Completed} \longrightarrow \text{Medicine Dispensed} \longrightarrow \text{Diagnostics Done} \longrightarrow \text{30-Day Follow-Up Review}$$

#### 2. AI & Analytics Engine (XGBoost Classifier)
* **Proactive Scoring:** Scores risk immediately after teleconsultation instead of waiting for a missed 30-day follow-up.   

* **Feature Inputs:**
  * **Demographic & Geographic:** Age, gender, distance to Ayushman Arogya Mandir (AAM), village remoteness.   
  * **Clinical Severity:** Baseline Systolic/Diastolic BP, RBS/HbA1c levels, total prescribed medications (pill burden), comorbidity status (Hypertension + Diabetes).   
  * **Behavioral History:** Delay (in days) between initial screening and teleconsultation, past missed appointments.   

* **Operational Risk Tiers:** Outputs a probability score $P(\text{LTFU}) \in [0, 1]$ categorized into:   
  * **High Risk ($P \ge 0.70$):** Flagged for an immediate ASHA home visit.   
  * **Medium Risk ($0.35 \le P < 0.70$):** Queued for automated Telugu voice calls (IVR).   
  * **Low Risk ($P < 0.35$):** Sent standard text reminders.   

#### 3. Operational Delivery & Health Worker Intervention
* **ASHA Priority Worklist App:**
  * Runs an offline-first Android App backed by an embedded SQLite database.   
  * Displays a simple, color-coded daily list of the Top 5 High-Risk Patients per village, eliminating complex dashboards.   
  * Features 1-Click Logging ("Meds Delivered", "Patient Refused", "Rescheduled").   

* **Automated Voice Calls (IVR):**
  * Sends pre-recorded Telugu voice calls to patient mobile phones 3 days before and on the day of their follow-up date.
  * Reduces manual calling burden for Community Health Officers (CHOs) and ASHAs.   

#### 4. Rural Constraints & Technical Feasibility
* **Zero-Connectivity Resilience:** The ASHA mobile app functions fully offline. Data synchronization uses lightweight delta-sync payloads over MQTT/REST APIs whenever 2G/3G connectivity becomes available.   

* **Low-Tech Hardware Optimization:** Designed for basic Android smartphones with minimal storage and battery consumption.   

#### 5. Privacy Safeguards & ABDM Compliance
* **Data Security at Rest:** Local mobile SQLite databases are encrypted using SQLCipher (AES-256).   

* **Consent Architecture:** Captures digital or paper-backed explicit consent via ABDM's Consent Manager API during initial registration.   

* **De-identification:** Central AI batch processing operates on anonymized patient UUIDs rather than raw personal details.   

#### 6. Metrics & Impact Assessment
* **Clinical Effectiveness:** Target $>30\%$ reduction in Patients Lost to Follow-Up (LTFU) and increased 30-day medication pickup rates.   

* **Feasibility & Adoption:** Measured by the ASHA Action Rate (% of flagged high-risk patients visited within 5 days).   

* **Financial Sustainability:** Evaluated using Cost Per Retained Patient (CPRP). Automated bulk IVR calls cost $\approx\text{₹}0.15$ per call, making the system cost-effective compared to treating chronic complications at district hospitals.   

---

### Summary Matrix

| Metric / Dimension | Finalized Solution Choice |
| :--- | :--- |
| **Primary Identifier** | ABHA ID + Metaphone Fuzzy Matcher |
| **Machine Learning Model** | XGBoost Classifier (Optimized for Tabular Data) |
| **Primary Interface** | Lightweight Offline Android App (SQLite) for ASHAs |
| **Patient Nudge Channel** | Pre-recorded Local Dialect (Telugu) IVR Voice Calls |
| **Data Security Standard** | ABDM Compliant + Local AES-256 Encryption |

```
