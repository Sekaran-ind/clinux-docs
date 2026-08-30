# Specification 11: ABDM M1-M4 Alignment — Prerequisite Layer, and Where ClinuxFlow Invests Next

## 1. Objective

Position ClinuxFlow's ABDM work against the real, official milestone structure the ecosystem is certified against — M1 through M4 — instead of treating "HFR registration," "HPR registration," and "ABDM integration" as one undifferentiated bucket of work. This spec states which milestone(s) ClinuxFlow pursues seriously, which it treats lightly on purpose, and why — a decision made explicitly this session, not left implicit. It also documents the prerequisite layer this same session built: simplified sign-up, role capture, and role-routed HFR/HPR self-service journeys.

## 2. ABDM's real milestone structure

Verified this session via direct research (not assumed from the "M1/M2/M3 exist" general awareness alone):

- **M1 — ABHA-based patient identity/discovery.** The facility's own software creates/verifies ABHA numbers, links patients to them, and responds to discovery requests from Consent Managers. This is about the *patient's* identity at the point of registration, not the facility's or professional's own.
- **M2 — becoming a Health Information Provider (HIP).** The facility shares its *own* records out — creating care contexts, handling consent notifications from the ABDM gateway, generating encrypted FHIR bundles (Fidelius/ECDH encryption) for authorized requesters.
- **M3 — becoming a Health Information User (HIU).** The facility pulls records *in* from other facilities — requesting consent, receiving and decrypting external FHIR bundles, presenting a unified record to the clinician.
- **M4 — NHCX.** Digital insurance claims: eligibility checks, pre-authorization, claim submission and response handling through the National Health Claims Exchange.

**HFR (Health Facility Registry) and HPR (Health Professional Registry) registration are prerequisites to M1, not a milestone of their own.** A facility needs a valid HFR ID (which becomes its HIP ID in the ABDM network) and a professional needs a valid HPR ID before any of M1-M4 can begin. Confirmed specifically for M4: NHCX onboarding leverages the facility's HFR ID as its core credential, plus a per-claim ABHA ID for verification — it does **not** require full M1/M2/M3 certification as a hard gate, which makes it a comparatively light-prerequisite target relative to M2/M3's heavy certification burden (STQC security assessment, Fidelius encryption, NRCeS-profile FHIR compliance).

## 3. What this pass builds: the prerequisite layer

Simplified sign-up (`clinux-frontend/src/pages/Index.vue`) now collects only email/password/confirm-password plus a **role**: `hospital_admin` | `health_professional` | `admin_and_health_professional`. Role determines which self-service registration journey(s) ClinicHome offers afterward:

| Role | HFR journey | HPR journey |
|---|---|---|
| Hospital Admin | Yes | No |
| Admin and Health Professional | Yes | Yes |
| Health Professional | No | Yes |

All three roles land on Clinic Home after sign-up (no forced wizard) — the appropriate journey card(s) are simply offered from the user menu there, both self-service and independently completable in any order. HFR's dedicated flow (`HospitalOnboarding.vue`, new) mirrors HPR's existing self-service pattern (`StaffOnboarding.vue`, built in SPEC-09) — both use `AbdmFieldForm.vue`'s controlled inputs (no LForms coded-field data-loss risk) against a canonical field schema (`abdmSchema.js`'s `HOSPITAL_FIELDS`/`STAFF_FIELDS`).

`clinics.name` (previously collected at sign-up) is now a server-derived placeholder until `HospitalOnboarding.vue` captures the real `hospital_name` and calls the new `PATCH /api/auth/clinic-name` — the concrete resolution for the regression this task's own scope change would otherwise have introduced (see clinux-frontend's `stores/auth.js`).

This layer alone does not constitute M1-M4 compliance — it's the prerequisite every one of them needs, built first on purpose.

## 4. M1, treated lightly

ClinuxFlow does not build a certified M1 identity-provider (Aadhaar/mobile-OTP ABHA creation, discovery endpoints, Patient Master Index) in the near term. Instead: **ABHA lookup** — capturing/verifying an already-existing ABHA number at Front Desk patient registration, the same capability already named in `docs/SPEC-05-DATA-TIER-AND-ABDM-BOUNDARY.md`'s free-tier line ("ABHA lookup + personal HPR still allowed"). This is prior art for the framing here, not a new decision — SPEC-05 already drew this line before M1-M4 vocabulary was applied to it explicitly.

## 5. M2/M3, pragmatic — not certified

**DigiLocker verified this session (not assumed): its real ABDM integration is a citizen-facing Health Locker/PHR app with genuine push and pull, but there is no evidence it lets a third-party facility's software inherit M2/M3 certification by proxy.** It's a destination/source within the ecosystem, not a stand-in for ClinuxFlow's own HIP/HIU status.

ClinuxFlow's actual near-term answer for M2/M3's *spirit* — portable, shareable records — is the already-built DigiLocker PDF+QR export (`digilockerExport.js`, `docs/SPEC-09-ABDM-ANCHORED-ONBOARDING-REBUILD.md` §5/§6a): a citizen self-uploads a generated PDF+QR to their own DigiLocker via its native Scan/Upload feature. For pulling a patient's history in, the near-term answer is manual — the clinician asks the patient to show their own DigiLocker-held records during consultation. **Neither of these is certified HIP/HIU compliance, and this spec does not claim they are.** Real M2/M3 (Fidelius encryption, NRCeS-profile FHIR bundles, STQC "Safe-to-Host" security assessment) stays explicitly deferred, unchanged from SPEC-09 §5's existing Enterprise-tier deferral.

## 6. M4, the real investment target

NHCX is where ClinuxFlow puts genuine near-term engineering effort once the prerequisite layer (§3) and lightweight M1 (§4) are in place — its own prerequisites are comparatively light (an HFR ID, already covered by §3, plus a per-claim ABHA lookup, already covered by §4), unlike M2/M3's heavy certification burden. It is also a genuine, verified capability gap versus the government-run e-Sushrut@Clinic competitor: `docs/SPEC-10-ESUSHRUT-CAPABILITY-ASSESSMENT.md`'s capability matrix found no billing/claims-integration equivalent in e-Sushrut@Clinic's documented feature set at all. Cashless insurance claims for small clinics is real, demonstrable value a free government product doesn't currently offer — the concrete answer to "price isn't a lever we can compete on" from that same spec.

Not built this pass — named here as the deliberate next real investment, once §3's prerequisite layer is live-verified and stable.

## 7. Relationship to what already exists

- `docs/SPEC-05-DATA-TIER-AND-ABDM-BOUNDARY.md` — the tier/pricing boundary this spec's M1 framing (§4) already anticipated.
- `docs/SPEC-09-ABDM-ANCHORED-ONBOARDING-REBUILD.md` — the canonical-schema/controlled-input pattern (`AbdmFieldForm.vue`, `abdmSchema.js`) this pass extends from Staff/HPR into Hospital/HFR, and the DigiLocker export §5/§6 leans on for M2/M3's pragmatic answer.
- `docs/SPEC-10-ESUSHRUT-CAPABILITY-ASSESSMENT.md` — the competitive analysis that makes M4's business case concrete.

## 8. Status

**Built and live-verified this session**: migration `0008_add_account_role.sql`; `POST /api/auth/register`'s new `{ email, password, role }` contract with server-derived `clinicName` placeholder and forced `facilityType: 'facility'`; `PATCH /api/auth/clinic-name`; `Index.vue`'s simplified sign-up with a 3-option role picker; `HospitalOnboarding.vue` (new, mirrors `StaffOnboarding.vue`'s pattern, splits `HOSPITAL_FIELDS`' flat values across 3 real YAML groups via `withGroupFields()`); `ClinicHome.vue`'s role-gated HFR/HPR entry points; `abdmSchema.js`'s `HOSPITAL_FIELDS` correction (13 fields across the correct 3 `groupLinkId`s, closing a real gap where `onboarding.js`'s `buildClinicProfile()` already read 12 `section_hospital` fields nothing had ever written).

**Explicitly deferred, not built this pass**: real M1 (certified ABHA identity-provider build), real M2/M3 (certified HIP/HIU — Fidelius encryption, NRCeS FHIR profiles, STQC assessment), M4/NHCX integration itself.

## 9. Open items

- `createTeammateAccount` (Phase D team invites) doesn't collect its own `role` at invite time — an invited teammate inherits D1's `role` default (`hospital_admin`), which is safe but not necessarily accurate for e.g. an invited nurse. Low priority, not addressed this pass.
- Router-level role gating was deliberately not added (UI-card-only, matching `isAdmin`'s existing "cosmetic/defense-in-depth" precedent) — a `health_professional` account can still reach `/hospital-onboarding` by typing the URL directly. Accepted, consistent with the existing permission model.
- Whether to raise the shared QR generation's `errorCorrectionLevel` for the DigiLocker export (see `clinux-digilocker-pdf-qr-export` memory note) — unrelated to M1-M4 directly, still an open tuning decision from the prior session.
