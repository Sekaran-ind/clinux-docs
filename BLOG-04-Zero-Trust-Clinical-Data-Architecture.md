# Security and Privacy for Clinical Apps in India, Without the Buzzwords

*Building Clinical Software That Deserves Trust, part 4*

**For:** CTOs, CISOs and data protection officers responsible for clinical software.

---

"Zero trust" has become a sticker on product pages. The idea behind it is simple and useful:
**no request is trusted because of where it comes from.** Being on the clinic's Wi-Fi, holding
an API key, or having logged in last Tuesday isn't enough. Every request is checked against who is
asking, what they're asking for, and whether that is allowed right now. NIST's SP 800-207 describes
this as policy decisions enforced at every point where a subject touches a resource.

This post turns that idea into concrete rules for a clinical app. It uses mistakes we made and
fixed in ClinuxFlow as examples, and names the parts we're still building.

## Who you are defending against

A threat model for small-clinic software is more mundane than most security marketing implies:

| Adversary | Realistic scenario |
|---|---|
| The next user of a shared device | A front-desk PC used by two clinics, or two staff with different permissions |
| A lost or stolen phone or tablet | Patient records in local storage, a signed-in session |
| A curious insider | Staff looking up a neighbor's or a celebrity's record |
| Another tenant | A user at clinic B probing identifiers that belong to clinic A |
| Automated abuse | Bots hitting endpoints that send OTPs or SMS, running up cost and harassing numbers |
| Supply chain | A compromised dependency in a frontend bundle |
| Configuration drift | A cloud database, bucket or key left more open than intended |

Note what isn't at the top: nation-state attacks on transport encryption. TLS solves transport.
Most healthcare breaches come from the list above.

## The legal frame in India

Not legal advice, but every team needs to know these instruments:
- **Digital Personal Data Protection Act, 2023, and its Rules (notified November 2025)**: notice
  and consent, purpose limitation, reasonable security safeguards, breach notification to the Data
  Protection Board and to affected individuals, and verifiable parental consent for children's
  data. That last one matters for every pediatric patient record.
- **ABDM's Health Data Management Policy**: consent-based sharing and data-minimization
  expectations for systems in the ABDM ecosystem.
- **The Aadhaar Act, 2016**, and UIDAI's rules: don't store Aadhaar numbers unless you must, and
  if you must, keep them in an Aadhaar Data Vault. The simplest compliant design never stores them.
- **CERT-In's 2022 directions**: report specified cyber incidents within six hours and retain
  system logs for 180 days. Your logging design has a legal deadline attached.

## Twelve rules, with the mistakes that taught them

### 1. Tenant identity comes from the verified session, never from the request
If an endpoint reads `clinicId` from the request body, a user can simply change it. ClinuxFlow's
API takes the clinic from the verified session token on every call.

### 2. Every lookup by id is also scoped by tenant
Taking the tenant from the token isn't enough if the query ignores it. `SELECT … WHERE id = ?` on a
multi-tenant table is a cross-tenant read waiting for someone to guess an id. Every read needs
`AND tenant_id = ?`, and every upsert must refuse to overwrite a row owned by another tenant. Ids
built from timestamps or account ids aren't secrets. Put this in code review checklists and write a
test per endpoint that tries another tenant's id.

### 3. The device is multi-tenant too
Our early local storage wasn't scoped by clinic, so on a browser shared by two clinics, one
clinic's records appeared in the other's lists. Every local record now carries its clinic, and
every local read filters by it.

### 4. A session is revocable, short-lived and hard to steal
Three properties, each of which we've had to think hard about:
- **Revocable.** A signed token that is only checked for signature and expiry stays valid after an
  account is disabled. When we built self-service joining (a new staff member redeems a token and
  waits for admin approval), our first version issued the pending account a session immediately.
  Login correctly refused pending accounts, but the already-issued token would have worked on every
  other endpoint, because the middleware never re-checked account status. We fixed that path by
  issuing no session until approval. The general fix is a server-side status or version check (or
  short token lifetimes plus refresh) so disabling an account takes effect immediately.
- **Short-lived.** Days-long tokens are convenient and widen every window.
- **Hard to steal.** A token in `localStorage` is readable by any script on the page, so one
  cross-site scripting bug is a full account takeover. HTTP-only cookies with proper CSRF defenses
  are the stronger default for web clients.

### 5. A key shipped to the browser is not a secret
Anything in a frontend bundle can be read by anyone who loads the page. A "service key" in the
bundle keeps out bots that don't bother to look; it does not authenticate users. Endpoints that
cost money or touch regulated systems (sending OTPs, calling a national registry) need a real
user session, or a short-lived token minted by a trusted backend for that user, plus per-user rate
limits.

### 6. Put regulated credentials in their own service
ClinuxFlow's ABDM integration runs in a separate service, the only component that holds ABDM
credentials or ever sees Aadhaar-linked payloads. Identifiers and OTPs are RSA-encrypted per
transaction with a public key fetched fresh from ABDM; Aadhaar numbers are never stored, only the
resulting identifiers (ABHA number, HPR ID, HFR facility id). Isolation shrinks what a compliance
review has to examine. Isolation plus rule 5 is what makes it safe.

### 7. Be precise about what encryption buys you
ClinuxFlow can move a record between devices as an encrypted string (a QR code or a chat
message). Each transfer uses a fresh AES-256-GCM key embedded in the string itself. That buys
tamper-evidence (any change makes decryption fail) and keeps plaintext out of the channel. It
doesn't buy access control: whoever holds the string can open it. The decision whether a
receiving clinic may import it is made after decryption, by an explicit consent check. Write down
what each use of cryptography protects against; teams often claim more than they get.

### 8. Relay, don't store, where you can
Staff chat is peer-to-peer over WebRTC. The server only relays the connection handshake and stores
no message content. When we needed durable delivery for join requests (the admin may be offline),
we queued only ciphertext, removed after the admin's client fetched and decrypted it. The server
never sees names or roles in plaintext.

### 9. Log reads, not just writes
The curious insider is the most common healthcare privacy incident, and it involves no write at
all. FHIR has `AuditEvent` for exactly this. A useful audit record says who, what record, when,
from where, and why (the workflow context). ClinuxFlow logs every workflow transition in an
append-only log mirrored to the cloud. Record-level read and edit auditing is still to come, and
it is the most important remaining piece of this list.

### 10. Protect the device you don't control
Local-first puts records on laptops and phones. Encrypt local data at rest where the platform
allows it, sign out on shared devices, and plan for remote wipe on managed hardware. The browser's
own storage isn't encrypted by the application; a packaged mobile app can use the platform
keystore.

### 11. Rate-limit anything that sends a message or costs money
OTP endpoints are abused for SMS pumping and harassment. Rate-limit per user, per phone number and
per IP, and alert on spikes.

### 12. Treat consent as data
A patient's consent is a record with a scope, a purpose and a period: FHIR `Consent`. Model it as
data your code checks, not a checkbox your UI shows.

## Breach readiness

Assume something will go wrong and rehearse it:
- Who decides it's a reportable incident, and how does the six-hour CERT-In clock start?
- What do your logs let you reconstruct, and are 180 days retained?
- How do you notify affected patients and the Data Protection Board?
- Can you revoke every session and rotate every key in an hour?

## Where ClinuxFlow is today

Built: tenant from the verified session on every API call; clinic-scoped local data; a separate
ABDM service holding all regulated credentials, with per-transaction encryption and no Aadhaar
storage; PBKDF2 password hashing; per-transfer authenticated encryption for device-to-device
records; peer-to-peer chat with a signaling-only server; an append-only workflow audit log; no
session for pending accounts.

Being closed before any clinic goes live: tenant scoping on every cloud lookup and write (rule 2);
real per-user authentication and rate limits in front of the ABDM service (rules 5 and 11);
session revocation and safer token storage (rule 4); record-level audit events (rule 9);
encryption of on-device data at rest (rule 10).

We would rather publish that second list than pretend it's empty.
