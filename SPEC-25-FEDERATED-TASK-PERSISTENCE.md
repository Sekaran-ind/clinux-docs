# SPEC-25: Federated Task Persistence

| | |
|---|---|
| **Status** | Built (local IndexedDB persistence, D1 mirror, append-only audit, single-writer locks, the `ClinuxFlowTask` profile). **Security gap**: the D1 reads and upserts are not clinic-scoped (§6.1). Per-action FHIR `Task` records are not produced (§5). |
| **Last reviewed** | 2026-09-26 |
| **Code** | `clinux-frontend/src/data/collections/{taskActorSnapshots,taskAuditLog}.js`, `src/data/runtime/taskSync.js`, `src/workflow/workflowRuntime.js`, `src/stores/entryWorkflow.js`; `clinuxflow-api/migrations/0010_add_task_persistence.sql`, `src/lib/runtime/task-db.js`, `src/routes/runtime.js` (`/api/tasks/*`), `data/structure-definitions/ClinuxFlowTask.json` |
| **Related** | SPEC-13 §2, SPEC-19 §4 and §9, SPEC-22 §2, SPEC-24 §3 |

## 1. Purpose

Durable persistence for the workflow runtime, so a journey survives a reload, a device switch and
a handoff between roles, across tiers (free/local versus paid/cloud) and across devices.

## 2. Two meanings of "Task", kept apart

- The **control layer** (SPEC-24's GraphDefinition and `next-best-action.js`) is stateless on
  purpose: "done" means "passes validation", recomputed on every call. Registration is not
  workflow-tracked (SPEC-23).
- The **runtime layer** (`workflowRuntime.js` + `planDefinitionRunner.js`) is the real XState
  engine. Persistence belongs here.

They connect one way: a runtime Task may cite the next-best-action suggestion that caused it
(`reasonReference` → `{linkId, sourceResourceId}`), without the control layer tracking anything.

## 3. Starting point

`registerPlan`, `onActionDone`, audit diffing and snapshot-compatibility checks already existed.
Snapshots were local-only (localStorage), the audit log in-memory only. The durable-mirror pattern
(`provider_composition`, `encounter_documents`) and the TTL lock primitive (`encounter_assignments`)
already existed for sibling concerns; this spec generalizes them.

## 4. Why single-writer, not merge

An XState snapshot is one interdependent state tree. Two divergent snapshots cannot be merged
safely; last-write-wins would silently lose a transition. So a running plan instance has **one
writer at a time**, enforced by a TTL lock (acquire, renew while active, release on completion or
handoff). Devices without the lock can read status but not advance the actor. The audit log is
append-only, so even a write that bypassed the lock remains reconstructable.

## 5. Data model

`ClinuxFlowTask` profiles FHIR `Task` for one action instance: `groupIdentifier` (the plan),
`status` (mapped from runtime states per SPEC-19 §9), `for`, `owner`, `businessStatus.text`,
`authoredOn`/`lastModified`, `note` (audit entries), and `reasonReference` (§2).

**As built, the runtime does not emit `Task` resources.** It persists the actor snapshot and one
audit row per transition. The profile is the target shape for when a worklist (SPEC-22 D1) needs
queryable Tasks.

## 6. Storage

**Local, every tier**: both collections use IndexedDB (`indexedDbCollectionFactory.js`):
- `taskActorSnapshots`: the current snapshot, one row per plan key, replaced on each change.
- `taskAuditLog` (`cf_task_audit_log_v1`): append-only, `{taskId, planId, actionId, from, to, at,
  accountId}`, never updated.

**Cloud mirror, paid tier** (`requirePaidTier()`), migration 0010:

| Table | Key | Purpose |
|---|---|---|
| `task_snapshots` | `plan_id` | Latest snapshot JSON, `clinic_id`, `updated_at` |
| `task_audit_log` | `id` | Append-only rows: plan, task, clinic, account, action, from, to |
| `task_locks` | `plan_id` | Single-writer lock: holder, `expires_at` (a sibling table with the same shape as `encounter_assignments`, rather than overloading that table) |

Routes: `GET/PUT /api/tasks/:planId/snapshot`, `POST/GET /api/tasks/:planId/audit`,
`POST /api/tasks/:planId/lock` (plus `/renew`, `/release`), `GET /api/tasks/:planId/lock`.
Client: `taskSync.js` (`pushTaskSnapshot`, `fetchTaskSnapshot`, `pushTaskAuditEntry`,
`fetchTaskAuditLog`, lock functions), called from `workflowRuntime.js`'s persistence hooks. The
local write always happens first; the cloud push is best-effort and a 403 means "stay local".

**Durable key**: plans with a shared template id would collide across accounts (a real bug found in
verification), so `entryWorkflow.js` uses `${planId}:${accountId}` as the durable key. Anonymous
(pre-login) transitions stay local.

### 6.1 Security gap: no tenant check on reads or upserts
`getSnapshot` and `listAuditLog` select by `plan_id` alone, and `upsertSnapshot` overwrites
`clinic_id` on conflict. A paid account that knows another account's plan key (it embeds the
account id, which linked affiliates can see) can read or replace that account's workflow snapshot
and read its audit trail. Fix: scope reads by the caller's clinic (and account, for per-account
plans) and reject upserts onto rows owned by another clinic. See SPEC-01 §10 item 0.

## 7. Federation across roles

No new authorization model: the same staff-or-linked-affiliate boundary as encounter assignment.
Ownership moves the way encounter assignment does. Patients never own Tasks; a Task `for` a
Patient is owned by the staff member acting for them.

## 8. What this does not change

GraphDefinition, next-best-action and the validator stay pure. Registration stays untracked. Free
tier behavior is unchanged, apart from the two new collections also syncing over the LAN.

## 9. Placement

Runtime layer in both repos (`src/routes/runtime.js`, `src/lib/runtime/task-db.js`;
`src/data/runtime/taskSync.js`).

## 10. Build record

1. ~~`ClinuxFlowTask` profile~~: done.
2. ~~Local IndexedDB audit log, wired into `logAudit()`~~: done. This also fixed a page-reload
   resumability race and needed a test-environment IndexedDB polyfill.
3. ~~Migration 0010~~: done.
4. ~~Routes, modeled on the encounter coordination accessors~~: done.
5. ~~`taskSync.js` wired into the runtime~~: done.
6. Live verification: done against a real `wrangler dev` + local D1 with curl (snapshot
   round-trip, audit append and list, lock acquire, contention and release). The two-browser,
   two-role handoff through the UI has not been run.
7. New: fix §6.1.
