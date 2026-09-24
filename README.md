# Teleconsultations
# Kestrel CareLoop

### Data-Driven Follow-Up Assurance for Rural Teleconsultations

Kestrel CareLoop is an AI-assisted care-continuity platform designed to identify and reduce **Loss to Follow-Up (LFU)** in rural teleconsultation workflows.

The platform integrates fragmented healthcare records, reconstructs patient care journeys, predicts LFU risk, identifies the likely point and cause of care disruption, and generates an actionable follow-up workflow for healthcare workers.

The system is designed around a closed-loop workflow:

> **Connect → Predict → Explain → Act → Close**

---

## 1. Problem

Rural teleconsultation involves multiple stages of care:

```text
Teleconsultation
      ↓
Prescription
      ↓
Medicine Collection
      ↓
Laboratory Tests
      ↓
Follow-up Review
      ↓
Care Completion
```

Information related to these stages may exist in separate systems such as:

* Teleconsultation records
* NCD records
* Prescription records
* Pharmacy and dispensing records
* Laboratory records
* Follow-up visits
* Outreach records
* Geographic information
* Facility information
* Medicine stock information

These datasets may use different identifiers and may contain inconsistencies such as spelling variations, incomplete records, or different representations of the same patient.

As a result, it can be difficult to determine:

* Which records belong to the same patient
* What happened during the patient's care journey
* Where the journey stopped
* Why the journey stopped
* Which patients require intervention
* What intervention is appropriate

---

# 2. Solution

Kestrel CareLoop creates a unified representation of each patient's care journey.

The system:

1. Ingests healthcare records from multiple sources.
2. Resolves records belonging to the same patient.
3. Reconstructs episode-level care journeys.
4. Measures record continuity using CCI-7.
5. Predicts LFU risk.
6. Identifies the likely dropout stage.
7. Determines the primary barrier contributing to the risk.
8. Selects an appropriate intervention using the Kestrel-7 intervention framework.
9. Generates an action queue for ASHAs and CHOs.
10. Tracks intervention outcomes and care closure.

---

# 3. System Workflow

```text
                    ┌──────────────────────┐
                    │   Healthcare Data    │
                    └──────────┬───────────┘
                               ↓
                    ┌──────────────────────┐
                    │  Data Normalization  │
                    └──────────┬───────────┘
                               ↓
                    ┌──────────────────────┐
                    │ Entity Resolution    │
                    │ & Patient Linking    │
                    └──────────┬───────────┘
                               ↓
                    ┌──────────────────────┐
                    │ Episode Reconstruction│
                    └──────────┬───────────┘
                               ↓
                    ┌──────────────────────┐
                    │       CCI-7          │
                    │ Care Continuity      │
                    └──────────┬───────────┘
                               ↓
                    ┌──────────────────────┐
                    │ Feature Engineering  │
                    └──────────┬───────────┘
                               ↓
                    ┌──────────────────────┐
                    │   LFU Risk Model    │
                    └──────────┬───────────┘
                               ↓
                    ┌──────────────────────┐
                    │ Dropout Stage &      │
                    │ Barrier Detection    │
                    └──────────┬───────────┘
                               ↓
                    ┌──────────────────────┐
                    │    Kestrel-7         │
                    │ Intervention Engine  │
                    └──────────┬───────────┘
                               ↓
                    ┌──────────────────────┐
                    │    Action Queue      │
                    └──────────┬───────────┘
                               ↓
                    ┌──────────────────────┐
                    │ ASHA / CHO Action    │
                    └──────────┬───────────┘
                               ↓
                    ┌──────────────────────┐
                    │    Care Closure      │
                    └──────────────────────┘
```

---

# 4. Core Components

## 4.1 Data Integration

The platform integrates records from multiple healthcare workflows.

Typical data sources include:

| Source           | Purpose                                 |
| ---------------- | --------------------------------------- |
| Teleconsultation | Consultation and episode information    |
| NCD              | Screening and chronic-condition context |
| Prescription     | Medicines prescribed                    |
| Pharmacy         | Medicine dispensing/collection          |
| Laboratory       | Test orders and results                 |
| Visits           | Follow-up attendance                    |
| Outreach         | ASHA/community outreach                 |
| Geography        | Distance and location-related features  |
| Facility         | Healthcare facility information         |
| Stock            | Medicine availability                   |

---

## 4.2 Entity Resolution

Different healthcare systems may identify the same patient differently.

Kestrel CareLoop resolves these records into a canonical patient identity.

Matching signals can include:

* Name
* Phone number
* Age
* Gender
* Village
* Block
* District
* Facility
* Temporal proximity

The process combines exact and fuzzy matching to produce a match confidence.

```text
Source Record A
       │
       ├── Name similarity
       ├── Phone similarity
       ├── Location similarity
       ├── Demographic similarity
       └── Temporal similarity
                 │
                 ↓
          Match Confidence
                 │
                 ↓
       Canonical Patient ID
```

---

# 5. Care Journey Reconstruction

After patient records are linked, related records are organized into care episodes.

A typical journey is:

```text
Teleconsultation
       ↓
Prescription
       ↓
Medicine
       ↓
Laboratory Test
       ↓
Follow-up
       ↓
Care Completion
```

The system tracks the state of each stage.

Example:

```text
Teleconsultation    ✓
Prescription        ✓
Medicine            ✗
Laboratory          ?
Follow-up           ?
```

This identifies medicine collection as the first unresolved stage.

---

# 6. CCI-7

## Continuity-of-Care Index

CCI-7 measures the completeness of the reconstructed care journey.

It evaluates seven relationships:

1. Patient → Teleconsultation
2. Teleconsultation → NCD context
3. Teleconsultation → Prescription
4. Prescription → Pharmacy
5. Test requirement → Laboratory
6. Episode → Follow-up
7. Episode → Outreach

Each relationship is evaluated as linked, partially linked, or unresolved.

Example:

```text
Patient → Teleconsultation       ✓
Teleconsultation → NCD           ✓
Teleconsultation → Prescription  ✓
Prescription → Pharmacy          ✓
Test → Laboratory                ✓
Episode → Follow-up              ?
Episode → Outreach               ✓

CCI-7 = 6 / 7
```

CCI-7 measures **data and care-record continuity**. It is not a clinical score.

---

# 7. LFU Risk Prediction

The risk engine estimates the likelihood of a patient becoming lost to follow-up.

Potential features include:

* Distance from facility
* Previous missed visits
* Previous medicine collection
* Current prescription
* Current test requirements
* Previous outreach
* Facility characteristics
* Medicine availability
* Historical care completion
* Relevant demographic and geographic characteristics

The final production model should use the official LFU target defined by the competition specification.

---

# 8. Dropout Stage Detection

LFU is represented as a failure in the care journey rather than only a binary outcome.

The system identifies the first unresolved or likely failure stage.

Examples include:

### Medicine

```text
Prescription
    ↓
Medicine not collected
```

### Laboratory

```text
Test advised
    ↓
Test not completed
```

### Review

```text
Follow-up required
    ↓
Review not attended
```

This allows the intervention system to select an appropriate response.

---

# 9. Barrier Detection

After identifying the likely failure stage, the system determines the primary contributing barrier.

Potential barriers include:

* Geographic distance
* Medicine availability
* Laboratory access
* Previous missed care
* Contact/connectivity difficulties
* Facility-related constraints
* Other observed operational factors

Example:

```text
LFU Risk: HIGH

Failure Stage:
Medicine Collection

Primary Barrier:
Medicine Availability

Supporting Factors:
• Long facility distance
• Previous missed collection
• Facility stock issue
```

---

# 10. Kestrel-7 Intervention Framework

Kestrel-7 maps predicted risk and barriers to progressively stronger operational interventions.

| Level | Intervention                    |
| ----- | ------------------------------- |
| K1    | Monitor                         |
| K2    | Reminder                        |
| K3    | ASHA Contact                    |
| K4    | Community/Home Follow-Up        |
| K5    | Operational Resolution          |
| K6    | CHO Intervention                |
| K7    | Appropriate Clinical Escalation |

The system selects an appropriate intervention based on the patient's risk, failure stage, barrier, and previous intervention history.

---

# 11. Action Queue

The intervention engine produces a structured action queue.

Example:

| Priority | Episode | Problem              | Recommended Action            | Assigned Cadre |
| -------- | ------- | -------------------- | ----------------------------- | -------------- |
| High     | EP1023  | Medicine unavailable | Resolve medicine availability | CHO            |
| High     | EP2041  | Review missed        | Contact patient               | ASHA           |
| Medium   | EP3092  | Test incomplete      | Test reminder                 | ASHA           |
| Low      | EP4112  | Review approaching   | Send reminder                 | System         |

The action queue is the primary operational output of the platform.

---

# 12. Patient View

The patient view combines the most relevant information into a single care-continuity record.

```text
Patient ID
    │
    ├── LFU Risk
    ├── CCI-7
    ├── Current Care Stage
    ├── Likely Failure Stage
    ├── Primary Barrier
    ├── Risk Factors
    ├── Recommended Action
    ├── Assigned Cadre
    └── Intervention Status
```

---

# 13. Intervention Feedback

The system records the outcome of each intervention.

```text
Risk Identified
      ↓
Action Assigned
      ↓
Action Performed
      ↓
Patient Contacted
      ↓
Care Step Completed
      ↓
Care Journey Closed
```

This creates a feedback loop between prediction and intervention.

---

# 14. Data Leakage Prevention

The model must only use information available at prediction time.

### Allowed

* Historical patient information
* Historical visits
* Historical medicine collection
* Historical laboratory completion
* Current prescription
* Current consultation information
* Existing facility information
* Available stock information
* Historical outreach

### Excluded from prediction

* Future medicine collection
* Future laboratory results
* Future follow-up visits
* Future outreach
* Future outcome labels

This prevents the model from using future information to predict the past.

---

# 15. Model Architecture

The platform uses machine learning for structured/tabular prediction.

Suitable model families include:

* CatBoost
* XGBoost
* LightGBM

The model produces a risk prediction which is then passed to the barrier and intervention layers.

```text
Patient Features
      ↓
ML Risk Model
      ↓
Risk Tier
      ↓
Explainability
      ↓
Barrier Detection
      ↓
Intervention Rules
```

Machine learning is used for prediction and pattern detection; operational actions remain governed by explicit rules and healthcare workflows.

---

# 16. Explainability

The platform exposes the major factors contributing to a prediction.

Example:

```text
LFU Risk: HIGH

Major contributing factors:

1. Previous missed follow-up
2. Long distance from facility
3. Medicine availability issue
4. Previous medicine non-collection
```

Explainability mechanisms such as feature importance or SHAP can be used during model analysis.

---

# 17. Evaluation

The system can be evaluated at two levels.

## Model Evaluation

* Precision
* Recall
* Macro F1
* Weighted F1
* Confusion matrix
* ROC-AUC
* PR-AUC
* Calibration
* Per-tier performance

## Operational Evaluation

* LFU rate
* Medicine collection rate
* Test completion rate
* Follow-up attendance
* Time to care closure
* Intervention success rate
* Worker workload
* Cost per successfully closed episode

A workload-aware measure such as **Recall@Worker Capacity** can evaluate how many high-risk cases are identified within a realistic daily intervention capacity.

---

# 18. Rural Deployment

Kestrel CareLoop is designed for environments with intermittent connectivity and limited device capabilities.

The worker application can support:

* Offline task access
* Offline action recording
* Local data synchronization
* Low-bandwidth communication
* Lightweight interfaces
* Mobile-first layouts
* Local-language content

Basic synchronization workflow:

```text
Server
   ↓
Task Download
   ↓
Offline Worker Operation
   ↓
Local Action Recording
   ↓
Connectivity Restored
   ↓
Synchronization
```

---

# 19. Security and Privacy

The system should follow privacy-by-design principles.

Key measures include:

* Role-based access
* Minimum necessary data
* Pseudonymous identifiers
* Encryption
* Secure storage
* Audit logging
* Controlled access
* Appropriate consent and governance

The implementation should follow the applicable healthcare data governance requirements specified by the deployment environment and competition documentation.

---

# 20. Repository Architecture

A recommended repository structure is:

```text
kestrel-careloop/
│
├── README.md
├── DOCUMENTATION.md
├── LICENSE
│
├── data/
│   ├── raw/
│   ├── processed/
│   └── README.md
│
├── backend/
│   ├── api/
│   ├── services/
│   ├── models/
│   ├── database/
│   └── main.py
│
├── ml/
│   ├── preprocessing/
│   ├── entity_resolution/
│   ├── feature_engineering/
│   ├── training/
│   ├── evaluation/
│   └── inference/
│
├── intervention/
│   ├── barrier_engine/
│   ├── kestrel7/
│   └── action_queue/
│
├── frontend/
│   ├── components/
│   ├── pages/
│   ├── services/
│   └── assets/
│
├── notebooks/
│   ├── exploration/
│   └── evaluation/
│
├── tests/
│   ├── backend/
│   ├── ml/
│   └── intervention/
│
└── docs/
    ├── architecture/
    ├── data_dictionary/
    └── api/
```

---

# 21. End-to-End System Example

```text
1. Patient completes teleconsultation
                ↓
2. Records are linked to canonical patient
                ↓
3. Care episode is reconstructed
                ↓
4. CCI-7 evaluates continuity
                ↓
5. Features are generated
                ↓
6. LFU risk is predicted
                ↓
7. Likely dropout stage is identified
                ↓
8. Primary barrier is identified
                ↓
9. Kestrel-7 selects intervention
                ↓
10. Action is assigned to ASHA/CHO
                ↓
11. Action is performed
                ↓
12. Outcome is recorded
                ↓
13. Patient continues care
                ↓
14. Care journey is closed
```

---

# 22. Design Principle

Kestrel CareLoop is designed around one central principle:

> **A prediction is useful only when it can be converted into an appropriate action and its outcome can be measured.**

Therefore, the system connects:

**Data Integration**

→ **Patient Identity**

→ **Care Journey**

→ **Risk Prediction**

→ **Barrier Identification**

→ **Intervention**

→ **Care Closure**

---

## Kestrel CareLoop

**From fragmented healthcare records to closed-loop continuity of care.**
