# SPEC-07: SNOMED CT-Coded Multidisciplinary Clinical Chat

| | |
|---|---|
| **Status** | Design. Nothing built. The only SNOMED content in the codebase is the specialty ConceptMap in `clinuxflow-api/data/clinic-specialities.json`. |
| **Last reviewed** | 2026-09-26 |
| **Related** | SPEC-05 §6.3 (enterprise tier), SPEC-06 (harness), SPEC-08 (sequence), BLOG-05, BLOG-07 |

## 1. Goal

Give clinical conversation semantic grounding in SNOMED CT: vocabulary scoped by role and
specialty, translation between disciplines, background safety checks, and specialty modules
(dentistry, optometry/ophthalmology) with their own anatomy and procedures.

The spec has three parts. **Part A** is the full hosted-infrastructure vision. **Part B** is the
path that fits the current architecture (Workers edge, local-first, no always-on servers).
**Part C** is the enterprise tier on owned hardware.

## Part A: full vision (hosted infrastructure)

| Component | Technology | Purpose |
|---|---|---|
| Terminology engine | Snowstorm | Lookups, expansion, updates |
| API | FHIR terminology services (`$expand`, `$lookup`, `$closure`) | App ↔ terminology |
| NLP | MedCAT or equivalent | Entity extraction and linking from chat text |
| Decision support | CQL + SNOMED value sets | Rule evaluation, cross-thread contradiction checks |
| Persistence | FHIR server (HAPI) or OMOP CDM | Structured messages with semantic bindings |

Role slices by ECL: surgeons (procedures `71388002`, morphology `49755003`, body structure
`123037004`); doctors (clinical findings `64572001`, products `373873005`); nurses (observables
`363787002`); lab (specimen `123038009`); imaging (`363679005`); dietitians (`22943007`,
`226887002`); admin (SNOMED → ICD-10 mapping for coding and billing).

UI: recognized terms shown as chips with preferred terms; hover shows the parent hierarchy;
inline safety banners when a drug conflicts with a condition or allergy recorded in another
thread. Workflow: background CDS, handover validation against a SNOMED template before sign-off,
audit of message + role + bindings + timestamps.

Specialty modules:
- **Dentistry**: FDI tooth numbering mapped to SNOMED tooth structures; clicking an odontogram
  injects the tooth concept into context; restorative material checked against allergies.
- **Optometry/ophthalmology**: structured Sphere/Cylinder/Axis/Add/VA per eye; refractive and
  retinal findings; an IOP spike routes an urgent alert to the ophthalmologist.

This is a different operating model from anything running today (Snowstorm is a Java +
Elasticsearch service). Read it as what a dedicated hosted deployment would run.

## Part B: the lean path that fits the current architecture

| Part A piece | Lean equivalent |
|---|---|
| Hosted Snowstorm | Pre-built, versioned SNOMED subsets per role and specialty, shipped as static reference data and refreshed periodically |
| MedCAT | Interim: the same keyword/NER matching as SPEC-06 §5, over SNOMED labels enriched with Wikidata synonyms. Enterprise: an embedding index (SPEC-06 §7) |
| CQL | A small local rules table (concept X contraindicated with active concept Y), evaluated against the patient's known structured record |
| FHIR server / OMOP | No new store: a `snomedConcepts: [{code, term}]` field on existing message records |
| ECL slicing | Load the subset matching the account's role and specialty |
| Chips, tooltips, banners | Buildable now; pure frontend |
| Odontogram | A custom form component emitting a tooth-structure code |
| Optometry block | A YAML form section |
| Urgent routing | The existing encounter assignment mechanism, triggered by a severity rule |

Even the lean path has running costs (maintaining subsets), so it sits in the paid tier as an
opt-in.

## Part C: enterprise tier on owned hardware

| Model | Hardware | Role |
|---|---|---|
| MedGemma | Mac mini, 24 GB unified memory | Text audit and reasoning |
| MedCAT / medspaCy | Five SBCs on Proxmox | CPU-bound clinical NER and linking |
| MedSAM | Windows PC with GPU | Image segmentation |

Neither MedGemma nor MedSAM is in the Workers AI catalog, so owned hardware is a necessity, not a
preference. Connectivity is a Cloudflare Tunnel (no public IP, no inbound ports). A dedicated
gateway (`clinuxflow-nano-dc-gateway`, not yet created) would be the only holder of nano-DC
credentials, the same pattern as the ABDM gateway. Shared or dedicated deployment is set by
`clinics.nano_dc_endpoint` (SPEC-05 §6.3); shared mode needs a real queueing and fair-scheduling
design first. Part C traffic carries PHI and needs encryption in transit, minimal retention and
access logging (SPEC-05 §7).

## Decisions already made

- **Licensing**: SNOMED CT in India is licensed through NRCeS (C-DAC Pune, the national release
  centre), which also provides free tooling. Deployments outside India need their own licence.
- **Starting specialties**: General Medicine and its cluster, using schema.org's
  `MedicalSpecialty` enumeration cross-checked against Wikidata and NEET-SS: PrimaryCare,
  Cardiovascular, Endocrine, Gastroenterologic (including hepatology), Hematologic, Infectious,
  Neurologic, Pulmonary, Renal, Rheumatologic, Geriatric, Genetic.

## Open items

- Size and contents of each role subset, scoped against real ECL queries.
- The nano-DC gateway's API surface.
- Shared-mode capacity and scheduling design.
- Reconcile the starting specialty list with the two specialty catalogs the app already has
  (SPEC-23 §3.2).
