# SPEC-16: Notebooks and Task-Primary Navigation

| | |
|---|---|
| **Status** | Design. Its blocking prerequisite (persisted Tasks) now exists (SPEC-25), but no notebook, Task-to-thread binding or navigator has been built. Cübo shows a "Notebooks" placeholder. |
| **Last reviewed** | 2026-09-26 |
| **Related** | SPEC-13 §3, SPEC-15 §3, SPEC-18, SPEC-23 §4, SPEC-25 |

## 1. Idea

The FHIR `Task` is the primary structure Cübo's left pane navigates. A "thread" is the
conversation that accumulates while working one Task, not a separately maintained list that has
to stay in sync. A **notebook** groups Tasks under one anchor. Chat with no anchor is ephemeral.

## 2. The inversion

Today a thread is primary (`category` plus an optional `encounterId`) and workflow state rides
along as metadata. Target: open a Task in the navigator and you are in its conversation. One
structure, not two that can drift.

## 3. Notebook types

| Type | Anchor | Lifecycle | Contents |
|---|---|---|---|
| Facility | The facility | Singleton, perpetual | Setup and later updates |
| Encounter | Encounter id | One visit; ends at Checkout | Front Desk, Consultation, Checkout Tasks |
| Affiliation / roster | A practitioner–facility or facility–facility relationship | The relationship's lifetime | Joining, credentialing changes, offboarding |
| EpisodeOfCare | `EpisodeOfCare` for one managed condition (for example a high-risk pregnancy or a rehab programme) | Bounded but long-running; spans many encounters | The encounter notebooks it groups (`Encounter.episodeOfCare`) and a governing `CarePlan`, which may instantiate a PlanDefinition protocol (`CarePlan.instantiatesCanonical`); see SPEC-23 §4 |

Since SPEC-23, facility and roster registration are not tracked workflows, so those two notebook
types would hold conversation history and join or credentialing Tasks, not a registration
checklist.

## 4. Ephemeral chat

Chat with no anchor is not persisted; it is discarded at session end. This is a behavior change
from today, where category-only threads (such as `general`) persist. Reasons: keep notebooks free
of ungroupable entries, and don't retain casual queries in a clinical app by default.

## 5. One left pane, not two widgets

Top level: notebook type (taking over the role of today's `category`, but backed by a real
anchor). Inside: the Task hierarchy with status shown inline. This is the journey map.

## 6. Build order

1. ~~Persist Tasks~~: done (SPEC-25: IndexedDB snapshots and audit, D1 mirror, locks,
   `ClinuxFlowTask` profile). PlanDefinition authoring: done (SPEC-18).
2. Extend `chatThreads.js` rows with a notebook anchor id and a Task id; add a non-persisted path
   for ephemeral chat.
3. Build the navigator in Cübo's left pane (replace the placeholder).
4. The first real notebook will be **Encounter**, once the outpatient visit runs as a
   PlanDefinition (SPEC-04 §4). The Facility registration flow originally planned as the first
   notebook no longer exists (SPEC-14 §7).

## 7. Related specs

SPEC-13 §3 (superseded by this model), SPEC-15 (the UI), SPEC-18 (authoring), SPEC-23 §4 (the
EpisodeOfCare row; affiliate-fulfilled services as a second reason Tasks need owners), SPEC-25
(persistence).

## 8. Open items

- A practitioner going inactive while a Task is assigned to them.
- Migrating existing category-only threads.
- Whether ephemeral chat needs a minimal "a conversation happened at time T" audit entry.
