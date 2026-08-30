# Specification 09: ABDM-Anchored Registration & Citizen Health Record Storage

## 1. Objective

Replace the current double-modeled Provider/Staff/Facility registration path — a generic YAML-authored form compiled to a FHIR `Questionnaire`, rendered by LForms, extracted, then hand-translated by `abdmAdapter.js` into ABDM's own bespoke request shapes — with a single canonical schema anchored directly on what HFR/HPR/ABHA actually require. This is the concrete answer to "is rebuilding the onboarding journey a planned item" — it now is, and this spec is the plan.

**The failure this fixes is not hypothetical.** Live-verified this session: a realistic user interaction (type a value into a coded/autocomplete LForms field, save without explicitly clicking a dropdown suggestion) silently discards the value — confirmed by reading the persisted record back out of `localStorage` after a real save. That bug exists specifically because Provider/Staff data is forced through a generic forms-rendering system built for arbitrary clinic-authored content, not for a fixed external schema. Removing the mismatch removes the bug class, not just this one instance of it.

## 2. The canonical schema — anchored on ABDM's field requirements, not ABDM's raw request shapes

Two ways to "anchor on ABDM" were considered; this spec picks the second deliberately:

- ❌ Store ABDM's literal request bodies as the canonical local record — zero mapping at submission time, but couples local storage to a schema that's externally versioned and, per `abdmAdapter.js`'s own header comment, "intentionally inconsistent... between endpoints" (e.g. `demographic-auth-mobile` wants `mobileNumber`, `mobile-otp` wants `mobile`). Building the whole app's storage around that is a second, incompatible modeling convention on top of everything else.
- ✅ **A clean, FHIR-consistent canonical schema whose field set is driven by ABDM's requirements** — collect exactly what HFR/HPR/ABHA need, nothing the current generic form adds beyond that — while staying compatible with SPEC-05 §4's reference-only model the rest of the app already follows. The submission step becomes a thin, near-1:1 mapping instead of today's real translation.

The starting field set isn't invented — it's already been reverse-engineered once, in `abdmAdapter.js`'s existing builders (verified against the real gateway routes): `hospital_name`, `hospital_ownership_code`, `hospital_facility_type`, `hospital_state_lgd_code`, `staff_first_name`, `staff_hp_category_code`, `staff_abdm_role`, and so on. The canonical schema is this same field list, promoted from "fields an adapter happens to read" to "the actual schema," with the generic composition's extra/unmapped fields dropped.

## 3. Scope — Provider/Staff/Facility only, not a rewrite of the forms system

The generic YAML → FHIR `Questionnaire` → LForms pipeline stays exactly as-is for Encounter, Vitals, SOAP, and custom clinic-authored forms — that's a legitimately different problem (arbitrary, clinic-defined structure) where the current architecture is the right fit. Only registration data — which has a *fixed, external* schema now — moves to a dedicated, purpose-built flow with controlled inputs (real dropdowns/searchable selects, not LForms' coded-autocomplete widgets), the same pattern SPEC-07's ABHA/HPR/HFR journeys already used. This sidesteps the coded-field data-loss bug for this data by construction, not by patching LForms' extraction behavior.

**The LForms coded-field bug itself stays open as a separate item** for wherever the generic system is still used (Encounter/Vitals fields with coded answers) — this rebuild doesn't touch that; it needs its own fix.

## 4. Free-tier data authority — two models, one already built

- **Local guardian + LAN sync**: the Administrator's device is authoritative; staff sync while on the clinic's LAN. This is the *existing* Tauri shared-server architecture (`sharedServerSync.js`, `ConnectionStatusControl.vue`) — no new build, just the correct framing of who's authoritative when it's live.
- **Admin's local device + Cübo Chat propagation** (no LAN server): the admin holds Hospital/Staff data locally and propagates it to each staff member via chat. Concretely, this reuses the *existing* session-transfer encoding (`main.js`'s AES-GCM+gzip scheme, already surfaced through `SessionShareModal`/`SessionImportModal`) — the encoded string is sent as a chat message over the already-built P2P DataChannel, and the recipient gets an "Import this data" action recognizing the same blob format `SessionImportModal` already decodes. Chat is an *added* delivery option, not a replacement — if the admin and staff member aren't online at the same time, the identical payload still works via the existing QR/paste fallback, so chat's "nothing stored in the interim" constraint doesn't turn into a hard requirement for both parties to be simultaneously present.

## 5. Citizen health record storage — DigiLocker, not ClinuxFlow

Extends the "don't be the permanent custodian" principle (chat's ephemeral design, ABDM's federated model) to citizen data: a citizen's health records live in **their own DigiLocker**, a real, government-operated, already-trusted service — not a database ClinuxFlow builds, hosts, or bears liability for.

Verified directly (PIB press release + corroborating coverage, not assumed): DigiLocker completed a "Level 2" ABDM integration making it a genuine **Health Locker / PHR app** within the ABDM ecosystem, with both directions real — pull (linking/accessing records from ABDM-registered facilities) and push (citizens scan/upload their own records, and share selected records out to registered professionals).

This splits into two tiers of effort matching what's already planned, with **no separate DigiLocker certification needed at either level**:

- **Free tier / near-term**: ClinuxFlow generates a well-formed PDF with an embedded QR encoding the record. The citizen uses DigiLocker's *own* existing "scan and upload" feature to store it. Zero DigiLocker API integration — buildable now, no external dependency.
- **Enterprise tier**: once ClinuxFlow is HFR-registered (already the Enterprise-tier gate per SPEC-05 §6), records can flow into a citizen's DigiLocker-as-PHR-app automatically via ABDM's standard consent/HIE exchange — again not a separate DigiLocker integration, just a consequence of the HFR work already scoped.
- **HAPI FHIR** (already named in SPEC-07 Part A's enterprise vision) stays explicitly deferred — a possible later upgrade for structured, queryable enterprise-tier storage, not a near-term decision.

**Open, not yet decided**: whether the QR embedded in the citizen-facing PDF reuses the same AES-GCM+gzip encoding already built for device-to-device transfer, or needs a distinct format appropriate for a document presented to an unrelated third-party provider. These likely should be two different mechanisms (internal app-to-app transfer vs. an externally-verifiable citizen document) rather than the same encoding serving both purposes — a real decision, not yet made.

## 6. Relationship to what already exists

- `AbdmOnboarding.vue` / `abdmAdapter.js` — the field-mapping logic here is the direct ancestor of this spec's canonical schema; not thrown away, promoted.
- `StaffOnboarding.vue`'s "My Staff Details" drawer (LForms-based) — replaced by the new dedicated flow for the fields this schema covers.
- `Onboarding.vue`'s Hospital Profile drawer — same; the Hospital/Facility portion moves to the new flow, while anything outside ABDM's scope (branding, marketing copy for the public ClinicHome page) stays in the existing local-first system, since ABDM has no opinion on it.

## 6a. Status — §2, §3, and the chat-propagation half of §4 are built and live-verified

- **Canonical schema** (§2): `src/data/abdmSchema.js` — `HOSPITAL_FIELDS`/`STAFF_FIELDS`, same linkIds `abdmAdapter.js` already reads, tested against exactly those linkIds so the existing HFR/HPR builders keep working unchanged.
- **Dedicated registration flow** (§3): `AbdmFieldForm.vue` (controlled `<select>`/`<input>`/checkbox, no LForms) + `appendGroupInstance()` (new, in `formData.js`) replacing LForms extraction. Wired into `StaffOnboarding.vue`'s "My Staff Details" drawer, which now shows only Staff fields — the Hospital Details section is gone from this page, since editing the hospital record was never this self-service page's job to begin with.
  **Live-verified with the exact scenario that found the bug**: a real login, plain realistic typing into Specialty (no dropdown-clicking needed at all — it's a native input now), save. The record persisted correctly: `staff_specialty: "Cardiology"`. The coded-field data-loss bug is gone for this flow, by construction.
- **Chat-based propagation** (§4, second half): `TeamChat.vue` gained a "Share Clinic Profile" button (calls `buildProviderProfileSharePayload`, sends the resulting key as a plain chat message) and renders any received transfer-key message as an actionable "Clinic profile transfer" card with an Import button (`decodeProviderProfileShareKey` → `isSameClinic` → `onboarding.importProviderProfile`) instead of raw ciphertext. **Live-verified with two real browsers**: Alice added a real staff entry via the fixed form, connected to Bob over a real P2P DataChannel, shared the profile, Bob imported it — his device's `localStorage` then held a real Provider record with Alice's staff data pulled across. No new transfer protocol was needed; this reused `sessionShare.js`/`sessionTransfer.js` exactly as they already existed.
- **Generic-system LForms coded-field fix** (§3's own "stays open" callout): fixed and live-verified, without moving Encounter/Vitals/custom forms off LForms. `useSystemForms.js` now attaches a capturing `focusout` listener per form container that catches a coded field's typed value *before* the widget's own blur handler wipes it (root cause turned out to be an active clear-on-blur, not just a non-commit), then `extractResponse()` grafts any still-missing field/group nodes back into the extracted response as a recovered `valueString`. See [[clinux-lforms-coded-field-data-loss-bug]] for the full mechanism and 3-path live verification (recovered fallback, exact-match auto-confirm, explicit click — all pass, no regressions).
- **UX pass**: `abdmSchema.js`/`AbdmFieldForm.vue` gained field grouping (`section`), plain-language `help` text, and inline per-field validation (fires on blur, or all-at-once via `touchAll()` when a blocked Save is attempted) — the concrete answer to "is the new onboarding actually very user friendly," live-verified in `StaffOnboarding.vue`'s drawer.
- **DigiLocker PDF+QR** (§5, free-tier path): built and live-verified. `src/data/digilockerExport.js` — reuses `sessionShare.js`'s existing `buildEncounterSharePayload()` transfer encoding for the embedded QR (resolves §7's open item in favor of reuse, not a new format), jsPDF visit summary matching `ConsultationDesk.vue`'s existing prescription-PDF conventions. Wired into `Checkout.vue` as a "Health Record (DigiLocker)" button. Live-verified full Front Desk → Checkout → download flow with real data; PDF text content and embedded QR image object both confirmed present and correct. Along the way, found and fixed a real QR-raster-density robustness gap (bumped `width: 320→480` in both this file and the pre-existing `SessionShareModal.vue`) and a real, separate free-tier regression that blocked reaching Checkout at all — see [[clinux-digilocker-pdf-qr-export]] and [[clinux-checkout-free-tier-lock-bug]].
- **Not yet built**: the local-guardian LAN-sync half of §4 (already exists as `sharedServerSync.js`, just needs no new work); the Hospital-side dedicated flow (only Staff is done — Onboarding.vue's Hospital Profile drawer still uses the old LForms path, deliberately deferred per §7).

## 7. Open items

- ~~QR format for the citizen-facing DigiLocker document~~ — resolved: reuses the device-transfer encoding. See §6a.
- Exact canonical field list still needs completing against the real HFR "Additional/Detailed Information" steps (`buildHfrAdditionalInfoBody`/`buildHfrDetailedInfoBody` are still stubs — `{ trackingId }` only, per the existing adapter's own "known-gap" comment) and the full HPR/ABHA field set, not just what's already been reverse-engineered.
- ~~The LForms coded-field data-loss bug remains open for the parts of the app still on the generic forms system~~ — fixed, see §6a.
- New, not yet decided: whether to raise the shared QR generation's `errorCorrectionLevel` from `'L'` to `'M'` for better real-world scan reliability at the cost of a lower max-capacity ceiling before falling back to text-only — measured as a real improvement in testing but affects `SessionShareModal.vue` broadly, not just this spec's scope. See [[clinux-digilocker-pdf-qr-export]].
