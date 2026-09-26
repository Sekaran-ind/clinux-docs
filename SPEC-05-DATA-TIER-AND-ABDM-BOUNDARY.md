# SPEC-05: Data Tiers and the ABDM Product Boundary

| | |
|---|---|
| **Status** | Current policy. Free/paid tiers built; enterprise tier not built; two policy lines are not yet enforced in code (§6.4). |
| **Last reviewed** | 2026-09-26 |
| **Code** | `clinuxflow-api/src/lib/shared/userAuth.js` (`requireUser`, `requirePaidTier`), `src/lib/shared/usageTracking.js`, `migrations/0003` (`clinics.tier`), `migrations/0007` (usage), `clinux-frontend/src/data/collectionFactory.js`, `src/data/sharedServerSync.js` |
| **Related** | SPEC-01 §6, SPEC-11 (ABDM milestones), SPEC-19 (modes), SPEC-21 (tier lifecycle), SPEC-25 (task mirror) |

## 1. Purpose

One authoritative answer to "where does this data live, and why". Before this spec, placement was
decided feature by feature, and that produced real bugs (for example ClinicHome falling back to a
hard-coded demo clinic because its profile and its staff roster lived in different tiers with no
rule for which wins).

## 2. The three storage tiers

| Tier | Mechanism | Reach | Present for |
|---|---|---|---|
| **Local** | TanStack DB collections on localStorage or IndexedDB | This device only | Every clinic, always |
| **LAN** | Tauri shared server (`src-tauri/shared_server.rs`, HTTPS `:47856`), mirrored by `sharedServerSync.js` | Devices on the clinic's network | Every clinic that runs it |
| **Cloud** | D1 behind clinuxflow-api | Anywhere with internet | Paid clinics (with the exceptions in §6.4) |

The LAN tier is the same collection interface backed by a server on the clinic's network instead
of the browser. Callers never know which tier served them.

## 3. Master-data principle

Cloud identity (`accounts`, `clinics`) is the root and exists from the moment an account
registers, whatever the tier. Local and LAN provider data (the Facility profile, the care-team
roster) enrich that identity. A missing enrichment falls back to root identity, never to a
placeholder. Once a facility is HFR-registered, the HFR record becomes the authority for the
fields HFR owns.

## 4. Reference-only relationships

Every cross-record relationship is a **one-directional stored reference** (Encounter → Patient,
Practitioner → HPR ID, Organization → HFR ID, PractitionerRole → Organization). Reverse lookups
(all encounters for a patient) are always computed queries, never stored back-links. This keeps
the graph acyclic by construction. `clinical.js`'s `patientRef` is the original instance; the
GraphDefinitions in SPEC-24 formalize the same rule.

## 5. ABDM's role: federated identity, not operational storage

HFR, HPR and ABHA answer "who is this facility, professional or patient, and are they verified".
They do not store operational data, and in ABDM's federated design they cannot: each facility
(as a Health Information Provider) keeps custody of its own visit records, ABHA correlates
patients across facilities, and the Consent Manager handles discovery and consent, not storage.

Consequences:
- Local, LAN and cloud tiering for encounters, vitals, notes and billing is unaffected by ABDM.
- ABDM adds a new reason for cloud durability: an HFR-registered facility takes on a standing
  duty to answer future health-information requests reliably. A device that is off or offline
  cannot do that.
- Health-record exchange (HIP push, HIU pull, consent artefacts) is out of scope here (SPEC-11 §5).

## 6. Three commercial tiers, one additive model

### 6.1 Free
- Local and LAN storage. Multi-staff coordination over the LAN is included. The line is "no reach
  beyond the premises and no health-information exchange", not "single user".
- ABHA lookup and creation for patients (a citizen convenience with no ongoing obligation).
- HPR registration for staff (a personal, portable credential).
- **Not included by policy**: HFR registration, because it creates the standing duty in §5.

### 6.2 Paid (`clinics.tier = 'paid'`)
Adds cloud durability and cross-location coordination, enforced by `requirePaidTier()` on:
- `/api/encounters/*` (documents mirror, assignment, stage locks)
- `/api/tasks/*` (snapshot and audit mirrors, locks, SPEC-25)
- `/api/resources/:type/search` and `/save` (the generic resource mirror)
- `POST /api/workflow/test-scribe` (Workers AI cost)

`requirePaidTier()` reads `clinics.tier` from D1 on every call, so an upgrade takes effect without
logging in again. Usage is metered per clinic per day from real `rows_written`
(`clinic_usage_daily`). Cloud durability and HFR/HIP obligations are bundled for now; revisit if
a customer wants one without the other.

### 6.3 Enterprise (design only)
Paid plus a provisioned nano data centre (SPEC-07 Part C) for on-premises inference. Modeled
additively: a nullable `clinics.nano_dc_endpoint`, with its own `requireNanoDcAccess()` gate, so
the binary paid gate stays untouched. Shared or dedicated hardware is a pricing decision, not a
code fork, but shared mode needs a real request-scheduling design (GPU inference is not elastic).
**None of this exists yet**: no column, no gate, no gateway.

### 6.4 Where code and policy differ today
1. **HFR registration is not gated.** Nothing checks the tier: not `FacilityHfrPanel.vue`, and
   not the gateway (which has no user identity at all, SPEC-01 §10). A free clinic can register
   with HFR today. Either enforce the policy (a tier claim on a gateway token) or change it.
2. **`provider_composition` is mirrored for every tier.** `GET`/`PUT /api/provider-composition`
   are `requireUser()` only. This was deliberate in SPEC-26: the server must know whether a
   facility has published before it issues join tokens, and staff on another device must be
   able to pull the clinic profile. It is a small, bounded exception (one document per clinic)
   and should be named as one in pricing.
3. Chat signaling, video join and Wikidata lookups are not tier-gated, by design (small, and
   relayed rather than stored).

## 7. Compliance containment

All Aadhaar- and ABDM-adjacent handling (RSA encryption of identifiers and OTPs, transaction
state, ABDM access tokens) lives in clinuxflow-abdm-gateway and nowhere else. Aadhaar numbers
and OTPs are encrypted per transaction and never stored; only resulting identifiers (ABHA number
or address, HPR ID, HFR facility ID) are kept.

The containment goal was that the free tier never touches this surface. §6.4 shows that is not
yet true for HFR, and SPEC-01 §10 shows the gateway's own access control is weaker than this
section assumes. Both need fixing before a compliance review can rely on this boundary.

The enterprise tier's nano-DC traffic will carry clinical text, audio and images across a
Cloudflare Tunnel. That is a PHI transit surface needing the same treatment (encryption in
transit, minimal retention, access logging). Owned hardware does not exempt it.

## 8. Open items

- Choosing between local-only and LAN/cloud modes: a staff-count trigger was proposed and
  retracted (a second account does not reliably mean a coordination need). Today the mode is an
  explicit user choice (ClinicHome's sync toggle). No automatic trigger is planned.
- Whether free and local-only facilities must register with ABDM once all three ABDM journeys
  are complete. Undecided; it needs a decision before any registration cutover.
- Trial, suspended and discontinued tier states (SPEC-21 §4): not modeled. `clinics.tier` is
  `CHECK (tier IN ('free','paid'))`.
- Resolve §6.4 items 1 and 2.

## 9. Where the ABDM journeys live in the frontend

The original single `AbdmOnboarding.vue` page was retired. Each journey is now a panel inside the
entity it belongs to, all calling the gateway through `abdmGatewayClient.js`:

| Journey | Panel | Hosted in |
|---|---|---|
| HFR (facility) | `FacilityHfrPanel.vue`, a 7-stage `RegistrationLedger` gated by `facilityHfrJourney.js` | Onboarding (Hospital section) |
| HPR (professional) | `ProviderHprPanel.vue`, gated by `hprRegistrationJourney.js` | Onboarding (Care Team section), StaffOnboarding |
| ABHA (patient) | `PatientAbhaPanel.vue` | `PatientBasicsHost.vue` (Front Desk, PatientHome) |

A standalone citizen-facing ABHA/PHR product would justify its own repo (a different audience,
auth model and hardening bar), but that would be a different product. Assisted enrolment at the
front desk stays inside clinux-frontend.
