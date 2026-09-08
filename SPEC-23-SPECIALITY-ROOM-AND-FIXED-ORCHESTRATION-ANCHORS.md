# Specification 23: Speciality Room as the Growing Dimension, Facility/Provider as Fixed Orchestration Anchors

## 1. Objective

Resolve a real scaling gap in SPEC-22 §5.11's just-built Room-Architect Designer: `ROOM_DEFINITIONS` is a static, hand-authored 5-entry array, but the real future is *every clinical specialty becomes a room* — a count that grows with the business, not a fixed set a developer hand-edits. Ground a user-provided "States and Transition Map" diagram against real code (not assumed), reconcile it with what `docs/SPEC-21-STATE-MACHINE-MAP-SIX-DIMENSION-ARCHITECTURE.md` §6 and `docs/SPEC-16-NOTEBOOKS-AND-TASK-PRIMARY-NAVIGATION.md` already resolved, and name precisely what's genuinely new. Nothing built yet — this is the write-up the user explicitly asked for before any more code.

## 2. What the diagram confirms that's already resolved — not re-litigated here

Most of the diagram's structure was already analyzed, against a near-identical earlier version, in SPEC-21 §6. Confirmed unchanged, not re-derived:

- The Public/Authenticated overlap crossing exactly at "onboard" (SPEC-21 §3's structural critique, §6's confirmation).
- ABDM (HFR/HPR/ABHA) as a side-structure inside Authenticated, not sequenced into Journey.
- Per-role Journey Stages — Hospital's `affiliates` / Provider's `linking` are the same real mechanism (`facility_affiliates` table, `addAffiliate`/`listAffiliatesByFacility`), not two. Hospital's `activate` is real (`onboarding.js`'s `publish()`). Provider's `activate` is still a real open question (SPEC-21 §6 already flagged this, unresolved) — likely HPR/account activation, not confirmed to be the same mechanism as clinic-publish.
- **Patient is not a fourth roster-shaped peer of Facility/Provider.** SPEC-21 §6 resolved this directly, by explicit instruction: *"Patient will not have a login to this application. Patient can see his health records via digi-locker no features more than that."* Patient never gets an `accounts.role` value, never has its own Auth-Stage/roster branch. The diagram's Patient-facing row (Onboard/Triage/Consult/SOAP/checkout, ABHA) is staff-mediated data flow, not an authenticated actor's own journey — this spec's own reading of "3 fixed parts" below is built on that resolution, not a reopening of it.
- The Storage-tier record lifecycle (Draft→Ready→Reviewed→Complete, Retrieve/Publish between Local and Server) — SPEC-21 §6 already named this as real, valuable, missing modeling that should reconcile with `docs/SPEC-19-LOCAL-FIRST-LOCAL-SERVER-AND-FEDERATED-MODES.md` §7's conflict-detection rather than become a second parallel design. Unchanged recommendation.

## 3. What's genuinely new: Speciality Room as an explicit, named dimension

The one structural piece absent from SPEC-21 §6's own summary and present here for the first time: **`Speciality Room: General Medicine` × `Service Category: Out-Patient`**, running its own real sequence (`Onboard → Triage → Consult → SOAP note → checkout`). This is the diagram's answer to the scaling problem SPEC-22 §5.13 named: rooms aren't a fixed set, they're one per specialty × service category, and the count grows as the business does.

### 3.1 The three fixed orchestration parts, precisely

Re-reading the user's own words against §2's Patient resolution: *"Facility and Providers remains fixated"* — the third fixed part is not Patient (already resolved as a non-actor), it's **the Room/Service mechanism itself**: not a fixed *count* of rooms, but a fixed *mechanism* that supports N specialties as data.

| Fixed part | Shape | Real status |
|---|---|---|
| **Facility** | roster-lifecycle (`Register→Onboard→Affiliates→Activate→Maintain`), perpetual | Mostly built — SPEC-09/11's ABDM HFR onboarding, `facility_affiliates`, `onboarding.js#publish()` |
| **Provider** | roster-lifecycle (`Register→Onboard→Linking→Activate→Maintain`), perpetual | Mostly built for Register/Onboard/Linking (HPR self-registration, same affiliate mechanism); `activate` unconfirmed (SPEC-21 §6's open item) |
| **Speciality Room mechanism** | pipeline (`Onboard→Triage→Consult→SOAP→checkout`), one instance per patient visit | **Not built** — this spec's real subject |

This maps directly onto `docs/SPEC-16-NOTEBOOKS-AND-TASK-PRIMARY-NAVIGATION.md` §3's own table, which already named two structurally different machine shapes and flagged the distinction as real: *"Affiliation/Roster: a specific Practitioner-Facility or Facility-Facility relationship, perpetual for that relationship's lifetime"* versus *"an Encounter's process/pipeline machine."* Facility/Provider's own tracks are the Affiliation/Roster shape — SPEC-16's own "Maintain" as a perpetual, non-terminal state is exactly what `planDefinitionRunner.js`'s existing `action.repeatable: true` already models; no new machine type is needed for this half. Speciality Room's `Onboard→Triage→Consult→SOAP→checkout` is the pipeline shape — the same shape `planDefinitionRunner.js` already runs for Facility setup today, just at a different anchor.

### 3.2 "Services rendered becomes the workflow" — a real, already-captured field, not a new concept

Confirmed directly in `tools/system-forms/system-provider-composition-v1.yaml`: the existing `section_services_matrix` group (Designer's "Services" card, one of the 8 Provider entities) already maps to real FHIR `HealthcareService`, and its `category` field is *already* a specialty dropdown:

```yaml
- id: "service_specialty_category"
  path: "HealthcareService.category"
  label: "Core Specialty Category"
  choices: ["Cardiology", "Pediatrics", "Ophthalmology", "Optometry", "Diet & Wellness", "General Medicine"]
```

"General Medicine" — the diagram's own worked example — is already one of these six hardcoded choices. This is the real, concrete mechanism behind "services rendered becomes the workflow": a Facility/Provider declaring a `HealthcareService` with category X is what should determine which Speciality Rooms exist for them, and each one is a workflow instance spawned from that declaration — not a developer hand-adding a room to a JS array.

**A real, precisely-identified gap, not assumed**: this 6-value hardcoded dropdown and `clinic-specialities.json` (the real, already-used-elsewhere catalog behind Cübo's persona picker — 42 real specialties: Allergist, Cardiologist, Clinical psychologist, Dentist, Dermatologist, ENT, Family practitioner, Gastroenterologist, ...) are **two divergent catalogs today**, different in both size and naming convention (`"Cardiology"` vs. `"Cardiologist"`). Reconciling these into one real source of truth is part of this spec's concrete scope, not a side note — exactly the two-sources-of-truth pattern this whole session line has repeatedly found and fixed (Hospital data across 3 surfaces; `journey` vs. the new `roomId`).

### 3.3 Composition: shared administrative sub-flows, authored once

The previous session turn named onboarding/imaging/laboratories/billing as steps shared across many specialty rooms, some potentially fulfilled by an affiliate. The FHIR-native mechanism for this already exists and requires no new invention: `PlanDefinition.action.definitionCanonical` — an action that *is* a reference to another PlanDefinition, not a leaf step authored inline. A Cardiology room's plan becomes mostly a sequence of `definitionCanonical` references (Onboarding, Imaging, Billing — each its own `flowsLibrary` entry, authored once) plus whatever steps are genuinely specialty-specific (e.g. an ECG step only Cardiology rooms have). This is real composition, not copy-paste across every specialty's own plan — directly closing the gap SPEC-22 §5.13 named.

**Not built**: `local-extractor.js`/`yaml-to-questionnaire.js` have never produced or consumed `action.definitionCanonical` — this needs real compiler/extractor support, and `planDefinitionRunner.js`'s own machine-building has never resolved a cross-plan reference at runtime (today, `registerPlan()` takes one flat `PlanDefinition.action[]` array; a `definitionCanonical` reference would need to be *resolved* — presumably by inlining the referenced plan's actions, or by registering it as a genuinely separate sub-plan and gating on its completion the same way `entryWorkflow.js`'s `facility_registration`↔`hospitalSetupWorkflow.allDone` watch already does). Which of those two resolution strategies is right is a real open design question, not decided here.

### 3.4 Affiliate fulfillment: a `Task`-level question, not a `PlanDefinition`-level one

"Some of these may become shared services offered by affiliates" is a *who*-fulfills-this-step question, and `PlanDefinition` only ever describes *what* should happen — it has no concept of which Organization actually does it. That's `Task.owner`/`Task.requester`, or `ServiceRequest.performer` — real FHIR fields for exactly this, and squarely inside SPEC-16 §6 step 1's still-open item: *"no `Task`/`PlanDefinition` resource is persisted anywhere yet... `Task` persistence itself remains open."* This spec doesn't resolve that gap; it adds a second, independent reason (beyond SPEC-16's own Notebook/left-pane-navigator motivation) that persisted Task records with a real owner field are the next big foundational piece, not a nice-to-have.

## 4. Encounter's role, confirmed — and refined: EpisodeOfCare as a real third machine shape

Encounter is not a sixth static room alongside Facility/Provider/Speciality-Room — it's the Task-level runtime instance of one patient running a Speciality Room's plan once, the same Definition-vs-instance split Facility's own plan (`HOSPITAL_SETUP_PLAN_DEFINITION`, definition) already has against a specific hospital's own progress through it (`hospitalSetupWorkflow.js`'s actor state, instance). `chatThreads.js`'s existing `encounter` category / `encounterId` prop routing already treats Encounter this way structurally (one thread per visit) — this spec doesn't change that, it gives it a formal PlanDefinition-vs-Task grounding it didn't have stated explicitly before.

That's the common case — a single, self-contained visit. A real refinement, correct and additive, not a correction of the above: **some conditions need multiple Encounters managed under one umbrella** — a high-risk pregnancy's prenatal visits, a 3-month physical-therapy rehab program's sessions. FHIR's real mechanism for this is `EpisodeOfCare`, and it's worth being precise about the actual field-level relationships rather than gesturing at the concept:

- `Encounter.episodeOfCare` — a real, optional (`0..*`) field. Most Encounters (a walk-in General Medicine visit) never set it. An Encounter that's one visit within a longer program does.
- `EpisodeOfCare.diagnosis.condition` — what ties the episode to the actual condition being managed (the pregnancy, the rehab diagnosis) — this is the real anchor, not a specialty-room label.
- **`CarePlan` is not directly referenced BY `EpisodeOfCare`** — there's no `EpisodeOfCare.carePlan` field in base FHIR. The real linkage is indirect: both anchor to the same `Condition` (`CarePlan.addresses` ↔ `EpisodeOfCare.diagnosis.condition`) and the same `Patient`, and a `CarePlan.encounter` can tie the plan to whichever visit it was authored during. Worth stating precisely now, since assuming a direct field exists would produce wrong extraction/composition logic later.
- **A genuinely elegant fit with what's already decided, not a new mechanism**: `CarePlan.instantiatesCanonical` is a real field referencing a `PlanDefinition`. The same SPEC-18 YAML-authoring pipeline this whole design line already runs on — and §3.3's proposed `definitionCanonical` composition — could, without inventing anything new, also be the source a patient-specific `CarePlan` gets instantiated from (a "High-Risk Pregnancy Management" protocol, authored once as a PlanDefinition the same way a Speciality Room's plan is, then instantiated per patient as a real CarePlan carrying the goals/interventions/medication/nutrition content).

**This is a real, third machine shape**, distinct from both of SPEC-16 §3's already-named ones — not perpetual like Affiliation/Roster, not a single bounded pipeline like Encounter, but **a bounded-yet-long-running series of Encounter-pipeline-runs, grouped around one condition**. See SPEC-16 §3's own table, extended with this row below, for the canonical notebook-type enumeration this implies.

EpisodeOfCare is **conditional, not universal** — it sits *above* Encounter, optionally, decided per patient/condition, not tied to which Speciality Room is running (the same "General Medicine" room could occasionally need one too, e.g. ongoing diabetes management across many GM visits — this isn't a property of the specialty, it's a property of whether the patient's condition needs longitudinal grouping at all).

## 5. Consequence for the Room-Architect Designer build (SPEC-22 §5.11/§5.13)

SPEC-22 §5.13 already recommended replacing "Design & Compile Room"'s YAML→compile→fill-a-form→extract pipeline with a native step-list editor, independent of this spec's own scope — that recommendation stands and is *sharpened*, not changed, by the above: the native editor needs to support picking "embed a shared sub-flow" (§3.3's `definitionCanonical`) alongside "add a leaf step," and Speciality Rooms themselves should be listed from the reconciled specialty catalog (§3.2), not `ROOM_DEFINITIONS`' static array. Facility/Provider/Account stay exactly as SPEC-22 §5.11 built them — this spec's changes are additive to the Speciality-Room dimension specifically, not a rewrite of what's already shipped and working.

## 6. Storage/tier and P2P — confirmed, minor unification, not a new initiative

The diagram's `APP TIER: ENTERPRISE STORAGE: HAPI` (a real, open-source FHIR server implementation — a concrete choice, not a placeholder) sits alongside the already-real Free/local and Paid/cloud tiers `docs/SPEC-05-DATA-TIER-AND-ABDM-BOUNDARY.md`/`docs/SPEC-19-LOCAL-FIRST-LOCAL-SERVER-AND-FEDERATED-MODES.md` already cover — SPEC-21 §4's "unified Storage state, thin unification not new mechanics" recommendation already anticipated exactly this. CHAT/Video-conf/LLM gated by tier is consistent with the already-shipped `requirePaidTier()` gating (SPEC-05) and the already-built P2P chat feature (`clinux-p2p-user-chat-feature` memory) — nothing here contradicts what exists; it's confirmation, not new scope.

## 7. Open questions, not resolved here

- **Specialty-catalog reconciliation** (§3.2): does `HealthcareService.category`'s dropdown get replaced with `clinic-specialities.json`'s real 42-entry catalog, or do the two stay deliberately distinct (a broader "what kind of practitioner" catalog vs. a narrower "what services does this specific facility offer" one)? Not obviously the same list even once reconciled in principle.
- **`definitionCanonical` resolution strategy** (§3.3): inline-expand a referenced sub-plan's actions into the parent at compile time, or register it as a genuinely separate runtime plan gated by a cross-plan completion watch (the pattern SPEC-22 §5.9's `facility_registration`↔`hospitalSetupWorkflow` link already uses once)? Different implementation cost and different resumability/persistence behavior.
- **Provider's `activate` step** — still unconfirmed real mechanism, carried forward unresolved from SPEC-21 §6.
- **Service Category's own scope** — is `Out-Patient` one of several categories every specialty room supports (In-Patient, Day-Care, ...), or specific to which combinations exist per facility? Not specified by the diagram alone.
- **EpisodeOfCare/CarePlan persistence shape** (§4): a fourth notebook type (SPEC-16 §3, extended below) with no persistence model of its own yet, same "not built" status as Task itself (SPEC-16 §6 step 1). Whether `CarePlan.instantiatesCanonical` reuses the SAME `flowsLibrary`/PlanDefinition storage Speciality Rooms use, or needs its own catalog (clinical protocols are authored/governed differently than operational workflows — different audience, different review process, arguably a different `resourceType` scope entirely) is not decided here.
- **Which Encounters actually need grouping** is a per-patient clinical judgment call (does THIS diabetes patient's care need an EpisodeOfCare, or is each GM visit still independent?) — not a system-decidable rule from Speciality Room alone. Where/how that judgment gets made (staff-prompted at checkout? Provider-initiated explicitly?) is unspecified.

## 8. Relationship to existing specs

- `docs/SPEC-21-STATE-MACHINE-MAP-SIX-DIMENSION-ARCHITECTURE.md` §6 — this spec is a direct continuation, not a replacement; §2 above states precisely what's reused unchanged.
- `docs/SPEC-16-NOTEBOOKS-AND-TASK-PRIMARY-NAVIGATION.md` §3's Affiliation/Roster-vs-Encounter-pipeline distinction is the structural backbone of §3.1 above; §6 step 1's still-open Task persistence is what §3.4's affiliate-fulfillment question AND §4's EpisodeOfCare persistence both depend on. §3's own notebook-type table gets a fourth row (EpisodeOfCare) per §4's refinement — worth adding there directly since that's the canonical enumeration, not duplicating it here.
- `docs/SPEC-22-PERSISTED-WORKFLOW-SYSTEM-FLOWS-CUBO-STATE-MIRROR-DRAWER-CAPTURE.md` §5.11-§5.13 — the Room-Architect Designer build this spec's §5 extends, not rewrites.
- `docs/SPEC-18-PLANDEFINITION-AUTHORING-VIA-YAML-PIPELINE.md` — §3.3's `definitionCanonical` support and §4's `CarePlan.instantiatesCanonical` fit are both new work on top of this pipeline, not yet built there either.
- `docs/SPEC-05-DATA-TIER-AND-ABDM-BOUNDARY.md`, `docs/SPEC-19-LOCAL-FIRST-LOCAL-SERVER-AND-FEDERATED-MODES.md` — §6's storage-tier confirmation.

## 9. Recommended sequencing

Given §5's native step-list editor was already recommended independent of this spec, and this spec's real new content is the Speciality-Room dimension itself: (1) reconcile the specialty catalog (§7's first open question — small, unblocks everything else naming a specialty consistently); (2) build the native step-list editor scoped to a single room's own leaf steps first (no `definitionCanonical` yet — proves the "hand-authoring YAML is too hard" fix independent of composition); (3) add `definitionCanonical` composition once a real second shared sub-flow (e.g. Onboarding) exists to compose against, not speculatively; (4) Task persistence (SPEC-16 §6 step 1 / decision #1) for the affiliate-fulfillment question, on its own track — already independently justified, doesn't block 1-3. EpisodeOfCare/CarePlan (§4) rides on the SAME Task-persistence work as (4) — real, but not its own separate sequencing slot; it's what Task persistence unlocks for the longitudinal-care case specifically, once built.
