# SPEC-04: Clinical Workflow Orchestration and Data Sync

| | |
|---|---|
| **Status** | Superseded. Rooms and sequencing: SPEC-12, SPEC-13, SPEC-23. Extraction and documents: SPEC-13 §5. Sync: SPEC-19, SPEC-25. This spec now records what the outpatient visit actually does. |
| **Last reviewed** | 2026-09-26 |
| **Code** | `clinux-frontend/src/pages/{ClinicHome,FrontDesk,ConsultationDesk,Checkout}.vue`, `src/stores/clinical.js`, `src/data/runtime/encounterCoordination.js`, `src/data/collections/encounterDocs.js`, `clinuxflow-api/src/routes/runtime.js` (`/api/encounters/*`) |

## 1. The original plan, and what changed

The original plan composed small YAML fragments per role and specialty into one Questionnaire,
rendered it with LHC-Forms, and posted the completed response to a HAPI FHIR server's
`Questionnaire/$extract` to create Patient, Observation and Condition resources server-side.

| Original | Now |
|---|---|
| HAPI server `$extract` | Local definition-based extraction (`local-extractor.js`), run in the Worker (SPEC-13 §5). No FHIR server is connected. |
| Runtime fragment composition per role | One compiled encounter composition (`system-encounter-composition-v1.yaml`) plus clinic-authored custom forms attached to the patient journey. Specialty-parameterized rooms are designed in SPEC-23 but not built. |
| Server as source of truth | The device is the source of truth; paid clinics mirror encounter documents to D1. |

## 2. The outpatient visit as built

The three stations are views inside `ClinicHome.vue` (not separate routes; the old routes
redirect to `/clinic-home`). Each is a two-pane layout: Cübo on one side, the working surface on
the other, collapsing to a toggle below 768px.

| Station | What happens |
|---|---|
| **Front Desk** | Find or register the patient (with optional ABHA lookup or creation, `PatientAbhaPanel.vue`), open an encounter, capture intake, attach additional custom forms, assign the next station. |
| **Consultation Desk** | Vitals and clinical record (LHC-Forms), SOAP drafting (Cübo, `test-scribe` on paid tier), prescriptions (jsPDF), imaging review (Cornerstone DICOM viewer), documents drawer, video consult (RealtimeKit). |
| **Checkout** | Billing (AG Grid payments table), prescription and visit PDFs, DigiLocker PDF+QR export, closing the encounter (`encounter_status: finished`). |

`encounter_status` (`arrived | in-progress | finished | cancelled`) is the open/closed signal. The
active-sessions list shows every encounter not in `finished`/`cancelled` (a deny-list, after an
allow-list bug hid every new session).

## 3. Coordination between staff

- **Locks**: `POST /api/encounters/:id/lock` (plus `renew`/`release`) gives one device at a time
  the right to work a stage. Locks carry a TTL so a crashed tablet cannot hold a patient forever.
- **Assignment**: `POST /api/encounters/:id/assign` routes an encounter to a named staff member
  or linked affiliate.
- **Documents**: `PUT /api/encounters/:id` mirrors the encounter's documents to
  `encounter_documents`, best-effort after the local save.
- **Security gap (SPEC-01 §10 item 0)**: the document read and write, and the encounter video
  join, do not check that the encounter belongs to the caller's clinic. Fix before any pilot.
- All three are paid-tier (`requirePaidTier()`). On the free tier, coordination happens on one
  device or over the LAN shared server, and a 403 from these routes is treated as "local only",
  not a failure (the Checkout lock bug taught this).

## 4. What is not built

- The visit is not a PlanDefinition. There is no persisted Task per station, no personal
  worklist, and no automatic "next station is ready" notification (SPEC-13 §2.3, SPEC-22 §2).
- No attested clinical document. Composition assembly exists (SPEC-13 §5.4) but is not called
  from Checkout, and nothing signs it.
- No lab orders, IPD, pharmacy stock, or notifications (SPEC-10 §4).
