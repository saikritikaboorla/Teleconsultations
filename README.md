# Mitra — Closed-Loop Rural Teleconsultation Follow-Up System

Mitra is a data-driven care-continuity platform designed to reduce Loss to Follow-Up (LTFU) after rural teleconsultations.

The system connects fragmented healthcare records, reconstructs each patient's care journey, identifies patients at risk of dropping out, determines the likely point of failure, and converts the prediction into an actionable task for ASHAs and Community Health Officers (CHOs).

The system is designed around a simple principle:

> **A completed teleconsultation should not be treated as completed care.**

The platform therefore follows the patient beyond the consultation:

```text
Teleconsultation
      ↓
Prescription
      ↓
Medicine Collection
      ↓
Diagnostic Testing
      ↓
Follow-up Review
      ↓
Care Closure
```

---

## 1. Problem

Rural chronic-care patients may successfully complete a teleconsultation but fail to complete one or more downstream steps.

The supplied dataset represents multiple operational systems containing separate records for:

* Patients
* Teleconsultations
* NCD screening
* Prescriptions
* Medicine dispensing
* Laboratory tests
* Follow-up visits
* Previous visits
* Outreach actions
* Medicine stock
* Healthcare facilities
* Geography
* Episode outcomes

These records are not provided as one unified patient timeline.

Mitra addresses this fragmentation by creating a unified patient and episode-level representation.

---

# 2. Solution Overview

Mitra consists of five major processing layers:

```text
┌─────────────────────────────────────────────┐
│              RAW DATA SOURCES               │
│ Teleconsultation | NCD | Pharmacy | Labs   │
│ Follow-up | Outreach | Geography | Facility│
└──────────────────────┬──────────────────────┘
                       ↓
┌─────────────────────────────────────────────┐
│          DATA LINKAGE ENGINE                 │
│ Canonical Patient Resolution                 │
│ Deterministic + Fuzzy Matching               │
└──────────────────────┬──────────────────────┘
                       ↓
┌─────────────────────────────────────────────┐
│          EPISODE RECONSTRUCTION             │
│ Consultation → Medicine → Lab → Review      │
└──────────────────────┬──────────────────────┘
                       ↓
┌─────────────────────────────────────────────┐
│          ANALYTICS ENGINE                   │
│ Feature Engineering → XGBoost → LFU Risk    │
└──────────────────────┬──────────────────────┘
                       ↓
┌─────────────────────────────────────────────┐
│          INTERVENTION ENGINE                │
│ Dropout Stage → Barrier → Action → Cadre    │
└──────────────────────┬──────────────────────┘
                       ↓
┌─────────────────────────────────────────────┐
│          MITRA ACTION LAYER                 │
│ ASHA Priority Queue | CHO Tasks | IVR       │
└─────────────────────────────────────────────┘
```

---

# 3. Core Workflow

Mitra processes every teleconsultation episode through the following workflow:

### Step 1 — Resolve Patient Identity

Records from different systems are mapped to the canonical patient record.

```text
source_patient_id
        ↓
entity resolution
        ↓
canonical patient_id
```

The supplied synthetic dataset uses `patient_360_reference.csv` as the canonical patient reference.

---

### Step 2 — Build the Care Episode

A teleconsultation becomes the central episode.

Related records are attached to that episode:

```text
Episode
 ├── NCD context
 ├── Prescription
 │    └── Medicine dispensing
 ├── Diagnostic tests
 ├── Follow-up review
 ├── Previous patient history
 ├── Outreach
 ├── Facility
 ├── Geography
 └── Medicine availability
```

---

### Step 3 — Construct Features

The system derives features representing:

* Patient characteristics
* Clinical context
* Geographic access
* Connectivity
* Medicine requirements
* Diagnostic requirements
* Previous care behaviour
* Facility conditions
* Medicine availability
* Continuity of care

---

### Step 4 — Predict LTFU Risk

An XGBoost classifier processes the episode-level features and generates:

```text
P(LTFU) ∈ [0,1]
```

The probability is then converted into an operational risk category.

The model is trained only using information available at the defined prediction point.

---

### Step 5 — Identify the Likely Dropout Stage

For patients identified as at risk, the system determines the likely point of failure:

```text
Medicine Collection
        OR
Diagnostic Test
        OR
Follow-up Review
```

---

### Step 6 — Identify the Operational Barrier

The system combines the predicted stage with available evidence.

Examples:

```text
Medicine not collected
        +
Stockout detected
        ↓
Medicine availability barrier
```

or:

```text
Follow-up overdue
        +
Previous missed visits
        ↓
Follow-up adherence barrier
```

---

### Step 7 — Generate an Intervention

The intervention engine converts the risk and barrier into an action.

Examples:

```text
High Risk + Medicine Stockout
        → CHO operational resolution

High Risk + Missed Review
        → ASHA follow-up

Medium Risk + Upcoming Review
        → Telugu IVR reminder

Incomplete Diagnostic Test
        → Test reminder / coordination
```

---

### Step 8 — Create the Action Queue

The final operational output contains:

```text
episode_id
predicted_patient_id
priority
reason
recommended_action
assigned_cadre
```

This forms the basis of the ASHA/CHO worklist.

---

# 4. Dataset Pipeline

The supplied datasets are not loaded independently into the application.

They are transformed into an integrated episode dataset.

```text
patient_360_reference
             │
             ↓
       Entity Resolution
             │
             ├──────── NCD Screening
             ├──────── Teleconsultations
             ├──────── Prescriptions
             ├──────── Medicine Dispensing
             ├──────── Laboratory Tests
             ├──────── Follow-up Visits
             ├──────── Visit History
             ├──────── Outreach Actions
             ├──────── Medicine Stock
             ├──────── Facility Reference
             └──────── Geography Reference
                         │
                         ↓
                Episode Feature Table
                         │
                         ↓
                     XGBoost
                         │
                         ↓
                    Risk Output
                         │
                         ↓
                Intervention Engine
```

---

# 5. Dataset Roles

| Dataset                     | System role                                |
| --------------------------- | ------------------------------------------ |
| `patient_360_reference.csv` | Canonical patient reference                |
| `teleconsultations.csv`     | Episode anchor                             |
| `ncd_screening.csv`         | Clinical/NCD context                       |
| `prescriptions.csv`         | Treatment prescribed                       |
| `medicine_dispensing.csv`   | Medicine collection outcome                |
| `lab_tests.csv`             | Diagnostic completion                      |
| `followup_visits.csv`       | Review completion                          |
| `visit_history.csv`         | Historical behaviour                       |
| `outreach_actions.csv`      | Previous intervention/contact              |
| `medicine_stock_status.csv` | Medicine availability context              |
| `facility_reference.csv`    | Facility/workforce context                 |
| `geography_reference.csv`   | Geographic/access context                  |
| `episode_outcomes.csv`      | Development labels and evaluation episodes |

---

# 6. Prediction Point

The model predicts LTFU **after the teleconsultation and before future care events occur**.

Therefore, the model may use:

```text
✓ demographic information
✓ geographic information
✓ historical behaviour
✓ NCD screening available before consultation
✓ consultation information
✓ prescription information available at prediction time
✓ facility context
✓ medicine availability known at prediction time
```

The model must not use:

```text
✗ future medicine collection
✗ future laboratory results
✗ future follow-up attendance
✗ future outreach actions
✗ future dropout labels
```

This prevents temporal leakage.

---

# 7. Development and Evaluation Data

The supplied outcome dataset separates episodes into development and evaluation cohorts.

Development records contain known outcomes and are used for:

```text
Exploratory analysis
       ↓
Feature engineering
       ↓
Model training
       ↓
Validation
```

Evaluation records contain hidden outcomes and are used only after the model pipeline is finalized.

The intended flow is:

```text
Development
     ↓
Train + Validate
     ↓
Freeze Pipeline
     ↓
Evaluation
     ↓
Generate Predictions
```

---

# 8. Outputs

Mitra produces three principal data outputs.

### Linkage Output

```text
submission_template_linkage.csv
```

Contains the mapping between source records and canonical patients.

---

### Episode Prediction Output

```text
submission_template_episode_predictions.csv
```

Contains model predictions for each episode.

---

### Action Queue

```text
submission_template_action_queue.csv
```

Contains operational tasks generated from the predictions.

---

# 9. Repository Structure

```text
mitra/
│
├── README.md
├── DOCUMENTATION.md
├── requirements.txt
├── .env.example
│
├── data/
│   ├── raw/
│   ├── processed/
│   └── README.md
│
├── ml/
│   ├── linkage/
│   ├── preprocessing/
│   ├── feature_engineering/
│   ├── training/
│   ├── inference/
│   └── evaluation/
│
├── backend/
│   ├── api/
│   ├── services/
│   ├── models/
│   └── database/
│
├── intervention/
│   ├── barrier_engine/
│   ├── priority_engine/
│   └── action_queue/
│
├── frontend/
│   ├── pages/
│   ├── components/
│   └── services/
│
├── notebooks/
│   ├── eda/
│   ├── linkage/
│   └── model_analysis/
│
├── tests/
│   ├── linkage/
│   ├── ml/
│   └── integration/
│
└── models/
    └── xgboost/
```

---

# 10. Design Principles

Mitra follows these principles:

### Data continuity

Every downstream care event should be associated with the correct patient and episode whenever the available data supports the association.

### Prediction-time integrity

No future information is used when generating an LTFU prediction.

### Explainability

A risk score should be accompanied by the stage and evidence that produced the operational recommendation.

### Actionability

The system does not stop at prediction. Each actionable high-risk episode can become a task.

### Capacity awareness

The system prioritizes cases according to operational capacity rather than generating an unlimited workload.

### Offline-first operation

The ASHA workflow is designed to support intermittent connectivity.

### Privacy

Patient identifiers and sensitive healthcare data are separated from model-processing outputs wherever possible.

---

# 11. Technology Stack

### Data and ML

* Python
* Pandas
* NumPy
* Scikit-learn
* XGBoost

XGBoost provides a Python scikit-learn interface suitable for the proposed classifier and supports probability prediction for binary classification.

### Backend

* Python
* FastAPI
* REST APIs

### Storage

* PostgreSQL for centralized data
* SQLite/SQLCipher for offline mobile operation

### Frontend

* Lightweight Android application
* Offline local storage
* Synchronization when connectivity becomes available

---

# 12. End-to-End Result

The final system transforms fragmented healthcare records into a closed-loop workflow:

```text
Fragmented Records
        ↓
Patient Linkage
        ↓
Episode Reconstruction
        ↓
Continuity Measurement
        ↓
Risk Prediction
        ↓
Dropout Stage
        ↓
Barrier Detection
        ↓
Intervention
        ↓
Healthcare Worker Action
        ↓
Care Completion
```

Mitra therefore shifts the system from:

> **Counting completed teleconsultations**

to:

> **Tracking whether the patient's care journey was actually completed.**
