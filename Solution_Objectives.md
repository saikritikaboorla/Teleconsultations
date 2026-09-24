### Core Problem Definition

The problem statement addresses a critical gap in rural primary healthcare delivery in Telangana's teleconsultation pipeline. At Ayushman Arogya Mandirs (AAMs), Community Health Officers (CHOs) connect chronic disease patients (hypertension and diabetes) with specialist doctors at district hospitals via eSanjeevani.

While the virtual consultation is marked as "completed" in the system, there is zero visibility into whether the patient actually received treatment, as a large portion of patients drop out immediately afterward. They fail to collect prescribed medicines, complete mandatory diagnostic tests, or return for their required review visits within weeks. The fundamental issue is **data fragmentation**: data across teleconsultation logs, NCD screening records, pharmacy dispensing registers, and ASHA diaries sits in isolated silos and is never linked together.

---

### Objectives Your Proposed Solution Must Satisfy

To solve this problem, your solution must address three main problem-statement questions while working under specific practical constraints:

#### 1. Data Linkage Architecture

* **Objective:** Establish a unified tracking system that links isolated data streams (eSanjeevani teleconsultation records, NCD screening data, medicine dispensing registers, visit registers, demographics, and geographic data) to establish a single, continuous record of each patient's care journey.


* **Deliverable:** Demonstrate how these disparate logs can be deterministically matched to track a patient's true follow-up status end-to-end.



#### 2. Dropout Identification & Risk Prediction Model

* **Objective:** Identify the exact stages where patients drop out of care and build a predictive, AI/data-driven model to calculate which patients are at risk of becoming "Lost to Follow-Up" (LTFU) before it happens.


* **Deliverable:** Use the provided synthetic dataset to perform exploratory analysis, identify key drop-out factors, and output risk scores/categories for patients.



#### 3. Data-Driven Intervention Framework

* **Objective:** Design an actionable tool or workflow (e.g., prioritized task list) that guides health workers (CHOs and ASHAs) to proactively bring high-risk patients back into care.


* **Deliverable:** Define how the system moves health workers from simply counting completed consultations to ensuring complete, closed-loop treatment.



#### 4. Measurement & Evaluation Framework

* **Objective:** Define explicit metrics to measure the solution's performance.


* **Deliverable:** Provide concrete evaluation methods for:
* **Effectiveness:** Reduction in LTFU rates, improved medication/test compliance.


* **Feasibility:** Adoption ease for health workers, operational workflow fit.


* **Cost:** Financial viability and resource efficiency of scaling the intervention.





---

### Mandatory Operational Constraints & Considerations

Your proposed design must operate within real-world rural healthcare limitations:

* **Hardware & Connectivity:** Must function on low-end smartphones with intermittent or no internet connectivity (offline-first capability).


* **Workforce Capacity:** Must respect health-worker time constraints without overburdening CHOs and ASHAs with complex data entry.


* **Compliance & Security:** Must be privacy-preserving and compliant with Ayushman Bharat Digital Mission (ABDM) standards, including explicit patient consent management.


* **Evidence Base:** All proposals and claims must be supported by empirical findings and data analysis derived directly from the provided synthetic dataset.v
