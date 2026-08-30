# Specification 05: Data Tier & ABDM Product Boundary

## 1. Objective

Give one explicit, authoritative answer to "where does this data live, and why" — replacing the ad hoc, feature-by-feature placement decisions that have been producing real bugs (e.g. ClinicHome falling back to a hardcoded demo clinic instead of a registered clinic's real data, because its profile lived in a different tier than its staff roster, with no stated rule for which one wins). This spec is the reference point for the three ABDM onboarding journeys (ABHA, HPR, HFR) and every future feature that touches storage placement. It supersedes case-by-case judgment calls made earlier in the project.

## 2. The three storage tiers

- **Local-first** (per-device TanStack DB / `localStorage`) — always present, zero network dependency, visible only to the device it's on.
- **LAN-shared** (Tauri live-server) — a shared store reachable by every device on the clinic's own network. Same mechanism as local-first, just backed by a server on the LAN instead of `localStorage`.
- **Cloud** (D1 in clinuxflow-api; R2 if a content-plane blob store is ever needed) — durable, reachable from anywhere with internet, the only tier that survives a device being off or off-site.

## 3. Master-data principle

Cloud/D1 identity (`accounts`, `clinics`) is root identity — real the moment an account is registered, regardless of connectivity mode. Local/LAN Provider-Profile data (Hospital Profile drawer, Team roster display) is *enrichment* layered on top of that root identity — never the reverse. A missing enrichment record should fall back to root identity, not to a hardcoded placeholder.

This principle governs the interim state only. Once the HFR journey (§5) exists, facility identity fields migrate to being sourced from the facility's own HFR record instead — see the open item in §8.

## 4. Reference-only data model

Every cross-record relationship — Encounter→Patient today, and Encounter→ABHA / Practitioner→HPR / Organization→HFR going forward — is a **one-directional stored reference**. Reverse lookups (all encounters for a patient, all facilities a professional is affiliated with) are always a computed filter/query, never a stored back-link. This is what keeps the data graph acyclic by construction, with no risk of the embedded-recursion problem that prompted this rule. Already the pattern in `clinical.js`'s `patientRef` / `encounter_patient_ref`; extend new ABDM identifiers the same way rather than inventing a different convention.

## 5. ABDM's role: federated identity, not operational storage

HFR/HPR/ABHA are identity/registry data — who is this facility/professional/patient, and are they verified. They are not, and per ABDM's federated architecture cannot be, a store for operational/clinical workflow data (active encounters, worklist assignment, vitals, SOAP notes, billing). Each HIP (each clinic) retains custody of its own visit records permanently; ABHA is the correlating identifier across clinics, not a data store; the Consent Manager is a discovery-and-consent layer, not storage.

Consequences:

- Local/LAN/cloud tiering for encounter/vitals/SOAP/billing data (§2–3) is **unaffected by ABDM** and stays exactly as designed — none of it competes with what HFR/HPR/ABHA do.
- ABDM adds a **new reason** cloud durability matters, distinct from internal multi-staff coordination: a clinic that completes HFR registration takes on a standing duty to serve future HIU pull requests reliably, at any future time — a local-only or offline device cannot do this.
- Health-record exchange itself (HIP push / HIU pull, consent artifacts) stays explicitly out of scope for this spec, same as `clinuxflow-abdm-integration-approach.md` — anticipated by the tier boundary in §6, not designed here.

## 6. Three pricing tiers, one additive model

**Free tier:**
- Local-only and/or LAN-shared (Tauri) storage only. No cloud durability requirement.
- Multi-staff coordination via LAN-shared mode is included — the gate is "no cross-location reach, no HIE participation," not walk-in vs. ongoing-patient relationship. A multi-staff, LAN-only specialty clinic is free tier just as much as a single-visit walk-in clinic.
- Patient **ABHA** lookup/creation is available — a citizen convenience that creates no ongoing obligation on the clinic, since an unregistered facility's records aren't HIE-discoverable regardless.
- Staff **HPR** registration is available — a personal, portable professional credential, independent of the clinic's own tier.
- Facility **HFR** registration is **not** available — this is the one registration that creates a standing duty to durably serve future HIU requests, which free tier cannot promise.

**Cloud tier** (`clinics.tier = 'paid'` — the existing `requirePaidTier()` gate, unchanged) **adds:**
- Cloud-durable storage (encounter data, provider profile become cloud-authoritative), enabling cross-location staff coordination — the `encounter_assignments` / lock system — beyond a single LAN.
- Facility **HFR** registration and ongoing, reliable participation as a real ABDM HIP.
- Cloud-durability and ABDM-HIP compliance obligations are bundled together for v1 rather than sold as separable add-ons. Revisit if a customer wants cloud reach without taking on ABDM registration overhead.

**Enterprise tier** = Cloud tier **plus** a provisioned nano data center (see `docs/SPEC-07-SNOMED-CLINICAL-CHAT.md` Part C — MedGemma/MedCAT-medspaCy/MedSAM real-time audit and safety warnings). Modeled additively, not as a third value on `clinics.tier` — a Cloud-tier clinic gains Enterprise capability the moment it has a `nano_dc_endpoint` provisioned (a new nullable field on `clinics`; null = no nano-DC access). This keeps the existing binary `requirePaidTier()` gate untouched for everything already built, and adds a second, independent gate for Part C's routes specifically.
- **Shared**: multiple enterprise clinics' `nano_dc_endpoint` all point at the same, centrally-operated cluster. Real capacity consideration, not assumed away: GPU inference isn't instantly parallel across tenants, so a shared deployment needs a request queue/fair-scheduling plan once more than one clinic uses it concurrently.
- **Dedicated**: an enterprise clinic gets its own physically separate hardware, with its own endpoint — `nano_dc_endpoint` points at that clinic-specific address instead.
- Which one, and at what price, is a sales decision per customer ("agreed price model") — architecturally both are the same code path, just a different value in one column.

## 7. Compliance containment

All Aadhaar/ABDM-adjacent PII handling — RSA encryption, OTP flows, audit logging, DSC certification scope — stays entirely within paid-tier code paths (`clinuxflow-abdm-gateway`, and the HFR journey specifically). Free tier never exercises this surface at all, which keeps its security/compliance review scope minimal by construction, not just by policy.

Same discipline applies to Enterprise tier's nano-DC surface: unlike chat signaling (which only ever relays connection-establishment metadata, never content — §9's sibling spec), Part C's requests carry real clinical text/audio/imaging across the Cloudflare Tunnel boundary to the nano DC. That's a real PII/PHI transit surface and needs the same encryption-in-transit, minimal-retention, access-logging treatment as the ABDM flows above — not assumed safe just because the compute happens on owned hardware rather than a third party's.

## 8. Open items

- **What triggers local-only vs. LAN-shared/cloud mode**: an earlier draft of this spec proposed a staff-count rule (1 account = local-only forever, 2+ accounts = server-mode becomes necessary). Retracted as too brittle — a second account doesn't reliably signal a real coordination need (e.g. a backup/admin login, a temporary contractor), and the rule forced a mode switch off a roster change rather than an actual usage need. Deferred; revisit with a better signal later.
- **Registration cutover scope**: once all three ABDM journeys are complete, the plan is full replacement of the current non-ABDM registration flow — but does that apply even to facilities that would otherwise stay local-only/free tier, forcing ABDM registration regardless of how they operate? Or does the free/local-only path stay exempt from *registration* even post-cutover, with ABDM only required to reach paid-tier capability? Unresolved — needs a decision before the actual cutover, not before starting the journey builds.
- **ClinicHome's facility-identity fallback** (currently the `DEMO_CLINIC` / `buildSeedFromRegistration()` question) should be re-scoped against HFR once that journey exists per §3, rather than patched under the pre-ABDM model.

## 9. Repo structure for the ABDM journeys

HPR and HFR already share one page (`AbdmOnboarding.vue` in clinux-frontend) against the isolated `clinuxflow-abdm-gateway` backend, which is the only component holding ABDM credentials. The ABHA (patient) journey follows the same pattern — a new route inside clinux-frontend, not a separate deployable project — because:

- Credential isolation is already satisfied by the gateway split (backend), independent of which frontend calls it; a separate frontend project buys no additional security containment.
- The scoped use case is assisted enrollment during a clinic visit (Front Desk), not a standalone citizen self-service product.
- It reuses existing infrastructure (`LhcFormHost`, `formData` collections, the shared visual system) instead of duplicating it across two repos.

A standalone citizen-facing ABHA/PHR product (public self-service, no clinic affiliation required, its own auth model, broader security-hardening needs for public Aadhaar input) would be a legitimate trigger to spin out a separate `clinux-shelf` project — but that's a different product with a different audience than assisted enrollment, and should wait for that need to actually materialize rather than being built ahead of it.
