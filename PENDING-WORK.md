# Pending Work — ClinuxFlow

First written 2026-09-26, when the workspaces moved off the MacBook to the Mac Mini + Windows PC.
Revised the same day after a full spec-versus-code review (see `SPEC-00-INDEX.md`). Test state at
that point: clinux-frontend 351/351, clinuxflow-api 383/383, clinuxflow-abdm-gateway 15/15.

Update this file as items close. One feature per machine, each on its own branch; see
"Working across machines" at the bottom.

---

## 0. Security: fix before any clinic uses the system

Found in the 2026-09-26 review. Details and fix sketches in SPEC-01 §10. BLOG-04 must not be
published until these are closed.

| # | Issue | Where | Severity |
|---|---|---|---|
| S1 | **Cross-tenant reads and writes by id.** `GET/PUT /api/encounters/:id`, `GET/PUT /api/tasks/:planId/snapshot` and `GET /api/tasks/:planId/audit` look up by id with no `clinic_id` condition; upserts overwrite `clinic_id`. `resource_records` upsert also reassigns `clinic_id` (its reads are scoped). Encounter ids are `rec-<timestamp>-<5 chars>`. | clinuxflow-api `src/lib/runtime/{encounter-coordination-db,task-db}.js`, `src/lib/control/resource-records-db.js`, `src/routes/runtime.js` | High (clinical data) |
| S2 | **Encounter video join has no clinic check** and no paid gate: any signed-in account that knows an encounter id gets a RealtimeKit token for that consultation. | `POST /api/realtime/join` | High |
| S3 | **ABDM gateway has no per-user auth or rate limiting.** Its only gate is `X-Service-Key`, which ships in the frontend bundle; `config.js` claims rate limiting that doesn't exist. OTP endpoints are abusable. | clinuxflow-abdm-gateway `src/index.js`, `src/lib/serviceAuth.js`; clinux-frontend `src/config.js` | High |
| S4 | Sessions can't be revoked (7-day JWT, no status re-check) and are stored in localStorage. | clinuxflow-api `userAuth.js`, `session.js`; clinux-frontend `config.js`, `stores/auth.js` | Medium |
| S5 | Staff created via join token get role `hospital_admin` (`createTeammateAccount` never sets `role`). | clinuxflow-api `accounts-db.js`, `control.js` redeem | Medium |
| S6 | `POST /api/facility/affiliates` still accepts writes with no client caller (SPEC-26 says it was retired). | clinuxflow-api `control.js` | Low–medium |
| S7 | No record-level `AuditEvent`s (reads and edits aren't logged). | both | Medium (compliance) |
| S8 | On-device data not encrypted at rest. | clinux-frontend | Medium |
| S9 | HFR registration is not tier-gated, contrary to SPEC-05 policy. Decide: enforce, or change the policy. | gateway / frontend | Policy |

After S1, audit every `*-db.js` accessor for the same "lookup by id without tenant" pattern and
add a cross-tenant test per endpoint.

## 1. Open build items

| Area | What's left | Repo(s) | Spec |
|---|---|---|---|
| Outpatient visit as a PlanDefinition | Front Desk → Consultation → Checkout runs on `encounter_status` and stage locks; no Tasks per station, no worklist, no "next station ready" | frontend, api | SPEC-04 §4, SPEC-13, SPEC-23 §9 |
| `ContactPoint.system` never captured | Makes `Patient.telecom:mobile` (and similar slices) unreachable, so `valid:true` is impossible for real patients | api (YAML), frontend (hosts) | SPEC-13 §5.3 |
| SDC-canonical `definition` URLs | Compiler emits `http://hl7.org/<Type>#path`; SDC expects `.../fhir/StructureDefinition/<Type>#...`; rebuild system forms after | api | SPEC-02 §4 |
| Specialty catalog in three places | YAML choices, `ServicesHost.vue` copy, `clinic-specialities.json` | api, frontend | SPEC-23 §3.2 |
| Generic `resource_records` migration | Facility / Provider / Affiliate still on their own routes | api | SPEC-24 §7 |
| Designer "Include cloud records" toggle | Optional | frontend | SPEC-24 |
| Cübo mobile-nav retrofit onto `AdaptiveSectionNav` | Not done | frontend | SPEC-24 §7 step 8 |
| ABHA certificate endpoint 404 | Blocks every ABHA route that encrypts; confirm the current endpoint with ABDM | abdm-gateway | SPEC-11 §4 |
| HPR `/api/v1/auth/cert` response shape | Unverified TODO in `encryption.js` | abdm-gateway | — |
| SPEC-26 join tokens | Two-browser UI walkthrough of the Cübo approval card not done; Worker not deployed with the queue consumer (the queue itself is provisioned) | frontend, api | SPEC-26 §11 |
| Personal worklist (SPEC-22 D1) | Persistence built (SPEC-25); per-action Task records and the worklist UI not built | frontend, api | SPEC-22 §2 |
| Wikidata specialty tagging at onboarding | Regressed in the SPEC-24 rebuild; re-add to `ProviderBasicsHost.vue` | frontend | SPEC-06 §6 |
| SPEC-08 phase 1 content batch | Not run | api | SPEC-08 |
| Conflict detection and resolution UI, Federation Actor | Design only | frontend | SPEC-19 §5–§8 |
| Composition attestation and CapabilityStatement | Not built | api | SPEC-13 §5.4, §6.4 |
| Mobile/sync/video roadmap | Production secrets not yet deployed | api, gateway | — |

## 2. Known bugs

- The LHC-Forms coded-field data-loss bug is fixed (the `useSystemForms.js` focusout capture).
- The duplicate Hospital Profile surface is resolved (SPEC-12 §3).
- Open items are the security list above and the `ContactPoint.system` gap.

## 3. Designed, not started

| Spec | Topic |
|---|---|
| SPEC-06 / 08 | Multi-intent, dispatch registry, group chat, unified inbox, enterprise embeddings |
| SPEC-07 | SNOMED clinical chat (subsets, CDS rules, nano-DC models) |
| SPEC-10 | e-Sushrut gaps: IPD, lab orders, MIS reporting, SMS/notifications, pharmacy stock (no roadmap decided) |
| SPEC-11 §6 | M4 / NHCX claims |
| SPEC-12 §4.2–§4.4 | Bounded-context scope stack, shared resolution component, outcome metrics |
| SPEC-15 §4–§7 | Markdown rendering, section cards, formal pop-out, retiring slot matching |
| SPEC-16 | Notebooks and Task-primary navigation |
| SPEC-17 | TanStack AI conversation layer |
| SPEC-21 §4 | Tier lifecycle (trial/suspend/discontinue), unified storage state |
| SPEC-23 | Speciality Rooms, `definitionCanonical` composition, EpisodeOfCare/CarePlan |
| BLOG-07 | Grammar-constrained local extraction (design written as a blog post) |

## 4. Parked / deferred by decision

- rendering-xhtml rich-text YAML field (compiler backlog).
- LForms data sharing / sync (the AES-GCM key scheme once in `main.js` was superseded by `sessionTransfer.js`).
- NIST ZTA physical service split: module boundary done; the physical split waits for a forcing function.
- TanStack Query + GraphQL: evaluated, not adopting broadly.
- Any Patient-authenticated surface: out of scope (Patient has no login, SPEC-21 §6).

## 5. Pull requests

No open PRs in any repo as of 2026-09-26 (checked with `gh pr list`). Memory notes that say
"PR #N (not merged)" are stale.

## 6. Housekeeping

- Dead code to delete: `clinux-frontend/src/pages/AiEngine.vue` (unrouted), `sliceQuestionnaireGroup`
  (no caller), `clinuxflow-api/src/lib/runtime/runtime-scribe-engine.js` (never imported; see SPEC-03 §5).
- `clinux-frontend/CLAUDE.md` is stale: it says every collection is localStorage, that `main.js`
  preloads every collection before mount, and that ABDM lives on `/onboarding-abdm`.
- `clinux-frontend/idb-race-repro.mjs`: debug repro for the SPEC-25 race; move under `scripts/` or delete.
- `docs/abdm-fhir-data-xchange/` holds both `definitions.json.zip` and its extracted folder
  (~71 MB together); keep one.

---

## Working across machines

- **Branch per feature, one machine per feature.** Mac Mini and the Windows PC must never commit to
  the same branch at the same time. `git pull` before starting, push when you stop.
- **Secrets don't travel via git.** Copy these by hand (use the `*.example` files as templates):
  `clinux-frontend/.env.local`, `clinuxflow-api/.dev.vars`, `clinuxflow-abdm-gateway/.dev.vars`.
- **D1 local state doesn't travel either.** `.wrangler/` is gitignored. On a new machine run
  `npx wrangler d1 migrations apply clinuxflow --local` in clinuxflow-api (migrations go up to 0015).
- **Line endings.** Each repo has a `.gitattributes` forcing LF, so the Windows checkout won't
  produce CRLF diffs. On Windows also run `git config --global core.autocrlf false`.
- **Dev ports.** clinuxflow-api = 8787, clinuxflow-abdm-gateway = 8788.
- **Claude memory** is its own repo (`Sekaran-ind/clinux-claude-memory`). On Windows it goes in
  `%USERPROFILE%\.claude\projects\<path-slug>\memory\`, where the slug is derived from the clone
  path, so it will differ from the Mac one.
