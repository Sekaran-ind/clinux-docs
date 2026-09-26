# SPEC-13: FHIR Workflow, Documents and Conformance

| | |
|---|---|
| **Status** | Partially built. Built: extraction hardening (§5.3), the PlanDefinition → XState runtime (§2, §2.3), document Bundle assembly (§5.4), local StructureDefinitions (via SPEC-24). Not built: Task-status lifecycle for rooms (§2.2), registration-authored plans and credentialing Tasks (§4), attestation, CapabilityStatement (§6). |
| **Last reviewed** | 2026-09-26 |
| **Code** | `clinux-frontend/src/workflow/{planDefinitionRunner,workflowRuntime,entryPlanDefinition,authPlanDefinition}.js`, `clinuxflow-api/src/lib/shared/{local-extractor,composition-assembler}.js`, `POST /api/workflow/{extract,assemble-document}` |
| **Related** | SPEC-12, SPEC-14 (runtime choice), SPEC-18 (authoring), SPEC-19 §9 (status mapping), SPEC-24 (profiles), SPEC-25 (persistence) |

## 1. Scope

SPEC-12 governs capture **within** a room. This spec governs what happens **between** rooms
(sequencing, assignment, credentialing: FHIR Workflow), **after** a room (the authorized record:
FHIR Documents), and how the resulting surface is **declared and checked** (conformance).

## 2. FHIR Workflow mapping

| Tier | Resource | Role |
|---|---|---|
| Definition | `PlanDefinition` | The protocol (for example "Standard OPD Visit"); its actions are rooms or steps |
| Definition | `Questionnaire` | What a step captures; referenced from `action.definitionCanonical` rather than adding `ActivityDefinition` |
| Request | `Task` | One running step: `focus` (the Questionnaire), `for` (the subject), `owner`, `status` |
| Event | `QuestionnaireResponse` | What was captured; `basedOn` the Task, and the Task's `output` points back |

**Built**: `planDefinitionRunner.js` compiles a PlanDefinition into an XState v5 `parallel`
machine, one region per action with states `pending → ready → active → done`. Features:
- `relatedAction` becomes an `always` guard with `stateIn`, AND-gated over all targets.
  **The relationship code is not distinguished**: `after-start`, `after-end` and the rest all
  wait for the target's `done`.
- `services[actionId]` lets `active` invoke a real async function (`fromPromise`, `onDone`,
  `onError`); the resolved value is kept in context as `result_<actionId>`, errors as
  `error_<actionId>`.
- `action.repeatable` makes `done` accept `FOCUS` again instead of being final (login, logout,
  change password can recur; register cannot).

`workflowRuntime.js` is the only thing that sends events to actors (one RxJS bus, one dispatcher,
factory-built so tests don't share state). It persists through `actor.subscribe()` (which captures
asynchronous invoke results), writes an audit entry per region transition, discards incompatible
snapshots, and offers `onActionDone(planId, actionId, cb)` for cross-plan triggers.

Live plans: `ENTRY_PLAN_DEFINITION` (register, login, forgot password, change password, logout;
SPEC-20) and `AUTH_PLAN_DEFINITION` (the closed-loop test fixture). The outpatient visit is not
yet a plan (SPEC-04 §4).

### 2.1 Sequencing and specialty conditioning
Use `PlanDefinition.action.relatedAction` for ordering (replacing SPEC-12 §4.7) and
`action.condition` for conditional inclusion, such as a specialty-specific step (closing SPEC-12's
specialty question). The runtime implements `relatedAction` only; `condition` is authored
(SPEC-18) but never evaluated.

### 2.2 Room lifecycle on Task.status
Bind a room's lifecycle to `Task.status`, not `QuestionnaireResponse.status`. SPEC-19 §9 maps the
runtime's four states onto the real R4 codes (`pending→draft`, `ready→requested`,
`active→in-progress`, `done→completed`); SPEC-25's `ClinuxFlowTask` profile and D1 mirror use that
mapping. `received/accepted/rejected`, `on-hold`, `failed`, `cancelled` and `entered-in-error` are
not modeled.

### 2.3 The closed loop
Captured response → Task completes → dependent actions re-evaluate → the next Task becomes
`ready` → Cübo shifts scope or notifies its owner → repeat, without anyone clicking "next".
**Built for the runtime half**: `always` transitions re-check on every change, so there is no
separate re-evaluation step, and `onActionDone` carries the result across plans (SPEC-21 §5 uses
it for role-based suggestions). **Not built**: owner notification and Task ownership (§4.3).

## 3. Cübo threads and the workflow tiers

A thread row is `{id, category, title, pinned, timestamp, encounterId, messages}`. A thread with no
`encounterId` works at the Definition tier (registration, authoring); one with an `encounterId`
works inside one encounter. SPEC-16 later inverts this so the Task is primary and the thread is
derived; that is still design.

## 4. Registration as workflow authoring and credentialing

### 4.1 The hospital authors its visit protocol
Once a facility declares its services (`HealthcareService`), a PlanDefinition should be
**templated** from its specialties and confirmed by the admin, not hand-authored. **Not built**,
and SPEC-23 later ruled that registration itself is not a tracked workflow; the templating idea
now belongs to Speciality Rooms.

### 4.2 Practitioner credentialing as a Task
Fine-grained privileges belong on `PractitionerRole.healthcareService`. Granting them is a `Task`
whose `for` is the Practitioner; completing it updates the role. Same machinery as patient work,
different subject. **Not built.**

### 4.3 Primary and supporting roles
Co-staffing is two linked Tasks: a primary (`owner` = the credentialed practitioner) and a
supporting one (`partOf` the primary), both with the same `focus`. At document time the primary
owner becomes `Composition.attester`, supporting participants become `Composition.author`.
**Not built.**

### 4.4 Open governance question
May `Task.owner` (who did the work, for example a resident) differ from `Composition.attester`
(who is legally responsible, for example the attending)? It should be allowed, but this is a
clinical-governance decision, not a technical one.

### 4.5 Worked example
A resident and an attending co-staff a cardiology consultation. The plan (cardiology-conditioned)
creates a Consultation Task. The resident's role doesn't yet cover independent cardiology consults,
so they get the supporting Task. On completion, extraction runs and the Composition is assembled
with the attending as attester and both as authors.

## 5. FHIR Documents

### 5.1 Why Composition is not redundant
`Questionnaire.item` nesting structures capture. A `Composition` is a different thing: an
authored, attestable, versioned document assembled from already-extracted resources.

### 5.2 Extraction
`ComprehensiveLocalExtractor.extract(questionnaire, response, options)` reads each item's
`definition`, walks the response, and produces linked resources (Organization, Location,
HealthcareService, Practitioner, PractitionerRole, OrganizationAffiliation, Consent, Patient,
Encounter, Observation, Condition, PlanDefinition and more), including references such as
`PractitionerRole.practitioner` and `Location.managingOrganization`.

### 5.3 Hardening (all built and regression-tested)
1. **No silent drops.** An answered item with no `definition`, or a malformed one, adds to the
   result's `.warnings` array (not serialized, so API responses are unchanged).
2. **Stable identity.** Pluggable `identityResolvers`; Practitioner resolves by name plus
   facility (not truly unique; a stronger anchor such as an HPR ID is the upgrade path). Generic
   saves avoid duplicates by requiring the caller's own record id (SPEC-24's resource API).
3. **Repeating groups no longer collapse.** The resource cache was keyed by type alone, so N
   staff produced one Practitioner. Groups are now classified as separate-instance or
   array-field automatically from path structure.
4. **Genuinely nested groups.** The extractor walks a stack of enclosing group frames, and nested
   instance counters are keyed by the full frame path (SPEC-18 §7 step 3, SPEC-22 §3).
5. **FHIR array cardinality.** `FHIR_ARRAY_PATHS` makes `telecom`, `identifier`, `address`,
   `type` and similar real arrays, gives each linkId its own slot (so `staff_phone` and
   `staff_email` both survive), and keeps every MultiSelect answer.
6. **Extensions carry URLs.** ABDM fields are distinct extensions, not anonymous entries.

**Still open**: `ContactPoint.system` is never set, because no YAML field supplies it. The
profiles slice `telecom` by `system`, so a mobile-number slice can never be satisfied and
`Patient` conformance cannot reach `valid: true` (SPEC-24 §7). Fix with Hidden-field defaults per
telecom field.

### 5.4 Document assembly
`FhirDocumentAssembler.assemble()` (`POST /api/workflow/assemble-document`) builds a
`Bundle{type:"document"}` with the Composition first and every referenced resource present as a
`urn:uuid:` entry. Sections default to one per resource type. `status` defaults to `preliminary`
and `Composition.type` is plain text (no invented LOINC code). **Nothing in the frontend calls it
yet**, nothing signs it, and nothing stores or sends it anywhere.

### 5.5 The one exception to "never copy"
A `final`, attested Composition is a deliberate snapshot; that is what signing means. Amendments
create a new version (`relatesTo: replaces`), which only works if resource identities are stable
(§5.3 item 2).

## 6. Conformance

### 6.1 CapabilityStatement
FHIR's machine-readable declaration of supported resources, interactions, profiles and search
parameters. `kind`: `requirements` (a target), `capability` (self-attested), or `instance` (one
deployment).

### 6.2 Regulatory weight
It becomes load-bearing when M2/M3 certification is pursued (SPEC-11 §5). Before that it is
internal documentation and a test anchor.

### 6.3 Value now
A precise declaration of what the system supports, and something to validate real instances
against systematically rather than by screenshot review.

### 6.4 Recommendation
Author a `kind: requirements` statement listing Questionnaire, QuestionnaireResponse, Task,
PlanDefinition, Composition, Organization, Practitioner, PractitionerRole, HealthcareService,
Location, OrganizationAffiliation, Patient, Encounter, Observation, Condition, MedicationRequest,
with `supportedProfile` pointing at the seven local StructureDefinitions. **Not built.**

### 6.5 StructureDefinitions
The deferred "graph → StructureDefinition" item was delivered differently: SPEC-24 hand-authored
seven profiles grounded in the ABDM specs, plus a validator. The dictionary and the profiles are
separate artifacts; generating one from the other is not planned.

## 7. Related specs

SPEC-12 (§4.6/§4.7 superseded here), SPEC-06 (the harness this loop gives something to dispatch
against), SPEC-18 (authoring), SPEC-19 §9 (status mapping), SPEC-24 (profiles), SPEC-25
(persistence).

## 8. Build order and state

1. ~~Harden extraction~~: done (§5.3).
2. ~~PlanDefinition compiler and runtime~~: done (§2), including invoke services and a real
   change-password backend built for the auth closed loop.
3. Task.status as room lifecycle (§2.2): mapping and persistence done (SPEC-25); no room uses it.
4. `relatedAction` relationship semantics and `condition` evaluation (§2.1): not built.
5. Registration-templated plans and credentialing Tasks (§4.1, §4.2): not built; §4.1 re-homed to
   SPEC-23.
6. Primary and supporting Tasks (§4.3): not built.
7. ~~Composition assembly~~: done (§5.4); attestation and storage not built.
8. CapabilityStatement (§6.4): not built.

## 9. Open questions

- Is `Questionnaire` a valid `action.definitionCanonical` target in R4 for this use? Not verified.
- Identity resolvers for Location, HealthcareService and PractitionerRole.
- `Task.owner` versus `Composition.attester` (§4.4).
- `ContactPoint.system` (§5.3).
- `relatedAction.relationship` semantics (§2).
- Automatic reference linking for `HealthcareService` once credentialing links roles to services.
- Resolved: side-effect wiring (`invoke`), cross-plan triggering (`onActionDone`), and capturing
  the login result (`result_<actionId>`).
