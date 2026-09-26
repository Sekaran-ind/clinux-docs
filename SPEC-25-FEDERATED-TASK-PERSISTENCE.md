# Specification 25: Federated Task Persistence

## 1. Objective

SPEC-22's own "sole large remaining item from the original four foundational decisions" — real,
durable persistence for the Task/PlanDefinition runtime, so a journey survives a reload, a device
switch, and a handoff between roles, not just a single tab's lifetime. The user's own framing named
the real constraints this has to be designed against together, not sequentially: the journey is
**federated across roles** (staff, affiliates, patient-mediated), **access tiers** (free/local vs.
paid/cloud — SPEC-05), and **devices** (this app is local-first by default, with Cloudflare D1
already durable for Facility's and Provider's own entities).

This spec is design-only, per explicit instruction — no code in this pass.

## 2. A real terminology clash, resolved up front

"Task" means two different things across this codebase's own history, and the request as phrased
("Task persistence for graph definition") sits right on top of the clash:

- **SPEC-24's `GraphDefinition`/`next-best-action.js`** (control layer — Facility/Provider/
  Affiliate×2/Patient) is deliberately, repeatedly-reaffirmed **stateless**: "done" is "passes
  validation," recomputed fresh on every call, no persisted actor, no PlanDefinition. SPEC-23's own
  correction was explicit and hard-won: *"state machine only for clinical journeys"* — onboarding
  registration is not workflow-tracked, on purpose.
- **SPEC-13 §2's runtime** (`clinux-frontend/src/workflow/workflowRuntime.js` +
  `planDefinitionRunner.js`) is a REAL, already-built, already-live XState-driven Task/PlanDefinition
  engine — register/login/change-password (SPEC-20's entry flow) is its first real case. It already
  has an injectable persistence hook (`data/collections/taskActorSnapshots.js`, confirmed **local-
  only** — a plain TanStack DB localStorage collection, no durable mirror at all today) and an
  audit log (`auditLog`, confirmed **in-memory only** — not persisted anywhere, lost on reload).

**Resolution**: Task persistence belongs to the RUNTIME layer's already-real engine, not a new
tracking mechanism bolted onto GraphDefinition. SPEC-23's correction stands unchanged — the control
layer stays stateless. What connects them: a Task's own record can **cite** a GraphDefinition link
(`sourceResourceId`/`linkId` from a `nextBestActions()` call) as *why* it exists — real traceability
("this Task was opened because next-best-action surfaced 'affiliation-from-facility'") — without
the control layer itself tracking anything. GraphDefinition stays the suggestion source; the
runtime engine is the only thing that remembers what happened next.

## 3. What already exists — the real starting point

Not building from scratch. Already real and working:

- `workflowRuntime.js`: `registerPlan()` (starts/resumes an XState actor per `planId`),
  `onActionDone()` (SPEC-21 §5 cross-plan triggering, live-verified), `diffAndLogTransitions()`
  (per-action status diffing → `auditLog` entries), snapshot-compatibility discard-and-restart on a
  changed PlanDefinition shape (a real bug this session's predecessor found and fixed).
- `taskActorSnapshots.js`: one local row per `planId` (`{planId, snapshot, updatedAt}`), via
  `createLocalCollection` — the SAME `collectionFactory.js` primitive `formData.js`/every other
  collection uses (localStorage + additive LAN sync via `sharedServerSync.js` when a Tauri shared
  server is reachable).
- The proven durable-mirror pattern, twice already: `provider_composition` (migrations/0005) and
  `encounter_documents` (migrations/0006) — one row per clinic/encounter, `PUT` on save, `GET` on
  load, "eventually consistent, best-effort." `encounter_assignments` (same migration) additionally
  proves a real **lock** primitive: atomic acquire via a conditional `ON CONFLICT ... WHERE`, TTL
  expiry as the crashed-device safety net, `renew`/`release` — already live, already tested.

Every piece Task persistence needs a version of already exists for a sibling concern. This spec is
mostly about generalizing and connecting them, not inventing new mechanisms.

## 4. The real gap in the existing snapshot model, found while designing this

`taskActorSnapshots.js` keys by `planId` alone and treats the actor's `getPersistedSnapshot()` as
one opaque blob to replace wholesale. That's fine for one device driving one plan start-to-finish
(SPEC-20's entry flow). It breaks for federation:

- **Two devices/roles advancing the same journey concurrently** — a whole-snapshot replace is a
  last-write-wins overwrite of an XState actor's internal state, not a merge. Unlike a flat FHIR
  document (where `mergeGroupResponseItems`' per-group replace is safe because groups are
  independent), a PlanDefinition actor's snapshot is one interdependent state tree — two divergent
  snapshots can't be reconciled after the fact without real risk of silently losing one side's
  transition.
- **The audit log isn't persisted at all** — every transition/blocked-attempt entry SPEC-20 §4
  designed for is gone on reload today.

**Resolution, reusing what's already proven rather than inventing conflict resolution**: a running
Task instance is **single-writer**, enforced the same way `encounter_assignments`'s `kind:'lock'`
already enforces "only one device actively works this encounter stage at a time" — generalize that
same table/primitive to lock a `(taskId)` for whichever device/actor is actively driving it, TTL-
renewed while active, released on completion or handoff. A device without the lock can still
**read** the durable mirror (see §6) to show current status, it just can't advance the actor until
it acquires the lock — the exact same shape Front Desk/Checkout's shared-worklist locking already
has live users depending on. No new conflict-resolution algorithm needed; the audit log itself
becomes the second half of the fix — append-only, never replaced (see §6), so even a lock-bypassed
double-write is at least fully reconstructable after the fact instead of silently lost.

## 5. Data model — a real Task, not an invented shape

FHIR's own `Task` resource is the real, standard fit ("a task to be performed, or a record that one
was performed") — same "real resource, not invented" discipline SPEC-24 held StructureDefinition/
GraphDefinition to. One `Task` per running (or completed) PlanDefinition **action** instance, not
one per whole plan — a plan with several actions is several Tasks sharing a `groupIdentifier` (real
FHIR field for exactly this: tasks that belong to the same overall request). Fields this app
actually needs, nothing speculative:

| Field | Source | Purpose |
|---|---|---|
| `id` | generated | this Task instance |
| `groupIdentifier` | `planId` | which running plan this belongs to |
| `status` | XState action status (`ready`/`active`/`done`/etc., mapped to Task's own real value set: `requested`/`in-progress`/`completed`/…) | current state, mirrors `snap.value[actionId]` |
| `for` | the real FHIR resource this Task concerns, when there is one (an `Organization`/`PractitionerRole`/`Encounter` id) | ties a Task back to a real entity, not just an abstract action name |
| `owner` | `{accountId}` of whoever currently holds the lock (§4), or last held it | federation across roles — who is/was doing this |
| `businessStatus.text` | free text | human-readable ("Waiting on ABHA verification") |
| `authoredOn` / `lastModified` | timestamps | |
| `note` | one entry per `auditLog` transition — see §6 | the durable audit trail SPEC-20 §4 asked for |
| `reasonReference` | when this Task was opened in response to a real `next-best-action.js` candidate (§2) — `{linkId, sourceResourceId}` from that call | traceability into the control layer, without the control layer tracking anything itself |

`clinuxflow-api/data/structure-definitions/` gets one more real profile,
`ClinuxFlowTask.json`, same differential-constraint shape every other profile there already uses —
not a special case.

## 6. Storage architecture — local-first primary, paid-tier durable mirror, same shape twice already

**Local (every tier, every device, always)**: replace `taskActorSnapshots.js`'s localStorage
collection with an **IndexedDB** one via `indexedDbCollectionFactory.js` (SPEC-19 §4's own
precedent) — not a new backend, the second real use of a protocol built to be reused, and the right
call here specifically because an append-only audit log genuinely can outgrow localStorage's 5-10MB
quota the way AI Engine's own bulk records did. Two collections, not one, matching §4's "snapshot
replace vs. audit append" split:
  - `taskActorSnapshots` (unchanged shape, just a different backend) — current XState snapshot,
    one row per `planId`.
  - `taskAuditLog` (new) — **append-only**, one row per audit entry (`{taskId, planId, actionId,
    from, to, at, accountId}`), never updated in place. This is what makes the single-writer lock's
    "at least reconstructable" property in §4 real: even a rejected/late write still lands as its
    own entry instead of overwriting anything.

**Durable mirror (paid tier only, `requirePaidTier()`-gated — same binary gate every cloud-durable
route in this app already uses, SPEC-05 §6)**: two new D1 tables, deliberately shaped like
`provider_composition`/`encounter_documents`'s own precedent, not a new pattern:

```sql
CREATE TABLE task_snapshots (
    plan_id TEXT PRIMARY KEY,
    clinic_id TEXT NOT NULL,
    snapshot TEXT NOT NULL,       -- JSON, the same actor.getPersistedSnapshot() blob
    updated_at TEXT NOT NULL DEFAULT (datetime('now'))
);

CREATE TABLE task_audit_log (
    id TEXT PRIMARY KEY,
    plan_id TEXT NOT NULL,
    task_id TEXT NOT NULL,
    account_id TEXT NOT NULL REFERENCES accounts(id),
    action_id TEXT NOT NULL,
    from_status TEXT,
    to_status TEXT,
    created_at TEXT NOT NULL DEFAULT (datetime('now'))
);
CREATE INDEX idx_task_audit_log_plan ON task_audit_log (plan_id, created_at);
```

`task_locks` is not a new table — it's `encounter_assignments` (migrations/0006) with its own
`kind`/`stage` CHECK constraints widened to cover a `planId` the same way they cover an
`encounterId` today (or, if that reads as overloading one table too far once actually scoped,
a sibling table with the identical column shape and the identical `acquireLock`/`renewLock`/
`releaseLock` functions from `encounter-coordination-db.js` copied, not redesigned).

Sync direction matches `encounterCoordination.js`'s own established shape exactly: explicit
`pushTaskSnapshot(planId, snapshot)` / `fetchTaskSnapshot(planId)` functions, called by
`workflowRuntime.js`'s own `persistSnapshot`/`loadPersistedSnapshot` injection points (already
built to accept exactly this kind of swap-in) — local write always happens first and always
succeeds; the durable push is best-effort on top, same "local-first collection is source of truth
between syncs" contract every other paid-tier mirror in this app already has. Free tier gets the
existing LAN-only `sharedServerSync.js` path for these two new collections for free, zero new code
— that wiring is already additive and automatic for anything built on `createLocalCollection`/
`indexedDbCollectionFactory.js`.

## 7. Federation across roles — visibility, not new access control

No new authorization model needed — reuse the exact staff-or-linked-affiliate trust boundary
`POST /api/encounters/:id/assign` and `GET /api/chat/signal` already enforce (same-clinic staff, or
an affiliate already linked via `facility_affiliates`). A Task's `owner` can be reassigned the same
way `EncounterCoordinationDb.assignEncounter` already reassigns an encounter — this is not a new
concept, just the same one, applied to a Task instead of an encounter. Patient-mediated journeys
(SPEC-21 §6's own resolution: Patient never holds a login) never own a Task directly — a Task
`for`-referencing a Patient resource is always `owner`-ed by the staff member acting on their
behalf, matching how Patient capture already works everywhere else in this app.

## 8. What this deliberately does not change

- GraphDefinition/`next-best-action.js`/`conformance-validator.js` — untouched, stay pure and
  stateless. §2's `reasonReference` is a one-way citation, never a write-back.
- No PlanDefinition/state-machine tracking is added for Facility/Provider/Affiliate/Patient
  registration itself — SPEC-23's correction is not being reversed by this spec.
- Free tier's actual behavior doesn't change — it already gets LAN sync for local-first collections
  today; it just also now gets it for these two new ones, automatically.

## 9. Module placement (this session's own NIST ZTA reorganization)

Squarely runtime layer, both repos — no new boundary decision needed, the reorg already drew this
line: `clinuxflow-api/src/routes/runtime.js` (+ `src/lib/runtime/task-db.js`, sibling to
`encounter-coordination-db.js`), `clinux-frontend/src/data/runtime/taskSync.js` (the push/fetch
functions §6 names), `src/workflow/` unchanged in shape — `workflowRuntime.js`'s own injection
points are the seam, not a rewrite.

## 10. Build sequencing (next pass, not this one)

1. `ClinuxFlowTask.json` StructureDefinition (§5) — grounds the shape before any code.
2. `taskAuditLog` local collection (IndexedDB) + wire `workflowRuntime.js`'s `logAudit()` to write
   through it instead of the in-memory array only — real, immediately-useful even before any
   durable mirror exists (survives a reload today).
3. Migration: `task_snapshots` + `task_audit_log` tables, plus the lock primitive (§6).
4. `clinuxflow-api` routes: GET/PUT task snapshot, POST audit append, lock acquire/renew/release —
   copy `encounter-coordination-db.js`'s own functions, don't redesign them.
5. `taskSync.js` push/fetch + wire into `workflowRuntime.js`'s injection points.
6. Live-verify: two simulated devices/roles trading the same `planId`'s lock, confirm the audit log
   reconstructs the real sequence across both.
