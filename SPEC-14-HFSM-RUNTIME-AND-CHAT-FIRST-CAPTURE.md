# SPEC-14: HFSM Runtime and Chat-First Capture for Guided Flows

| | |
|---|---|
| **Status** | Runtime decision current (XState + RxJS). The chat-first Hospital Registration slice (§7) was built, then retired and deleted. The scope rules in §4–§6 stand. |
| **Last reviewed** | 2026-09-26 |
| **Code** | `clinux-frontend/src/workflow/{planDefinitionRunner,workflowRuntime}.js`; dependencies `xstate`, `@xstate/vue`, `rxjs` |
| **Related** | SPEC-12 §4.3, SPEC-13 §2, SPEC-15, SPEC-22 §5.14, SPEC-23 |

## 1. Purpose

Choose the technology that executes SPEC-13's PlanDefinition/Task design, and decide where
one-question-at-a-time chat capture is appropriate.

## 2. Runtime: XState + RxJS

- **XState** is the hierarchical state machine: nested and parallel states, guards, actions,
  invoked services, serializable snapshots. That is exactly the statechart formalism needed.
- **RxJS** is event plumbing around it: one subject as the workflow event bus in
  `workflowRuntime.js`. It is the only RxJS use in the app, deliberately modest.
- **ObservableHQ's runtime** was considered and rejected: it is a reactive dataflow graph for
  computation and visualization, with no states, guards or transitions as primitives.
- Pinia keeps general UI state; TanStack DB collections keep data; XState holds only workflow
  status (its context never holds a copy of a record).

A bundle-size and mount-stability check was required before these sat on Cübo's critical path
(Cübo mounts on every page, and an earlier NLP dependency nearly broke mounting). It was never
formally measured; both libraries have run on every page since, with no mount failures observed.

## 3. The HFSM executes SPEC-13, it doesn't replace it

A PlanDefinition compiles into an XState machine, the same relationship YAML has to a
Questionnaire. A running Task corresponds to a persisted actor snapshot, which is what makes an
interrupted step resumable. The FHIR path dictionary and the StructureDefinitions remain the
authority on what fields exist; the machine only sequences.

## 4. Scope: chat-first only for guided flows

One-question-at-a-time capture suits first-time, low-repetition flows. Dense stations (Front
Desk, Consultation, Checkout) keep dense surfaces, because a clerk in a walk-in rush needs fast
entry, not a conversation. The machine exposes "current state and what's needed next" and stays
independent of any renderer.

**Later narrowing (SPEC-22 §5.14, SPEC-23)**: facility, practitioner and patient registration
are not state machines at all. They are pages with stateless conformance checks. Today the only
chat-hosted guided flow is the entry journey (register, login, forgot and change password,
logout; SPEC-20).

## 5. Coded fields never become free text

"One at a time" is about sequencing, not about turning every field into a free-text bubble. A
coded field renders an inline picker or quick replies inside the chat turn (SPEC-12 §4.3).

## 6. What "learning" may and may not change

- **Safe**: personalization within a fixed topology, such as ordering optional steps or surfacing
  shortcuts by role, specialty and history. Role and specialty are **read** from data, not
  learned.
- **Not safe by default**: the topology itself changing from usage data (states, transitions or
  guards added or rerouted automatically). A workflow whose shape drifts silently cannot be
  audited or certified. Usage data may **propose** a structural change for a human to approve,
  never apply one.

## 7. The first slice, and why it was retired

`hospitalRegistrationMachine.js` + `ChatCapture.vue` + `HospitalOnboardingChat.vue` implemented
hospital registration as a hand-authored hierarchical machine with one chat turn per field, RxJS
debounced text commit, and quick-reply chips for coded fields. It also converted the LGD code
fields from free text to pickers with labeled demo options.

It was retired because (a) SPEC-15 found it was a standalone page, not hosted in Cübo; (b) the
generic PlanDefinition runtime replaced hand-authored machines; and (c) SPEC-22 §5.14 and
SPEC-23 then took registration out of workflow tracking altogether. All three files are deleted.
The LGD fix lives on as real cascading pickers in `FacilityHfrPanel`.

## 8. Related specs

SPEC-12 (principles unchanged), SPEC-13 (the resource model this executes), SPEC-06 §7 (the
mount-stability precedent).

## 9. Open items

- Formally measure the bundle and mount cost of `xstate` and `rxjs` if Cübo's first paint regresses.
- Resolved: the general PlanDefinition → XState compiler (SPEC-13 §8 step 2); real LGD master data
  (live gateway lookups).
