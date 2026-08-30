# Specification 04: Clinical Workflow Orchestration & Data Sync

## 1. Objective
Define the multi-specialty clinical workflow state manager, layout composition framework, and persistence flow using LHC-Forms and a local HAPI FHIR server instances.

## 2. Multi-Specialty Modular Configuration Engine
- **Strategy**: Avoid building monolith forms for each clinic type. Implement a composition system that assembles smaller YAML layout fragments based on **User Roles** (Admin, Nurse, Specialist) and **Clinical Specialty Contexts**.
- **Execution Workflow**:
  - The application reads a shared session context state: `EncounterID` and `PatientID`.
  - When an endpoint is loaded, a module loader selects layout manifests (e.g., `core-demographics.yaml` + `triage-vitals.yaml` + `cardio-exam.yaml`) and resolves them into a unified, composite FHIR Questionnaire JSON document.
  - The compiled result is passed directly to the browser UI layer using the following native initialization hook: `LForms.Util.getFormDefFromFHIRQuestionnaire(compositeQuestionnaire)`.

## 3. Persistence Loop & Data Synchronization Matrix
1. **Layout Load**: The UI pulls structural information from the HAPI FHIR server, populating patient contexts using standard REST calls.
2. **Form Render**: LHC-Forms renders the dynamic interface elements inside the clinical console container.
3. **Data Ingestion**: The runtime scribe engine captures data inputs and updates the corresponding target nodes using the `linkId` mapping references.
4. **Payload Finalization**: When a user submits a record, the client platform invokes native data compilation routines to extract a valid, structured `QuestionnaireResponse` resource payload.
5. **HAPI Server Processing**:
  - The application issues an HTTP POST containing the completed payload to the following target endpoint: `POST [HAPI_SERVER_BASE]/Questionnaire/$extract`.
  - The server handles the automated decomposition rules outlined by the inline metadata constraints, generating separate, linked clinical assets (`Patient`, `Observation`, `Condition`) natively in the database.
  - The client UI tracks the server transaction outcome, surfacing errors clearly if any constraints or data relationships fail verification rules.
