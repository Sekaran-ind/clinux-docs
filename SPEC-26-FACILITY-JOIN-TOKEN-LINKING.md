# SPEC-26: Facility Join-Token Linking

| | |
|---|---|
| **Status** | Built for three relationship kinds (staff, affiliate practitioner, affiliate organization). Not done: a two-browser UI walkthrough of the Cübo approval card, deploying the Worker with its queue consumer. Known defects in §11.3. |
| **Last reviewed** | 2026-09-26 |
| **Code (API)** | `migrations/0012` (tokens, `accounts.status`, `provider_composition.published_at`), `0014` (backfill), `0015` (organization kind, `facility_organization_affiliates`); `src/lib/control/{join-tokens-db,facility-setup-stage,cubo-task-queue-consumer}.js`; `src/routes/control.js` (`/api/facility/join-tokens/*`, `/api/facility/organization-affiliates*`); `wrangler.toml` (`CUBO_TASK_QUEUE`) |
| **Code (frontend)** | `src/data/control/{facilitySetupMachine,personSetupMachine,joinTokenAdapter}.js`, `src/components/control/JoinLinkPanel.vue`, `src/components/auth/JoinTokenRedeemForm.vue`, `src/components/cubo/CuboContactConversation.vue`, `src/data/sessionShare.js`, `clinuxflow-api/tools/system-forms/system-join-request-v1.yaml` |
| **Related** | SPEC-05 §6.4, SPEC-06 §3, SPEC-09 §4, SPEC-11 §9, SPEC-24 |

## 1. Purpose

Replace two uneven ways of attaching a person to a facility (an admin inventing a teammate's
password and passing it on out of band; an admin looking up an existing account by email) with
**one** self-service, token-based flow, gated on readiness and finalized by explicit admin
approval. Later extended to facility-to-facility partnerships.

## 2. Starting point

Data sharing was already solved: staff on another device pull the clinic profile from
`GET /api/provider-composition` (SPEC-09 §4). The friction was authorization: how someone gets onto
a clinic's roster. Constraints kept: `MAX_ACCOUNTS_PER_CLINIC`, and `clinicId` always taken from
the caller's token.

## 3. Requirements

1. The joining person sets their own password.
2. Staff and affiliates join through one mechanism.
3. Tokens expire and can be renewed.
4. Redemption requires a ready facility and explicit admin approval, with enough detail shown to
   judge authenticity.

## 4. Facility readiness: `facilitySetupMachine`

A pure XState classifier over stored data (never persisted, never sent events, and not in the
workflow runtime, per SPEC-23): `draft` (no name) → `basics_saved` (named, not published) →
`published` → `hfr_registered` (published plus a real HFR facility id).
`canAcceptFacilityJoinToken` is true from `published`. HFR is not required, because most free
clinics will never register. The server mirrors this in `facility-setup-stage.js` (a plain
function; no XState on the Worker) using `provider_composition.published_at`, and checks it at
both issue and redeem.

## 5. Person readiness: `personSetupMachine`

There is no per-person publish step, so stages key off account and identity signals, and the bar
depends on the relationship:
- **Staff**: `basics_saved` (a name and a reachable contact) is enough. Many hires are
  administrative and will never hold an HPR ID.
- **Affiliate practitioner**: `published` requires a verified HPR login (`ProviderHprPanel`'s "log
  in as yourself"). This doubles as authenticity evidence for the admin.

## 6. The join request: delivered and reviewed in Cübo chat

Lifecycle: `issued → redeemed → (admin reviews) → approved | rejected`; `issued → expired`;
`revoked`.

1. The redeemer's client validates the token (`.../redeem`), maps their details onto a small
   compiled form (`system-join-request-v1`, `resourceType: Consent`, a real fit for "a record of a
   choice to permit an action"), and encodes the response with `sessionTransfer.js` (AES-GCM, fresh
   key per transfer).
2. It sends the ciphertext over the live P2P channel and also calls `.../deliver`, which enqueues
   `{type: 'facility-join-request', token, ciphertext}` on `cubo-task-queue`. The consumer writes
   the ciphertext to `facility_join_tokens.pending_payload`. The queue guarantees durable ingestion
   when the admin isn't online, not delivery to a browser.
3. The admin's Cübo lists pending requests as distinct contacts (`GET .../pending`), fetches the
   ciphertext (`GET .../payload`), decrypts it client-side, and renders it with `LhcFormHost` with
   Approve and Reject buttons.
4. Approve or Reject calls `POST .../decide`, the only place membership is granted.

Only ciphertext ever rests server-side, briefly; plaintext name, role and stage never reach D1.

Discovery problem solved: contacts used to be only already-linked people. `.../redeem` returns the
issuing admin's identity so the redeemer's client can show a pending contact, and the admin's side
sources pending requests from the token table.

## 7. Expiry and renewal

`expires_at` on every token. `renew` extends it only while the token is still `issued`; a dead
token is replaced, not revived. Consumed or expired tokens keep their row as a record.

## 8. Data model

`facility_join_tokens`: `token`, `facility_clinic_id`, `link_kind` (`staff | affiliate |
organization`, widened in 0015 by table rebuild), `issued_by_account_id`, `redeemed_by_account_id`,
`pending_payload`, `status` (`issued | redeemed | approved | rejected | expired | revoked`),
`expires_at`, `decided_by_account_id`, `decided_at`.

Outcomes on approval:
- **staff, new account**: `accounts.status` `pending → active` (a pending account cannot log in).
- **staff, existing independent practitioner**: a `facility_affiliates` row (account `clinic_id`
  is immutable). Before this fix, approval was a silent no-op; migration 0014 backfilled the rows
  those approvals should have written.
- **affiliate**: a `facility_affiliates` row.
- **organization**: a `facility_organization_affiliates` row linking two clinics (`status active |
  revoked`), managed at `/api/facility/organization-affiliates`. ABDM registration status is
  independent of the link.

## 9. Routes

| Route | Purpose |
|---|---|
| `POST /api/facility/join-tokens` | Issue (caller's facility must be published) |
| `GET /api/facility/join-tokens` | List for management |
| `GET /api/facility/join-tokens/pending` | Pending requests with redeemer identity |
| `POST .../:token/renew` | Extend expiry |
| `POST .../:token/redeem` | New staff: email, password, name (no session is issued). Existing account: bearer token. Returns the issuing admin's identity. |
| `POST .../:token/deliver` | Enqueue the ciphertext |
| `GET .../:token/payload` | Admin fetches the ciphertext |
| `POST .../:token/decide` | Approve or reject (admin of that facility only) |

## 10. What it replaced

`POST /api/auth/invite` is deleted. `TeamSettingsModal.vue` is deleted; its join-link UI became
`JoinLinkPanel.vue`, embedded in Onboarding's Care Team and affiliate sections and in
`FacilityStatusCard`. The store's `inviteTeammate()` and `linkAffiliate()` are gone. The redeem form
is reachable from Index (a Join Code entry, the register and login modals, an authenticated "Join
Another Facility" item, and `?join=TOKEN` links). It is deliberately not part of the tracked entry
plan.

## 11. Build report

### 11.1 Verified
The staff and affiliate lifecycle (publish gate, issue, redeem, pending blocks login, approve, login
succeeds, affiliate row written) was verified end to end with curl against `wrangler dev` and local
D1. Two bugs found only there: a SQL statement with four placeholders and three bound values; and
a real security gap: `.../redeem` minted a session for a pending account, and because
`requireUser()` never re-checks account status, that token would have worked everywhere. Fixed by
never issuing a session at redeem.

### 11.2 Queue
`cubo-task-queue` is provisioned (Standard Queues: 10,000 operations a day free, then $0.40 per
million). The consumer is a `type → handler` table with one handler and a retry path for unknown
types, named generically because more chat-triggered actions will follow (§12). The Worker has not
been deployed with it.

### 11.3 Known defects (2026-09-26 review)
1. **New staff accounts get the `hospital_admin` role.** `createTeammateAccount` doesn't set
   `role`, so the column default applies (SPEC-11 §9).
2. **`POST /api/facility/affiliates` still exists** and accepts writes, although nothing calls it
   and this spec says it was retired. Delete it or restrict it to the decide path's internal use.
3. The general session-revocation gap (SPEC-01 §10) means an approved-then-removed member keeps a
   working token for up to seven days.

## 12. Cübo as a Task-driven event bridge (not designed)

The review UI in `CuboContactConversation.vue` is a hard-coded branch with direct imports, which is
fine for one action type and wrong for many. Target: chat-triggered actions dispatch off a Task
specification, with a generic card renderer. This should be the same registry as SPEC-06 §3's
dispatch layer. The queue is already named and structured for it.
