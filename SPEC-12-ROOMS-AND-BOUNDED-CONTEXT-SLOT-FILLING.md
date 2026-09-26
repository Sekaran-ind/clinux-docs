# SPEC-12: Rooms and Bounded-Context Slot Filling

| | |
|---|---|
| **Status** | Partially superseded. §4.1–§4.4 are the standing design principles for capture (mostly not built as shared components). §4.6 and §4.7 are superseded by SPEC-13 §2. §4.5's room model is superseded by SPEC-23. The four defects in §3 are resolved or reduced. |
| **Last reviewed** | 2026-09-26 |
| **Related** | SPEC-13, SPEC-14, SPEC-15, SPEC-23, SPEC-24 |

## 1. Idea

Each stable working surface (a **room**: Front Desk, Consultation, Checkout, and the registration
entities) is a bounded context anchored to its own compiled document. Cübo's slot filling works
**inside** that context, scoped to its fields, rather than guessing across every visible field on
a page. Rooms stay real, addressable, resumable pages; Cübo is another way in, not a replacement.

## 2. Terminology

A YAML form's `composition:` array is an authoring-time list of resource blocks, not a FHIR
`Composition`. What is canonical per room is a compiled `Questionnaire` plus its
`QuestionnaireResponse`. `Questionnaire.item` nests recursively, which is what the section model
below relies on. A real FHIR `Composition` is produced only later, as a document (SPEC-13 §5).

## 3. The four defects that motivated this spec, and their status

| # | Defect | Status |
|---|---|---|
| 1 | ClinicHome's care-team section showed members with a blank name and photo | **Fixed.** `buildClinicProfile()` read a `staff_name` field that no longer existed after the Practitioner split; it now joins first and last name. |
| 2 | A stray, unstyled placeholder below the "Meet Our Experts" card | Not re-verified since the ClinicHome redesign (PR #15); treat as closed unless seen again. |
| 3 | Two unreconciled capture surfaces for the Hospital Profile | **Reduced to one guided surface plus one deliberate bulk editor.** `HospitalOnboarding.vue` and the chat slice are deleted; `Onboarding.vue`'s hosts are the guided surface. Designer's Provider card still opens a whole-document LHC-Forms editor for bulk edits (kept on purpose, SPEC-22 §5.4). |
| 4 | LGD state/district codes captured as free text | **Resolved in capture**: `FacilityHfrPanel` uses live cascading pickers from the gateway (`/hfr/master/lgd/*`). The YAML field stays `TextInput` because it stores the resolved code. |

## 4. Design principles

### 4.1 Read live, never materialize a copy
No page holds its own derived copy of another room's data; it reads live and filters for what it
may see. `publishedClinic` as a computed over `formData` is the reference instance. The single
legitimate exception is an attested document, which is a deliberate snapshot (SPEC-13 §5.5).

### 4.2 Bounded context with a scope stack
Chat writes go only to the active section. Out-of-scope input gets "that belongs in X, switch?",
never a silent write elsewhere. Reads may cross the whole document and other rooms. Because
sections nest, scope is a stack (Branch → Staff → one practitioner's services).
**Not built.** Today Cübo's slot filling matches across whatever fields the page registered.

### 4.3 One resolution primitive, three uses
A single component for:
- **Coded-value confirmation**: a closed-vocabulary field never accepts raw text as final; typed
  text filters the option list; cascades (state → district → sub-district) are linked.
- **Ambiguous-slot disambiguation**: show candidate fields with context and require one target.
  One fact is written once and referenced elsewhere.
- **Record-instance disambiguation**: pick which of several repeating instances (which staff
  member, which consent) a write targets.

**Partly built, not shared.** The rule is followed per component (controlled selects in the hosts,
cascading LGD pickers, suggestion highlights that need acceptance) but there is no single shared
component.

### 4.4 Measure the resolution outcomes
Log the unambiguous-fill rate, coded-confirm abandon rate, disambiguation skew (the same field
repeatedly resolved away from the default means the schema boundary is wrong), and "unactionable
here" recurrence, deduplicated by distinct session and clustered by meaning (several different
users wanting the same missing field is a schema signal; one user retrying is not). **Not built.**

### 4.5 Rooms as documents
Each room gets its own document with nested sections; cross-room data is referenced, not copied.
Superseded in its specifics by SPEC-23: rooms are specialty × service-category instances of a
fixed mechanism, and registration entities are not workflow rooms at all.

### 4.6 Room lifecycle
Superseded by SPEC-13 §2.2: lifecycle binds to `Task.status`, not `QuestionnaireResponse.status`.
Keep "draft" (lifecycle) separate from "local only" (data residency); they are different
properties.

### 4.7 Preconditions between rooms
Superseded by SPEC-13 §2.1: `PlanDefinition.action.relatedAction`, not an invented precondition
syntax.

### 4.8 Cübo as orchestrator, not sole surface
Once scope and resolution are solid, Cübo becomes the default way in, but dense surfaces (grids,
forms) stay first-class for work that chat is wrong for, such as a front-desk clerk during a
walk-in rush.

## 5. Related specs

SPEC-06 (the harness that consumes §4.2–§4.4), SPEC-13 (sequencing and documents), SPEC-14
(chat-first capture for guided flows), SPEC-23 (room model), SPEC-24 (registration capture).

## 6. Consolidation note

The duplicated Hospital Profile surface (§3, defect 3) was the most consequential finding: an
assistant built on two competing sources of truth inherits that ambiguity. It was resolved
before any bounded-context work started, as recommended.

## 7. Remaining build order

1. Shared resolution component (§4.3), starting with coded-value confirmation.
2. Scope stack for Cübo slot filling (§4.2).
3. Instrument outcomes (§4.4) on the component's events.
4. Specialty rooms per SPEC-23.

## 8. Open questions

- Practical nesting-depth limits for LHC-Forms rendering (the compiler supports any depth).
- Tier-gating implications of Cübo as the central interface.
