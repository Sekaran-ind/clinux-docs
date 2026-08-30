# Specification 07: SNOMED CT-Coded Multidisciplinary Clinical Chat

## 1. Objective

Give clinical chat (Cübo's agentic harness, per `docs/SPEC-06-CUBO-AGENTIC-HARNESS.md`) semantic grounding in SNOMED CT: role-scoped clinical vocabulary per specialty, cross-disciplinary translation between roles, background contraindication/safety checks, and two independent modules (dentistry, optometry/ophthalmology) with their own anatomical/procedural vocabulary.

Recorded in two parts, per explicit decision: **Part A** is the full spec as proposed — the enterprise-tier aspiration, assuming dedicated new infrastructure. **Part B** is the reconciled, near-term-actionable path that fits ClinuxFlow's actual architecture (Cloudflare Workers edge, local-first TanStack DB, no hosted always-on servers today) and SPEC-06's semantic layer. Part B is what's buildable without a separate infrastructure commitment; Part A is what a genuine enterprise/hosted deployment tier could grow into.

---

## PART A — Full spec, as proposed (enterprise tier)

### A.1 Architecture

| Component | Technology | Purpose |
|---|---|---|
| Terminology Engine | Snowstorm (SNOMED International FHIR server) | Ontology lookups, concept expansion, updates |
| API Standard | HL7 FHIR Terminology Services (`$expand`, `$lookup`, `$closure`) | App backend ↔ terminology engine |
| NLP Pipeline | MedCAT, Clinithink, or cloud healthcare NLP APIs | Real-time entity extraction/linking from chat text |
| CDS Logic Layer | CQL + SNOMED value sets | Real-time rule evaluation, cross-thread contradiction checks |
| Data Persistence | OMOP CDM or FHIR-native store (e.g. HAPI FHIR) | Structured JSON chat payloads with semantic bindings |

**This is a genuinely different operational model than anything running today** — Snowstorm is a hosted Java/Elasticsearch service, not something that runs on Workers; OMOP/HAPI is a separate persistence system from the TanStack DB/D1 model SPEC-05 already reconciled. Treat this column as "what a dedicated hosted deployment would run," not an extension of the current stack.

### A.2 Role-based ontology slicing (ECL)

- **Surgeons**: Procedures (`71388002`), Morphologic abnormalities (`49755003`), Body structures (`123037004`)
- **Attending/resident doctors**: Full clinical findings/disorders (`64572001`), Pharmaceutical/biologic products (`373873005`)
- **Nurses**: Nursing care activities, Observable entities (`363787002`) — vitals, pain tracking, administration entries
- **Lab technicians**: Specimen (`123038009`), specimen collection procedures, analytical observables
- **Imaging technicians**: Diagnostic imaging (`363679005`), imaging findings, anatomical focus areas
- **Dietitians**: Dietary procedures (`22943007`), nutritional observables, dietary substances/formulas (`226887002`)
- **Admin**: Automated SNOMED→ICD-10/CPT/OPCS mapping from chat audit logs, for coding/billing compliance

### A.3 UI/UX

- **Intent-aware input**: async NLP as the user types; recognized terms shown as subtle interactive chips; raw concept IDs hidden, preferred terms shown by default.
- **Cross-disciplinary tooltips**: hovering a tagged term shows its SNOMED preferred term and "is a child of" parent hierarchy — bridges vocabulary gaps between specialties.
- **In-chat safety banners**: an unmissable inline alert when background logic detects a conflict (e.g. a drug contraindicated by an active condition/allergy logged in a *different* thread).

### A.4 Core workflow

- **Background CDS**: active threads monitored against SNOMED logical relationships (finding site, causative agent, using substance).
- **Handover validation**: shift-change/inter-department threads checked against a SNOMED-backed template before sign-off is allowed.
- **Audit logging**: message text + author role + structured SNOMED bindings + timestamps + thread IDs, for QA/analytics.

### A.5 Independent module: Dentistry

- **Ontology**: dental structures (maxilla/mandible/individual teeth, FDI numbering mapped to SNOMED tooth structures, e.g. `113317003`), dental procedures (`12779000` — endodontics, periodontics, orthodontics, restorative), dental findings/materials (caries `53931007`, restorative materials `418464009`).
- **Odontogram-linked threads**: clicking a tooth on a visual dental chart auto-injects the matching SNOMED anatomical concept into the chat context.
- **Material compatibility alerts**: a planned restorative material is cross-checked against the patient's coded allergy record.

### A.6 Independent module: Optometry & Ophthalmology

- **Ontology**: eye structures (descendants of `39006000`), ophthalmic procedures (`103175005` — refraction, visual field testing, tonometry), visual findings/refractive errors (descendants of `404684003` — myopia, hyperopia, astigmatism, presbyopia).
- **Structured optometric parameter blocks**: a dedicated input template for Sphere/Cylinder/Axis/Add/Visual Acuity OD/OS, mapping to SNOMED observables.
- **Cross-specialty urgent routing**: a progressive retinal finding or IOP spike auto-triggers a routed alert to the attending ophthalmologist/emergency stream, using SNOMED severity modifiers.

---

## PART B — Reconciled, near-term path (fits the current architecture)

| Part A component | Lean equivalent |
|---|---|
| Hosted Snowstorm + `$expand`/`$lookup`/`$closure` | **Pre-baked, versioned SNOMED CT subsets** — just the ECL-sliced concepts each role actually needs (§A.2/A.5/A.6), bundled as static reference data and refreshed periodically. Same "cache reference data, don't hit a live server per request" principle already used for ABDM master data. No live terminology server. |
| MedCAT/Clinithink/cloud NLP | **Interim, revised**: the same node-nlp/NER matching SPEC-06 §5 uses for multi-intent, against SNOMED concept labels enriched with SPEC-06 §6's Wikidata-sourced synonyms — no embedding model, free/Cloud-tier-safe. **Enterprise tier upgrade**: once SPEC-06 §7's sentence-transformer lands, SNOMED matching moves to a proper concept-embedding index (preferred term + synonyms embedded once offline, messages matched at send-time against the role-scoped subset). One semantic system serving both general intent-matching and SNOMED linking at that point, not two pipelines — just not from day one. |
| CQL + value sets | **A small local rules table** (plain structured data: concept X contraindicated with active concept Y) evaluated against the patient's already-known structured record. The one concrete example given (drug vs. active allergy) is a bounded lookup, not a case for a general clinical-rules runtime yet. |
| OMOP CDM / HAPI FHIR | **No new persistence system** — a `snomedConcepts: [{code, term}]` field added to existing message records (`userChats.js`/`chatThreads.js`), following SPEC-05's reference-only, already-tiered model. |
| Role-based ECL slicing | Same outcome, different mechanism: **load whichever pre-baked subset matches the account's role/speciality**, rather than a live ECL query. |
| UI (chips, tooltips, banners) | Directly buildable now — these are Vue-side concerns regardless of backend, and don't depend on any of the infrastructure above. |
| Odontogram-linked threads | A custom LHC-Forms component/form emitting a SNOMED tooth-structure code on click — same pattern this app already uses for specialty-specific structured forms. |
| Structured optometric parameter block | A YAML-authored form section (Sphere/Cylinder/Axis/Add/VA OD/OS) — exactly what LHC-Forms/YAML forms already do well. |
| Urgent cross-specialty routing | The existing `encounter_assignments` routing/notification system (already built, see the encounter-coordination work), triggered by a severity rule instead of a manual Triage pick. |

**Tier placement**: even the lean path has a real cost (maintaining pre-baked concept subsets, an embedding index) beyond the base free tier's zero-cloud-cost posture — this fits as a **paid-tier, opt-in capability** under `docs/SPEC-05-DATA-TIER-AND-ABDM-BOUNDARY.md`, not a default for every clinic.

---

## PART C — Enterprise tier: nano data center (MedGemma + MedCAT/medspaCy + MedSAM)

Real-time audit and safety warnings on top of Part B's semantic layer, running on real, already-owned hardware — not a cloud service ClinuxFlow would need to build or rent GPUs for. Gated to Enterprise tier per `docs/SPEC-05-DATA-TIER-AND-ABDM-BOUNDARY.md` §6, explicitly the last feature in this whole spec's build order, enabled only once built.

### C.1 Hardware, confirmed real (not hypothetical)

| Model | Hardware | Role |
|---|---|---|
| MedGemma | Mac Mini, 24GB unified memory | Text-based clinical audit/reasoning |
| MedCAT / medspaCy | 5 SBCs on Proxmox, behind a switch | CPU-bound clinical NER/entity-linking (Part B's heavier fork, if chosen — §PART B) |
| MedSAM | 1 Windows PC (GPU) | Imaging segmentation, once/if image uploads join Cübo's chat surface |

Confirmed via direct check (not assumed): neither MedGemma nor MedSAM is in Cloudflare Workers AI's model catalog, and both require real GPU-class compute Workers cannot provide — this hardware is the deployment target by necessity, not preference.

### C.2 Connectivity: Cloudflare Tunnel, not a public endpoint

The nano DC sits behind a switch with no public IP. `cloudflared` (Cloudflare Tunnel) is the natural fit — it's built by the same vendor `clinuxflow-api` already runs on, specifically for exposing a private/on-prem service to Cloudflare's edge without opening inbound firewall ports. `clinuxflow-api` reaches the nano DC as an outbound call to the tunnel's address, the same directional pattern as every other external integration in this stack.

### C.3 A dedicated gateway, same role as `clinuxflow-abdm-gateway`

Not folded into `clinuxflow-api` directly — a separate service (name TBD, e.g. `clinuxflow-nano-dc-gateway`) that is the *only* thing holding whatever access/credentials reach the nano DC, translating `clinuxflow-api`'s request shape into whatever MedGemma/MedCAT-medspaCy/MedSAM's own services expose. Same justification that split the ABDM gateway out originally: a genuinely separate, differently-hosted external system, not something that benefits from living inside the Workers-based API.

### C.4 Shared vs. dedicated — a pricing lever, not a design fork

Both modes are the same code path, differing only in what `clinics.nano_dc_endpoint` points at (see SPEC-05 §6):
- **Shared**: multiple Enterprise clinics' `nano_dc_endpoint` all resolve to the one cluster described in C.1. Needs a real request queue/fair-scheduling plan once concurrent enterprise clinics are actually using it — GPU inference throughput is finite, not assumed elastic.
- **Dedicated**: a clinic-specific `nano_dc_endpoint`, its own physically separate (or logically isolated) instance.

Which a given customer gets, and at what price, is a sales decision ("agreed price model") — the architecture only needs the endpoint to be a per-clinic configurable value.

### C.5 Compliance

Unlike chat signaling (pure relay, never sees content), Part C requests carry real clinical text/audio/imaging across the Tunnel boundary — a genuine PHI transit surface needing the same encryption-in-transit/minimal-retention/access-logging discipline as the ABDM flows, per SPEC-05 §7. Owned hardware doesn't exempt this from that treatment.

## 2. Open items

- ~~SNOMED CT licensing~~ — **resolved for India-geography deployment**: licensed via **NRCeS** (National Resource Centre for EHR Standards, run by C-DAC Pune — India's official SNOMED International National Release Centre, confirmed current: [snomed.org/members/india](https://www.snomed.org/members/india), [nrces.in](https://www.nrces.in/services/national-releases)), which also provides free tooling/SDKs for integration. Scoped to India specifically — any future deployment outside India geography needs its own national-release or international-affiliate license; this doesn't cover that automatically.
- **Revised**: Part B's NLP piece now depends on SPEC-06 §6 (Wikidata tagging, foundational/first) for its interim matching, not §7 (sentence-transformer, now Enterprise-tier/last) — the concept-embedding upgrade specifically waits for Enterprise tier, but the pre-baked subsets, CDS rules, and UI don't need to.
- Exact size/scope of the pre-baked subsets per role — needs scoping against real ECL queries, now that licensing is resolved.
- ~~Supported specialty list~~ — **resolved, starting scope agreed**: General Medicine plus its cluster, using **schema.org's `MedicalSpecialty` enumeration** (a genuinely finite, stable, 41-value taxonomy — [schema.org/MedicalSpecialty](https://schema.org/MedicalSpecialty)) as the standing vocabulary, cross-checked live against a real Wikidata SPARQL query (subclasses/parts of `Q11180` "internal medicine") and the India section of [en.wikipedia.org/wiki/Medical_specialty](https://en.wikipedia.org/wiki/Medical_specialty), each resolving gaps the other two missed:

  | schema.org value | Specialty | Corroborated by |
  |---|---|---|
  | `PrimaryCare` | General Medicine (base) | — |
  | `Cardiovascular` | Cardiology | Wikidata Q10379, India page |
  | `Endocrine` | Endocrinology | Wikidata Q162606, India page |
  | `Gastroenterologic` | Gastroenterology (incl. Hepatology — schema.org doesn't break Hepatology out separately) | Wikidata Q120569, India page |
  | `Hematologic` | Hematology | Wikidata Q103824 |
  | `Infectious` | Infectious Diseases | India page (Wikidata's subclass query alone missed this) |
  | `Neurologic` | Neurology | schema.org + India page (Wikidata's subclass query alone missed this too) |
  | `Pulmonary` | Pulmonology | Wikidata Q203337, India page |
  | `Renal` | Nephrology | Wikidata Q177635, India page |
  | `Rheumatologic` | Rheumatology | Wikidata Q327657 |
  | `Geriatric` | Geriatric Medicine | schema.org |
  | `Genetic` | Medical Genetics | NEET-SS (confirms it sits under General Medicine in India) |

  Both the Wikidata pre-training (SPEC-06 §6) and the SNOMED subset slicing above proceed against this list first. schema.org's remaining ~29 `MedicalSpecialty` values (Dermatology, Oncologic, Psychiatric, Surgical, etc.) are the standing taxonomy for whatever gets added next under the incremental approach — not undefined, just not yet in scope.
- **Part C gateway's exact API surface** — not yet designed, just its role and connectivity mechanism (§C.3).
- **Shared-mode capacity plan** — a real queueing/fair-scheduling design is needed before more than one enterprise clinic shares the nano DC concurrently (§C.4), not just noted as a consideration.
