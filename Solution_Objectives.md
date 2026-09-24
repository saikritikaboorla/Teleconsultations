### Core Problem Definition

The problem statement addresses a critical gap in rural primary healthcare delivery in Telangana's teleconsultation pipeline. At Ayushman Arogya Mandirs (AAMs), Community Health Officers (CHOs) connect chronic disease patients (hypertension and diabetes) with specialist doctors at district hospitals via eSanjeevani.

While the virtual consultation is marked as "completed" in the system, there is zero visibility into whether the patient actually received treatment, as a large portion of patients drop out immediately afterward. They fail to collect prescribed medicines, complete mandatory diagnostic tests, or return for their required review visits within weeks. The fundamental issue is **data fragmentation**: data across teleconsultation logs, NCD screening records, pharmacy dispensing registers, and ASHA diaries sits in isolated silos and is never linked together.

---

### Objectives Your Proposed Solution Must Satisfy

To solve this problem, your solution must address three main problem-statement questions while working under specific practical constraints:

#### 1. Data Linkage Architecture

* **Objective:** Establish a unified tracking system that links isolated data streams (eSanjeevani teleconsultation records, NCD screening data, medicine dispensing registers, visit registers, demographics, and geographic data) to establish a single, continuous record of each patient's care journey.
* **Deliverable:** Demonstrate how these disparate logs can be deterministically matched to track a patient's true follow-up status end-to-end, generating `submission_template_linkage.csv` and validating completeness using the **Continuity-of-Care Index (CCI-7)**.

#### 2. Dropout Identification & Risk Prediction Model

* **Objective:** Identify the exact stages where patients drop out of care and build a predictive, AI/data-driven model to calculate which patients are at risk of becoming "Lost to Follow-Up" (LTFU) before it happens.
* **Deliverable:** Use the provided synthetic dataset to perform exploratory analysis, identify key drop-out factors, and output risk scores/categories for patients (`submission_template_episode_predictions.csv`) with strict prediction-time boundaries ($T_{\text{pred}}$) and monotonicity guardrails.

#### 3. Data-Driven Intervention Framework

* **Objective:** Design an actionable tool or workflow (e.g., prioritized task list) that guides health workers (CHOs and ASHAs) to proactively bring high-risk patients back into care.
* **Deliverable:** Define how the system moves health workers from simply counting completed consultations to ensuring complete, closed-loop treatment, outputting the operational action queue (`submission_template_action_queue.csv`) using the **Kestrel-7 Escalation Ladder**.

#### 4. Measurement & Evaluation Framework

* **Objective:** Define explicit metrics to measure the solution's performance.
* **Deliverable:** Provide concrete evaluation methods for:
  * **Effectiveness:** Reduction in LTFU rates (target $>30\%$), improved medication pickup and test completion.
  * **Feasibility:** Adoption ease for health workers (ASHA Action Rate $>85\%$ within 48 hours), operational workflow fit.
  * **Cost:** Financial viability and resource efficiency of scaling the intervention (Cost Per Retained Patient $\approx\text{₹}42.50$/year via ₹0.15 automated Telugu IVR calls).

---

### Mandatory Operational Constraints & Considerations

Your proposed design must operate within real-world rural healthcare limitations:

* **Hardware & Connectivity:** Must function on low-end smartphones with intermittent or no internet connectivity (offline-first capability using embedded SQLite with SQLCipher AES-256 and lightweight delta sync).
* **Workforce Capacity:** Must respect health-worker time constraints without overburdening CHOs and ASHAs with complex data entry, strictly capping worklists to the **Top 5 High-Risk Patients per village**.
* **Compliance & Security:** Must be privacy-preserving and compliant with Ayushman Bharat Digital Mission (ABDM) standards, the **Telangana Rural Data Stewardship Charter (TRDSC-7)**, explicit patient consent management, and immutable SHA-256 audit logging.
* **Evidence Base:** All proposals and claims must be supported by empirical findings and data analysis derived directly from the provided synthetic dataset.

---

### Objective-to-Dataset & Decision Tree Implementation Matrix

The matrix below shows how each core objective and constraint maps directly to specific dataset tables, decision trees, and guardrails:

| Solution Objective / Constraint | Dataset Files Utilized | Decision Mapping / Tree Applied | Applied Guardrails & Constraints | Key Output / Deliverable |
| :--- | :--- | :--- | :--- | :--- |
| **1. Data Linkage Architecture** | `patient_360_reference.csv`, `teleconsultations.csv`, `ncd_screening.csv`, `prescriptions.csv`, `lab_tests.csv`, `followup_visits.csv`, `outreach_actions.csv` | **Entity Resolution Decision Tree:** ABHA-first $\to$ Deterministic Phone+Name $\to$ Double Metaphone(Name) + Sex + Age $\pm 2$ + Village Jaro-Winkler $\ge 0.85$ | Minimum confidence threshold ($0.80$); unlinked entities quarantined for review | `submission_template_linkage.csv`, Continuity-of-Care Index (CCI-7 $\ge 6/7$) |
| **2. Risk Prediction Model** | `episode_outcomes.csv`, `teleconsultations.csv`, `prescriptions.csv`, `ncd_screening.csv`, `visit_history.csv`, `geography_reference.csv` | **Risk Tier Decision Tree:** Calibrated $P(\text{LTFU})$ evaluated at $T_{\text{pred}}$ mapped to Tier 4 (Critical), Tier 3 (High), Tier 2 (Medium), Tier 1 (Low), Tier 0 (Monitor) | **Temporal Boundary:** Zero post-consult leakage; **Monotonicity:** $\frac{\partial P}{\partial \text{dist}} \ge 0$, $\frac{\partial P}{\partial \text{burden}} \ge 0$; **Calibration:** Isotonic regression | `submission_template_episode_predictions.csv` |
| **3. Dropout Stage & Barrier Attribution** | `teleconsultations.csv`, `medicine_dispensing.csv`, `lab_tests.csv`, `followup_visits.csv`, `medicine_stock_status.csv`, `facility_reference.csv` | **Dropout Stage & Barrier Tree:** Checks advised flags $\to$ isolates stage (`Medicine not collected`, `Test not completed`, `Review not attended`) $\to$ maps stockout, transport, or phone barriers | **Clinical Sanity Guardrail:** Cannot assign stage if doctor advised against it; cannot blame patient if facility experienced drug stockout | Stage & root-cause barrier reason codes |
| **4. Data-Driven Intervention Framework** | `outreach_actions.csv`, `facility_reference.csv`, `geography_reference.csv` | **Operational Cadre Dispatch Tree:** Maps Tier + Stage + Barrier $\to$ Kestrel-7 Ladder (K1 Monitor to K7 Specialist Referral) | **Capacity Throttling:** Strict cap of Top 5 daily visits per ASHA village; dynamic spillover to IVR | `submission_template_action_queue.csv` |
| **5. Offline-First Mobile & Rural Hardware** | `patient_360_reference.csv`, `geography_reference.csv` | **ASHA Top-5 Task Generation Tree:** Generates minimal offline task cards from action queue | Embedded SQLite with SQLCipher AES-256; lightweight delta-sync over MQTT/REST | ASHA "NCD-Mitra" Offline Android App |
| **6. Ethics, Privacy & Governance** | All datasets | **TRDSC-7 & ABDM Governance Protocol:** De-identification, consent management, and audit tracking | Pseudonymized patient UUIDs for ML; SHA-256 state hashing; mandatory worker override reason codes | ABDM & TRDSC-7 Audit Trail |
