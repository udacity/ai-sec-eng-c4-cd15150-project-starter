# HealthGuard — Data Flow Diagram (revised)

A corrected version of artifact A6 (`data_flow_diagram.png`), redrawn to match
the component descriptions (§5) and data flow narrative (§6) in
`case_study_brief.md`. Edge labels cite the narrative step they come from.

```mermaid
flowchart TB
    subgraph hospital["Hospital (external)"]
        EHR["Hospital EHR<br/>HL7 FHIR R4<br/>clinical + registration"]
        BILL["Hospital Billing System"]
    end

    subgraph klariova["Klariova Health (regional)"]
        ING["1. Data Ingestion<br/>mTLS, Kafka buffer"]
        PRE["2. Preprocessing<br/>feature eng. + demographic and insurance join"]
        API["3. Inference API<br/>FastAPI + scikit-learn"]
        DASH["4. Clinician Dashboard<br/>web UI, HITL sign-off"]
        REG[("5. Model Registry<br/>model versions, training datasets and manifests")]
        LOG[("6. Logging and Monitoring<br/>requests, responses, overrides, drift - 7y")]
        PDB[("Prediction Database<br/>EU: Frankfurt / US: AWS US-East - 7y")]
    end

    USR["US Region<br/>(secondary deploy)"]

    EHR -- "patient records: clinical + demographics, every 15 min, mTLS (steps 1-2)" --> ING
    BILL -- "insurance_type (step 3)" --> PRE
    ING -- "raw clinical + demographics (step 3)" --> PRE
    PRE -- "feature vector incl. insurance_type (step 4)" --> API
    REG -- "model artifact load (joblib)" --> API
    API -- "prediction + feature vector + patient ID (step 5)" --> PDB
    API -- "inference request and response" --> LOG
    PDB -- "prediction by patient ID (step 6)" --> DASH
    DASH -- "clinician overrides (step 7)" --> REG
    DASH -- "overrides" --> LOG
    USR -. "cross-border telemetry US to EU: request metadata for drift monitoring" .-> LOG

    classDef external fill:#fbe8d8,stroke:#a0682d,color:#222
    classDef process fill:#d9e8f5,stroke:#1f4e79,color:#222
    classDef personal fill:#fbe6e6,stroke:#a02a2a,color:#222
    classDef usregion fill:#fff4c4,stroke:#9a8420,color:#222

    class EHR,BILL external
    class ING,PRE,API,DASH process
    class PDB,REG,LOG personal
    class USR usregion
```

**Legend:** orange = external system · blue = processing component ·
red = stores personal data · yellow = secondary region · dashed = cross-border flow

## Changes from A6

| A6 edge / element | Revised | Basis in brief |
|---|---|---|
| Dashboard → Registry labelled "log access" | Labelled "clinician overrides" | Step 7: overrides are logged back to the Model Registry for retraining |
| Preprocessing → Registry "model load" | Registry → Inference API "model artifact load" | Component 3 loads the trained joblib artifact; Component 5 stores artifacts |
| Inference API → Dashboard "score" | Inference API → Prediction DB → Dashboard | Step 5 stores predictions; step 6 says the Dashboard pulls by patient ID |
| Logging → Prediction DB "predictions+IDs" | Inference API → Prediction DB | Step 5: inference output is stored in the prediction database |
| Registry → Logging "events" | Removed; Inference API → Logging and Dashboard → Logging added | Component 6 captures inference requests, responses and clinician overrides |
| Model Registry and Logging shown as purple "supporting" storage | Shown red (personal data) | Registry holds training datasets + overrides; logs hold full requests and overrides (patient data) |
| "raw + demo" | "raw clinical + demographics" | Component 2 / step 3: demographics joined with clinical data |
| No billing source | Hospital Billing System → Preprocessing "insurance_type" | Step 3: `insurance_type` is sourced from the hospital's billing system and may be stale |

**Note on the billing flow:** the brief doesn't say how billing data reaches
Klariova. Only the EHR feed is described as FHIR over mTLS, so the transport,
authentication and data-processing agreement coverage for this flow are open
questions worth flagging in Task 2.
