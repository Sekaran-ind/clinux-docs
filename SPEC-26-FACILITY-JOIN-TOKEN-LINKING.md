# Specification 26: Facility Join-Token Linking

## 1. Objective

Replace two separate, uneven ways a person ends up attached to a facility —
`POST /api/auth/invite` (staff: the admin invents the new teammate's password and hands it over
out of band) and `POST /api/facility/affiliates` (affiliate: the admin looks the person up by
email and 404s if they don't already have an account) — with **one** consistent, token-based,
self-service linking flow for both relationship types, gated by two real readiness checks and a
real admin-approval step.

Origin: a design discussion this session about whether QR-code hospital-profile sharing should be
replaced by login-time tokens. §2 below records what that discussion actually found — the
premise needed a correction before a design could be built on it.

**Status: BUILT** (revised from the original design-only pass, per explicit "build this now"
instruction once the approach was settled). §11 at the end of this document is the real build
report — what was built, what was live-verified against a real `wrangler dev` + D1, two real bugs
found and fixed along the way, and what's still not done (full browser E2E of the Cübo chat card,
actually provisioning the Cloudflare Queue resource). Sections §2-§10 below are kept as the
as-designed record; §11 is authoritative on what actually shipped where it differs.

**Revision** (mid-design, before anything was built): the original draft had the admin-approval
step as a server-side `facility_join_requests` queue + REST `approve`/`reject` routes. Explicit
instruction changed that: request **delivery and review** goes through the real, already-live
"LForms data via Cübo chat" mechanism (`sessionShare.js` + `CuboContactConversation.vue`'s
clinic-profile-transfer pattern) instead — judged more robust, since the admin reviews the
redeemer's actual mapped details inline, in context, rather than a separate queue page. §6, §8, §9
reflect this; §7 collapses to one table. A real structural gap this surfaced (contacts are
strictly already-linked people today, and P2P chat has no offline delivery) is documented in §6,
not glossed over — then solved for real in the build (§11) via Cloudflare Queues.

## 2. What already exists — the real starting point, corrected

Investigation before designing this found the user's framing was half right: real Cloudflare-
persisted entity data *has* already replaced device-to-device QR transfer as the primary
mechanism — but that happened for the **data** problem, not the **authorization** problem, and
it's already shipped:

- `StaffOnboarding.vue`'s `tryDirectProviderCompositionFetch()` already pulls the clinic's real
  Provider composition via an authenticated `GET /api/provider-composition`, deriving `clinicId`
  from the caller's own JWT. QR/chat-based sharing is already demoted to "a manual fallback...
  for the genuinely offline case" — this is done, not a gap.

The **actual** remaining friction is one layer up, in how a person gets *authorized* onto a
clinic's roster at all:

- **Staff** (`clinuxflow-api/src/routes/auth.js` `POST /api/auth/invite`, doc comment quoted
  verbatim): *"This app has no mail server, so there's no invite email/link: the inviting admin
  sets the new teammate's email+password directly... and shares it with them out of band."* The
  admin picking a stranger's password and texting it to them is the real UX wart — not QR codes.
- **Affiliate** (`clinuxflow-api/src/routes/control.js` `POST /api/facility/affiliates`, backing
  `clinux-frontend/src/stores/auth.js` `linkAffiliate()`): admin-initiated only, looks the
  practitioner up by email via `AccountsDb.getAccountByEmail`, and returns 404 if no account
  exists yet — there is no self-service join path for this relationship type at all today.
- Both are also capped and clinic-scoped already (`MAX_ACCOUNTS_PER_CLINIC` in `auth.js`;
  `clinicId` always taken from the caller's own JWT, never the request body) — real constraints
  this design keeps, not replaces.

This spec targets that authorization layer: **one token a facility issues, a person redeems (with
their own password, no account-inventing by an admin), gated on two readiness checks, finalized
only by explicit admin approval.**

## 3. The four requirements, as given

1. Remove the password field from the staff invite path — the joining person sets their own,
   same as anyone using the token would.
2. Make staff and affiliate linking **one** consistent mechanism, not two.
3. The token has an expiry and can be renewed periodically (not a one-shot, not permanent).
4. Redemption is gated (facility must have completed setup — §4) **and** requires explicit admin
   approval, with enough shown to the admin to judge whether the request is authentic.

## 4. `facilitySetupMachine` — built this pass

**Real, working code** (not a sketch): `clinux-frontend/src/data/control/facilitySetupMachine.js`
+ `facilitySetupMachine.test.js` (9 tests, all passing). A pure XState v5 machine, same
`createMachine`/inlined-guard idiom `src/workflow/planDefinitionRunner.js` already uses, but
deliberately **not** registered into `workflowRuntime.js`'s tracked Task/PlanDefinition runtime —
that runtime is reserved for clinical journeys (SPEC-23's "state machine only for clinical
journeys" boundary, which facility/staff registration was explicitly pulled out of). This machine
never receives an event and is never persisted; it classifies a snapshot of already-stored data
into a lifecycle stage the instant it's created.

States, driven by real signals already in `clinux-frontend/src/stores/onboarding.js`:

| Stage | Real signal |
|---|---|
| `draft` | `buildClinicProfile().name` empty — no profile saved yet |
| `basics_saved` | a name exists but `everPublished` is still false |
| `published` | `everPublished` is true (the existing "go live" gate) |
| `hfr_registered` | `published`, plus a real `hospital_facility_id` (written by `FacilityHfrPanel.vue` on a successful HFR submit) |

`canAcceptFacilityJoinToken(stage)` is `published || hfr_registered` — **not** full HFR
registration, deliberately, matching how the rest of this product already treats HFR as optional
supplementary registration, never required to operate (`checkConformance()`'s own "most clinics
won't see this go green... that's expected, not required to publish" precedent). Requiring real
HFR completion here would lock every free-tier clinic out of inviting anyone.

Wired into `onboarding.js` as three new exports (`facilitySetupStage`, `facilitySetupStageLabel`,
`canAcceptJoinToken`), recomputed on the same `dataVersion` reactive dependency
`buildClinicProfile()`/`publishedClinic` already use — and surfaced today as a real "Setup stage"
badge on `Onboarding.vue`'s Review card, so this isn't inert code even though nothing consumes
`canAcceptJoinToken` for actual gating yet (§8's redemption route is the real future consumer).

## 5. `personSetupMachine` — design only

The mirrored gate for the **redeeming person**, not just the facility. Two real complications
found while designing it, both worth stating plainly rather than forcing a false parallel with §4:

**No individual "publish" action exists today.** Facility has a real, single "Publish Clinic
Page" moment (`onboarding.js`'s `publish()`). An individual staff/affiliate record is just a
`section_staff` group instance inside the ONE shared Provider composition document — there's no
per-person save/publish step to key a state off. This machine's stages therefore key off
**account + identity signals**, not a publish click:

| Stage | Real signal |
|---|---|
| `draft` | account exists (`accounts` row), no declared name/contact beyond signup |
| `basics_saved` | `admin_name` present and a contact channel (email/mobile) reachable — the same fields already captured at registration |
| `published` | role-dependent — see below |

**Staff and affiliates need different bars for `published`, honestly.** An affiliate is by
definition a licensed practitioner; a staff hire may be purely administrative (reception, billing)
and will never hold an HPR ID. So:
- **Staff default bar**: `basics_saved` is enough — name + a verified contact channel. Requiring
  HPR here would incorrectly block the majority of real hires.
- **Affiliate default bar**: `published` requires a real, verified HPR login — i.e.
  `ProviderHprPanel.vue`'s already-built "log in as yourself" step (`myToken` set via
  `buildHprPasswordLoginBody`) succeeding. This reuses a genuinely real government-verified
  identity signal that already exists in this codebase, and doubles as the authenticity evidence
  §6's admin-approval step needs — a request from someone with a verified HPR login is
  materially more trustworthy than a bare email claim.

This machine is role-aware by construction (its `input` should include which `link_kind` is being
requested), not a single universal bar — same two-speed shape §4 already uses (a default bar
everyone can reach, a stricter one for where real government verification is meaningful).

## 6. `joinRequestMachine` — delivered and reviewed via the real Cübo chat/LForms mechanism (design only)

The lifecycle a single token redemption attempt goes through, from the moment someone uses a
token to the moment (if ever) a real `facility_affiliates` row or `clinic_id` reassignment exists.
Same states as originally drafted — `issued → redeemed → pending_admin_review →
approved/rejected/expired` — but **what each transition physically does** now rides on a real,
already-live mechanism instead of a new queue page, per explicit instruction.

```
issued ──(redeemed, passes personSetupMachine's bar)──▶ redeemed ──(LForms-mapped card
                                                                      sent + opened in
                                                                      admin's Cübo chat)──▶ pending_admin_review
                                                                                                    │
                                                       ┌─────────────────────────────────────────────┴──────────────────┐
                                                       ▼                                                                 ▼
                                                   approved                                                          rejected
                                          (real link created via                                              (no link created;
                                           POST .../decide:                                                    token stays
                                           facility_affiliates row,                                            consumed — admin
                                           or accounts.clinic_id                                               can issue a fresh
                                           reassignment for staff)                                              one)

issued ──(expires_at passes, never redeemed)──▶ expired
```

**The mechanism, concretely** — a new `kind: 'facility-join-request'` payload alongside
`sessionShare.js`'s existing `kind: 'provider-profile'`/`'encounter-session'` ones:

1. Once a redeemer's own client has validated the token (`POST .../redeem`, §9) and computed their
   own `personSetupMachine` stage, it maps their key facility/personal details — name, `link_kind`,
   declared role, the `personSetupStage` they redeemed at — onto answers against a small real
   Questionnaire (a new `system-join-request-v1`, same authoring pipeline SPEC-18 already built),
   not a bare JSON blob. That QuestionnaireResponse is what `buildJoinRequestSharePayload()` (new,
   mirrors `buildProviderProfileSharePayload()`) encodes via `sessionTransfer.js`'s existing
   `encodeSessionTransfer()` into one compact string.
2. Sent over the real P2P DataChannel exactly like `CuboContactConversation.vue`'s
   `shareProviderProfile()` does today (`session.value.send(key)`) — no new transport.
3. The admin's own `CuboContactConversation.vue` recognizes the prefix (mirroring `isTransferKey()`)
   and renders it as an actionable card instead of raw text — opening it renders the actual
   QuestionnaireResponse via `LhcFormHost.vue` (the same component that renders every other form in
   this app), with **Approve**/**Reject** buttons, not just an "Import" button. This is the "opens
   the form and approves or rejects" review experience, delivered inline in a tool the admin
   already uses daily, matching the reasoning already written for `shareProviderProfile()`'s own
   card.

**A real structural gap this surfaced — stated plainly, not glossed over.** `Cubo.vue`'s own
contacts list (`contacts.value = [...team, ...affiliates]`, sourced only from `auth.fetchTeam()`/
`auth.fetchAffiliates()`) is strictly people **already** linked to the clinic — a join-token
redeemer isn't a contact of the admin yet, and the admin isn't one of theirs, so neither side can
open a `CuboContactConversation` with the other through today's UI at all. Compounding it:
`p2pChat.js`'s own header is explicit that there is **no offline delivery** — "if the peer isn't
actively connected to the signaling room right now" a message simply doesn't arrive, no store-
and-forward. Two real, separate problems, not one:

- **Discovery**: `POST .../redeem`'s response (§9) carries the issuing admin's own `account_id`/
  `name`/facility name specifically so the redeemer's client can synthesize a one-off "pending"
  contact entry for that admin — `P2PChatSession`/`userChats.js` key purely by an account-id pair,
  not by a pre-existing `facility_affiliates`/team row, so this is a Contacts-*sourcing* change on
  both ends (the admin's Cübo needs to surface "pending join request from X" too, sourced from
  §8's token row, not from `fetchTeam`/`fetchAffiliates`), not a new chat primitive.
- **Offline delivery — Cloudflare Queues, not a bare retry-on-reconnect guess.** Revised from the
  original draft of this section, which left the retry trigger as an unresolved build-time
  question. A Cloudflare Queue is the right primitive, with one precision worth being exact
  about: no queue product makes a Worker "wait until a specific browser reconnects" — that's not
  what Queues do. What it actually buys is reliable, retried, at-least-once **ingestion into
  durable storage**, decoupled from whether the P2P session happened to be live at send time.
  Concretely: the redeemer's client still builds the exact same `encodeSessionTransfer()`-produced
  ciphertext string as the live-chat path (§6 step 1 above, unchanged) and *also* enqueues it (a
  new `JOIN_REQUESTS` queue binding, `{token, adminAccountId, ciphertext}`) alongside attempting
  live delivery. A consumer Worker durably writes that ciphertext onto the token row (§8's new
  `pending_payload` column) — Cloudflare's own retry/backoff covers transient failures on *that*
  write, which is the real guarantee, not a promise about the admin's browser. The admin's Cübo
  then picks up any `redeemed`-status token carrying a `pending_payload` on next load/reconnect
  and decrypts it client-side exactly as it would a live chat message — same review card either
  way, just sourced from a durable pull when live delivery didn't land in time.

**The actual authorization write is still a real server call, deliberately** — the two devices
share no database, and only the admin's own JWT may grant clinic membership. Clicking
Approve/Reject on the chat card fires `POST /api/facility/join-tokens/:token/decide` (§9) at that
moment. The chat/LForms mechanism is the **delivery and review UX** — familiar, in-context, no
separate queue page to remember to check, which is the real "more robust" this buys — not a
replacement for the write itself.

One more coherence point, **corrected from the previous draft of this section**: with the queue in
the picture, ciphertext *can* now rest briefly in `facility_join_tokens.pending_payload` (cleared
once the admin's client has pulled and decrypted it) — so "the rich content never touches D1 at
all" was too strong a claim and is retracted. What still holds, and is the property that actually
matters, is `sessionTransfer.js`'s own design intent: only **ciphertext** ever transits or rests
server-side (AES-GCM, tamper-evident, a fresh per-transfer key embedded in the payload itself, per
its own header comment) — plaintext name/role/`personSetupStage` details are never visible to
Cloudflare or D1 at any point, only to the two devices that hold the key. That's the real
guarantee worth stating precisely, not "zero server storage" — which this design no longer is.

## 7. Token expiry and renewal

Reuses the exact TTL primitive already proven twice in this schema — `encounter_assignments`'
`kind = 'lock'` rows (migration `0006`) and `task_locks` (migration `0010`, `docs/SPEC-25-
FEDERATED-TASK-PERSISTENCE.md` §6) — rather than inventing a third expiry mechanism:

- `expires_at`, nullable-cleared-not-deleted on consumption (same reasoning `task_locks`' own
  `release` uses: a consumed/expired token keeps its row as a record of what happened, rather
  than disappearing).
- A `renew` action extends `expires_at` from an admin action, only while the token is still
  `issued` (not yet redeemed/expired) — "periodically renewed" per requirement 3, not resurrecting
  an expired one; an admin renewing a dead token instead issues a fresh one, same "explicit
  recovery, not implicit resurrection" reasoning as rejected requests above.

## 8. Data model sketch (not created this pass)

**One** table now, not two — the revision in §6 means no request *content* is ever stored
server-side, so there's nothing left to justify a separate `facility_join_requests` table; it
collapses into a few extra columns/status values on the token row itself, which is purely an
existence-and-decision record:

```sql
-- migrations/0012_add_facility_join_tokens.sql (sketch)
CREATE TABLE facility_join_tokens (
    token TEXT PRIMARY KEY,                 -- short, human-typeable (e.g. 8 chars) + wrapped in a
                                             -- QR/link for convenience, reusing SessionShareModal's
                                             -- existing qrcode generation for a URL, not a data blob
    facility_clinic_id TEXT NOT NULL REFERENCES clinics(id),
    link_kind TEXT NOT NULL CHECK (link_kind IN ('staff', 'affiliate')),
    issued_by_account_id TEXT NOT NULL REFERENCES accounts(id),
    redeemed_by_account_id TEXT REFERENCES accounts(id),  -- set on redeem; who the admin's Cübo
                                                            -- should expect the LForms card FROM
    pending_payload TEXT,                    -- the SAME encodeSessionTransfer() ciphertext string
                                              -- the live P2P path sends — the Queue consumer's
                                              -- durable fallback write (§6), cleared once the
                                              -- admin's client has pulled + decrypted it. Never
                                              -- plaintext; see §6's corrected coherence note.
    status TEXT NOT NULL DEFAULT 'issued'
        CHECK (status IN ('issued', 'redeemed', 'approved', 'rejected', 'expired', 'revoked')),
    expires_at TEXT NOT NULL,
    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    decided_by_account_id TEXT REFERENCES accounts(id),
    decided_at TEXT
);
```

**Queue binding** (new, no existing binding to generalize — `clinuxflow-api/wrangler.toml` has D1/
AI bindings today but no Queue): a `JOIN_REQUESTS` producer/consumer pair,
`[[queues.producers]]`/`[[queues.consumers]]` in `wrangler.toml`, the consumer handler living
alongside the other `src/lib/*-db.js` modules (e.g. `src/lib/join-tokens-db.js`) doing the single
`UPDATE facility_join_tokens SET pending_payload = ? WHERE token = ?` write §6 describes.

`facilitySetupMachine`'s gate is checked at **issue** time (an admin can't generate a token for an
unpublished facility at all — fail fast, don't let a token exist that can never legally be
redeemed) and again at **redemption** time (the facility's stage could have regressed — not
possible today since `everPublished` is one-way, but worth the belt-and-suspenders check since
this route runs server-side, not trusting client state).

## 9. Route sketch (not built)

- `POST /api/facility/join-tokens` — issue (`requireUser()`, checks the caller's own facility
  against `facilitySetupMachine` server-side before minting a token; `link_kind` in the body).
- `POST /api/facility/join-tokens/:token/renew` — extend `expires_at` (§7).
- `POST /api/facility/join-tokens/:token/redeem` — **replaces both `/api/auth/invite`'s
  password-setting and `/api/facility/affiliates`' email-lookup.** Body: `{ email, password,
  adminName }` for a new account, or nothing extra if the caller is already authenticated (an
  existing affiliate practitioner linking to an additional facility). Validates the token and
  `facilitySetupMachine`'s gate, marks the token `redeemed` with `redeemed_by_account_id` set, and
  returns the issuing admin's `account_id`/`name` + facility name — just enough for the redeemer's
  client to open a chat with them (§6's discovery fix). It does **not** create the
  `facility_affiliates` row or reassign `clinic_id`; that only happens on `.../decide` below.
  Request *content* (name, role, `personSetupStage`) is never in THIS call's body — the client
  builds that payload afterward, once it has the admin's identity back from this response.
- `POST /api/facility/join-tokens/:token/deliver` — body `{ ciphertext }`, the SAME
  `encodeSessionTransfer()` string the client sends over live P2P chat (§6). Called unconditionally
  right after building the payload, whether or not the live send also succeeds — enqueues onto
  `JOIN_REQUESTS` (§8) rather than writing `pending_payload` synchronously itself, so a transient
  D1 failure doesn't drop the redeemer's only durable copy. Idempotent by `token` (a redeemer
  reopening the app and calling this again just overwrites the same `pending_payload`, not a
  growing list — there is only ever one live request per token, since a token is single-use).
- `POST /api/facility/join-tokens/:token/decide` — `{ decision: 'approved' | 'rejected' }`,
  `requireUser()` restricted to the token's own `facility_clinic_id` admin. The real mutation
  point, fired from the chat card's Approve/Reject buttons (§6): on `approved`, creates the
  `facility_affiliates` row or reassigns `clinic_id` for `redeemed_by_account_id`, exactly the
  same write `linkAffiliate()`/`inviteTeammate()` already make today, just reached from a chat
  card instead of an admin-typed form. This is where `POST /api/auth/invite` can be retired
  outright — its password-setting job moves into `.../redeem`, its "who gets added to my clinic"
  decision moves here.

## 10. What this replaces, once built

- `POST /api/auth/invite` — retired. Staff join the same way affiliates do: token in, own
  password, admin approval via chat.
- `POST /api/facility/affiliates` — retired as the *only* path; kept, if at all, as an
  already-know-each-other's-accounts admin shortcut, or dropped in favor of always going through
  the token flow for consistency (requirement 2). Worth deciding at build time, not guessed here.
- `MAX_ACCOUNTS_PER_CLINIC` and clinic-scoping (`clinicId` from the caller's JWT, never the
  request body) carry over unchanged — real constraints this design keeps, not removes.
- No admin-approval queue page — the chat/LForms card in `CuboContactConversation.vue` *is* the
  review surface (§6), so there is no `GET /api/facility/join-requests`-shaped route to build at
  all, and one fewer table than the original draft.

## 11. Build report (this pass)

**clinuxflow-api** — `migrations/0012_add_facility_join_tokens.sql` (the `facility_join_tokens`
table exactly as §8 describes, plus `provider_composition.published_at` and `accounts.status`,
both real additions found necessary while building, not in the original §8 sketch — see below).
`src/lib/control/join-tokens-db.js`, `facility-setup-stage.js` (the server-side mirror of
`facilitySetupMachine.js`, a plain function rather than a ported XState machine — no existing
xstate dependency on the Worker side, and pulling one in for a four-branch classifier wasn't
worth it), `cubo-task-queue-consumer.js`. Six routes in `src/routes/control.js`: `POST
/api/facility/join-tokens` (issue), `GET /api/facility/join-tokens` (list, management UI), `GET
/api/facility/join-tokens/pending` (§6's discovery fix, joined with redeemer identity), `POST
.../renew`, `POST .../redeem`, `POST .../deliver`, `POST .../decide`, `GET .../payload`.

**Update, later in the same session**: the Cloudflare Queue is now real — `cubo-task-queue`
(explicit go-ahead, after confirming Standard Queues pricing: 10,000 ops/day included free,
$0.40/million after), `wrangler deploy --dry-run` confirms the `CUBO_TASK_QUEUE` binding resolves
against it. Named generically, not `join-request-delivery` — the user's own framing (§12) is that
Cübo should eventually dispatch EVERY P2P-chat-triggered action off a real Task specification, not
one hardcoded handler per feature, so the queue that carries those actions shouldn't be named
after the first one. The consumer (`cubo-task-queue-consumer.js`) is now a small `type -> handler`
dispatch table with exactly one registered handler (`'facility-join-request'`) and a real
unknown-type path (retries, doesn't crash the batch) — genuinely ready for a second type to
register later, not a cosmetic rename. `.../deliver` tags its message with that same `type`.
Deploying the Worker itself is still a separate action, not done.
`POST /api/auth/invite` and `POST /api/facility/affiliates` deleted outright (§10) — nothing
called either anymore once the new flow's own UI replaced their only callers. 341/341 backend
tests passing (up from 296 before this session's SPEC-25 pass); the whole staff+affiliate lifecycle
— publish-gate, issue, redeem, pending-blocks-login, decide-approve, login-succeeds, and the
affiliate path through to a real `facility_affiliates` row — was live-verified end to end against
a real `wrangler dev` + local D1 with curl, not just unit-mocked.

**Two real bugs found and fixed during that live verification, neither of which the mocked unit
tests caught**:
1. `JoinTokensDb.decide`'s SQL had 4 `?` placeholders but only 3 bound values — `D1_ERROR: Wrong
   number of parameter bindings`, only surfaced by an actual D1 call.
2. **A real security gap**: the original design (§9) had `.../redeem` mint a session token for a
   brand-new (`'pending'`) staff account immediately. Since `requireUser()` only validates a JWT's
   signature/expiry and never re-checks account status against D1 on every request, that token
   would have passed on every OTHER authenticated route (team list, provider-composition, ...)
   even though `POST /api/auth/login` correctly refused to issue a *new* one for a pending
   account. Fixed by never mining a session token for a newly-created account at redeem time at
   all — found by continuing to reason through the redeem→pending→approve chain after the happy
   path already worked, not by a test failing.

**clinux-frontend** — `data/control/personSetupMachine.js` (§5, built — the role-aware two-speed
XState machine, staff bar = basics, affiliate bar = a real verified HPR login via
`ProviderHprPanel.vue`'s existing "log in as yourself" step). `data/control/joinTokenAdapter.js`
(all 8 routes above). `data/sessionShare.js` gained the `facility-join-request` payload kind
(`buildJoinRequestSharePayload`/`decodeJoinRequestShareKey`/`joinRequestAsSyntheticRecord`),
mapped onto a real new compiled form — `tools/system-forms/system-join-request-v1.yaml`
(`clinuxflow-api`), `resourceType: Consent` (a genuine semantic fit — FHIR's own "record of a
choice to permit or deny... performance of one or more actions" — using real, dictionary-confirmed
leaf paths, not guessed; `Basic` was tried first and rejected by the compiler as an un-indexed
resourceType). `stores/onboarding.js` now pushes to `PUT /api/provider-composition` from both
`publish()` and `saveProviderRecord()` — a real, separate gap found while wiring the server-side
gate: that route existed with exactly this stated purpose since migration 0005 but nothing ever
called it. `components/TeamSettingsModal.vue` rebuilt — both Staff and Affiliate tabs now share
one "Join Links" panel (issue/copy/renew), gated on `onboarding.canAcceptJoinToken`; the old
password-invite and email-lookup forms are gone. `components/auth/JoinTokenRedeemForm.vue` (new,
standalone — deliberately *not* wired into `entryFormRegistry.js`/`entryWorkflow.js`'s tracked
runtime, same "state machine only for clinical journeys" reasoning that already excludes
onboarding), reachable from `Index.vue` (a "Join Code" button, an entry in the Register/Login
modals, an authenticated "Join Another Facility" menu item, and `?join=TOKEN` prefill for a shared
link). `components/Cubo.vue`'s contacts list now also sources `listPendingJoinRequests()`,
rendering a visually distinct "pending join request" entry. `components/cubo/
CuboContactConversation.vue` gained the actual review panel (§6) — fetches the durably-delivered
ciphertext, decodes it, renders it via the same `LhcFormHost.vue` every other form in this app
uses, Approve/Reject calling `.../decide`. `stores/auth.js`'s `inviteTeammate()`/`linkAffiliate()`
removed (no callers left). 264/264 frontend tests passing; a full `npm run build` compiles clean.

**Not done this pass, stated plainly**: full browser (Playwright) verification of the Cübo
chat-card review flow end to end (two real accounts, live redemption, opening the pending contact,
approving) — the backend routes it depends on ARE live-curl-verified individually, and the whole
frontend compiles and unit-tests clean, but the two-account/two-browser-context UI path itself
wasn't clicked through live, a real, named limitation rather than an assumed success (same honesty
precedent as SPEC-24's own Patient/ABHA panel note). The `cubo-task-queue` Cloudflare Queue
resource IS now provisioned (explicit go-ahead, see the update in §11 above); deploying the Worker
itself is the one piece still not done.

## 12. Cübo as a Task-driven event bridge (raised, not yet designed)

Real critique, same session, of `CuboContactConversation.vue`'s own shape: the pending-join-
request handling is a hardcoded `v-if` branch with direct imports (`getJoinRequestPayload`,
`decodeJoinRequestShareKey`, `decideJoinToken`, ...) — fine for one action type, not for "many
actions triggered in a P2P share mode," the user's own framing for where this is headed. The ask:
Cübo should be a generic event bridge dispatching off a real Task specification (this codebase's
own SPEC-13 Task/PlanDefinition language), not accreting one new hardcoded branch per feature.
`cubo-task-queue`'s naming and its dispatch-table consumer (§11's update) are provisioned toward
this already; the actual generic-card-renderer redesign in `CuboContactConversation.vue` itself is
real, substantial future work — not designed or built in this pass.
