# SPEC-01: Architecture Overview (As Built)

| | |
|---|---|
| **Status** | Current |
| **Last reviewed** | 2026-09-26, against clinux-frontend `7deb3e5`, clinuxflow-api `e89630a`, clinuxflow-abdm-gateway `1905185` |
| **Replaces** | The earlier SPEC-01, which held only a pasted list of 42 specialty labels (that list lives in `clinuxflow-api/data/clinic-specialities.json`) |
| **Related** | SPEC-05 (tiers), SPEC-13 (workflow/documents), SPEC-19 (local-first), SPEC-24 (conformance), SPEC-25 (task persistence) |

## 1. What ClinuxFlow is

ClinuxFlow is clinic software for Indian outpatient practices, from single-doctor clinics to
small nursing homes. It covers facility and practitioner registration (including ABDM's HFR and
HPR registries), patient registration (including ABHA), the outpatient visit (Front Desk →
Consultation → Checkout), prescriptions, billing, imaging review, and staff chat and video. Three
design commitments shape everything else:

1. **FHIR R4 as the data model, not an export format.** Forms are compiled from FHIR paths,
   captured as `QuestionnaireResponse`, extracted to FHIR resources and validated against local
   StructureDefinitions.
2. **Local-first.** Every record is written to the device first. Network sync is additive and,
   for the cloud, paid.
3. **ABDM through a sealed gateway.** Only one Worker ever holds ABDM credentials or touches
   Aadhaar-linked payloads.

## 2. Repositories and deployables

| Repo | Runtime | Role |
|---|---|---|
| `clinux-frontend` | Vue 3 + Vite SPA; Capacitor for Android/iOS; Tauri desktop wrapper (`src-tauri/`) | All UI, all local storage, the workflow runtime, Cübo |
| `clinuxflow-api` | Cloudflare Worker (Hono) + D1 + KV + Durable Objects + Queues + Workers AI | Accounts and auth, form compiler, extraction, conformance, cloud mirrors, locks, chat signaling, video tokens |
| `clinuxflow-abdm-gateway` | Cloudflare Worker (Hono) + Durable Objects | ABHA, HPR, HFR API proxy: session token, RSA encryption, transaction state, master data |
| `clinux-docs` | Markdown | Specs, blog drafts, ABDM reference PDFs and FHIR IG definitions |
| `clinixflow` (sandbox) | Alpine.js + Express | The original prototype; read-only reference, not developed further |

Dev ports: clinuxflow-api `8787`, clinuxflow-abdm-gateway `8788`, Vite `5173`, Tauri LAN server
`47856` (HTTPS, self-signed).

## 3. Layers

```
┌──────────────────────────── clinux-frontend (browser / Capacitor / Tauri) ────────────────────────────┐
│ Pages: Index · ClinicHome (hosts FrontDesk / ConsultationDesk / Checkout) · Onboarding · StaffOnboarding│
│        PractitionerHome · PatientHome · Designer · Dashboard · /ai-engine (Cübo THREE_PANE)            │
│ Cübo (components/Cubo.vue): threads, contacts, P2P chat, next action, entry journey                     │
│ Workflow runtime (src/workflow): PlanDefinition → XState actor, RxJS event bus, audit log              │
│ Control (src/data/control, components/control): hosts, conformance clients, ABDM adapters, join tokens │
│ Runtime (src/data/runtime): encounter coordination, task sync, DigiLocker export                        │
│ Collections (src/data/collections): TanStack DB on localStorage or IndexedDB, clinic-scoped             │
│ Sync: sharedServerSync.js (LAN, all tiers) · explicit push/fetch mirrors (cloud, paid tier)             │
└───────────┬──────────────────────────────────────────────┬─────────────────────────────────────────────┘
            │ HTTPS + Bearer JWT + X-Service-Key           │ HTTPS + X-Service-Key
┌───────────▼─────────────── clinuxflow-api ─────────────┐ ┌▼──────────── clinuxflow-abdm-gateway ──────────┐
│ routes/auth.js      accounts, sessions, team, usage    │ │ routes/abha.js  enrollment, login, find,       │
│ routes/control.js   roster, join tokens, conformance,  │ │                 profile                        │
│                     generic resource API, valuesets    │ │ routes/hpr.js   registration, professional,    │
│ routes/runtime.js   encounters, locks, tasks, chat     │ │                 documents, password, masters   │
│                     signal, video, compile/extract/    │ │ routes/hfr.js   search, 4-stage create, masters│
│                     assemble, scribe                   │ │ DOs: SessionTokenManager,                      │
│ routes/platform.js  system forms/flows catalog, health │ │      RegistrationTransaction                   │
│ D1 · KV (forms library, Wikidata cache) · DO (chat)    │ │ lib/encryption.js  RSA-OAEP per transaction    │
│ Queue (cubo-task-queue) · Workers AI                   │ └─────────────┬──────────────────────────────────┘
└────────────────────────────────────────────────────────┘               ▼
                                                           ABDM sandbox / production endpoints
```

The **control / runtime split** came out of a review against NIST SP 800-207 (zero-trust
architecture). It is a module boundary inside each repo, not separate services:
`clinuxflow-api/src/routes/{auth,control,runtime,platform}.js` with
`src/lib/{control,runtime,shared}/`, and `clinux-frontend/src/data/{control,runtime}/` plus
`src/components/control/`. Control is identity and roster (who may act); runtime is clinical and
session work (what is being done). Splitting into physical services is deliberately deferred
until there is a concrete reason (separate scaling, separate compliance scope).

## 4. The FHIR pipeline

```
YAML form ──► yaml-to-questionnaire.js ──► Questionnaire ──► capture ──► QuestionnaireResponse
 (tools/system-forms,   (validated against the          (LHC-Forms for clinical rooms;       │
  samples/, Designer)    FHIR R4 path dictionary)        hand-authored hosts for registration) │
                                                                                               ▼
 FHIR document Bundle ◄── composition-assembler.js ◄── conformance-validator.js ◄── local-extractor.js
 (Composition + entries,   (status: preliminary)         (against 7 local SDs)       (definition-based
                                                                                      SDC extraction)
```

- **Dictionary** (SPEC-02): `tools/dictionary-builder.js` walks the vendored `@smile-cdr/fhirts`
  R4 TypeScript classes with `ts-morph` for the 31 resources in `tools/clinixflow.config.json`,
  producing per-resource path graphs (`data/graphs/*.graph.json`) and the YAML authoring schema.
- **Compiler**: each YAML field names a FHIR path; the compiler rejects paths not in the graph,
  emits nested groups, `enableWhen`, `answerValueSet`, and a `definition` per item.
- **Capture**: clinical rooms render through `LhcFormHost.vue` (LHC-Forms). Registration
  entities (Facility, Provider, Affiliate Organization, Patient) use hand-authored hosts inside
  `AdaptiveSectionNav` (SPEC-24), which write the same `QuestionnaireResponse` item shape, so
  everything downstream is shared.
- **Extraction** (SPEC-13 §5): `ComprehensiveLocalExtractor` produces linked FHIR resources and
  handles repeating and nested groups, FHIR array cardinality (`FHIR_ARRAY_PATHS`), extension
  URLs, and pluggable identity resolvers.
- **Conformance** (SPEC-24): seven StructureDefinitions (`ClinuxFlowFacility`,
  `ClinuxFlowProvider`, `ClinuxFlowProviderRole`, `ClinuxFlowAffiliatePractitionerRole`,
  `ClinuxFlowAffiliateOrganization`, `ClinuxFlowPatient`, `ClinuxFlowTask`) and two
  GraphDefinitions (onboarding, patient). The validator checks cardinality, fixed values and
  slices; `next-best-action.js` walks the graph to suggest what to capture next.
- **Documents**: `FhirDocumentAssembler` builds a `Bundle{type:document}` with the Composition as
  the first entry. Nothing is attested (`final`) yet, and nothing is sent to an external FHIR
  server.

## 5. Workflow runtime

Only **clinical and session journeys** run as state machines ("state machines only for clinical
journeys", SPEC-23). Registration of facilities, practitioners and patients is plain pages plus
stateless conformance checks.

- `planDefinitionRunner.js` compiles a FHIR `PlanDefinition` into an XState v5 parallel machine:
  one region per action, `pending → ready → active → done`, `relatedAction` as guards, optional
  `invoke` services for real side effects, opt-in `repeatable` actions.
- `workflowRuntime.js` owns all actors behind one RxJS event bus. It persists snapshots, diffs
  transitions into an audit log, discards snapshots whose shape no longer matches the plan, and
  exposes `onActionDone` for cross-plan triggers.
- The live plan today is `ENTRY_PLAN_DEFINITION` (register, login, forgot password, change
  password, logout), hosted in Cübo (SPEC-20). `system-flows-library.json` also ships a compiled
  hospital-setup plan that nothing currently runs (SPEC-22 §5.14).
- Persistence (SPEC-25): snapshots and an append-only audit log in IndexedDB; paid clinics mirror
  both to D1 (`task_snapshots`, `task_audit_log`) with single-writer locks (`task_locks`).

The outpatient visit itself (Front Desk → Consultation → Checkout) is **not** yet a
PlanDefinition. It is coordinated by `encounter_status` plus server-side stage locks
(`encounter_assignments`).

## 6. Storage

| Tier | Where | Mechanism | Who |
|---|---|---|---|
| Device | localStorage (most collections); IndexedDB (the three AI Engine sandbox collections, `taskActorSnapshots`, `taskAuditLog`) | `collectionFactory.js`, `indexedDbCollectionFactory.js`; every record carries `clinicId` | Everyone |
| LAN | Tauri shared server, HTTPS on `:47856` | `sharedServerSync.js`: poll-and-merge with a local-push grace window | Everyone (the free tier's multi-device path) |
| Cloud | D1 | Explicit mirrors (`provider_composition`, `encounter_documents`, `resource_records`, `task_*`) plus source-of-truth tables for accounts, clinics, affiliates, join tokens, locks, meetings, usage | Paid tier (`requirePaidTier()`), except accounts and auth |

D1 tables (migrations 0001–0015): `clinics`, `accounts`, `facility_affiliates`,
`facility_organization_affiliates`, `provider_composition`, `encounter_assignments`,
`encounter_documents`, `encounter_meetings`, `contact_meetings`, `clinic_usage_daily`,
`task_snapshots`, `task_audit_log`, `task_locks`, `resource_records`, `facility_join_tokens`,
plus the legacy `partner_profile`, `room_workflows` and `local_holding_queue`.

Local data is the source of truth between syncs. Cloud writes are best-effort pushes after a
successful local save, and a 403 (free tier) means "stay local", never an error.

## 7. Identity, sessions, and access

- **Accounts** are staff only. Roles: `hospital_admin`, `health_professional`,
  `admin_and_health_professional`, labeled with the HPR role vocabulary (`hprRoles.js`). Patients
  never log in; their records reach them through the DigiLocker PDF+QR export (SPEC-21 §6).
- **Passwords**: PBKDF2-SHA256 via Web Crypto (`passwordHash.js`). Recovery is by a hashed
  security question; there is no email infrastructure.
- **Sessions**: HS256 JWT with a 7-day TTL carrying `{sub, clinicId, email}`, stored in the
  browser's localStorage and sent as `Bearer`. `requireUser()` verifies signature and expiry;
  `requirePaidTier()` reads `clinics.tier` from D1 on every call so upgrades apply immediately.
- **Tenancy**: `clinicId` always comes from the verified JWT, never the request body. Every local
  record is clinic-scoped (added after a real cross-clinic leakage bug on shared browsers).
- **Roster linking** (SPEC-26): staff, affiliate practitioners and affiliate organizations join by
  redeeming a facility-issued token. The request reaches the admin as an encrypted card in Cübo
  chat and is durably queued through `cubo-task-queue`.
- **Device-to-device transfer**: `sessionTransfer.js` encrypts with a fresh AES-256-GCM key per
  transfer, embedded in the payload. That gives tamper-evidence and keeps plaintext out of the
  transport. It is not access control: whoever holds the string can open it. Access is decided by
  `sessionShare.js`'s clinic consent check after decryption.

## 8. Real-time

- **Chat**: peer-to-peer WebRTC DataChannels (`p2pChat.js`). The `ChatSignalingRoom` Durable
  Object relays only SDP/ICE and stores nothing. It is capped at two sockets per room, so there is
  no group chat, and there is no offline delivery (join requests use the queue instead).
- **Video**: Cloudflare RealtimeKit, per-encounter meetings and contact-to-contact calls
  (`/api/realtime/*`).

## 9. AI and NLP, as they actually are

| Piece | What it is | Where |
|---|---|---|
| Intent classifier | `node-nlp`, two trained intents (`generate_soap`, `switch_room`), single-label | `nlp/intents.js` |
| Slot filling | `@nlpjs` NER over harvested field labels, plus deterministic `/shortcuts` | `nlp/formSlotEngine.js`, `fieldShortcuts.js` |
| Scribe | One Workers AI call to Llama 3.3 70B for SOAP drafting and field suggestions, paid tier | `POST /api/workflow/test-scribe` |
| Semantic tagging | Wikidata lookup at design and onboarding time, KV-cached | `wikidataTagging.js` |
| Planned | Constrained local decoding, SNOMED subsets, MedGemma/MedCAT/MedSAM on owned hardware | SPEC-03, SPEC-06, SPEC-07, BLOG-05, BLOG-07 |

## 10. Known gaps that matter

Found during the 2026-09-26 review. None are fixed yet; all are tracked in `PENDING-WORK.md`.

0. **Cross-tenant reads and writes on the cloud mirrors (highest priority).**
   `GET /api/encounters/:id`, `GET /api/tasks/:planId/snapshot` and `GET /api/tasks/:planId/audit`
   look rows up by id alone, with no `clinic_id` condition. The matching `PUT`s upsert
   `ON CONFLICT(id) DO UPDATE SET clinic_id = excluded.clinic_id`, so any paid account can also
   overwrite and take over another clinic's row. Encounter ids are
   `rec-<timestamp>-<5 random chars>` and plan keys embed account ids, so they are guessable or
   discoverable, not secret. Encounter documents are clinical data. The same flaw appears in two
   more places: `POST /api/realtime/join` reuses an encounter's existing video meeting without
   checking the caller's clinic (and is not paid-gated), so any signed-in account that knows an
   encounter id can join another clinic's consultation; and `resource_records`' upsert reassigns
   `clinic_id` on conflict (its reads are correctly scoped). Fix: add `AND clinic_id = ?` to every
   read, refuse (403) any write or join whose existing row belongs to another clinic, and audit
   every `*-db.js` accessor for the pattern.
1. **The ABDM gateway has no per-user authentication.** Its only gate is `X-Service-Key`, whose
   value is compiled into the public frontend bundle. A comment in `config.js` says rate limiting
   backs this up, but neither Worker implements any. Anyone who extracts the key can drive
   OTP-sending ABDM endpoints. Fix: require a valid clinuxflow-api session, or a short-lived
   token minted by it, at the gateway, plus per-account rate limits.
2. **Sessions cannot be revoked.** A JWT stays valid for 7 days after an account is disabled or
   its password changes, because `requireUser()` never re-checks D1. SPEC-26 worked around one
   instance (pending accounts get no token); the general gap remains.
3. **Session tokens live in localStorage**, so any XSS is a full account takeover.
4. **On-device data is unencrypted at rest** (localStorage and IndexedDB).
5. **`POST /api/facility/affiliates` still accepts writes** even though SPEC-26 retired its only
   client caller. It is a live, unused write path.
6. **No FHIR `AuditEvent`s.** The workflow audit log covers Task transitions only, not record
   reads or edits.
7. **The ABHA certificate endpoint returns 404 on the current sandbox**, blocking every ABHA route
   that encrypts (enrolment, login, find).
8. **The compiler's `definition` URLs are not SDC-canonical.** They read
   `http://hl7.org/Patient#Patient.name`; SDC expects
   `http://hl7.org/fhir/StructureDefinition/Patient#Patient.name`. The local extractor accepts
   either form, but an external `$extract` server would not (SPEC-02 §4).

## 11. Test and build baseline (2026-09-26)

clinux-frontend 351 tests in 38 files; clinuxflow-api 383 in 17; clinuxflow-abdm-gateway 15 in 2.
`npm run build:kernel` in clinuxflow-api regenerates the dictionary, graph bundle, system forms
and system flows.
