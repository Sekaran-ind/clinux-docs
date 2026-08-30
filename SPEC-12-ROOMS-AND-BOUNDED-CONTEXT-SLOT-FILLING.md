# Specification 12: Rooms & Bounded-Context Slot-Filling — Cübo as the Central Interface Over a Composition Graph

## 1. Objective

Reframe the application's navigation model: instead of a static page/route sequence with data captured incidentally along the way, each stable page ("room" — Front Desk, Consultation, Checkout, Hospital Profile, and any future one) is a bounded context anchored to its own compiled FHIR document, and Cübo's existing chat-based slot-filling becomes the primary way to navigate and fill it, scoped strictly to that document's own fields. Nothing in this spec is built yet except where explicitly marked. It originated from live defects found this session while reviewing running screenshots of `clinux-frontend` (`localhost:5173`), not from abstract design first — §3 is the grounding evidence, §4 is the design it motivated.

## 2. Terminology correction — verified this session, not assumed

The working conversation that produced this spec used "FHIR Composition" and "section.section" as the mental model throughout. Checking the actual compiler changes the precise vocabulary, not the design:

- `clinuxflow-api/src/lib/yaml-to-questionnaire.js:82-134` — a YAML form's `composition:` array is an **authoring-time list of resource blocks** (each targeting one FHIR resource type, e.g. Organization, PractitionerRole). The compiler flattens each block into a `sectionGroupItem` and pushes it onto **one compiled `Questionnaire`'s `item` array** (line 153-302). There is no runtime `Composition` resource produced anywhere in this pipeline — "composition" here names the YAML input shape, not the FHIR resource type.
- What's actually canonical per room/section is a compiled **`Questionnaire`** (the form definition, versioned in `formsLibrary.js`) plus its captured **`QuestionnaireResponse`** (the data, currently flattened via `getAnswer`/`getAnswers`-style readers per `abdmSchema.js`'s own header comment).
- This doesn't change anything structurally: `Questionnaire.item` nests recursively (`item.item`) exactly like `Composition.section.section` would have — the parent-child section design in §4.2/§4.5 holds unchanged, it's built on `Questionnaire.item` nesting, not `Composition.section`.

Everywhere below, "the room's document" means a compiled `Questionnaire` + its `QuestionnaireResponse`, not a `Composition` resource instance.

## 3. Defects found this session — the grounding evidence

Found live in `localhost:5173`'s running app (Clinic Home, `/designer`'s Forms Library → Hospital Profile), screenshots reviewed directly, not reported secondhand. Two were confirmed against real source this session; two remain open findings.

- **Defect 1 — Care Team record count is correct, the record's own name/photo never render on the public page.** Clinic Home's hero says "1 care team member," Designer's Forms Library card badge agrees ("Care Team — 1 added"), but the public "Our Medical Experts" section renders a blank avatar and an empty name. **Verified this session**: `onboarding.js:44-47` — `publishedClinic` is already a live `computed(() => buildClinicProfile())`, not a stale copy; the earlier staleness bug this pattern describes was already fixed (see `clinux-cross-clinic-leakage-stale-cache-fix` memory). `ClinicHome.vue:475-486`'s Care Team template (`v-for="s in clinic.staff"`, `{{ s.name }}`, `s.name?.charAt(0)`) is structurally sound. **Not yet located**: the actual gap is inside `buildClinicProfile()` itself (`onboarding.js:168`) — its staff-array construction isn't populating `name`/photo correctly for at least one real record. This is a live-view *mapping* bug, distinct from the staleness bug already fixed — proof that fixing "copy vs. view" doesn't automatically fix every propagation defect underneath it.
- **Defect 2 — stray unstyled placeholder** below the "Meet Our Experts" hero card, no label, no content. Cosmetic, unlocated this session.
- **Defect 3 — revised from the live diagnosis given mid-session.** Originally read as "the Hospital Profile section contains wrongly-scoped Practitioner fields." **Verified this session, and this changes the finding**: `abdmSchema.js:41-100`'s `HOSPITAL_FIELDS` (the canonical schema SPEC-09/SPEC-11 built) is correctly Organization-scoped only — every field is `hospital_*` under `section_hospital` / `section_hospital_abdm_facility_type` / `section_hospital_abdm_location`. The Practitioner-shaped "Staff Details" fields seen in the screenshot (Full Name, Qualification, License, Role, Specialty, "First Name (as on Aadhaar)") are not sourced from this schema — `staff_first_name`/`staff_last_name` live in a separate `STAFF_FIELDS` block (`abdmSchema.js:104-106`) used by the purpose-built `HospitalOnboarding.vue`/`AbdmFieldForm.vue` flow, not by Designer's generic Forms Library card. **The real finding**: the screenshot's Designer → Forms Library → "Hospital Profile" drawer is exercising a *different, older* capture path (almost certainly the generic `system-provider-composition-v1.yaml`, LForms-rendered) than the newer canonical one SPEC-09/11 built. **Two parallel, unreconciled Hospital Profile capture surfaces exist in the app right now** — this supersedes the original framing and is elevated in §7 below.
- **Defect 4 — confirmed real and current, independent of which surface is used.** `abdmSchema.js:98-100` — `hospital_state_lgd_code` and `hospital_district_lgd_code` are declared `type: 'text'`, plain free-text, despite each field's own `help` string naming the exact HFR/LGD master list it's meant to resolve against. Compare `hospital_ownership_code` at line 69, correctly `type: 'select'` — the schema already distinguishes coded from free-text fields elsewhere, these two just weren't given the treatment. This is in the *current, canonical* schema, not legacy code — needs the modal-resolution fix (§4.3) regardless of which capture surface eventually wins in §7's reconciliation.

## 4. Target design

### 4.1 Live view, not materialized copy
Already the right instinct and already partially proven in this codebase: `publishedClinic`'s fix (§3, Defect 1) turned a stale snapshot into a live computed. Generalize this as a standing rule for every consumer of a room's document — no page holds its own derived/cached copy of another room's data; it reads live, filtered for what it's allowed to see.

### 4.2 Bounded context = the active section, with a scope stack
Chat/slot-filling's write scope is the currently active `groupLinkId`/section — not the whole graph. Out-of-scope input gets an "unactionable here, want me to switch you to \[section\]?" response, never a silent guess into the wrong group. Read access can reach across the whole document (and across rooms, §4.5) so the assistant isn't amnesiac about already-captured values; only writes are scope-limited. Because sections nest (§2), scope is a **stack**, not a flat pointer — e.g. Branch A → Staff → a specific practitioner's Services.

### 4.3 One shared modal-resolution primitive, three consumers
A single reusable component, not three bespoke ones:
- **Coded-value confirm** — any field whose schema declares a closed vocabulary (Defect 4's LGD/HFR-master fields, `type: 'select'` fields generally) always routes here; raw free text is never accepted as the final value, confidence-gated or not. Typed chat text pre-filters the option list rather than being discarded. Cascading fields (state → district → sub-district) must be linked, not independent pickers. This also closes the existing `clinux-lforms-coded-field-data-loss-bug` — a modal that blocks on explicit confirm is strictly stronger than a dismissible dropdown overlay.
- **Ambiguous-slot disambiguate** — when an utterance or value plausibly matches more than one candidate slot/section, show candidates with context (reuse Forms Library's existing card subtitles as disambiguating text) and require a single canonical target — never write to two sections for one fact; use cross-section/cross-room reference (§4.1, §4.5) instead.
- **Record-instance disambiguate** — same component, applied to collection-type sections (Care Team has many practitioners; Legal Consents has many consent types) when a write needs to target one existing record among several.

### 4.4 Metrics across all four resolution outcomes
Log every modal show/resolution and every scope-boundary decision:
- unambiguous-fill rate (happy-path health)
- coded-confirm abandon rate (master list too big/poorly filtered)
- disambiguate resolution skew (repeated non-default resolution of the same field ⇒ real schema-boundary defect, same shape as Defect 3's finding — fix the schema, don't keep asking users forever)
- unactionable recurrence, **deduped by distinct session with a semantically-clustered pattern** (one confused user retrying isn't a signal; several different users independently wanting the same missing field is) — a genuine "this needs to become a real field in the schema" signal, same logic as failed-search-query-driven catalog expansion in other domains.

### 4.5 Room = its own compiled Questionnaire + QuestionnaireResponse
Front Desk, Consultation, Checkout, and any future room each get their own document, following §4.2's recursive section pattern internally — same primitive as Provider's Hospital Profile/Care Team/Office Hours/Legal Consents, not a special case per room. Consultation's section graph is **specialty-parameterized**, not fixed — open item, §8. Cross-room data (Consultation needing Front Desk's demographics, Checkout needing both) is referenced, never copied — same rule as §4.1, applied one level up. Rooms remain real, addressable, resumable, printable pages/routes; Cübo (§4.8) is an additional entry point onto them, not a replacement for them.

### 4.6 Room lifecycle: draft / published / complete, as two independent axes
`QuestionnaireResponse.status` is the native FHIR home for this (`in-progress | completed | amended | entered-in-error | stopped`), and it maps cleanly onto **one** of the two axes actually needed:
- **Visibility** ("can another room or the public page trust this data yet") — draft → published. FHIR's native status has no middle "published-but-still-being-worked" value; this needs a small extension/flag orthogonal to `status`, not a repurposing of it.
- **Workflow completion** ("has this room's job finished, so the next room's precondition is satisfied") — this is what `QuestionnaireResponse.status`'s `in-progress`/`completed` natively expresses.
Keep both as one ordered progression (draft < published < complete, completion implies published) for simplicity, but document that they answer different questions for different consumers — don't let a later cleanup collapse them assuming redundancy. Also: don't reuse the word "local" for the draft state — the app already ships an unrelated "Local Only" sync-scope toggle (device-local vs. LAN/cloud-synced data residency); lifecycle stage and data residency are orthogonal and must stay separate properties.

### 4.7 Declared preconditions between rooms, not a hardcoded route order
"Not linear" must not mean "no sequencing" — Checkout billing before a consultation happened, or a room unlocking before required consent, are real constraints to keep. Express them as declared preconditions ("Room X requires Room Y's document at `status ≥ published` for section Z"), filtered in at read time (§4.1), rather than a fixed router sequence.

### 4.8 Cübo as central orchestrator, not sole surface
Once §4.2-§4.7 are proven, Cübo becomes the default entry point — it tracks scope, resolves ambiguity, and decides what reveals — but dense, high-volume surfaces (AG Grid tables, forms) stay first-class for exactly the workflows chat is wrong for (a front-desk clerk during a walk-in surge). This is the destination §6 of `SPEC-06-CUBO-AGENTIC-HARNESS.md` already named (multi-intent classification, semantic layer, agentic dispatch); this spec is the concrete substrate that makes it safe to build on, arrived at by fixing the defects in §3 first rather than building the harness ungrounded.

## 5. Relationship to existing specs

- `docs/SPEC-06-CUBO-AGENTIC-HARNESS.md` — §4.8 here is that spec's "target" realized on top of this one's primitives; the agentic dispatch layer it describes is the eventual consumer of §4.2-§4.4.
- `docs/SPEC-08-CUBO-BUILD-SEQUENCE.md` — this spec's §7 build order needs threading into that doc's phase table once work starts; not done yet.
- `docs/SPEC-09-ABDM-ANCHORED-ONBOARDING-REBUILD.md` / `docs/SPEC-11-ABDM-M1-M4-ALIGNMENT.md` — `abdmSchema.js`'s `HOSPITAL_FIELDS`/`STAFF_FIELDS` and the `AbdmFieldForm.vue` controlled-input pattern are the canonical-schema precedent §4.3's modal primitive builds on; §3's Defect 3 finding is a direct consequence of that work not yet having replaced the older generic Designer path.
- `docs/SPEC-05-DATA-TIER-AND-ABDM-BOUNDARY.md` — any tier-gating implications of Cübo-as-central-interface (§4.8) are unexamined here, open item.

## 6. Consolidation finding that changes the priority order

§3's Defect 3 revision is the most consequential thing found this session: **two parallel, unreconciled Hospital Profile capture surfaces exist** — Designer's generic Forms Library card (LForms-rendered, presumed backed by `system-provider-composition-v1.yaml`) and the purpose-built `HospitalOnboarding.vue`/`AbdmFieldForm.vue`/`abdmSchema.js` flow SPEC-09/11 already built specifically to fix the LForms coded-field data-loss bug. Building bounded-context scoping (§4.2) on top of a still-duplicated capture surface means the assistant would inherit "which of the two Hospital Profile forms is canonical" as a standing ambiguity, on top of every ambiguity this spec is designed to resolve. This needs reconciling first, or at minimum first in §7's order, not discovered as a surprise mid-build.

## 7. Build order

1. **Reconcile the two Hospital Profile capture surfaces** (§6) — confirm the generic Designer path's actual source, decide whether Forms Library's card routes to `HospitalOnboarding.vue` or is retired.
2. **Find and fix Defect 1's root cause** inside `buildClinicProfile()` (`onboarding.js:168`)'s staff-array construction.
3. **Build the shared modal-resolution component** (§4.3) — fixes Defect 4 immediately in whichever surface wins step 1, and closes the existing LForms coded-field bug as a side effect.
4. **Wire bounded-context scope** (§4.2) using native `Questionnaire.item` nesting, on the now-reconciled, now-correctly-mapped Provider document.
5. **Instrument metrics** (§4.4) across all four resolution outcomes — cheap, rides on step 3's modal event points.
6. **Generalize to rooms** (§4.5-§4.7) — Front Desk/Consultation/Checkout each their own compiled document, cross-referencing not copying, declared preconditions instead of hardcoded route order. Prove on Provider's document first; this is the largest single step, don't start it before 1-5 are stable.
7. **Cübo becomes the central interface** (§4.8) — only once rooms and the resolution primitives are live-verified; this is the point where `SPEC-06`'s agentic dispatch layer has real ground to stand on.

## 8. Open design questions

- Exact mapping gap inside `buildClinicProfile()` causing Defect 1 — not located this session, needs direct inspection.
- Exact source of the "Staff Details" fields rendered inside Designer's Hospital Profile drawer — presumed `system-provider-composition-v1.yaml`, not confirmed this session.
- Where the real HFR/LGD master lists (states, districts, sub-districts, facility-types, `OWNER` codes) should be sourced from for §4.3's coded-value picker — `abdmAdapter.js` may already fetch/cache some HFR master data; not checked this session.
- Whether existing `groupLinkId` sections already map 1:1 to Forms Library cards cleanly enough for §4.2, or need restructuring first.
- How Consultation's specialty-parameterized section graph (§4.5) gets selected/generated — whether `yaml-to-questionnaire.js` needs new capability for this or it's composable from existing pieces.
- Any practical nesting-depth limit in `yaml-to-questionnaire.js`'s current `item` handling — untested at more than the existing shallow grouping.
- Tier-gating implications of Cübo-as-central-interface (§4.8) against `SPEC-05`'s boundary — unexamined.
