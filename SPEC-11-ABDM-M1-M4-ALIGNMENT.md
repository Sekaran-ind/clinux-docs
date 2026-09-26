# SPEC-11: ABDM M1–M4 Alignment: Prerequisites First, and Where to Invest

| | |
|---|---|
| **Status** | Current strategy. The prerequisite layer (§3) is built; its UI has since been rebuilt twice (SPEC-23, SPEC-24). M1 is light, M2/M3 are deferred, M4 is chosen but not started. |
| **Last reviewed** | 2026-09-26 |
| **Code** | `clinuxflow-api/src/routes/auth.js` (register, clinic-name), `migrations/0008_add_account_role.sql`, `clinux-frontend/src/data/control/hprRoles.js`, `src/router/homeDestination.js`, `src/workflow/onboardingJourneys.js`, the ABDM panels listed in SPEC-05 §9 |
| **Related** | SPEC-05, SPEC-09, SPEC-10, `clinuxflow-abdm-integration-approach.md` |

## 1. Purpose

Place ClinuxFlow's ABDM work against the milestones ABDM actually certifies (M1–M4), instead of
treating "ABDM integration" as one bucket, and say which milestones get serious investment.

## 2. The milestones

- **M1: ABHA-based patient identity and discovery.** Create or verify ABHA, link patients,
  answer discovery requests from Consent Managers.
- **M2: Health Information Provider (HIP).** Share the facility's records: care contexts,
  consent notifications, encrypted FHIR bundles (Fidelius ECDH) for authorized requesters.
- **M3: Health Information User (HIU).** Pull records in: request consent, decrypt external
  bundles, present them to the clinician.
- **M4: NHCX.** Digital insurance claims: eligibility, pre-authorization, claim submission and
  response via the National Health Claims Exchange.

**HFR and HPR registration are prerequisites, not milestones.** A facility's HFR ID becomes its
HIP ID; a professional's HPR ID identifies them. NHCX onboarding uses the HFR ID plus a per-claim
ABHA, and does not require M1–M3 certification first. That makes M4 light on prerequisites
compared with M2/M3, which need STQC security assessment, Fidelius encryption and NRCeS-profile
FHIR bundles.

## 3. The prerequisite layer (built)

Sign-up collects email, password and a **role** (`POST /api/auth/register`):

| Role (`accounts.role`) | Label (HPR vocabulary) | Journeys offered | Lands on after login |
|---|---|---|---|
| `hospital_admin` | Facility Manager | Register Your Facility (`/onboarding`) | `/clinic-home` once a clinic profile exists, otherwise `/` |
| `health_professional` | Healthcare Professional | Add My Details (`/practitioner-home`) | `/practitioner-home` |
| `admin_and_health_professional` | Healthcare Professional and Facility Manager | Both | `/clinic-home` or `/` |

Journeys are role-gated links (`ONBOARDING_JOURNEYS`), suggested by Cübo after register or login
(SPEC-21 §5). They are deliberately not tracked PlanDefinitions (SPEC-22 §5.14). The clinic name
is a placeholder until the facility profile supplies it (`PATCH /api/auth/clinic-name`). Journeys
are self-service and can be completed in any order.

HFR and HPR themselves run as stage-gated panels (SPEC-05 §9), live-verified against the ABDM
sandbox.

## 4. M1, deliberately light

No certified M1 identity-provider build (no Patient Master Index, no discovery endpoints). What
exists is **assisted ABHA at the front desk**: create or look up a patient's ABHA and store the
number or address on the Patient (`PatientAbhaPanel`, with seven ABHA verification-status
extensions on `ClinuxFlowPatient`). This is currently blocked by the sandbox certificate 404
(SPEC-01 §10).

## 5. M2/M3, pragmatic and explicitly not certified

DigiLocker is a real Health Locker with push and pull, but there is no evidence a third-party
facility's software inherits HIP or HIU status through it. ClinuxFlow's near-term answer to M2/M3's
purpose (portable records) is the DigiLocker PDF+QR export (SPEC-09 §5), plus asking the patient to
show their own DigiLocker records during a visit. **Neither is M2/M3 compliance, and ClinuxFlow does
not claim it is.** Certified HIP/HIU work stays deferred.

## 6. M4 (NHCX) is the next real investment

Its prerequisites are already covered (HFR ID from §3, ABHA from §4). It is also a real gap in the
government competitor: SPEC-10 found no claims integration in e-Sushrut@Clinic's documented
features. Cashless claims for small clinics is value a free government product doesn't offer,
which matters because ClinuxFlow cannot compete on price (SPEC-10 §1). **Not started.** The FHIR
path dictionary already includes Claim, Coverage and Invoice.

## 7. Related specs

- SPEC-05: the free-tier ABHA/HPR line this spec's M1 framing builds on.
- SPEC-09: the DigiLocker export §5 relies on.
- SPEC-10: the competitive case for M4.

## 8. History

The first build (2026-08) used `HospitalOnboarding.vue` and `AbdmFieldForm.vue` over
`abdmSchema.js`. SPEC-22 briefly moved hospital setup into Cübo; SPEC-23 moved it back to page
routes; SPEC-24 replaced the field forms with StructureDefinition-backed hosts. The role model,
the clinic-name placeholder and the milestone strategy were unchanged throughout.

## 9. Open items

- **Staff who join through a join token get the wrong role.** `createTeammateAccount` never sets
  `role`, so a new account created by `POST .../redeem` takes the column default,
  `hospital_admin`. A receptionist who joins by token is then offered "Register Your Facility"
  and routed like a facility manager. Fix: capture the role on the redeem form (or derive it from
  the token) and pass it through.
- Role gating is enforced in the UI (journey visibility, home routing) and in specific server
  routes, not in the router. A `health_professional` can open `/onboarding` by URL. Accepted for
  now.
- M4: scope the NHCX integration (eligibility first).
