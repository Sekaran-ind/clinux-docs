# SPEC-09: ABDM-Anchored Registration and Citizen Health Record Storage

| | |
|---|---|
| **Status** | Principles current; original mechanism superseded. The canonical-schema form (`abdmSchema.js` + `AbdmFieldForm.vue`) was replaced by SPEC-24's StructureDefinitions and hand-authored hosts and then deleted. §4 chat propagation and §5 DigiLocker export are built. |
| **Last reviewed** | 2026-09-26 |
| **Code** | `clinux-frontend/src/data/sessionShare.js`, `src/data/sessionTransfer.js`, `src/components/cubo/CuboContactConversation.vue`, `src/components/TeamChat.vue`, `src/data/runtime/digilockerExport.js`, `src/pages/Checkout.vue`, `src/data/useSystemForms.js` |
| **Related** | SPEC-11, SPEC-23 (build notes), SPEC-24 |

## 1. Problem

Registration data for providers, staff and facilities used to go through a generic pipeline
(YAML → Questionnaire → LHC-Forms → extraction), then get hand-translated by `abdmAdapter.js`
into ABDM's request shapes. That double modeling caused a real, live-confirmed bug: typing a
value into a coded or autocomplete LHC-Forms field and saving without clicking a suggestion
silently discarded the value. Registration data has a **fixed external schema**; it should not
flow through machinery built for arbitrary clinic-authored forms.

## 2. Canonical schema: driven by ABDM's requirements, not its request bodies

Two options were weighed:
- Store ABDM's literal request bodies. Rejected: ABDM's shapes are externally versioned and
  inconsistent between endpoints (`mobileNumber` in one, `mobile` in another).
- **A clean, FHIR-consistent schema whose field set is driven by what HFR, HPR and ABHA need.**
  Chosen. Submission becomes a thin mapping.

This principle survived; the implementation changed. The schema is now a set of StructureDefinitions
(`ClinuxFlowFacility`, `ClinuxFlowProvider`, `ClinuxFlowProviderRole`, `ClinuxFlowPatient`) with
ABDM fields as distinct, URL-identified extensions (18 of them, re-homed in SPEC-23's build). The
submission mapping lives in `src/data/control/abdmAdapter.js` (HFR/HPR) and `abhaAdapter.js`.

## 3. Scope: registration only

The generic YAML → Questionnaire → LHC-Forms pipeline stays for encounters, vitals, SOAP and
clinic-authored forms, where arbitrary structure is the point. Only registration moved to
purpose-built capture (SPEC-24's hosts), with real selects and controlled inputs.

The LHC-Forms coded-field bug was also fixed for the generic system:
`useSystemForms.js` attaches a capturing `focusout` listener that saves a coded field's typed
value before the widget's own blur handler clears it, and `extractResponse()` grafts any missing
value back as `valueString`.

## 4. Free-tier data authority

Two ways data reaches other staff without cloud storage:
- **LAN shared server**: the administrator's device and the Tauri shared server are authoritative
  while on the clinic network (`sharedServerSync.js`).
- **Chat propagation**: the admin shares the clinic profile as an encrypted transfer key over the
  P2P DataChannel; the recipient sees a "Clinic profile transfer" card with Import
  (`buildProviderProfileSharePayload` / `decodeProviderProfileShareKey` / `isSameClinic`). Built in
  `TeamChat.vue` and in Cübo's `CuboContactConversation.vue`. The same string still works through
  the QR and paste fallback, so both parties don't need to be online together.

Since SPEC-26, staff joining a published clinic can also pull the clinic profile directly from
`GET /api/provider-composition`, so chat propagation is the offline fallback, not the main path.

## 5. Citizen health records live in DigiLocker, not ClinuxFlow

ClinuxFlow does not become the long-term custodian of a citizen's records. DigiLocker is an ABDM
Health Locker (PHR app) with real push and pull. Two levels, neither needing a separate DigiLocker
integration:

- **Built (free tier)**: Checkout's "Health Record (DigiLocker)" button generates a visit-summary
  PDF (jsPDF) with an embedded QR carrying the encounter as a `sessionTransfer` string. The
  citizen stores it with DigiLocker's own Scan/Upload. QR at 480 px width (measured: 320 px failed
  to decode reliably), error correction level `L`.
- **Later**: once a facility is HFR-registered and ClinuxFlow is a certified HIP, records flow to
  the citizen's PHR app through ABDM's consent-based exchange (SPEC-11 §5).

A HAPI FHIR store for queryable enterprise records stays deferred.

## 6. Relationship to current code

| Original piece | What happened |
|---|---|
| `abdmSchema.js` (`HOSPITAL_FIELDS`, `STAFF_FIELDS`) | Deleted; replaced by StructureDefinitions (SPEC-24) |
| `AbdmFieldForm.vue` | Deleted; replaced by hand-authored hosts in `AdaptiveSectionNav` |
| `AbdmOnboarding.vue` | Deleted; replaced by `FacilityHfrPanel`, `ProviderHprPanel`, `PatientAbhaPanel` |
| `HospitalOnboarding.vue` | Deleted; `/hospital-onboarding` redirects to `/onboarding` |
| `abdmAdapter.js` | Kept and extended (real HFR additional and detailed information builders) |

## 7. Open items

- ~~Canonical field list vs HFR additional/detailed information~~: done in `FacilityHfrPanel`'s
  seven-stage ledger and the adapter's builders.
- ~~QR format for the DigiLocker document~~: reuses the device-transfer format.
- Raise the QR `errorCorrectionLevel` from `L` to `M`? Better scan reliability, lower capacity
  before falling back to text. It affects `SessionShareModal.vue` too. Undecided.
- The QR in the DigiLocker PDF is decodable only by ClinuxFlow (it is a `sessionTransfer`
  string). A third-party provider scanning it gets nothing useful. A plain-text summary is in the
  PDF itself, but a standards-based QR (for example a FHIR document link) is worth considering
  once HIP certification is pursued.
