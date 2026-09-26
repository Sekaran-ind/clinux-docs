# SPEC-23: Speciality Rooms as the Growing Dimension; Facility and Provider as Fixed Anchors

| | |
|---|---|
| **Status** | Design for Speciality Rooms (§3–§5), nothing built. The onboarding rebuild this spec triggered **is built** (§8), and one rule from it governs the whole system: *state machines only for clinical journeys*. |
| **Last reviewed** | 2026-09-26 |
| **Related** | SPEC-16 §3, SPEC-18, SPEC-21 §6, SPEC-22 §5.11–§5.14, SPEC-24, SPEC-25 |

## 1. Problem

SPEC-22 §5.11 shipped five hand-authored rooms. The real future is that every clinical specialty
becomes a room, a count that grows with the business, and each specialty room shares
administrative steps (onboarding, imaging, labs, billing), some provided by affiliates. A static
array cannot represent that.

## 2. Already settled elsewhere (not reopened)

- Public and Authenticated overlap at "onboard" (SPEC-21 §3, §6).
- ABDM registries sit beside the journey, not in it.
- A hospital's `affiliates` and a provider's `linking` are one mechanism (SPEC-26's join tokens and
  `facility_affiliates`).
- **Patient is not an actor** (SPEC-21 §6).
- The storage lifecycle belongs in SPEC-19 §7's conflict design.

## 3. Speciality Room: the new dimension

A room is **specialty × service category** (for example General Medicine × Out-Patient), running
`Onboard → Triage → Consult → SOAP → Checkout`.

### 3.1 Three fixed parts

| Fixed part | Shape | State |
|---|---|---|
| **Facility** | Roster lifecycle (register → onboard → affiliates → activate → maintain), perpetual | Built as pages plus conformance (SPEC-24) |
| **Provider** | Roster lifecycle (register → onboard → linking → activate → maintain), perpetual | Built; "activate" still undefined |
| **Speciality Room mechanism** | Pipeline, one run per patient visit | **Not built** |

The fixed part is the *mechanism*, not the number of rooms. Roster lifecycles and visit pipelines
are different machine shapes (SPEC-16 §3).

### 3.2 "Services offered become the workflow"
`HealthcareService.category` already captures a specialty. A facility declaring a service in
category X is what should create room X, as data, not code.

**Gap, now worse than when written**: the specialty vocabulary exists in three places:
1. `system-provider-composition-v1.yaml` → `service_specialty_category` choices (6 values:
   Cardiology, Pediatrics, Ophthalmology, Optometry, Diet & Wellness, General Medicine).
2. `ServicesHost.vue` → a hard-coded copy of the same 6 (`SPECIALTY_CHOICES`), since SPEC-24's
   hand-authored hosts don't read the YAML.
3. `data/clinic-specialities.json` → 42 practitioner-type entries ("Cardiologist", "Dentist", …)
   with SNOMED mappings, used by Cübo's virtual-room persona picker.

These are different concepts (a service line versus a kind of practitioner), but they need one
governed source per concept, published as a ValueSet the YAML, hosts and Cübo all read.

### 3.3 Shared sub-flows by reference
A specialty room's plan should mostly be references to shared plans (Onboarding, Imaging,
Billing) via `PlanDefinition.action.definitionCanonical`, plus the steps unique to that specialty
(an ECG step for cardiology). **Not built**: the compiler, extractor and runtime have never handled
`definitionCanonical`. The open choice is to inline the referenced plan at compile time, or to run
it as a separate plan and gate on its completion (the `onActionDone` pattern).

### 3.4 Affiliates fulfill Tasks, not PlanDefinitions
Who performs a step is `Task.owner` or `ServiceRequest.performer`, not part of the plan. This is a
second reason (besides SPEC-16's notebooks) that persisted Tasks with owners matter. SPEC-25
persists runtime snapshots and audit; per-action Task ownership is still not built.

## 4. Encounter and EpisodeOfCare

An Encounter is one run of a Speciality Room's plan for one patient. Some conditions group many
encounters under an `EpisodeOfCare` (a high-risk pregnancy, a rehab programme, long-term diabetes
care):

- `Encounter.episodeOfCare` (0..*) links a visit into an episode; most visits never set it.
- `EpisodeOfCare.diagnosis.condition` anchors the episode to what is being managed.
- There is **no** `EpisodeOfCare.carePlan` field. `CarePlan` links indirectly through the same
  Condition (`CarePlan.addresses`) and Patient.
- `CarePlan.instantiatesCanonical` can reference a PlanDefinition protocol authored through
  SPEC-18's pipeline, instantiated per patient.

This is a third machine shape: bounded but long-running, grouping many visit pipelines. Whether an
episode is needed is a clinical judgment per patient, not a property of the specialty.

## 5. Consequence for Designer

Replace hand-written room YAML with a native step-list editor that can "add a step" or "embed a
shared sub-flow", and list rooms from the reconciled specialty catalog rather than
`ROOM_DEFINITIONS`.

## 6. Storage and tiers

The diagram's "Enterprise: HAPI" storage fits the existing additive tier model (SPEC-05 §6.3).
Chat, video and LLM gating by tier match what exists. Nothing new here.

## 7. Open questions

- One catalog or two (§3.2).
- `definitionCanonical` resolution strategy (§3.3).
- What a provider's "activate" means.
- Service categories beyond Out-Patient (In-Patient, Day-Care) and which combinations each facility
  offers.
- Where CarePlan protocols are governed: the same library as operational flows, or a separate
  clinical-protocol catalog with its own review.
- Who decides that encounters should be grouped into an episode, and where in the UI.

## 8. What was built from this spec (the onboarding rebuild)

Working through this spec led to a rebuild of Facility, Provider and Patient onboarding:

1. **Extraction fixes**: sibling fields sharing a path (phone and email on `telecom`) and bare
   objects where FHIR requires arrays, fixed with `FHIR_ARRAY_PATHS`; MultiSelect answers no longer
   truncated (SPEC-13 §5.3).
2. **FHIR document assembly** (SPEC-13 §5.4).
3. **Real `extension.url`s**: 18 ABDM-specific fields re-homed onto distinct extensions; a real
   `Organization.active` boolean bug fixed.
4. **One Facility surface**: `HospitalOnboarding.vue` and its chat variant deleted;
   `/hospital-onboarding` redirects to `/onboarding`.
5. **One Provider surface**: `AbdmFieldForm.vue` and `abdmSchema.js` deleted.
6. **Away from LHC-Forms for registration**: first a generic `CustomFormHost.vue`, which SPEC-24
   then replaced with hand-authored hosts.
7. **The rule**: registration is not a tracked PlanDefinition; journeys are role-gated page links
   (`onboardingJourneys.js`). State machines are for clinical journeys only (SPEC-22 §5.14).

## 9. Recommended order

1. Reconcile the specialty catalog (§3.2). Small, and it unblocks naming everything else.
2. Run the outpatient visit as a real PlanDefinition for one specialty (General Medicine),
   producing persisted Tasks (SPEC-04 §4, SPEC-25).
3. Native step-list editor for one room's own steps.
4. `definitionCanonical` composition once a second shared sub-flow exists.
5. Per-action Task ownership for affiliate fulfillment; EpisodeOfCare builds on the same work.
