# Building on ABDM: Lessons from Integrating ABHA, HPR and HFR

*Building Clinical Software That Deserves Trust, part 8*

**For:** engineers at Indian health-tech companies integrating with the Ayushman Bharat Digital
Mission.

---

The Ayushman Bharat Digital Mission gives India something few countries have: national registries
for patients (ABHA), health professionals (HPR) and facilities (HFR), plus a consent-based
framework for exchanging records between them. For software vendors, integrating is increasingly
table stakes. It is also harder than the API documents suggest.

This post collects what we learned integrating ClinuxFlow with all three registries in the ABDM
sandbox. None of it is secret, and most of it would have saved us days.

## First, know what you're certifying

ABDM certifies you as a **Digital Solution Company**: the vendor whose software patients,
professionals and facilities use. Your work splits into milestones:

- **M1**: ABHA-based patient identity: create or verify ABHA, link patients, answer discovery.
- **M2**: act as a Health Information Provider: share records under consent, as encrypted FHIR
  bundles.
- **M3**: act as a Health Information User: request and receive records from other facilities.
- **M4**: NHCX, the National Health Claims Exchange for insurance claims.

HFR and HPR registration are **prerequisites**, not milestones. A facility's HFR ID becomes its
identity in the network; a professional's HPR ID identifies them. M2 and M3 bring the heavy
requirements: security assessment, the consent manager's encryption scheme, and FHIR bundles
conforming to the national profiles. M4 needs mainly an HFR ID and ABHA per claim, which makes it a
comparatively light-prerequisite target that is valuable to clinics.

Our sequencing: HFR and HPR first (built), light M1 (ABHA at the front desk), M4 next, M2 and M3
later. Pick yours deliberately; "do ABDM" is not a plan.

## Architecture: put ABDM behind its own gate

Everything ABDM-specific lives in one small service:
- **A session-token manager.** Your client credentials are exchanged for an access token that
  expires. Refresh before expiry, share one token across requests, and keep it out of the browser.
  We keep it in a single-instance Durable Object.
- **Per-transaction encryption.** Aadhaar numbers, mobile numbers, OTPs and passwords are
  RSA-encrypted (`RSA/ECB/OAEPWithSHA-1AndMGF1Padding`) with a public key fetched from ABDM. Fetch
  it per operation; don't cache it long-term.
- **Transaction state.** Multi-step flows (OTP, verify, create) are correlated by a `txnId`. Keep
  that state server-side, keyed by transaction.
- **Master data.** States, districts, facility types, ownership codes, councils and similar lists
  rarely change. Cache the static ones and call the hierarchical ones (states → districts →
  sub-districts) live.

The service is the only component that holds ABDM credentials or sees Aadhaar-linked payloads.
**Isolation is only half of it.** The service also needs to know which of your users is calling
and to rate-limit them. An endpoint that triggers an OTP to any mobile number is an abuse target,
and a shared key compiled into your frontend doesn't identify anyone (part 4).

## Lessons from the sandbox

### The documents and the sandbox disagree
Several times the sandbox behaved differently from the PDF:
- The ABHA public-certificate endpoint rejected unauthenticated requests with a 401, although it
  serves a "public" key. The Postman samples showed an Authorization header; the prose didn't.
- The same endpoint later started returning 404 at the URL the (by then year-old) document gives.
  Every ABHA flow that encrypts depends on it.

Treat the documents as a starting point. Log every request and response (minus sensitive values)
with its `REQUEST-ID`, and when behavior changes, compare against the newest document version
before assuming your code is wrong. Keep in touch with ABDM support; some answers only come from
them.

### Field names are inconsistent between endpoints
One endpoint wants `mobileNumber`, the next wants `mobile`. Store data in **your own** clean model
and map to each request shape at the edge. Never make ABDM's request bodies your storage format.

### "Yes/No" isn't always boolean
HFR's facility questions ("has a dialysis centre?") look like booleans and aren't: the master data
defines three codes (`YALL`, `YIN`, `N`). Read the master-data type for every "flag" before
modeling it.

### Gate stages on real dependencies, not the manual's screen order
HFR registration runs in stages: search, basic information, additional information, detailed
information, submit. Our first UI followed the user manual's screen order and locked each stage
until the previous one succeeded. But search itself needs the ownership code, state LGD code and
facility name, fields that lived in a later "stage". Users hit `Required OwnershipCode Field is
empty` with no way to enter it. We rebuilt the stage logic from the API's actual prerequisites (what
data each call needs, and which ids it returns) and show locked stages with the reason they're
locked.

### Two kinds of token
Some HFR operations need a user-level token from a professional's own HPR login (the facility
manager acting as themselves), on top of your client-credentials token. Design for both from the
start.

### Coverage creeps
Our first HPR integration covered registration and professional fetch. Real users then needed to
update their professional details, list and upload qualification documents, and verify an email:
all in the spec, none in our gateway. Diff your gateway's routes against the full API list, not
only the flows in your first design.

### Declare your software
HFR's facility record has an `abdmCompliantSoftware` field where a facility names the certified
software it uses, chosen from NHA's list. Until you're certified and listed, your facilities
can't point at you. Getting listed is the real finish line of DSC certification; start that
conversation early.

## Handling Aadhaar and identity data

- Never store Aadhaar numbers. Encrypt them per transaction for ABDM and discard them. If you truly
  must store them, UIDAI requires an Aadhaar Data Vault; the better design avoids that entirely.
- Store what the registries give back: ABHA number and address, HPR ID, HFR facility id.
- Model the verification state precisely. ABHA returns several independent flags (KYC-verified,
  verification status, email verified, mobile verified). They don't imply each other: a child's ABHA
  can be verified without KYC.
- Registration submits legally meaningful declarations. We put the final HFR and HPR submit behind
  an explicit attestation card that summarizes exactly what is being declared, styled so it can't be
  mistaken for routine data entry.

## What ABDM is not

ABDM's registries are identity, not storage. They answer "who is this facility, professional or
patient, and are they verified?" They don't hold your visit records. Each facility keeps custody of
its own records and shares them under consent. Two consequences:
- Your local, offline or cloud storage design is still yours to make (part 3).
- A facility registered with HFR has taken on a duty to answer future record requests reliably. An
  offline laptop can't do that, so HFR registration and durable storage go together.

DigiLocker is a real personal health record app within ABDM, and a practical way for patients to
keep their own records (we export a PDF with a QR code the patient can upload). It doesn't make your
software a certified provider or user of records. Don't claim M2/M3 through it.

## A checklist for your integration

1. Which milestones are you pursuing, in what order, and why?
2. Is every ABDM credential in one isolated service, and does that service authenticate your users?
3. Are you storing your own model, not ABDM's request shapes?
4. Are multi-step flows keyed by `txnId` server-side?
5. Are master lists modeled from the master-data API, not from the UI's labels?
6. Are stages gated on real API prerequisites?
7. Do you never store Aadhaar?
8. Do you log every request and response with its `REQUEST-ID`?
9. Have you started the software-listing conversation?

## Where ClinuxFlow is today

Built and exercised against the ABDM sandbox: HFR facility registration through all stages, HPR
registration plus professional update, documents and email verification, the ABHA routes, and a
separate gateway with token, transaction and encryption handling. Blocked: ABHA encryption flows,
by the sandbox certificate endpoint. Not started: M2, M3, M4 and certification. The gateway's own
per-user authentication and rate limiting are being added before any production use.
