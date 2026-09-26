# SPEC-24: StructureDefinitions and GraphDefinitions for the Registration Entities, and a Shared Adaptive Section Nav

| | |
|---|---|
| **Status** | Built, except §7 step 8 (retrofitting Cübo's own mobile navigation onto `AdaptiveSectionNav`). |
| **Last reviewed** | 2026-09-26 |
| **Code (API)** | `data/structure-definitions/*.json` (7 profiles), `data/graph-definitions/{ClinuxFlowOnboardingGraph,ClinuxFlowPatientGraph}.json`, `src/lib/control/{conformance-validator,next-best-action,resource-registry,resource-records-db}.js`, routes `POST /api/{facility,provider,affiliate-organization}/conformance`, `GET /api/facility/affiliates/conformance`, `POST /api/resources/:type/conformance`, `GET /api/resources/:type/search`, `POST /api/resources/:type/save`, `migrations/0013` |
| **Code (frontend)** | `src/components/AdaptiveSectionNav.vue` + `adaptiveSectionNav.js`, `src/components/control/*Host.vue`, `src/data/control/*Conformance.js`, `src/data/control/resourceRecords.js`, `src/pages/{Onboarding,StaffOnboarding,PractitionerHome,PatientHome}.vue` |
| **Related** | SPEC-13 §5, SPEC-23 §8, SPEC-25 §2 |

## 1. Problems this solved

1. "Done", "required" and policy for onboarding were scattered: ad-hoc JS checks and `required`
   flags in YAML, with no standard conformance artifact. Once SPEC-23 removed workflow tracking
   from registration, nothing expressed sequence either.
2. `CustomFormHost.vue`/`FormField.vue` (built to escape LHC-Forms' cramped rendering) was the same
   mistake again: a second generic schema-driven form engine. The fix is hand-authored panels with
   a shared navigation shell, and a standard profile for validation.

## 2. Seven profiles

"Affiliate" turned out to mean two FHIR shapes: a visiting practitioner (a `PractitionerRole` at a
facility that isn't their home) and a partner organization (`OrganizationAffiliation`: org A
provides service X to org B).

| Entity | Resource | Profile |
|---|---|---|
| Facility | `Organization` | `ClinuxFlowFacility` |
| Provider (person) | `Practitioner` | `ClinuxFlowProvider` |
| Provider (role at a facility) | `PractitionerRole` | `ClinuxFlowProviderRole` |
| Affiliate practitioner | `PractitionerRole` | `ClinuxFlowAffiliatePractitionerRole` |
| Affiliate organization | `OrganizationAffiliation` | `ClinuxFlowAffiliateOrganization` |
| Patient | `Patient` | `ClinuxFlowPatient` |
| Workflow task | `Task` | `ClinuxFlowTask` (added by SPEC-25) |

Splitting Provider into Practitioner + PractitionerRole (the YAML used to put role fields on
`Practitioner.extension`) lets one person hold roles at several facilities, which affiliation
requires.

Profiles give standard cardinality, fixed values, bindings and **slicing** (for example
`telecom` sliced by `system`). The 18 ABDM extension fields are declared as extension slices.
`ClinuxFlowFacility` and `ClinuxFlowProvider` are grounded in the HFR and HPR API docs;
`ClinuxFlowPatient` in the ABHA V3 API and M2/M3 docs, including `Patient.contact` and seven ABHA
verification extensions (`abha-kyc-verified`, `abha-verification-status`, `abha-verification-type`,
`abha-email-verified`, `abha-mobile-verified`, `abha-status`, `abha-auth-methods`). KYC-verified and
verification status are independent flags: a child ABHA can be `VERIFIED` without KYC.

## 3. Sequence without a state machine

A GraphDefinition describes references between resource types; it has no notion of time. Two
sources of order, both **pure functions recomputed on demand**, so registration stays untracked:

- **Structural dependency** from the graph's links: a PractitionerRole needs its Organization.
  `ClinuxFlowOnboardingGraph` starts at Organization with links to PractitionerRole (→
  Practitioner) and OrganizationAffiliation (→ participating Organization).
  `ClinuxFlowPatientGraph` has one reverse link, Patient ← Encounter.subject.
- **Business rules** as profile invariants, so "done" means "passes validation".

`nextBestActions(graphDefinition, bundle, validationResults)` returns, for each link whose source
is present and valid but whose target is missing, `{resourceType, reason, linkId,
sourceResourceId}`. No actor, no snapshot. SPEC-25 lets a runtime Task cite such a suggestion as
its reason.

## 4. Knowledge-graph note

A GraphDefinition is a schema, not a knowledge graph. The instances are the persisted resources.
Wikidata QIDs on specialties are the one existing link to an external knowledge graph.

## 5. `AdaptiveSectionNav`

One responsive navigation shell, built on Reka UI primitives:
- `sections: [{id, label, icon, badge?, badgeTone?}]`, `mode: 'tabs' | 'sidebar' | 'accordion' | 'panes'`.
- At 768px and above: the chosen mode. Below: one content area plus a "⋮" menu of sections.
- It owns only navigation; each section's content is a named slot filled by hand-written markup.
- The chosen mode persists per instance (`storage-key`).

Used by Onboarding (page-level sidebar over Hospital, Care Team, Services, Hours, Consents,
Affiliate Organizations, with badges such as "Saved" and "2 added"), inside each host
(Basics / Address / Contact), and by StaffOnboarding, PractitionerHome, FrontDesk,
`FacilityStatusCard` and `BottomSheet`.

## 6. `CustomFormHost` retired

Deleted once every consumer moved to hand-authored hosts. The hosts still save the same
`QuestionnaireResponse` item shape through `mergeGroupResponseItem`/`appendGroupResponseItem`, so
extraction, assembly and conformance are unchanged. Only rendering changed.

Hosts: `FacilityBasicsHost`, `LocationsHost`, `ServicesHost`, `HoursHost`, `ConsentsHost`,
`AffiliateOrganizationHost`, `ProviderBasicsHost`, `ProviderPersonalDetailsHost`,
`ProviderQualificationsHost`, `ProviderWorkExperienceHost`, `ProviderDocumentsHost`,
`PatientBasicsHost`. ABDM panels (`FacilityHfrPanel`, `ProviderHprPanel`, `PatientAbhaPanel`) sit
beside them. HFR and HPR run as `RegistrationLedger` stage lists gated by real API prerequisites
(`facilityHfrJourney.js`, `hprRegistrationJourney.js`), and end in an `AttestationCard`.

## 7. Build record

1. ~~Profiles and graph~~: done.
2. ~~Conformance validator with tests~~: done (`validate(structureDefinition, resource)`).
3. ~~Next-best-action~~: done.
4. ~~`AdaptiveSectionNav` with tests~~: done.
5. ~~Facility reference rebuild~~: done. Onboarding.vue's drawer was later replaced by the
   page-level nav; `LocationsHost` was added because nothing had ever created the locations that
   services and staff reference.
6. ~~Provider, affiliate practitioner, affiliate organization, patient~~: done.
   - Provider: `PractitionerHome.vue` is a resume-style profile page (photo, headline,
     qualifications, experience, documents, HPR credential), with StaffOnboarding as the editing
     surface.
   - Patient: `PatientHome.vue` (`/patient-home`) with a Design Page (a searchable directory: local
     records, plus a read-only "on other devices" list from the paid-tier search) and a Data Page
     (`PatientBasicsHost`), plus Cübo in a `patient-directory` thread. Staff reach it from
     ClinicHome's navigation.
   - **Generic resource API**: `resource_records` (one D1 table keyed by resource type and id,
     clinic-scoped, JSON plus three promoted search columns) and `RESOURCE_REGISTRY`. `save`
     requires the client's stable `recordId` because the extractor mints a fresh id on every call.
     `save` does not require `valid: true`, because the telecom slicing gap (SPEC-13 §5.3) makes
     that unreachable for real patients. The old `POST /api/patient/conformance` was removed.
     Patient is the only registered type; Facility, Provider and affiliates still use their own
     routes.
7. ~~Delete `CustomFormHost.vue`/`FormField.vue`~~: done.
8. Retrofit Cübo's mobile navigation (THREE_PANE's tab strip and the overlay toggles) onto
   `AdaptiveSectionNav`: **not done**.

## 8. Related specs

SPEC-23 (the "no workflow for registration" rule this respects), SPEC-13 (extraction and assembly
unchanged), SPEC-09 (the ABDM extension fields carried forward), SPEC-25 (Task citing a suggestion).

## 9. Open items

- Set `ContactPoint.system` so telecom slices can validate (SPEC-13 §5.3).
- Move Facility, Provider and affiliates onto `resource_records`.
- Designer's optional "include cloud records" toggle.
- §7 step 8.
