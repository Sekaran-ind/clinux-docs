
# ClinuxFlow – ABDM Integration Approach (Design Phase)

**Scope:** ABHA (patient identity), HPR (health professional registry), HFR (health facility registry) — based on the ABDM ABHA V3 Integrator Guide and NHPR Sandbox API docs supplied. HIP/HIU consent-manager and FHIR health-record exchange are intentionally out of scope for this document, since specs for those weren't part of this review, and should be scoped separately once ClinuxFlow's data-exchange requirements are firmed up.

---

## 1. Why this matters for ClinuxFlow

ABDM certifies ClinuxFlow not as a patient or a facility, but as a **Digital Solution Company (DSC)** — the software vendor whose product other actors (patients, doctors, facilities) use to interact with the ecosystem. Concretely this means three separate, mostly independent integration surfaces:

- **ABHA APIs** so patients using ClinuxFlow can create/link their ABHA address and be identified consistently across the ecosystem.
- **HPR APIs** so doctors/nurses/lab techs using ClinuxFlow can register as verified professionals and get an HPR ID.
- **HFR APIs** so the health facilities running ClinuxFlow can register themselves and, importantly, declare ClinuxFlow as their **ABDM-compliant software** (the `abdmCompliantSoftware` field in HFR's facility creation API). This last point is the actual certification hook — NHA maintains a master list of certified software products, and a facility can only link itself to software already on that list. ClinuxFlow getting listed there is the end goal of the DSC certification process, not just calling the APIs correctly.

All three surfaces share the same auth/session mechanics and the same encryption pattern, which is why a shared integration layer (Section 3) makes more sense than three independent point integrations.

---

## 2. Environments and identity

| Environment | Auth/session endpoint | Domain(s) to whitelist |
|---|---|---|
| Sandbox (dev/test) | `dev.abdm.gov.in/api/hiecm/gateway/v3/sessions` | `apihspsbx.abdm.gov.in` (HPR/HFR), `abhasbx.abdm.gov.in` (ABHA) |
| Production | Same path, prod host | Prod equivalents, issued after certification |

Before any coding, ClinuxFlow needs to sign up on the ABDM Sandbox portal to receive a `clientId`/`clientSecret` pair. That pair is exchanged for a bearer access token via the gateway sessions API (`grantType: client_credentials`), valid for `expiresIn` seconds (10 hours in sandbox) with a separate `refreshToken`. This token is the `Authorization` header for essentially every downstream ABHA/HPR/HFR call, alongside two mandatory correlation headers on each request: `REQUEST-ID` (a fresh UUID per call) and `TIMESTAMP` (ISO-8601, request time). HFR/HPR calls also need `X-CM-ID` (`sbx` or `abdm` depending on environment).

Separately, some HFR/HPR write operations (e.g. facility creation) require a second, user-level token obtained by authenticating an individual's own HPR ID/password (`/v1/auth/authPassword`) — this is distinct from the DSC's client-credential token and represents the acting facility manager, not the DSC application itself. The design needs to account for both token types living side by side.

---

## 3. Proposed architecture: an ABDM Integration Gateway

Rather than scattering ABDM calls across ClinuxFlow's patient, doctor, and facility modules, the recommendation is a dedicated internal service (or a well-isolated module if a full service is premature at this stage) that owns everything ABDM-specific:

**Session & token manager** – acquires and caches the client-credential access token, refreshes proactively before `expiresIn` elapses, and separately manages short-lived user-level tokens (HPR auth) per acting user session. Tokens should never be persisted in plaintext; cache in memory or an encrypted store with short TTL.

**Encryption service** – ABDM requires several fields (Aadhaar number, mobile number, OTP, password) to be RSA-encrypted (`RSA/ECB/OAEPWithSHA-1AndMGF1Padding`) using a public key fetched fresh per transaction from `/v3/profile/public/certificate`. This needs to be a shared utility, not duplicated per flow, since the public key should be re-fetched per encryption operation (ABDM rotates it) rather than cached long-term.

**Master data cache** – ABDM exposes read-only reference data (states, districts, sub-districts, medical/nurse councils, universities, facility types/sub-types, ownership types, LGD codes, system-of-medicine codes, etc.) that changes rarely. These should be pulled and cached locally (with a periodic refresh job) rather than hitting ABDM on every form load — both for latency and to avoid rate-limit exposure.

**Flow orchestrators** – one per business flow (ABHA creation, ABHA login, HPR registration, HFR onboarding), each a state machine over the multi-step OTP/verify/create sequences ABDM requires, with the transaction ID (`txnId`) as the correlation key carried across steps.

**Audit/compliance log** – every ABDM request/response (minus raw Aadhaar/OTP values) logged with `REQUEST-ID`, timestamps, and outcome, both for debugging and because handling Aadhaar-linked data carries regulatory obligations (see Section 6).

This gateway sits behind ClinuxFlow's existing backend and is the only component allowed to hold ABDM credentials.

---

## 4. Module-by-module flow

### 4.1 ABHA (patient identity)

ABDM supports several ABHA creation paths: Aadhaar OTP, Driving Licence, demographic auth, and biometric (fingerprint/face/iris). For a typical DSC serving walk-in patients and OPD registration, **Aadhaar-OTP creation** and **mobile-OTP-based creation/login** cover the large majority of real-world usage; biometric flows matter mainly if ClinuxFlow will run kiosk/Ayushman-card hardware at a facility, and DL-based creation is a fallback for patients without Aadhaar. Recommend prioritizing Aadhaar-OTP and mobile-OTP paths for MVP, deferring biometric and DL flows to a later increment unless a specific facility use case demands them now.

The core creation sequence is: fetch public key → encrypt Aadhaar/mobile → generate OTP (`/v3/enrollment/request/otp`) → verify OTP (`/v3/enrollment/auth/byAbdm`) → mobile verification → ABHA-address suggestion/creation. Login/verification for returning patients follows a parallel but shorter sequence (login via Aadhaar OTP, ABHA OTP, or mobile OTP). Profile management (update mobile/email, deactivate/reactivate, re-KYC) and "Find ABHA" (search by mobile/Aadhaar/biometric) round out the patient-facing surface and matter mainly for the patient portal / front-desk registration screens.

Design implication: ClinuxFlow's patient record should store the ABHA address/number as the canonical external identifier but must not persist encrypted Aadhaar payloads beyond the transaction; only the ABHA number/address is meant to live in ClinuxFlow's database long-term.

### 4.2 HPR (health professional registry)

Registration mirrors the ABHA Aadhaar flow closely: generate Aadhaar OTP → verify OTP → check if an HPID account already exists (to avoid duplicates) → demographic-auth-via-mobile or generate/verify mobile OTP → HPID suggestion → create HPR ID with pre-verified details, optionally attaching qualification/registration documents (uploaded separately via the documents API). A large set of master-data lookups (system of medicine, medical/nurse councils, universities, colleges, courses) feeds the registration form and should come from the cached master-data layer above.

Design implication: every doctor/nurse/lab-tech profile in ClinuxFlow needs an `hprId` field, and the doctor-facing onboarding UI in ClinuxFlow needs to walk this same OTP sequence — this can reuse the same orchestrator pattern as ABHA creation since the underlying mechanics (OTP, txnId, RSA-encrypted mobile) are structurally identical.

### 4.3 HFR (facility registry)

Facility onboarding is sequential and stateful: **Basic Information** (creates a draft with a `facilityId`/tracking ID) → **Additional Information** → **Detailed Information** → **Submit**, with the path after Basic Information branching on `facilityOperationalStatus` (non-functional facilities can skip straight to Submit). Before any of this, the facility must already have an HPR ID created with the "Facility Manager" role and a user-level auth token from `/v1/auth/authPassword` — this is the second token type referenced in Section 2. Search-before-create (`/facility/search`) is mandatory to avoid duplicate facility records, driven by ownership code, state/district LGD code, and facility name.

The field to design around specifically is `abdmCompliantSoftware`: this is where the facility declares which certified software product it runs, keyed to a code from the `get-Software-details` master API. ClinuxFlow needs to (a) get itself listed as a certified software product through the DSC certification process, and (b) surface that software code in the facility-onboarding UI so ClinuxFlow-run facilities correctly attribute themselves to ClinuxFlow in HFR's records.

---

## 5. Suggested phased plan

| Phase | Focus | Key exit criteria |
|---|---|---|
| 0 – Sandbox setup | Register on ABDM Sandbox, obtain client credentials, whitelist domains, stand up the Integration Gateway skeleton (token manager, encryption service, master-data cache) | Can successfully call `/sessions` and fetch the public certificate end-to-end |
| 1 – ABHA MVP | Aadhaar-OTP creation, mobile-OTP creation/login, Get Profile | A test patient can create/link an ABHA address from ClinuxFlow's registration screen |
| 2 – HPR | Aadhaar-based HPR registration, fetch/update professional details | A test doctor account has a working HPR ID stored against their ClinuxFlow profile |
| 3 – HFR | Facility search, basic/additional/detailed info, submit, software linkage | A test facility is created in HFR sandbox and linked to ClinuxFlow's (sandbox) software code |
| 4 – Certification prep | Security review of credential/PII handling, run through NHA's published test scenarios for each registry, apply for production access | NHA sign-off / production credentials issued |
| 5 – Production rollout | Swap sandbox endpoints for production, monitored go-live with a limited facility set | Stable error rates, audit logging in place |

Consent-manager/HIP-HIU integration (health record exchange) is deliberately not in this plan — recommend scoping that as its own phase once those specs are available, since it introduces a materially different data model (FHIR bundles, care-context linking, consent artefacts) rather than being an extension of the ABHA/HPR/HFR work above.

---

## 6. Cross-cutting risks and open items

Aadhaar-linked data handling is the biggest compliance surface here: even though ClinuxFlow never stores raw Aadhaar numbers (they're RSA-encrypted client-side/in-transit and only used transactionally), the design should still treat every ABHA/HPR creation flow as touching sensitive PII, with encryption-in-transit, minimal retention, and access logging built in from day one rather than bolted on before certification.

Token and credential management deserves its own security review before production — client secret storage, token refresh races, and the dual-token model (client-credential token vs. per-user HPR auth token) are easy to get subtly wrong.

Two things need direct confirmation from NHA/ABDM rather than being inferred from the docs: the exact test scenarios and evidence NHA expects during DSC certification review, and the process/lead time for getting ClinuxFlow listed in the `get-Software-details` master list — since that listing is what actually completes the "DSC" side of certification, and neither document supplied here fully specifies that process.
