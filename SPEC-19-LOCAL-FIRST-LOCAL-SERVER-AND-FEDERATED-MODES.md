# SPEC-19: Local-First Architecture: Local/Server and Federated Modes, Conflicts, and the FHIR Workflow Triad

| | |
|---|---|
| **Status** | Partially built. Built: the layering rules (§3), an IndexedDB collection backend used by five collections (§4), the Task.status mapping (§9, used by SPEC-25). Design only: the mode machine (§5), Federation Actor (§6), conflict sandbox (§7), audit extension (§8), versioned sync (§11). |
| **Last reviewed** | 2026-09-26 |
| **Code** | `clinux-frontend/src/data/{collectionFactory,indexedDbCollectionFactory,sharedServerSync}.js`, `src/data/collections/*.js`, `src-tauri/src/shared_server.rs`, `src/workflow/*` |
| **Related** | SPEC-05, SPEC-13, SPEC-21 §6 (storage lifecycle), SPEC-25 |

## 1. Purpose

Set the rules for two data modes: **Local + Server** (the device, optionally a LAN or cloud
server) and **Federated** (peers exchanging records directly). Specify conflict handling, and
state how the PlanDefinition runtime serves as one engine for several domains.

## 2. Ground truth (checked against the code)

- Most collections use `localStorage` through `collectionFactory.js`
  (`localStorageCollectionOptions`). TanStack DB ships no IndexedDB option; §4 adds one.
- TanStack Query is installed but used in one place (`ActiveSessionsLanding.vue`, cursor
  pagination).
- R4 `Task.status` has 12 codes. The runtime uses four internal states; §9 maps them.
- LAN sync already follows the "additive, callers don't change" rule: collections stay local and
  are mirrored to the Tauri shared server once it is detected (poll-and-merge, with a local-push
  grace window against the one race found live). `ChatSignalingRoom` shows the signaling pattern
  but deliberately stores nothing and allows two sockets.
- `buildPlanDefinitionRunnerMachine` has no domain-specific code, so "one engine, many domains" is
  already structurally true. What's missing is domain content and `ActivityDefinition`.

## 3. Layering

- **Pinia**: light, synchronous UI state (session, selections, toggles). Never mirrors data.
- **TanStack DB collections**: the single source of truth for records.
- **TanStack Query**: reading real server APIs in server mode (pagination, staleness, retry).
  Results are written into local collections, so components read from one place. Not the
  federation transport.
- **XState**: orchestration only. Its context holds status and errors, never a dataset.

## 4. Storage backend: IndexedDB, adopted per collection

**Built.** `indexedDbCollectionFactory.js` implements TanStack DB's `CollectionConfig` protocol
(`sync` with `begin/write/commit/markReady`, plus `onInsert/onUpdate/onDelete`) over `idb-keyval`,
with the same shared-server sync wiring as the localStorage factory. Callers don't change.

In use by: `aiEnginePatients`, `aiEngineStaff`, `aiEngineEncounters` (the bulk-data sandbox, the
first reason to leave localStorage's roughly 5–10 MB synchronous quota), and `taskActorSnapshots`
and `taskAuditLog` (SPEC-25: an append-only log outgrows localStorage).

Two bugs were caught by tests before shipping:
1. `idb-keyval`'s `createStore(db, store)` only creates the store on a database's first open, so
   collections sharing one database lost their stores. Each collection now gets its own database
   (`${prefix}:${storageKey}`).
2. `crypto.randomUUID()` is undefined in non-secure contexts (a LAN IP over plain HTTP); TanStack
   DB's `safeRandomUUID()` is used instead.

Cross-tab updates use a `BroadcastChannel` per collection (IndexedDB fires no `storage` event).
Tests run under Node with `fake-indexeddb`.

**§4.2 Remaining migration (deferred).** Move the remaining localStorage collections (`formData`,
`encounterDocs`, `users`, `chatThreads` and others), with a one-time copy from the old key that
leaves the old key in place. Deliberately postponed until the journeys are stable.

## 5. Mode machine (design)

A parallel machine with independent regions:

```
mode:          local_and_server | federated
connectivity:  online | offline
interaction:   viewing | editing
```

Mode changes are explicit user choices (like today's local-only versus server-sync toggle),
never automatic. Entering `federated` spawns the Federation Actor and leaving stops it.
`interaction` exists so conflict detection knows whether someone holds unsaved edits.

## 6. Federation Actor (design)

- Reuse the SDP/ICE signaling pattern, lifted from two sockets to a small mesh.
- A **data** channel carrying record deltas (for example Task.status and Observation changes).
  Unlike chat, it needs enough durability for a reconnecting peer to catch up by comparing
  `meta.versionId`s.
- Components never know whether data came locally, from a server or from a peer: deltas go
  through the same collection `insert`/`update` calls as every other write.

## 7. Conflict detection and the resolution sandbox (design)

Today's only protection is a timing heuristic (the grace window), which silently resolves in
favor of whichever side wins. Target:
- **Detection**: a conflict exists when both the incoming and the local copy changed since their
  last common `meta.versionId`, not merely when the incoming one is newer.
- **Lifecycle per record**: `detected → locked → comparing → resolved`. While locked, local writes
  are guarded no-ops. `resolved` is reached only by an explicit user choice per field.

### 7.1 Comparison view
LHC-Forms has no built-in compare mode. Because every item has a stable `linkId`, a compare view
is app-level UI: walk the Questionnaire's `linkId`s, show local and incoming values side by side,
offer keep mine, take theirs, or edit, per differing field.

## 8. Audit extension for overridden values (design)

Before a resolution is saved, record the losing value on the resource:

```
http://yaxb.ai/clinixkernel/StructureDefinition/conflict-resolution-audit
  overriddenValue · overriddenSource (local|peer|server) · resolvedBy (Reference(Practitioner))
  resolvedAt · priorVersionId
```

To be proven on one small case (a Practitioner name conflict) before generalizing.

## 9. The FHIR Workflow triad

- **Definition**: `PlanDefinition` (authoring built, SPEC-18); `ActivityDefinition` not built.
- **Request**: `Task`. The runtime-to-FHIR mapping, used by SPEC-25's `ClinuxFlowTask` and the D1
  mirror:

  | Runtime state | `Task.status` |
  |---|---|
  | `pending` | `draft` |
  | `ready` | `requested` |
  | `active` | `in-progress` |
  | `done` | `completed` |

  Not modeled: `received/accepted/rejected` (claiming work), `on-hold` (clinical pause),
  `failed` (a failed invoke returns to `ready`, so failures aren't auditable as Task states),
  `cancelled`, `entered-in-error`. If built, do accept/reject and on-hold first.
- **Event**: `Encounter`, `Observation`, `Procedure`: the extracted results of captured responses.

## 10. Domains

The engine is domain-agnostic; each domain needs content:
- **Hospital administration** (beds, financial checkpoints): nothing authored.
- **Clinical workflow**: the outpatient visit is not yet a plan (SPEC-04 §4); a five-room draft
  exists as a test fixture.
- **User and security management**: the entry plan (SPEC-20) is live. Authorization stays in
  `requireUser()`/`requirePaidTier()`; plan conditions should call into it, not replace it.

## 11. Sync strategy gaps (design)

- **Offline**: no persisted per-record `syncStatus`; the grace window is in memory and lost on
  reload.
- **Server**: poll-and-merge pulls whole rows; no `versionId` comparison, so no real conflict
  detection. The explicit cloud mirrors (SPEC-25, encounters) are last-write-wins, protected by
  single-writer locks rather than merge.
- **Federated**: nothing streams field-level deltas over WebRTC yet.

## 12. Sequencing

1. This spec (principles) — done.
2. AI Engine and sandbox redesign as a three-pane Cübo surface. **Outcome**: `/ai-engine` now
   renders Cübo itself in `THREE_PANE` (SPEC-22 §5.1). The first three-pane sandbox page
   (`AiEngine.vue`) is unreferenced by the router and can be deleted. Designer and the Cübo
   workspace are separate pages.
3. Data-layer and forms-library cleanup — largely done through SPEC-23 and SPEC-24 (duplicate
   capture surfaces removed, CustomFormHost retired).
4. Stabilize — ongoing.
5. Roll out to Hospital, Provider and Patient journeys — done as page journeys (SPEC-24), not as
   PlanDefinitions.

## 13. Related specs

SPEC-05 (tiers under these modes), SPEC-13 §2 (the runtime), SPEC-18 (authoring), SPEC-21 §6 (the
per-record storage lifecycle, to be reconciled with §7 here), SPEC-25 (Task persistence).
