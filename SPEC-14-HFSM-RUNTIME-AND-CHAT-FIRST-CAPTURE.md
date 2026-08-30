# Specification 14: HFSM Runtime & Chat-First Capture for Guided Flows

## 1. Objective

Answer how `docs/SPEC-13-FHIR-WORKFLOW-DOCUMENTS-AND-CONFORMANCE.md` §2's `PlanDefinition`/`Task` design actually executes, and replace the side-drawer capture UI with sequential, one-question-at-a-time chat for guided flows specifically (registration, credentialing) — not everywhere. This is a scope-narrowing spec, not a reset: SPEC-12's bounded-context/modal-resolution design and SPEC-13's resource mapping are unchanged; this spec picks the runtime technology and draws the UI boundary those two left open.

## 2. Runtime: RxJS + XState, not ObservableHQ

**Verified this session**: neither `xstate`, `rxjs`, nor `@observablehq/*` exist in `clinux-frontend/package.json` — clean choice, not a migration.

- **XState** is the HFSM itself — it implements Harel statecharts (nested/hierarchical states, guards, actions, history states), which is the actual formalism "HFSM" names, not something RxJS or ObservableHQ provide. `@xstate/vue` integrates directly alongside the existing Pinia stores (`onboarding.js`, `clinical.js`, `auth.js` are unaffected — Pinia keeps owning general app state; XState owns specifically workflow-sequencing state).
- **RxJS** is the event plumbing around it (chat input streams, debounced matching) — not a state-machine library itself.
- **ObservableHQ's runtime is a reactive dataflow graph for computation/visualization** (cells recomputing on dependency change), not a state-machine tool — no hierarchical states, guards, or transitions as primitives. Ruled out: would mean hand-rolling statechart semantics on a tool designed for a different problem.
- **Checkpoint before either lands on Cübo's critical path**: `docs/SPEC-06-CUBO-AGENTIC-HARNESS.md` §7 already documents a real near-miss in this codebase — a previous NLP dependency's fragile CJS graph nearly broke the app's ability to mount, specifically because Cübo sits on every page. Bundle-size/mount-stability needs verifying for both new dependencies before they're load-bearing on that path, same discipline, not assumed safe by default.

## 3. The HFSM is SPEC-13 §2's execution engine, not a replacement for it

`Task.status` (SPEC-13 §2.2) is already a state value. `PlanDefinition.action.relatedAction` (SPEC-13 §2.1) is already a transition graph. This spec names the technology that executes them: **`PlanDefinition` compiles into an XState machine config**, the same relationship `yaml-to-questionnaire.js` already has compiling YAML into a `Questionnaire` — declarative FHIR resource in, executable machine out. **A `Task` instance corresponds to a persisted machine snapshot.** XState machines are natively serializable, which is the concrete mechanism for the resumable-checkpoint requirement this whole design line has held since its first turn (a room/task must survive being interrupted mid-encounter).

The YAML shard-path graph (`yaml-to-questionnaire.js`'s allowed-paths validation, `abdmSchema.js`'s field/master-data definitions) remains the FHIR-correctness authority — unchanged by this spec. The HFSM is compiled *from* it plus the PlanDefinition's sequencing, not a competing source of truth for what fields exist or what they map to.

## 4. Scope: guided flows only, decided explicitly

Chat-first, one-at-a-time capture replaces the drawer for **Hospital registration and Practitioner registration/credentialing** (SPEC-13 §4) — both genuinely first-time, low-repetition, guided by nature. **Front Desk / Consultation / Checkout keep their existing dense surfaces** (AG Grid, the Forms Library table view) — a front-desk clerk mid-surge needs fast dense entry, not a sequential conversation; this was SPEC-12 §4.8's conclusion already and this spec doesn't reopen it.

Practically: build the HFSM's state/transition logic independent of the chat renderer (it emits current state + what's needed next; the UI is one consumer of that), so extending to a dense-grid surface later doesn't require rebuilding the machine — cheap to do now, not part of this pass's UI scope.

## 5. Modal-resolution primitive survives, changes presentation only

SPEC-12 §4.3's rule — closed-vocabulary/coded fields never accept raw free text as final, always resolve through an explicit picker — holds unchanged. "One at a time" means sequencing (one active question, driven by the machine's current state), not "every field becomes a free-text bubble." A coded-value state renders an inline picker/quick-reply within the chat turn instead of a side-drawer select. This rule exists because of a confirmed, live bug (Defect 4, §7 below); the UI simplification must not reopen it.

## 6. "Learning progressively" — the safe/unsafe split, stated as a hard rule

Two different things were bundled under "learning" and only one is safe to build without a human in the loop:

- **Safe — personalization within a fixed topology.** States/transitions the machine can be in are fixed (compiled from the PlanDefinition); *ordering* of optional slots, which shortcuts surface, adapts per user from role/specialty/history. Most of "skills, speciality, role" isn't something to learn from usage patterns — it should be **read** from data SPEC-13 §4 already designed a place for (`PractitionerRole.healthcareService`, the Wikidata-tagged specialty from SPEC-08 Phase 1), not re-derived. The genuinely new learned layer is narrower than it sounds: ordering/preference, not identity/credentialing.
- **Unsafe — the topology itself changing from usage data.** States/transitions/guards being added, removed, or rerouted autonomously. Rejected as a default: a workflow whose shape silently drifts is very hard to audit or certify, and undermines SPEC-13 §6's `CapabilityStatement` work directly (conformance can't be declared against a moving target). Keep SPEC-12 §4.4's existing pattern instead: usage data can *surface* a suggested structural change, routed to a human for confirmation — never a silent self-modification of the live compiled machine.

## 7. First build slice, built this session

Chose Hospital Registration as the first real HFSM+chat-first flow, on the actual current schema, before generalizing into a `PlanDefinition`→machine compiler (SPEC-13 §8 step 2 remains a larger, later step — this slice hand-authors one machine to prove the pattern first, same "prove small before generalizing" discipline as every prior spec in this series).

- **`hospitalRegistrationMachine.js`** — hierarchical: parent states are `HOSPITAL_FIELDS`' four display sections (Basics, Contact & Address, Facility Type & Ownership, ABDM Location), child states are one per field, sequenced with a `required`-field guard blocking advance. Final state performs the same `groupLinkId`-split-and-save `HospitalOnboarding.vue`'s `saveDrawer()` already does (`withGroupFields`, `saveDataRecord`, `onboarding.publish()`, `auth.updateClinicName()`) — same data path, not a fork.
- **`ChatCapture.vue`** — renders the machine's current field as one chat turn at a time; `select`-type fields render inline quick-reply chips (never free text) per §5; free-text commit is debounced via RxJS (`fromEvent`+`debounceTime`) before advancing — the first genuine use of RxJS in this codebase, deliberately modest rather than forced.
- **Real defect fixed as part of this slice, not deferred**: `abdmSchema.js`'s `hospital_state_lgd_code`/`hospital_district_lgd_code` were plain `type: 'text'` (Defect 4, `docs/SPEC-12-ROOMS-AND-BOUNDED-CONTEXT-SLOT-FILLING.md` §3) — converted to `type: 'select'` with an explicitly-labeled **placeholder/demo option set**, not fabricated real LGD codes. This codebase's own real official LGD master data isn't sourced yet (same gap `abdmSchema.js`'s header comment already flags for its other option lists); inventing plausible-looking numeric codes here would recreate exactly the failure Defect 4 was about — a value that looks authoritative but isn't. Demo values use non-numeric placeholder codes (e.g. `TN-DEMO`) specifically so nothing downstream can mistake them for real LGD codes, with an explicit comment naming the real ABDM LGD master-data endpoint (`clinuxflow-abdm-gateway`'s `/master/...` routes, per `abdmSchema.js`'s existing comment) as the follow-up.
- Wired as an **additive** entry point from `HospitalOnboarding.vue` — the existing drawer flow is untouched and still works; this is a parallel path to prove out, not a replacement shipped without comparison.

## 8. Relationship to existing specs

- `docs/SPEC-12-ROOMS-AND-BOUNDED-CONTEXT-SLOT-FILLING.md` — §4.1-§4.4 (live-view, bounded context, modal primitive, metrics) unchanged; this spec is the runtime + UI-scope decision layered on top.
- `docs/SPEC-13-FHIR-WORKFLOW-DOCUMENTS-AND-CONFORMANCE.md` — §2 gets its execution engine here (§3); §4's registration-as-authoring work is where §4 (this spec) draws the chat-first boundary; §6's `CapabilityStatement` work is why §6 (this spec) treats topology self-modification as unsafe.
- `docs/SPEC-06-CUBO-AGENTIC-HARNESS.md` §7 — the bundle-size/mount-stability precedent §2 (this spec) holds the new dependencies to.

## 9. Open items

- Bundle-size/mount-stability check for `xstate`+`rxjs` on Cübo's critical path — not yet measured.
- The general `PlanDefinition`→XState compiler (SPEC-13 §8 step 2) — this slice hand-authored one machine; generalizing it is still open.
- Wiring the real ABDM LGD master-data endpoint to replace §7's placeholder option set.
- Practitioner registration's credentialing-Task flow (SPEC-13 §4.2) — not built this slice, Hospital registration only.
- Whether `hospital_type`'s free-text-adjacent fields would benefit from the same RxJS-debounced fuzzy-match pattern used for text commit, once there's a second real case to generalize from.
