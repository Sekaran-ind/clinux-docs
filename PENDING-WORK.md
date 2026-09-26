# Pending Work — ClinuxFlow

Snapshot taken 2026-09-26, when the workspaces moved off the MacBook to the Mac Mini + Windows PC.
All four repos (clinux-frontend, clinuxflow-api, clinuxflow-abdm-gateway, clinux-docs) were
committed and pushed to `main` at that point. Test state at handoff: clinux-frontend 351/351,
clinuxflow-api 383/383, clinuxflow-abdm-gateway 15/15.

Update this file as items close. One feature per machine, each on its own branch — see
"Working across machines" at the bottom.

---

## 1. Open build items (started, not finished)

| Area | What's left | Repo(s) | Spec |
|---|---|---|---|
| Patient Registration journey | Patient GraphDefinition + PatientHome.vue exist; the full Patient Registration journey on the SPEC-23 FHIR-native onboarding pattern is the next un-started piece | frontend, api | SPEC-23/24 |
| `Patient.telecom:mobile` slicing gap | Known-deferred: makes `valid:true` unreachable for any real patient (same gap as `identifier:abhaNumber`). `/api/resources/:type/save` deliberately doesn't gate on validity until fixed | api | SPEC-24 |
| Generic `resource_records` migration | Registry is designed for it; Facility / Provider / Affiliate* still use their own hand-copied routes + tables | api | — |
| Designer "Include cloud records" toggle | Optional nice-to-have from the PatientHome plan | frontend | — |
| CustomFormHost retirement (Tier B) | Remaining LForms/CustomFormHost surfaces not yet moved to the schema-driven hosts | frontend | SPEC-24 |
| Cübo mobile-nav retrofit | AdaptiveSectionNav pattern not yet applied inside Cübo | frontend | SPEC-24 |
| ABHA certificate endpoint 404 | `abhasbx.abdm.gov.in/abha/api/v3/profile/public/certificate` returns 404 on the current sandbox; blocks every ABHA route that encrypts (enrolment, login, find). Confirm the current endpoint with a newer ABHA doc or ABDM support | abdm-gateway | — |
| HPR `/api/v1/auth/cert` response shape | Still an open TODO in `encryption.js` — unverified against a real sandbox response | abdm-gateway | — |
| SPEC-26 join tokens | Cübo browser E2E not run; Cloudflare Queue not actually provisioned | frontend, api | SPEC-26 |
| SPEC-22 decision 1 | Persisted Task records is the one large remaining item of the four decisions (SPEC-25 covers part of the persistence layer) | api, frontend | SPEC-22/25 |
| SPEC-22 room model | 5-static-room model found not to scale; replace with SPEC-23 Speciality Room (catalog-driven) | frontend | SPEC-23 |
| SPEC-21 role-based next action | Built for register/login; UI wiring for the wider PlanDefinition runtime still open | frontend | SPEC-13/21 |
| SPEC-19 §5+ | Conflict sandbox + Federation Actor — design only | — | SPEC-19 |
| SPEC-09 Hospital-side flow | Staff side done; Hospital-side ABDM-anchored flow still open | frontend | SPEC-09 |
| SPEC-08 phase 1 | Content pre-build batch still open | api | SPEC-08 |
| Mobile/sync/video roadmap | Production secrets not yet deployed | api, gateway | — |

## 2. Known bugs (not yet fixed)

- **LForms coded-field data loss** — coded/autocomplete LForms fields drop a typed value unless a
  dropdown suggestion is clicked before save. Confirmed live. Shrinks as LForms surfaces are retired.
- **SPEC-12 capture-surface defects** — duplicate, unreconciled Hospital Profile form.

## 3. Specs written, not started

| Spec | Topic |
|---|---|
| SPEC-06 | Cübo agentic harness: multi-intent classification, semantic layer, unified inbox, agentic dispatch |
| SPEC-07 | SNOMED clinical chat (Parts A/B/C incl. nano-DC on owned hardware) |
| SPEC-10 | e-Sushrut gap list: IPD, lab orders, reporting, SMS/notifications, pharmacy inventory — no roadmap decided |
| SPEC-12 | Rooms as bounded-context Questionnaire/Response documents |
| SPEC-13 | Workflow/Documents/Conformance: extraction hardening, `requirements` CapabilityStatement |
| SPEC-15/16/17 | Cübo-central 3-pane surface, notebooks + Task-primary navigation, TanStack AI layer |
| SPEC-23 | EpisodeOfCare/CarePlan as a 4th notebook type; Encounter state machine (Register→Triage→Consult→SOAP→Checkout) |

## 4. Parked / deferred by decision

- rendering-xhtml rich-text YAML field (compiler backlog).
- LForms data sharing / sync (AES-GCM key scheme in `main.js` is parked WIP, not dead code).
- NIST ZTA physical service split — module boundary done; physical split waits for a forcing function.
- TanStack Query + GraphQL — evaluated, not adopting (no CRUD API for clinical data yet).
- Any Patient-authenticated surface — out of scope (Patient has no login, SPEC-21 §6).

## 5. Pull requests

No open PRs in any repo as of this snapshot (checked with `gh pr list`). Several Claude memory
notes still say "PR #N (not merged)"; those are stale.

## 6. Housekeeping

- `clinux-frontend/idb-race-repro.mjs` — debug repro for the SPEC-25 IndexedDB resumability race.
  Committed so it isn't lost; move under `scripts/` or delete when no longer useful.
- `docs/abdm-fhir-data-xchange/` holds both `definitions.json.zip` and its extracted folder
  (~71 MB together) — keep one.

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
