# Kestrel CareLoop — Technical Documentation

## Data-Driven Follow-Up Assurance for Rural Teleconsultations

---

## 1. Introduction

Kestrel CareLoop is an AI-assisted care-continuity platform designed to reduce **Loss to Follow-Up (LFU)** in rural teleconsultation workflows.

The system integrates fragmented healthcare records, reconstructs patient-level care journeys, predicts the likelihood of loss to follow-up, identifies the stage and contributing factors associated with potential care disruption, and generates an actionable intervention workflow for healthcare workers.

The platform follows a closed-loop approach:

> **Connect → Predict → Explain → Act → Close**

The objective is not limited to predicting which patients are at risk. The system connects prediction with an operational response and tracks whether the required care step is eventually completed.

---

# 2. Problem Definition

Rural teleconsultation involves several interconnected stages.

A typical care journey may contain:

```text
Patient
   ↓
Teleconsultation
   ↓
Prescription
   ↓
Medicine Collection
   ↓
Laboratory Testing
   ↓
Follow-up Review
   ↓
Care Completion
```

Information generated during these stages may be stored in separate systems.

For example:

* Teleconsultation systems store consultation information.
* Prescription systems store prescribed medicines.
* Pharmacy systems store dispensing information.
* Laboratory systems store test information.
* Visit systems store follow-up attendance.
* Outreach systems store community-level interactions.
* Geographic datasets describe patient and facility locations.
* Facility and stock datasets describe healthcare resource availability.

These records may not share a common patient identifier and can contain variations in names, phone numbers, locations, dates, and other attributes.

This creates four major challenges:

### 2.1 Fragmented patient identity

The same patient may appear differently across different datasets.

### 2.2 Incomplete care visibility

A healthcare worker may not have a single view of the patient's complete care journey.

### 2.3 Unknown dropout causes

Identifying that a patient was lost to follow-up does not necessarily explain why.

### 2.4 Limited intervention capacity

Healthcare workers have limited time and cannot manually investigate every patient.

Kestrel CareLoop addresses these challenges by combining data integration, machine learning, explainability, and an intervention engine.

---

# 3. System Objectives

The system is designed to achieve the following objectives:

1. Integrate fragmented healthcare records.
2. Resolve patient identities across multiple data sources.
3. Reconstruct healthcare episodes.
4. Measure continuity of the reconstructed records.
5. Predict LFU risk.
6. Identify the likely stage at which care may fail.
7. Identify the major contributing barrier.
8. Prioritize patients requiring intervention.
9. Recommend an appropriate operational action.
10. Assign actions to the appropriate healthcare cadre.
11. Track intervention outcomes.
12. Measure whether the care journey was successfully closed.

---

# 4. High-Level Architecture

```text
                         ┌──────────────────────┐
                         │   Healthcare Data    │
                         └──────────┬───────────┘
                                    │
                                    ▼
                         ┌──────────────────────┐
                         │ Data Normalization   │
                         └──────────┬───────────┘
                                    │
                                    ▼
                         ┌──────────────────────┐
                         │ Entity Resolution    │
                         │ & Patient Linking    │
                         └──────────┬───────────┘
                                    │
                                    ▼
                         ┌──────────────────────┐
                         │ Episode Reconstruction│
                         └──────────┬───────────┘
                                    │
                                    ▼
                         ┌──────────────────────┐
                         │       CCI-7          │
                         │ Care Continuity      │
                         └──────────┬───────────┘
                                    │
                                    ▼
                         ┌──────────────────────┐
                         │ Feature Engineering  │
                         └──────────┬───────────┘
                                    │
                                    ▼
                         ┌──────────────────────┐
                         │   LFU Risk Model    │
                         └──────────┬───────────┘
                                    │
                                    ▼
                         ┌──────────────────────┐
                         │ Failure Stage &      │
                         │ Barrier Detection    │
                         └──────────┬───────────┘
                                    │
                                    ▼
                         ┌──────────────────────┐
                         │ Kestrel-7            │
                         │ Intervention Engine  │
                         └──────────┬───────────┘
                                    │
                                    ▼
                         ┌──────────────────────┐
                         │    Action Queue      │
                         └──────────┬───────────┘
                                    │
                         ┌──────────┴───────────┐
                         ▼                      ▼
                    ASHA Workflow          CHO Workflow
                         │                      │
                         └──────────┬───────────┘
                                    ▼
                         ┌──────────────────────┐
                         │    Care Closure      │
                         └──────────────────────┘
```

---

# 5. Data Layer

The data layer consists of multiple healthcare information sources.

## 5.1 Teleconsultation Data

Represents the initial remote healthcare interaction.

It provides information such as:

* Patient reference
* Consultation information
* Episode information
* Date/time
* Facility or provider information
* Relevant consultation attributes

---

## 5.2 NCD Data

Provides information related to non-communicable disease screening and existing health context.

It can provide historical or contextual information that may influence follow-up requirements.

---

## 5.3 Prescription Data

Represents medicines prescribed during a healthcare episode.

Important relationships include:

```text
Teleconsultation
       ↓
Prescription
       ↓
Medicine
```

---

## 5.4 Pharmacy / Dispensing Data

Represents whether prescribed medicines were dispensed or collected.

This is important for identifying medicine-related care gaps.

Example:

```text
Prescription exists
        ↓
Medicine not collected
        ↓
Potential care disruption
```

---

## 5.5 Laboratory Data

Represents tests associated with a patient's care episode.

The system can distinguish between:

* Test requirement
* Test order
* Test completion
* Result availability

---

## 5.6 Visit Data

Represents follow-up or subsequent healthcare visits.

This allows the system to determine whether a required review was completed.

---

## 5.7 Outreach Data

Represents community-level contact or outreach activities.

This provides information about previous attempts to engage with the patient.

---

## 5.8 Geographic Data

Geographic information can be used to derive factors such as:

* Distance
* Accessibility
* Facility proximity
* Geographic grouping

Distance can be an important contextual feature in rural follow-up.

---

## 5.9 Facility and Stock Data

Facility data describes the healthcare environment associated with an episode.

Stock data provides information about resource availability, particularly medicines.

This allows the system to distinguish between:

> Patient-level barriers

and

> System-level barriers.

For example, a patient may fail to collect medicine because the required medicine was unavailable.

---

# 6. Data Normalization

Before records can be linked, raw data is normalized.

Typical normalization operations include:

### Names

* Convert to consistent case
* Remove unnecessary whitespace
* Normalize common formatting differences
* Handle minor spelling variations

### Phone numbers

* Remove formatting characters
* Normalize country/region prefixes where appropriate
* Compare standardized values

### Locations

* Normalize village/block/district strings
* Handle common spelling differences

### Dates

* Convert to a consistent date/time format
* Validate invalid or missing values

### Categorical values

Normalize representations such as:

```text
Male
male
M
```

into a consistent representation where appropriate.

---

# 7. Entity Resolution

## 7.1 Purpose

Entity resolution determines whether records from different datasets represent the same real-world patient.

For example:

```text
Teleconsultation:

Lakshmi R
9876543210
Rampur
52

Pharmacy:

Laxmi R
9876543210
Rampur
52
```

The system should identify these records as a probable match.

---

## 7.2 Matching Strategy

The entity-resolution layer can combine:

### Exact matching

Used for highly reliable normalized attributes.

### Fuzzy matching

Used for attributes such as:

* Names
* Villages
* Locations

### Multi-field matching

Combines several attributes rather than depending on a single field.

A conceptual confidence calculation is:

```text
Name similarity
        +
Phone similarity
        +
Location similarity
        +
Demographic similarity
        +
Temporal consistency
        ↓
Match Confidence
```

---

## 7.3 Canonical Patient Identity

After matching, records are mapped to a canonical patient identifier.

```text
Source A ─┐
Source B ─┤
Source C ─┼──→ Canonical Patient ID
Source D ─┤
Source E ─┘
```

This allows the rest of the system to work with a unified patient representation.

---

# 8. Episode Reconstruction

A patient may have multiple healthcare interactions.

The system groups related records into care episodes.

A simplified episode may contain:

```text
Episode
│
├── Teleconsultation
├── NCD Context
├── Prescription
├── Pharmacy
├── Laboratory
├── Follow-up
└── Outreach
```

Each component is associated with the episode using available identifiers, dates, relationships, and matching logic.

---

# 9. Care Journey Model

Each episode is represented as a sequence of care stages.

Example:

```text
Teleconsultation
       ↓
Prescription
       ↓
Medicine
       ↓
Test
       ↓
Review
       ↓
Completed
```

Each stage can have a state such as:

* Completed
* Pending
* Not required
* Failed/unresolved
* Unknown

Example:

```text
Teleconsultation    COMPLETED
Prescription        COMPLETED
Medicine            PENDING
Laboratory          UNKNOWN
Review              UNKNOWN
```

This provides a structured representation of the patient's current care position.

---

# 10. CCI-7

## Continuity-of-Care Index

CCI-7 measures the completeness of the connected care journey.

It evaluates seven relationships:

| # | Relationship                    |
| - | ------------------------------- |
| 1 | Patient → Teleconsultation      |
| 2 | Teleconsultation → NCD          |
| 3 | Teleconsultation → Prescription |
| 4 | Prescription → Pharmacy         |
| 5 | Test Requirement → Laboratory   |
| 6 | Episode → Follow-up             |
| 7 | Episode → Outreach              |

Each relationship can be represented as:

* Linked
* Partially linked
* Unresolved

Example:

```text
Patient → Teleconsultation       ✓
Teleconsultation → NCD           ✓
Teleconsultation → Prescription  ✓
Prescription → Pharmacy          ✓
Test → Laboratory                ✓
Episode → Follow-up              ?
Episode → Outreach               ✓
```

Result:

```text
CCI-7 = 6 / 7
```

CCI-7 is a **data continuity indicator** and should not be interpreted as a medical or clinical score.

---

# 11. Feature Engineering

The unified episode representation is converted into model features.

Potential feature groups include:

## Patient features

* Age
* Gender
* Relevant demographic information

## Geographic features

* Distance to facility
* Geographic accessibility indicators

## Historical behavior

* Previous missed visits
* Previous medicine collection
* Previous test completion
* Previous outreach

## Current episode

* Medicine prescribed
* Test required
* Follow-up required
* Relevant consultation information

## Facility factors

* Facility characteristics
* Medicine availability
* Operational constraints

The feature pipeline must ensure that only information available at prediction time is used.

---

# 12. Prediction-Time Boundary

The system defines a prediction timestamp.

For example:

```text
Teleconsultation completed
        ↓
Prediction generated
```

Only information known at or before this point may be used.

### Permitted information

* Historical patient information
* Historical visits
* Historical pharmacy activity
* Historical laboratory activity
* Current prescription
* Current consultation
* Existing facility information
* Available stock information
* Historical outreach

### Excluded information

* Future medicine collection
* Future test results
* Future follow-up attendance
* Future outreach
* Future LFU outcome

This prevents future information from leaking into the model.

---

# 13. LFU Risk Engine

The LFU risk engine estimates the probability or risk tier associated with loss to follow-up.

The supplied problem specification identifies `lfu_risk_tier_k7` as the intended target for the final prediction task.

The implementation should use the official target provided by the competition.

The prediction pipeline is:

```text
Unified Episode
       ↓
Feature Vector
       ↓
LFU Model
       ↓
Risk Tier
```

---

# 14. Model Layer

Because the data is primarily structured/tabular, suitable model families include:

* CatBoost
* XGBoost
* LightGBM

The final model can be selected based on validation performance, calibration, interpretability, and operational requirements.

A baseline model should also be maintained for comparison.

---

# 15. Risk Explainability

The system should provide interpretable factors associated with each prediction.

Example:

```text
LFU Risk: HIGH

Important contributing factors:

• Previous missed follow-up
• Long distance to facility
• Medicine availability issue
• Previous medicine non-collection
```

Model explainability can be implemented using:

* Feature importance
* SHAP
* Rule-based reason codes

The purpose is to make predictions understandable to healthcare workers and evaluators.

---

# 16. Dropout Stage Detection

The system identifies where the care journey is likely to fail.

The main observed care-gap categories include:

### Medicine Collection

```text
Prescription
     ↓
Medicine not collected
```

### Laboratory Completion

```text
Test advised
     ↓
Test not completed
```

### Follow-Up Review

```text
Review required
     ↓
Review not attended
```

The stage information is used by the intervention engine.

---

# 17. Barrier Detection

The barrier engine determines the most relevant factor associated with the predicted failure.

Potential categories include:

### Geographic barrier

Long distance or accessibility limitations.

### Medicine barrier

Medicine unavailable or difficult to obtain.

### Laboratory barrier

Difficulty completing a required investigation.

### Follow-up barrier

Missed or delayed review.

### Contact barrier

Difficulty reaching or engaging the patient.

### Operational barrier

Facility-side or workflow-side constraints.

The barrier engine produces structured reason codes.

Example:

```text
Risk:
HIGH

Failure Stage:
Medicine Collection

Primary Barrier:
Medicine Availability

Secondary Factors:
Distance
Previous non-collection
```

---

# 18. Intervention Engine

The intervention engine converts:

```text
Risk
+
Failure Stage
+
Barrier
+
Previous Actions
```

into:

```text
Recommended Action
+
Assigned Cadre
+
Priority
```

---

# 19. Kestrel-7 Intervention Framework

Kestrel-7 defines seven operational intervention levels.

## K1 — Monitor

Used for cases that do not currently require direct intervention.

---

## K2 — Reminder

Automated or low-intensity communication.

Possible channels include:

* SMS
* IVR
* Other supported communication mechanisms

---

## K3 — ASHA Contact

The case is assigned to an ASHA for direct contact.

---

## K4 — Community/Home Follow-Up

Used when direct community-level engagement is appropriate.

---

## K5 — Operational Resolution

Used when the barrier originates from an operational constraint.

Example:

```text
Medicine required
       ↓
Medicine unavailable
       ↓
Operational resolution task
```

---

## K6 — CHO Intervention

Persistent or operationally complex cases can be escalated to a Community Health Officer.

---

## K7 — Appropriate Clinical Escalation

Cases requiring clinical escalation are routed through the applicable clinical/referral workflow.

Kestrel-7 does not replace clinical judgement.

---

# 20. Action Queue Generation

The intervention engine generates a structured action record.

Conceptually:

```text
{
    episode_id,
    patient_id,
    priority,
    risk_tier,
    failure_stage,
    barrier,
    reason,
    recommended_action,
    assigned_cadre,
    status
}
```

Example:

```text
Episode: EP1023
Priority: HIGH
Risk: Tier 4
Failure Stage: Medicine
Barrier: Stock Availability
Action: Resolve Medicine Availability
Cadre: CHO
Status: Pending
```

---

# 21. Healthcare Worker Workflow

The operational workflow is:

```text
New Case
   ↓
Risk Assessment
   ↓
Failure Stage
   ↓
Barrier
   ↓
Recommended Action
   ↓
Assigned Worker
   ↓
Action Performed
   ↓
Outcome Recorded
   ↓
Care Status Updated
```

---

# 22. Intervention Feedback Loop

The system records what happens after an intervention.

Example:

```text
High Risk
   ↓
ASHA Contact
   ↓
Patient Reached
   ↓
Medicine Collected
   ↓
Care Stage Completed
```

The outcome is stored as part of the episode history.

This allows the platform to measure intervention effectiveness and create feedback for future analysis.

---

# 23. Care Closure

A care journey is considered closed when the required care stages for that episode have been completed according to the available workflow information.

Example:

```text
Teleconsultation       ✓
Prescription           ✓
Medicine               ✓
Laboratory             ✓
Follow-up              ✓
                       ↓
                CARE CLOSED
```

The exact closure criteria depend on the requirements of the episode.

---

# 24. Action Prioritization

Patients can be prioritized based on:

* LFU risk
* Failure stage
* Barrier severity
* Time sensitivity
* Previous unsuccessful interventions
* Operational feasibility

Example:

| Priority | Failure  | Barrier      | Action                 |
| -------- | -------- | ------------ | ---------------------- |
| High     | Medicine | Stock issue  | Operational resolution |
| High     | Review   | Missed visit | ASHA contact           |
| Medium   | Test     | Pending      | Test reminder          |
| Low      | Review   | Due soon     | Automated reminder     |

---

# 25. Worker Dashboard

The primary dashboard is centered around actionable cases rather than raw data.

A dashboard can contain:

### Summary

* Total active episodes
* High-risk episodes
* Pending interventions
* Completed interventions
* Open care gaps

### Action Queue

Prioritized patient cases.

### Patient Journey

Timeline of individual care stages.

### Intervention Status

* Pending
* In progress
* Completed
* Escalated
* Unresolved

---

# 26. Patient Care Timeline

Each patient can be represented through a timeline.

Example:

```text
Day 0
│
├── Teleconsultation ✓
│
├── Prescription ✓
│
Day 1
│
├── Medicine ✗
│
Day 5
│
├── Outreach initiated
│
Day 6
│
├── Medicine obtained ✓
│
Day 20
│
├── Follow-up ✓
│
└── Care Closed ✓
```

This provides a simple visual representation of the complete episode.

---

# 27. Offline-First Workflow

Rural deployment may involve intermittent connectivity.

The worker interface can therefore support local operation.

```text
Server
   ↓
Task Synchronization
   ↓
Local Device
   ↓
Worker Views Tasks
   ↓
Worker Records Action
   ↓
Local Storage
   ↓
Network Available
   ↓
Data Synchronization
```

This reduces dependence on continuous internet connectivity.

---

# 28. Localization

The system can use available patient language information to provide localized communication.

The interface can support:

* Local-language action descriptions
* Localized reminders
* Simplified worker-facing instructions

Localization should not alter the underlying structured clinical or operational information.

---

# 29. Security and Privacy

The system should implement appropriate security controls.

### Access control

Different user roles should have access only to the information required for their responsibilities.

### Data minimization

Only necessary patient information should be exposed to each workflow.

### Encryption

Sensitive information should be protected during storage and transmission.

### Auditability

Important actions should be logged, including:

* Record access
* Intervention assignment
* Intervention completion
* Escalation
* Data updates

### Pseudonymization

Analytics and machine-learning workflows should use pseudonymous identifiers where possible.

---

# 30. Model Evaluation

The prediction model should be evaluated using standard classification metrics.

### Classification

* Precision
* Recall
* F1-score
* Macro F1
* Weighted F1
* Confusion matrix

### Ranking

* ROC-AUC
* PR-AUC
* Recall@K

### Calibration

Predicted risk should be evaluated for calibration so that risk levels correspond meaningfully to observed outcomes.

### Per-tier analysis

Performance should be examined separately for each risk tier.

---

# 31. Operational Evaluation

Model performance alone does not measure the success of the system.

The platform should also measure:

### Care outcomes

* LFU rate
* Medicine collection
* Test completion
* Follow-up attendance
* Care closure

### Intervention outcomes

* Successful patient contact
* Successful intervention
* Escalation rate
* Resolution rate

### Efficiency

* Cases handled per worker
* Worker time per case
* Outreach attempts
* Cost per successfully closed episode

---

# 32. Capacity-Aware Evaluation

Healthcare workers have limited capacity.

Therefore, the system can evaluate:

## Recall@Worker Capacity

For example:

```text
Available ASHA capacity = 20 cases

Top 20 cases selected
        ↓
How many are genuinely high-risk?
```

This measures whether the system can prioritize useful cases within realistic operational limits.

---

# 33. Validation Strategy

A chronological split is preferred for deployment-oriented evaluation.

```text
Historical Data
      ↓
Training

Later Historical Data
      ↓
Validation

Evaluation Cohort
      ↓
Final Evaluation
```

This better reflects the actual deployment scenario in which historical data is used to predict future outcomes.

---

# 34. Technical Components

A possible implementation consists of the following layers.

```text
Frontend
   ↓
Backend API
   ↓
Application Services
   ├── Patient Service
   ├── Episode Service
   ├── Risk Service
   ├── Intervention Service
   └── Action Queue Service
   ↓
Data Layer
   ├── Patient Records
   ├── Episodes
   ├── Predictions
   ├── Interventions
   └── Outcomes
   ↓
ML Pipeline
   ├── Preprocessing
   ├── Entity Resolution
   ├── Feature Engineering
   ├── Training
   └── Inference
```

---

# 35. Recommended Repository Structure

```text
kestrel-careloop/
│
├── README.md
├── DOCUMENTATION.md
├── LICENSE
├── requirements.txt
├── .env.example
│
├── data/
│   ├── raw/
│   ├── processed/
│   └── README.md
│
├── backend/
│   ├── main.py
│   ├── api/
│   ├── services/
│   ├── models/
│   ├── schemas/
│   └── database/
│
├── ml/
│   ├── entity_resolution/
│   ├── preprocessing/
│   ├── feature_engineering/
│   ├── training/
│   ├── inference/
│   └── evaluation/
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
│   ├── exploratory_analysis/
│   └── model_evaluation/
│
├── tests/
│   ├── unit/
│   ├── integration/
│   └── ml/
│
└── docs/
    ├── architecture/
    ├── api/
    └── data_dictionary/
```

---

# 36. End-to-End Example

The following illustrates how a single episode moves through the system.

### Input

A patient completes a teleconsultation.

The patient receives:

* A medicine prescription
* A laboratory test requirement
* A follow-up requirement

---

### Record Integration

The system links:

```text
Patient
   ↓
Teleconsultation
   ↓
Prescription
   ↓
Pharmacy
   ↓
Laboratory
   ↓
Follow-up
```

---

### Continuity

CCI-7 determines that six of seven required relationships are successfully linked.

---

### Prediction

The LFU model identifies the episode as high risk.

---

### Failure Stage

The system identifies medicine collection as the likely failure stage.

---

### Barrier

Facility stock information indicates a medicine availability problem.

---

### Intervention

The intervention engine generates:

```text
Priority: High

Barrier:
Medicine availability

Action:
Operational medicine-resolution task

Cadre:
CHO
```

---

### Outcome

The action is completed and the medicine is subsequently collected.

The episode status is updated:

```text
Medicine Collection: Completed
Care Status: Continuing
```

After all required stages are completed:

```text
Care Status: Closed
```

---

# 37. Core Data Flow

The complete data flow can be summarized as:

```text
                    RAW RECORDS
                         │
                         ▼
                 DATA NORMALIZATION
                         │
                         ▼
                 ENTITY RESOLUTION
                         │
                         ▼
                  PATIENT IDENTITY
                         │
                         ▼
                EPISODE RECONSTRUCTION
                         │
                         ▼
                    CARE GRAPH
                         │
                         ▼
                       CCI-7
                         │
                         ▼
                 FEATURE ENGINEERING
                         │
                         ▼
                  LFU RISK MODEL
                         │
                         ▼
              FAILURE STAGE DETECTION
                         │
                         ▼
                 BARRIER DETECTION
                         │
                         ▼
                KESTREL-7 ENGINE
                         │
                         ▼
                   ACTION QUEUE
                         │
                         ▼
                 ASHA / CHO ACTION
                         │
                         ▼
                 INTERVENTION RESULT
                         │
                         ▼
                   CARE CLOSURE
```

---

# 38. Design Principles

Kestrel CareLoop follows the following principles:

### Data continuity

Fragmented records should be connected before downstream analysis.

### Explainability

Predictions should have understandable contributing factors.

### Actionability

Every high-priority prediction should translate into an operational action.

### Human oversight

The system supports healthcare workers rather than replacing clinical judgement.

### Resource awareness

Interventions should consider limited healthcare-worker capacity.

### Rural suitability

The system should function under low-connectivity and low-resource conditions.

### Feedback

Interventions and outcomes should be recorded to measure actual care closure.

### Privacy

Patient information should be protected throughout the data lifecycle.

---

# 39. System Summary

Kestrel CareLoop transforms fragmented healthcare data into a closed-loop follow-up system.

The complete workflow is:

```text
CONNECT
Connect fragmented records into a unified patient identity.

↓

RECONSTRUCT
Build the patient's episode-level care journey.

↓

MEASURE
Evaluate continuity using CCI-7.

↓

PREDICT
Estimate LFU risk.

↓

EXPLAIN
Identify the likely failure stage and contributing barrier.

↓

ACT
Use Kestrel-7 to generate and assign an intervention.

↓

TRACK
Record intervention progress and outcomes.

↓

CLOSE
Confirm completion of the required care journey.
```

---

## Final System Principle

> **Kestrel CareLoop does not stop at predicting who may be lost to follow-up. It connects fragmented records, identifies where and why care may break, converts that insight into an actionable intervention, and tracks the journey until care is closed.**
