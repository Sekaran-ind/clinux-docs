# Specification 08: Cübo Agentic Harness + SNOMED + Enterprise — Consolidated Build & Test Sequence

## 1. Objective

Tie `SPEC-06-CUBO-AGENTIC-HARNESS.md`, `SPEC-07-SNOMED-CLINICAL-CHAT.md`, and SPEC-05's Enterprise tier into one build order with explicit dependencies and testing checkpoints. Each spec has its own internal phase list; none of them individually own the cross-spec sequencing, which is what this doc is for.

**Foundation already shipped, not part of this sequence**: P2P 1:1 chat (`userChats.js`/`p2pChat.js`/`TeamChat.vue`/`ChatSignalingRoom.js`), `requirePaidTier()` on the encounter-coordination routes, and per-clinic usage metering. Everything below builds on top of that, live-verified baseline.

**Pre-pilot status changes the testing bar, deliberately**: no production users or accumulated real data yet, so migration-safety and "prove nothing existing changed" concerns are lighter than they'd be post-launch — a phase can be reworked or a schema rebuilt outright rather than carefully migrated, if that's simpler. Checkpoints below still require real, live verification that new behavior is *correct* (this session's standing discipline) — they just don't need to additionally prove old behavior is byte-for-byte preserved the way a live production system would demand. Revisit this posture once there's a real pilot user base.

## 2. Sequence

Numbered = the critical path (each depends on the one before it). "∥" marks phases that can run in parallel with the critical path once their own prerequisite is met, not serialized behind everything above them.

### Phase 0 — Unblockers (start immediately, resolve before the phase that needs them)
- ~~SNOMED CT licensing~~ — **resolved for India geography**, via NRCeS/C-DAC (SPEC-07's Open Items). Remaining practical step, not a blocker: obtain the actual NRCeS release/tooling access ahead of Phase 4.
- ~~Supported specialty list~~ — **resolved**: General Medicine + 11-specialty cluster (Cardiology, Endocrinology, Gastroenterology, Hematology, Infectious Diseases, Neurology, Pulmonology, Nephrology, Rheumatology, Geriatric Medicine, Medical Genetics), sourced from schema.org's finite `MedicalSpecialty` taxonomy and cross-checked live against Wikidata + Wikipedia's India section (full list and sourcing in SPEC-07's Open Items). Unblocks Phase 1/4's content pre-build sub-tracks now.
- ~~`node-nlp`'s multi-label result shape~~ — **verified live**, and it changed Phase 2's plan: `classifications` exists but is softmax-normalized (scores sum to 1 across trained intents, confirmed: 0.6465+0.3535=1.0 on a mixed input) — independent per-intent thresholding doesn't work. Revised mechanism: split on conjunctions/clause boundaries, classify each clause independently through the existing single-intent classifier, union the results (SPEC-06 §5). Also surfaced a real, separate finding: the classifier shows zero discrimination on out-of-scope input with only 2 trained intents (confidently misclassified unrelated text) — flagged as a Phase 2 checkpoint item now, not a deferred concern.
- ~~Group-chat mesh vs. hub decision~~ — **resolved: full P2P mesh**, consistent with "nothing stored in the interim" being the feature's founding principle, not traded away for scale this app doesn't need at clinic-group sizes (2–8 people).
- Wikidata concept-matching disambiguation UX for the Designer flow (blocks Phase 1's forms-design call site)
- **Checkpoint**: none of these produce code — the checkpoint is a written decision/verification/agreement for each, not a test.

### Phase 1 — Wikidata design-time/onboarding semantic tagging (SPEC-06 §6, revised from "sentence-transformer")
- **Backend done and live-verified**: `WikidataTagging` (clinuxflow-api `src/lib/wikidataTagging.js`) + `GET /api/nlp/wikidata-search` / `GET /api/nlp/wikidata-concept`, KV-cached (`WIKIDATA_CACHE`), compliant `User-Agent`, 429/Retry-After handled explicitly. Two-step by design (search → confirm → getConcept) so nothing auto-picks a clinically-wrong match. Live-verified against the real Wikidata endpoint: a real cache-miss search took 619ms, the identical repeat call 3ms (cache hit confirmed by timing, not assumed); real concept lookup for Q11180 came back with zero aliases — an honest finding that not every concept has synonyms, not a bug to paper over.
- **Both call sites now built and live-verified end-to-end**:
  - **Designer.vue** — a "Suggest from Wikidata" button next to each field's keyword-training input (Step 2). Live-verified: created a real form, searched Wikidata for the "Full Name" field, confirmed a candidate, and the real returned aliases ("first and last name, personal name, orthonym, person's name, name, prosoponym") appeared in the keyword input.
  - **StaffOnboarding.vue** — a post-save "Tag your specialty" panel (runs after save, not during — `section_staff` is LForms-rendered, not a Vue template this file controls, so there's no safe way to inject into its live editing surface). Live-verified with a real specialty entry ("Cardiology"): the suggest button returned real Wikidata candidates, and confirming one **persisted correctly** to the actual saved record — `staff_specialty_wikidata_qid: "Q10379"`, `staff_specialty_wikidata_aliases: "cardiovascular medicine"` — read back directly from `localStorage`, not just observed in the UI.
  - One real test-mechanics finding along the way, not a product bug: the Specialty field is an LForms autocomplete (CODING datatype) — a plain programmatic `.fill()` never registers with LForms' own Angular change detection; only real keystrokes (`pressSequentially`) followed by clicking the actual autocomplete option persist a value at all. Worth remembering for any future LForms-driven live verification.
- Content pre-build batch for the 12-specialty list still not yet run.
- **Content pre-build sub-track (incremental, runs alongside the mechanism build, not gating it)**: once Phase 0's starting specialty list exists, batch-run Wikidata tagging against each supported specialty's common field vocabulary, so the first specialties that ship have real synonym coverage rather than an empty cache waiting to be filled organically. Expand specialty-by-specialty over time, not all-at-once.
- **Checkpoint**:
  - Unit tests for the cache-hit/cache-miss/429-retry logic.
  - Live test against the real Wikidata endpoint (not mocked) confirming a real query resolves and caches correctly.
  - Live Designer.vue test: tagging a field appends real synonyms to its harvested keywords, and `formSlotEngine.js` matching measurably improves on a paraphrase it previously missed.
  - Confirm zero bundle-size/runtime cost on Cübo itself — this phase touches Designer and onboarding, not the chat runtime path.

### Phase 2 — Multi-intent classification, interim mechanism (SPEC-06 §5, revised after Phase 0's live finding — clause-splitting, not per-intent thresholding)
- Build: split the utterance on conjunctions/clause boundaries; classify each clause independently through the existing single-intent classifier; union the results. Separately: address the out-of-scope-discrimination gap Phase 0 surfaced (broader/more diverse training data, explicit negative examples, or a top-2-score-margin check instead of a bare threshold).
- **Checkpoint**:
  - Fixture tests proving the concrete acceptance case: "generate the SOAP note and switch to cardiology" fires both intents.
  - A specific test for the out-of-scope finding: a clearly unrelated utterance no longer confidently misclassifies into a trained intent.
  - Spot-check existing single-intent cases still classify sensibly — pre-pilot, this doesn't need to be an exhaustive byte-for-byte regression suite, just confirmation nothing obviously broke.
  - Live Cübo test confirming both actions visibly fire from one message, not just that classification returns two labels.

### Phase 3 — Agentic harness dispatch layer (SPEC-06 §8.3, depends on Phase 2)
- Build: orchestration layer replacing `Cubo.vue`'s hardcoded chain; registers existing actions (`generate_soap`, `switch_room`, slot-fill, shortcut resolution) plus new ones (e.g. "message a colleague"); role/preference-aware, reading from Phase 1's onboarding tag.
- **Checkpoint**:
  - Integration tests proving every existing single-action behavior still works correctly under the new dispatch layer — pre-pilot, "still works" is the bar, not "produces byte-identical output to the old chain."
  - New test: one multi-intent message triggers multiple registered actions correctly, in the right order.
  - Live Playwright pass exercising a real multi-action message end-to-end in the actual Cübo UI.

### Phase 4 — SNOMED Part B (SPEC-07, depends on Phase 1 for its interim matching; licensing no longer a blocker, resolved via NRCeS)
- Build: pre-baked ECL-sliced subsets per role; interim keyword/NER matching enriched by Phase 1's Wikidata synonyms (not an embedding index — that's deferred to Phase 9); role-scoped subset loading; UI chips/tooltips/safety banner; local CDS rules table.
- **Content pre-build sub-track (incremental, same specialty list as Phase 1)**: ECL-slice the NRCeS SNOMED CT release per supported specialty, starting with whichever specialties Phase 0 agreed first — genuine clinical-content work, not just an engineering task, and reasonable to do a few specialties deep rather than all of them before anything ships.
- **Checkpoint**:
  - Unit tests against a curated phrase→expected-concept fixture set, per role.
  - A test proving role-scoping actually restricts results (a nurse's input never surfaces a surgeon-only concept).
  - Live test of the safety banner firing on the concrete example (a drug contraindicated by an active allergy logged in a different thread).
  - Bundle-size check — pre-baked subsets add real static weight even without an embedding model.

### Phase 5 ∥ — Group chat (SPEC-06 §4, mesh transport now resolved — unblocked, not dependent on Phases 1–4)
- Build: extend `ChatSignalingRoom.js` from its current 2-socket assumption to N sockets with broadcast relay; each client establishes one `RTCPeerConnection` per other participant (full mesh, per the resolved decision).
- **Checkpoint**:
  - Live multi-browser test (3+ real Chromium contexts via Playwright) — the same pattern already proven for 1:1 chat, extended to N participants: all participants receive all messages, join/leave is handled correctly.
  - Explicit confirmation "nothing stored in the interim" still holds for group messages, same as 1:1.

### Phase 6 — Unified inbox UI (SPEC-06 §8.5, depends on Phase 3 AND Phase 5)
- Build: merge `chatThreads.js` + `userChats.js` + group threads into Cübo's own thread list; `TeamChat.vue`'s functionality folds into `Cubo.vue`.
- **Checkpoint**:
  - Full live pass over Cübo's existing fragile-prone behaviors (slot-fill highlighting, pacer, slash shortcuts have each broken live before) — worth actually re-verifying given the history, even without full regression-suite formality pre-pilot.
  - Live Playwright pass confirming AI chat + 1:1 chat + group chat threads coexist correctly in one list.
  - Confirm encounter-scoped thread routing (the `encounterId` prop behavior Cübo already depends on) still works correctly.

### Phase 7 ∥ — Dentistry/optometry modules (SPEC-07 Part A/B, depends on Phase 4 for their SNOMED subsets, otherwise independent of Phases 5–6)
- Build: odontogram component (LHC-Forms-based, emits a SNOMED tooth-structure code on click); optometric structured parameter block (YAML form: Sphere/Cylinder/Axis/Add/VA OD/OS); urgent-routing rule wired into the existing `encounter_assignments` system.
- **Checkpoint**:
  - Live test: clicking a tooth on the odontogram injects the correct concept into chat context.
  - Live test: a simulated IOP-spike/severity value correctly triggers the existing routing/notification path, not a new one.

### Phase 8 ∥ — Enterprise tier plumbing (SPEC-05 §6, independent, can start anytime, needed before Phase 9)
- Build: `nano_dc_endpoint` migration on `clinics`; a `requireNanoDcAccess()` gate mirroring `requirePaidTier()`'s shape; admin path to provision shared vs. dedicated endpoint per clinic.
- **Checkpoint**:
  - Unit tests directly mirroring the existing `requirePaidTier()` test pattern: free clinic denied, Cloud-tier-without-endpoint denied, Cloud-tier-with-endpoint allowed.
  - Migration applied and verified against local D1, same live-apply discipline as every prior migration this session.

### Phase 9 — Enterprise-tier last features: sentence-transformer + nano-DC gateway (SPEC-06 §7 + SPEC-07 Part C, depends on Phase 8 and real Cloudflare Tunnel setup against the physical hardware)
- Build (two related pieces, same tier gate, natural to ship together):
  - **Sentence-transformer embedding layer** (SPEC-06 §7) — the client-side model deferred from Phase 1, now Enterprise-gated. Upgrades Phase 2's multi-intent and Phase 4's SNOMED matching from keyword/NER to embedding similarity, same consumers underneath.
  - **Nano-DC gateway** (SPEC-07 Part C) — new `clinuxflow-nano-dc-gateway` service; Cloudflare Tunnel connectivity; routes for MedGemma (text audit), MedCAT/medspaCy (if chosen for Phase 4's upgrade), MedSAM (imaging).
- **Checkpoint**:
  - Bundle-size/mount-stability check for the embedding model, same discipline originally planned for the old Phase 1 — just run here instead, since it's the same risk regardless of when it ships.
  - Confirm Phase 2/4's interim mechanisms still work correctly for non-Enterprise clinics, and that Enterprise clinics get the upgraded embedding-based results — this is a permanent tier split by design (not a migration-era compromise), so both paths genuinely need to keep working, even pre-pilot.
  - Live smoke test reaching each of the three nano-DC models through the real Tunnel from a real request — not mocked.
  - Concurrent-load test against the shared-mode capacity/queueing design (SPEC-07 §C.4's flagged open item).
  - One full real scenario end-to-end: a chat message → MedGemma audit → safety banner shown in the actual UI, not just an API response checked in isolation.

## 3. Summary table

| Phase | Depends on | Parallelizable with |
|---|---|---|
| 0 — Unblockers | — | everything (start now) |
| 1 — Wikidata tagging | Phase 0 | — |
| 2 — Multi-intent (interim) | Phase 0 (verify node-nlp shape) | — |
| 3 — Agentic dispatch | Phase 2 | — |
| 4 — SNOMED Part B (interim) | Phase 1, Phase 0 (specialty list) | Phase 5 |
| 5 — Group chat | — (mesh decision resolved) | Phases 2–4 |
| 6 — Unified inbox | Phase 3, Phase 5 | — |
| 7 — Dentistry/optometry | Phase 4 | Phases 5–6 |
| 8 — Enterprise plumbing | — | any phase |
| 9 — Sentence-transformer + Nano-DC gateway | Phase 8 | — |
