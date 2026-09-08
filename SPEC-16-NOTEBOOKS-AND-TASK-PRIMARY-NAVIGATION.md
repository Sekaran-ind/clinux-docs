# Specification 16: Notebooks & Task-Primary Navigation

## 1. Objective

Revise `docs/SPEC-13-FHIR-WORKFLOW-DOCUMENTS-AND-CONFORMANCE.md` §3: `Task` (the FHIR Workflow resource) is the primary structure Cübo's left pane navigates, not an independently-bookkept conversation thread with Workflow data attached as metadata. Introduce the **notebook** as the grouping container between "the whole navigator" and "one Task node." Define ephemeral, unanchored chat as the explicit exception outside this structure. Nothing built yet.

## 2. The inversion: Task primary, thread derived

SPEC-13 §3 mapped `chatThreads.js`'s existing shape (`category` + optional `encounterId`) onto Definition vs. Request/Event tiers, treating the thread as the primary object and Workflow state as metadata riding along. This spec inverts that: **the Task is primary; a "thread" is the conversation log that accumulates while working a given Task node**, not a separately-maintained list that has to stay in sync with the Task graph. Opening a Task in the left-pane navigator and opening its thread are the same action — there is one structure, not two that could drift apart.

## 3. Notebook: the container between "whole tree" and "one Task"

A notebook is anchored to a `PlanDefinition` instance, identified by an anchor-id. Four notebook types, matching what's already been established in this design line:

| Notebook type | Anchor-id | Lifecycle | Subtasks |
|---|---|---|---|
| Facility | the facility itself | singleton, perpetual | initial onboarding + later updates (e.g. replacing SPEC-14 §7's demo LGD codes with real ones) |
| Encounter | encounter-id | one per patient visit, completes when Checkout does | Front-Desk, Consultation-Desk, Checkout Tasks (SPEC-13 §2.1's `relatedAction` sequence) |
| Affiliation/Roster | a specific Practitioner-Facility or Facility-Facility relationship | perpetual for that relationship's lifetime | registration, credentialing changes, eventual offboarding — the roster-lifecycle machines, structurally different in shape from an Encounter's process/pipeline machine |
| EpisodeOfCare | the episode itself (`EpisodeOfCare.diagnosis.condition` — the specific condition being longitudinally managed, e.g. a high-risk pregnancy, a rehab program) | bounded but long-running — spans multiple Encounters, ends when the condition/program does, not perpetual like Affiliation/Roster and not single-visit-bounded like Encounter | the individual Encounter notebooks it groups (`Encounter.episodeOfCare`, real FHIR field), plus a governing `CarePlan` (goals/interventions/meds/nutrition) — the CarePlan is the one subtask here NOT anchored to a PlanDefinition the way the other three notebook types' own workflows are: it's patient-specific content, though `CarePlan.instantiatesCanonical` CAN reference a PlanDefinition-authored protocol template the same SPEC-18 pipeline produces (see SPEC-23 §4 for the full reasoning, including why EpisodeOfCare→CarePlan isn't a direct FHIR field — the real link is via shared Condition/Patient) |

## 4. Untethered chat is ephemeral, deliberately outside the notebook structure

No anchor-id, nothing to file it under — by design. **Ephemeral means not persisted** — discarded at session end, not kept in `chatThreads.js`. This is a real behavior change from what's shipped today: today's `category`-only threads (e.g. `ai-engine`, visible in the earlier Designer screenshots) *do* persist. Chosen deliberately here, not preserved silently, for two reasons: keeps the notebook structure clean of un-groupable entries, and avoids retaining casual/unimportant queries by default in a clinical app's data footprint.

## 5. Left pane = one structure, not two

Confirms `docs/SPEC-15-CUBO-UNIFIED-INTERACTION-SURFACE.md` §3's left pane is a single navigable tree: notebook type as the top grouping (reusing `chatThreads.js`'s existing `category` field's role, now backed by real anchor-id/notebook identity rather than a label string), Task hierarchy nested within each notebook, node status shown inline — this *is* the journey map from earlier discussion, not a separate widget alongside it.

## 6. Build order

1. Design the Task/notebook persistence model itself — the actual gap: no `Task`/`PlanDefinition` resource is persisted anywhere yet (SPEC-13 §2, still true as of this spec). Notebooks can't be built before Tasks are real, persisted objects. **The `PlanDefinition` half of this is now answered**: `docs/SPEC-18-PLANDEFINITION-AUTHORING-VIA-YAML-PIPELINE.md` — authored via the existing YAML/Questionnaire/extraction pipeline, not a new tool. `Task` persistence itself remains open.
2. Revise `chatThreads.js`'s schema: replace/supplement `category`+`encounterId` with notebook-anchor-id + Task-id references; add an explicit non-persisted path for untethered chat.
3. Build the left-pane navigator UI against the new model.
4. Migrate SPEC-14's built Hospital Registration flow to be a Facility notebook's Task, not a standalone thread — same rework SPEC-15 §8 step 1 already requires, same underlying fix.

## 7. Relationship to existing specs

- `docs/SPEC-13-FHIR-WORKFLOW-DOCUMENTS-AND-CONFORMANCE.md` §2 — this spec's Task-primary model depends on `Task`/`PlanDefinition` actually being built; not done yet. §3 is superseded by this spec.
- `docs/SPEC-15-CUBO-UNIFIED-INTERACTION-SURFACE.md` — the left pane's UI; this spec is the data model underneath it.
- `docs/SPEC-18-PLANDEFINITION-AUTHORING-VIA-YAML-PIPELINE.md` — answers this spec's §6 step 1 for `PlanDefinition` specifically; `Task` persistence is still this spec's own open item.
- `docs/SPEC-23-SPECIALITY-ROOM-AND-FIXED-ORCHESTRATION-ANCHORS.md` §4 — the EpisodeOfCare notebook type added to §3's table above; also names a second, independent reason (affiliate-fulfilled services) `Task` persistence is needed, beyond this spec's own Notebook motivation.

## 8. Open items

- Whether a mid-encounter roster change (a provider going inactive while a Consultation Task is already assigned to them) needs explicit handling — flagged, not resolved, from the prior session's discussion.
- Exact `chatThreads.js` schema migration path for existing persisted `category`-only threads under the new notebook model.
- Whether ephemeral chat needs any minimal audit trail (even if content isn't retained, should "a conversation happened, at this time" be logged) — a real compliance-adjacent question, not addressed here.
