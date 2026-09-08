# Specification 19: Local-First Architecture — Local/Server and Federated Modes, Conflict Resolution, and the FHIR-Driven Dynamic Workflow Engine

## 1. Objective

Formalize the principles this session proposed — before any functional feature work resumes — for how the app behaves across its two data modes: **Mode 1 (Local + Server)** and **Mode 2 (Federated, peer-to-peer with other users)**. Reconcile the proposal against what's actually built today (§2), correct a few claims that don't match the real code, then specify: the parallel-state mode architecture (§5), the Federation Actor (§6), conflict detection and the resolution sandbox (§7-8), and how the already-built `PlanDefinition`/Task runtime (SPEC-13, SPEC-18) becomes the **data-driven** engine spanning Hospital Administration, Clinical Workflow, and User/Security Management (§9-10) rather than three separate hand-built machines. Nothing in §5-11 is built yet — this is the formalization pass the user explicitly asked for before building resumes. The sequencing after this (ai-engine/sandbox redesign, Cübo SPEC-06/07, the 3-pane surface SPEC-15/16/17, data-layer cleanup, then rollout to Hospital/Provider/Patient journeys) is captured in §12.

## 2. Verified against real code — corrections to the starting proposal

The proposal was accurate in its overall shape but stated several things as if already true, or used terminology that doesn't match this codebase. Checked directly against the real installed packages and source, not assumed:

- **Storage is `localStorage`, not IndexedDB.** `clinux-frontend/src/data/collectionFactory.js` wraps `@tanstack/vue-db`'s `localStorageCollectionOptions` — every collection (`formData.js`, `taskActorSnapshots.js`, `encounterDocs.js`, etc.) is backed by `localStorage`. `@tanstack/db` and `@tanstack/vue-db` (the installed versions) export exactly three collection-storage options: `localOnlyCollectionOptions`, `localStorageCollectionOptions`, `liveQueryCollectionOptions` (the last is a derived/computed view over other collections, not its own storage backend) — no built-in IndexedDB option ships. §4 addresses whether to change this.
- **TanStack Query is a real, installed dependency (`@tanstack/vue-query@^5.101.4`) — but its current real usage is narrow.** `VueQueryPlugin` is installed globally in `main.js`; `useInfiniteQuery`/`useQueryClient` are used in exactly one place, `ActiveSessionsLanding.vue`, for cursor pagination. It is not today the general "IndexedDB & SQLite bridge" the proposal describes. This corrects the `clinux-tanstack-query-graphql-evaluation` memory's blanket "don't adopt" verdict, which was accurate for bulk clinical data (still true — see §3) but is now stale as an absolute statement given this one real usage and given the actual API surface (Task/PlanDefinition routes, encounter routes) has grown since that evaluation.
- **The FHIR-R4 `Task.status` value set is 12 codes, not 4.** Verified directly against the vendored `@smile-cdr/fhirts` R4 types (`node_modules/@smile-cdr/fhirts/src/FHIR-R4/classes/task.ts`, confirmed R4 — not R5 — is the version `tools/clinixflow.config.json` actually targets): `draft | requested | received | accepted | rejected | ready | cancelled | in-progress | on-hold | failed | completed | entered-in-error`. The already-built, already-tested `planDefinitionRunner.js` (SPEC-13) uses its own internal 4-state region vocabulary — `pending → ready → active → done` — which is **not** the same vocabulary. §9 resolves this with a mapping, not a rename.
- **There is real, working prior art for both the "actor model for federation" and "additive sync" principles — this isn't greenfield.** `collectionFactory.js` already does exactly the "additive, callers don't change" pattern the proposal asks for: every collection stays `localStorage`-backed as before, and *also* gets mirrored to/from a Tauri LAN shared server once `ensureSharedModeDetected()` confirms one is reachable (`sharedServerSync.js`, `clinux-mobile-sync-multiuser-video-roadmap` memory) — poll-based merge, with a local-push grace window used to avoid the one real race condition found live. This is Mode 2's simplest case (server-mediated LAN sync) already shipped, and the pattern — a background sync layer wired onto an unchanged Collection interface — is the direct model for the Federation Actor in §6. Separately, `clinuxflow-api/src/durable-objects/ChatSignalingRoom.js` is a real, live-tested WebRTC signaling relay (`clinux-p2p-user-chat-feature` memory) — but it is deliberately capped at exactly 2 sockets and stores nothing (`state.storage` is never called; its own comment says "nothing stored in the interim... at the transport level"). That's correct for ephemeral chat text and wrong for federated *data records*, which need at least enough durability to compare a local version against an incoming one. §6 reuses its signaling/negotiation half, not its statelessness.
- **`buildPlanDefinitionRunnerMachine` is already domain-agnostic.** It takes a `PlanDefinition` and compiles a parallel-region machine from whatever `action`/`relatedAction` entries it contains — nothing about it assumes clinical content. §10's "one Dynamic Core Machine across three domains" principle is therefore already structurally true of the runtime; what's actually missing is (a) real `PlanDefinition`s authored for the Hospital-Admin and Security domains (only Facility/Provider/Front-Desk/Consultation/Checkout rooms and the auth loop exist today) and (b) `ActivityDefinition` support, which does not exist anywhere in the current runtime.

## 3. Layering: Pinia / local collections / TanStack Query / XState

Confirmed and adopted, with the roles stated precisely against what each piece actually is in this codebase:

- **Pinia** (`stores/auth.js`, `stores/onboarding.js`) stays exactly what it already is — light, synchronous client config (active session, selected document id, UI toggles). It already does not mirror database content; no change needed here, just keep it that way as new stores are added.
- **The local-first store is TanStack DB's collections**, not a separate "IndexedDB layer" conceptually distinct from Pinia — `formData.js` and friends *are* the single source of truth for data entities, exactly as principle 4 requires. XState must not duplicate this — verified already true: `planDefinitionRunner.js`'s machine `context` holds only `error_<actionId>` strings, never a copy of any record.
- **TanStack Query's real, correct role going forward**: reading from the real server API (SPEC-13's Task/PlanDefinition endpoints, `/api/encounters/*`) in Server mode — pagination, staleness, retry/backoff over an actual CRUD surface, which now genuinely exists where it didn't at the time of the earlier evaluation. It is **not** the Federation Actor's transport (peers aren't a REST API to page over) and it does not replace the local collections as the source of truth — a `useQuery` result gets written *into* the relevant local collection on success, the same as any other write, so every component keeps reading from one place regardless of where the data came from. This is principle 5's "unified transformer" idea, generalized to the server case too, not just the federated one.
- **XState is the orchestrator only** — confirmed as the already-established pattern (`workflowRuntime.js`'s own header comment: "no UI component ever calls `.send()` on an actor directly"). It holds mode/connectivity/UI-state status, invokes services, and routes; it never holds a dataset.

## 4. Storage backend: migrate to IndexedDB, scoped and additive

**Status: built and shipped for the AI Engine sandbox collections only** (`aiEnginePatients`, `aiEngineStaff`, `aiEngineEncounters`) — explicit user instruction: do this one component first, defer every other collection until this spec's build-out (§12) is complete. The other ~40 collections (`formData.js`, `encounterDocs.js`, `taskActorSnapshots.js`, etc.) stay on `collectionFactory.js`'s `localStorage` backend for now — see §4.2's TODO.

Reasoning for AI Engine specifically, not a blanket rewrite: `localStorage`'s ~5-10MB synchronous quota was an acceptable constraint for most of the app's record shapes, but AI Engine (`AiEngineSandbox.vue`) is the one place that plausibly generates enough bulk synthetic clinical records to matter against that quota, and it's already named in §12 as the first of the "2 untouched components" to redesign. It's also the natural place to prove out the backend §7's conflict-resolution sandbox will need — holding both a local and an incoming conflicting version of a record at once (worst-case roughly doubling footprint for records under dispute) is exactly the kind of load `localStorage`'s synchronous, small-quota API is the wrong tool for.

**§4.1 Implementation, verified.** Since neither `@tanstack/db` nor `@tanstack/vue-db` ships an IndexedDB collection option (§2), built a custom one: `clinux-frontend/src/data/indexedDbCollectionFactory.js` — `indexedDbCollectionOptions()`/`createIndexedDbCollection()`, implementing the exact same `CollectionConfig` protocol `localStorageCollectionOptions` uses (read directly from `@tanstack/db`'s own TypeScript source before writing this, not guessed): a `sync(params)` function calling `begin()/write()/commit()/markReady()` to hydrate, plus `onInsert`/`onUpdate`/`onDelete` handlers that persist a mutation then re-confirm it through that same channel. Storage engine is `idb-keyval` (not raw IndexedDB or Dexie — a single ~1kb dependency, no query layer needed for a get/put/delete-by-key access pattern). Same `createLocalCollection()`-shaped call, including the identical shared-LAN-server additive-sync wiring (`ensureSharedModeDetected`/`wireSharedSync` — storage backend is orthogonal to that layer) — every caller (`useLiveQuery`, every `.insert()`/`.update()`/`.delete()` call site, `main.js`'s `Promise.all(collections.map(c => c.preload()))`) is unchanged, the same "additive, callers don't change" discipline `sharedServerSync.js` already established.

**Two real bugs found and fixed via failing tests, not assumed correct:**
- `idb-keyval`'s `createStore(dbName, storeName)` only creates `storeName` on that database's *first-ever* open (`indexedDB.open(dbName)` with no version argument never re-fires `onupgradeneeded` for an already-existing database) — the original design (one shared `dbName` across all AI Engine collections, one store per collection) silently left every collection but the first with a missing object store, caught by a real `NotFoundError` in a second-collection test, not hypothesized. Fixed by giving each collection its own fully independent database (`${dbNamePrefix}:${storageKey}`), sidestepping multi-store creation entirely.
- Used `crypto.randomUUID()` directly at first; caught before it shipped that `@tanstack/db` itself exports `safeRandomUUID()` specifically because `crypto.randomUUID` is `undefined` in non-secure contexts (a LAN IP over plain HTTP) — exactly the Tauri LAN-shared-server scenario this app already supports (`clinux-mobile-sync-multiuser-video-roadmap` memory). Switched to the exported helper instead of reinventing or ignoring that gap.

Cross-tab sync uses `BroadcastChannel` per collection (IndexedDB writes fire no native `storage` event the way `localStorage` does) — real-tested with two independent collection instances on the same key in one process (Node's `BroadcastChannel` is a native global, confirmed), not simulated.

Testing needed its own real fix: this repo's test suite runs under plain Node with no DOM (confirmed — no `jsdom`/`happy-dom` dependency anywhere, `localStorageCollectionOptions` falls back to its own documented in-memory shim under Node). `idb-keyval` has no such fallback, so `fake-indexeddb` was added as a dev dependency and imported (`fake-indexeddb/auto`) at the top of `indexedDbCollectionFactory.test.js` only — vitest isolates globals per test file, confirmed the other 122 tests are unaffected, no global `vitest.config.js` change needed. 6 new tests (insert/update/delete persistence verified by reopening a second collection instance on the same key, real cross-tab BroadcastChannel propagation, missing-storageKey rejection). `clinux-frontend` 122/122 passing, build clean.

**§4.2 TODO — explicitly deferred, not forgotten**: migrate the remaining ~40 `collectionFactory.js`-backed collections (`formData.js`, `encounterDocs.js`, `taskActorSnapshots.js`, `users.js`, etc.) to `indexedDbCollectionFactory.js`, plus the existing-`localStorage`-data migration-on-first-load step (read the old key, write to the new store, leave the old key alone rather than deleting it) neither of which is needed yet since AI Engine's collections had no prior `localStorage` data worth preserving. Per explicit instruction: **do this only after the rest of this spec's build-out (§12) is complete and stable** — Hospital/Provider/Patient journeys keep using `localStorage` via `collectionFactory.js` unchanged until then.

## 5. The two-mode parallel machine (State Isolation)

Adopted as proposed, structured as XState `type: 'parallel'` regions — the exact mechanism `buildPlanDefinitionRunnerMachine` already uses for independent action regions, so this isn't a new pattern for the codebase, just a new machine using it:

```
type: 'parallel'
regions:
  mode:          local_and_server | federated
  connectivity:  online | offline
  interaction:   viewing | editing
```

`mode` and `connectivity` are genuinely orthogonal — a clinic can be in Federated mode while offline (peers unreachable, falls back to local-only until reconnect) or in Local+Server mode while offline (works exactly as it already does today, nothing new). `interaction` (viewing/editing) is the UI-state region the proposal named — it exists to gate whether a conflict (§7) can even be detected against a record ("has anyone got unsaved local edits on this record right now"), not to duplicate what any individual form component already tracks internally.

Mode transitions are **explicit, not automatic** — no auto-detecting "other users are nearby, switch to federated" — matching the already-shipped LAN-sync UX pattern (`ClinicHome.vue`'s profile-menu sync toggle, `clinux-mobile-sync-multiuser-video-roadmap` memory) where the user explicitly chooses local-only vs. server-synced. Entering `federated` spawns the Federation Actor (§6); leaving it stops that actor cleanly — this is principle 2, stated as a real `invoke`/`stop` lifecycle rather than a manual boolean flag, the same discipline `planDefinitionRunner.js`'s `invoke`/`onDone`/`onError` already established for register/login/change-password's real async side effects.

## 6. Actor Model for Federation

A **Federation Actor**, spawned on entering `mode: federated`, generalizing `ChatSignalingRoom.js`'s signaling half (§2) rather than reusing it unchanged:

- **Signaling/negotiation**: reuse the SDP-offer/ICE-candidate relay pattern directly — it's already correct and already live-tested for exactly this job (peer discovery + WebRTC handshake). The 2-socket cap needs lifting for N-way federation (a room with 3+ clinicians), which changes the "who initiates" glare-avoidance logic (§2's `wasAlreadyOnePresent` check) into something that needs a real topology decision — a mesh (every peer connects to every peer, simplest, fine at clinic scale — a handful of concurrent users, not hundreds) is the right default rather than introducing an SFU for this.
- **Data channel, not chat channel**: unlike `ChatSignalingRoom`, the Federation Actor's RTCPeerConnection DataChannel carries actual record deltas (Task.status changes, Observation mutations — exactly the two examples the proposal names), which means it needs the durability `ChatSignalingRoom` deliberately opts out of. A peer that reconnects after a drop needs to catch up on what it missed — the Federation Actor, not the signaling relay, is responsible for that catch-up (comparing local `meta.versionId`s against what peers broadcast on rejoin, §11), not the relay server itself.
- **Traffic-cop routing (principle 3)**: components never know whether a read/write is being served locally, from the server, or from a peer — they call the same collection methods regardless. The Federation Actor's only job on the write path is: receive a peer's delta over the DataChannel → run it through the transformer (principle 5, §3) → hand it to the *same* local collection `.update()`/`.insert()` call every other write path uses → the collection's own change-detection (already wired for the LAN-sync case) decides whether that's a clean merge or a conflict (§7).

## 7. Conflict detection and the resolution sandbox

Today's only conflict-avoidance mechanism (`sharedServerSync.js`'s local-push grace window) is a **timing heuristic** — "don't overwrite a record I just pushed for the next N seconds" — good enough to avoid the one race condition found live, but it is not field-level conflict *detection*, and it silently resolves in favor of whichever side wins the timing window rather than ever surfacing a real conflict to a human. What's being formalized here is a genuine escalation path for when two sides both hold real, different, uncommitted changes to the same record — which the LAN-sync case mostly avoids by design (single clinic, short poll interval) but Federated mode (independent peers, no shared server clock) cannot assume away.

**Detection**: a conflict exists when an incoming record (server pull or peer delta) and the local record have both changed since their last common `meta.versionId` — not merely "the incoming one is newer," which is what a naive last-write-wins check does today.

**The sandbox lifecycle**, as a child machine invoked per conflicted record (not a global mode — multiple records can be independently in conflict at once):

```
detected → locked → comparing → resolved
```

- `locked`: the record is frozen for further local edits the instant a conflict is detected — the proposal's "prevents impossible states" framing, made concrete: no write can land on a record while its own conflict is still open, enforced by the machine (a write attempt while `locked` is a guarded no-op, not a race to lose).
- `comparing`: both versions are held side by side (this is what makes the IndexedDB migration in §4 worth doing now rather than later) and surfaced in the UI (§7.1).
- `resolved`: only reachable via an explicit `RESOLVE` event carrying the user's actual field-by-field choices — never automatic, matching the proposal's "user explicitly reviews... and signs off."

### 7.1 The comparison view — correcting one framing

The proposal's "LHC-Forms provides specific hooks for this" overstates what LForms ships: there is no built-in side-by-side/compare mode in the vendored `lforms` package. What's real and usable is that every `Questionnaire.item` the form was built from carries a stable `linkId` (verified already — this is exactly how `local-extractor.js` and `LhcFormHost`'s existing `scrollToLinkId` prop both key off `linkId` today), so a comparison view is buildable as app-level UI: iterate the same Questionnaire's `linkId`s, look up each one's value in both the local and incoming `QuestionnaireResponse`, and render a per-field radio choice ("keep mine" / "take theirs" / edit a merged value) wherever they differ, skipping fields that already agree. This is new UI work, not a hook LForms exposes — stated accurately so it isn't assumed to be less effort than it is.

## 8. FHIR audit trail extension

Before a `RESOLVE_CONFLICT` event reaches the collection, the losing side's value is appended as an audit extension on the resolved resource — never silently discarded, per the proposal's HIPAA/GDPR framing. New extension, under the same canonical namespace `condition-types.json`/`specialities.json` already established (`http://yaxb.ai/clinixkernel/`, confirmed the real convention in use, not invented fresh):

```
http://yaxb.ai/clinixkernel/StructureDefinition/conflict-resolution-audit
  extension: overriddenValue     (the losing side's raw value)
  extension: overriddenSource    ('local' | 'peer' | 'server')
  extension: resolvedBy          (Reference(Practitioner) — the account that signed off)
  extension: resolvedAt          (dateTime)
  extension: priorVersionId      (the meta.versionId this override superseded)
```

One extension entry per overridden field, appended to the field's own element where FHIR's extension model allows it (or to the resource root with a `linkId`-equivalent path marker where it doesn't) — not built yet; needs one real worked example (a Practitioner name conflict is the smallest realistic case) before generalizing the shape further.

## 9. The FHIR Workflow triad as the Dynamic Core Machine's data source

Confirmed as the right decomposition, and it's the one SPEC-13 already committed to conceptually — this section makes the Definitional/Request/Event split explicit as the thing the Dynamic Core Machine actually reads at runtime, and resolves the one real vocabulary gap from §2:

- **Definitional** (`PlanDefinition`, `ActivityDefinition`): the blueprint. `PlanDefinition` authoring already works end to end (SPEC-18). `ActivityDefinition` — the proposal's "mandatory forms, conditional routing rules" at the level of one individual action rather than a whole plan — does not exist in the runtime yet; this is real, not-yet-started work, needed before an action can carry its own required-form/routing metadata independent of the plan around it.
- **Request** (`Task`, `ServiceRequest`): the active instance. **Adopted mapping**, resolving §2's vocabulary gap without touching the tested `planDefinitionRunner.js` internals — a pure translation function, not a rename:

  | internal region state | real `Task.status` |
  |---|---|
  | `pending` (dependencies unmet) | `draft` |
  | `ready` (eligible, unclaimed) | `requested` |
  | `active` (being worked / invoke running) | `in-progress` |
  | `done` | `completed` |

  Six real `Task.status` codes are **not** modeled by this mapping, named explicitly rather than silently dropped: `received`/`accepted`/`rejected` (an explicit claim/hand-off step between "requested" and "in-progress" — matters once `Task.owner` assignment from multiple eligible people is real, not yet built); `on-hold` (a genuine clinical pause, different from a failed `invoke` — e.g. "waiting on labs," today indistinguishable from any other in-progress action); `failed` (today an `invoke` failure returns straight to `ready` for an immediate retry — a reasonable UX choice, but it means the FHIR record never persists a `failed` Task, so "this step failed twice before succeeding" isn't auditable); `cancelled`/`entered-in-error` (no cancellation flow exists in the runtime at all). Priority for closing these, if asked to build: accept/reject (needed for real multi-actor ownership) and on-hold (real clinical need) before the other three.
- **Event** (`Encounter`, `Procedure`, `Observation`): the audited reality of what happened — the `QuestionnaireResponse` payload each room already captures, per SPEC-12/SPEC-13's existing design. Unchanged by this spec; noted here only to complete the triad.

## 10. Domain mapping (Hospital Administration / Clinical Workflow / User & Security Management)

Per §2, the "one engine, three domains" principle is already structurally satisfied by `buildPlanDefinitionRunnerMachine` — it compiles whatever `PlanDefinition` it's handed, with no domain-specific code inside it today (confirmed by reading it: the only per-action specialization is the optional `services[action.id]` invoke hook, itself domain-agnostic). What's actually needed per domain is **content**, not new runtime code:

- **Hospital Administration** — bed allocation, financial verification checkpoints, clearance parameters: no `PlanDefinition` authored for this domain yet.
- **Clinical Workflow** — the five rooms (Facility/Provider Registration, Front Desk, Consultation Desk, Checkout) already exist as a real draft `PlanDefinition` (SPEC-18's `workflow-definition-v1.draft.yaml`); triage/post-op/labs-specific `ActivityDefinition` paths depend on §9's `ActivityDefinition` work landing first.
- **User & Security Management** — the register/login/change-password loop (SPEC-13's validated closed loop) is this domain's first real instance already. SMART-on-FHIR scope checks and programmatic sign-off nodes are not built — today's `requireUser()`/`requirePaidTier()` middleware (`clinuxflow-api/src/index.js`) is a simpler, real, working authorization layer that predates this spec and should be the thing a Security-domain `PlanDefinition`'s conditions eventually call into (via the same `condition-type` `CodeSystem` mechanism SPEC-18 built for `specialty-equals`), not replaced.

## 11. Unified sync strategy (offline / server / federated)

Three real gaps, named precisely against what exists today rather than assumed built:

- **Offline**: no formal `syncStatus: 'pending'` field exists on records today — `sharedServerSync.js`'s grace-window tracking is in-memory only (lost on reload) and specific to the LAN-sync case. A real, persisted `syncStatus` per record (written at save time, cleared once a server/peer ack is confirmed) is needed for the Federated case, where "was this actually seen by anyone else yet" can't be inferred from a short in-memory timer.
- **Server sync (SQLite/D1)**: today's shared-server sync is poll-and-merge, not a `meta.versionId` differential check — every collection's `pollOnce()` pulls the full current row and merges it, rather than comparing version identifiers first. A real `versionId` comparison (skip the merge entirely when the local and remote versionId already agree, and route to §7's conflict sandbox specifically when they've diverged with both sides having local changes) is new work, and is what actually makes conflict *detection* (§7) possible instead of the current implicit last-write-wins.
- **Federated mesh**: `Task.status` updates and `Observation` mutations streaming over the Federation Actor's DataChannel, bypassing the backend entirely for same-room-right-now updates — this is genuinely new; nothing today streams field-level deltas over WebRTC, only whole-record poll/merge (LAN case) or opaque signaling blobs (chat case).

## 12. Build order and sequencing

This spec is the formalization step the user asked for before resuming feature work. Explicit sequencing for what comes after, as stated:

1. **This spec** — principles, layering, conflict-resolution design, FHIR triad mapping. Formalized here; nothing in §4-11 built yet.
2. **AI Engine + Sandbox redesign** — the two components explicitly named as untouched by the new principles — rebuilt against this architecture, with Cübo as the agentic harness (SPEC-06/07) and the 3-pane interaction surface (SPEC-15/16/17).

   **§2.1, first slice done: the 3-pane shell.** Scoped, per explicit instruction, to layout only — classification/dispatch logic (SPEC-06/07) and rendering refinements (SPEC-15 §4-7: URL pop-out, markdown narrative, section-level cards, retiring `formSlotEngine.js`) all deliberately deferred, not built here. `src/pages/AiEngine.vue` (new) replaces the former `AiEngineSandbox.vue` tab-content component with SPEC-15 §3's real three-pane layout: **left** — a flat category nav (Patients/Staff/Encounters, live counts) that is honestly *not* SPEC-16's real Task/notebook navigator (that needs real persisted `Task`/`PlanDefinition` instances, still not built, and this sandbox has no workflow graph at all — a flat list is the closest useful analog for a context with nothing to sequence); **middle** — a dedicated `<Cubo>` instance, reusing the already-existing `.cubo-inline-host` confined-pane CSS pattern Front Desk/Consultation Desk/Checkout/Designer already established (verified — no changes needed to `Cubo.vue` itself, purely a matter of where the wrapper div is placed); **right** — the existing AG Grid data table for whichever category is selected, one grid visible at a time instead of the old three-stacked-accordion layout.

   **Corrected mid-build, per explicit instruction: Designer and AI Engine are separate pages, no dependency** — reverses the earlier AI Engine+Designer merge (`clinux-ai-engine-designer-merge-tanstack-table` memory note) rather than nesting the new 3-pane shell inside Designer.vue's existing tab structure as first attempted. `/ai-engine` is a real route again (`src/pages/AiEngine.vue`, no `requiresAuth` — sandbox data stays reachable by anyone, matching its prior behavior) instead of redirecting to `/designer`. `Designer.vue` reverted to Forms-Library-only content (its own tab-switcher, `primaryTab` ref, and cross-page Cübo-emit-forwarding via a template ref all removed) and now requires auth (`meta: { requiresAuth: true }`), since Sandbox Data's "safe for an unauthenticated visitor" case moved out with it — verified live that hitting `/designer` unauthenticated now correctly redirects to `/` (via `resolveGuard`), and that `/ai-engine`'s three nav categories, Cübo's confined-pane expansion, and AG Grid rendering all work with zero console errors (Playwright, real dev server, screenshots taken).
3. **Data-layer, forms-library, and PlanDefinition cleanup** — removing redundancies accumulated across the sessions that built the current system (the two-parallel-capture-surface pattern flagged early in this project's defect review is the known example; a fuller redundancy audit is part of this step, not assumed already scoped).
4. **Stabilize and test** steps 2-3 before step 5.
5. **Roll out to Hospital, Provider, and Patient journeys** — only after 2-4 are stable, per the user's explicit sequencing.

## 13. Relationship to existing specs

- `docs/SPEC-05-DATA-TIER-AND-ABDM-BOUNDARY.md` — the free/local-or-LAN vs. paid/cloud tier line this spec's Local+Server-vs-Federated mode split sits on top of; §10's Security-domain note on `requirePaidTier()` connects directly to SPEC-05's tier boundary.
- `docs/SPEC-13-FHIR-WORKFLOW-DOCUMENTS-AND-CONFORMANCE.md` §2 — the runtime this spec's §9 maps onto; no changes to `planDefinitionRunner.js`'s internals, only a translation layer.
- `docs/SPEC-18-PLANDEFINITION-AUTHORING-VIA-YAML-PIPELINE.md` — the authoring pipeline §10's per-domain `PlanDefinition` content will be authored through, unchanged.
- `clinux-mobile-sync-multiuser-video-roadmap` / `clinux-p2p-user-chat-feature` memories — the real, shipped prior art §2/§6/§7 build on and generalize, not replace.
