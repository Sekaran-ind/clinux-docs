# SPEC-00: Specification Index and Reading Guide

| | |
|---|---|
| **Status** | Current |
| **Last reviewed** | 2026-09-26, against clinux-frontend `7deb3e5`, clinuxflow-api `e89630a`, clinuxflow-abdm-gateway `1905185` |
| **Audience** | Engineers working on ClinuxFlow, and anyone deciding whether a spec is still true |

## 1. How to read this set

The specs were written one at a time over several months, each recording what one build pass
decided. Many were later overtaken by the code. On 2026-09-26 every spec was re-checked against
the three repos and rewritten so that its opening status block and its body describe the system
**as it is now**, with earlier decisions kept only as a short history.

Each spec opens with a status block:

| Status | Meaning |
|---|---|
| **Built** | The described behavior exists in code and is covered by tests. |
| **Partially built** | Some sections are real; the spec says which. |
| **Design** | Agreed direction, nothing (or almost nothing) in code yet. |
| **Superseded** | Kept for its reasoning. Another spec, named in the status block, is authoritative. |
| **Reference** | Research or analysis, not a build plan. |

**Section numbers are stable.** About 500 code comments cite specs by section (for example
`SPEC-24 §7 step 6` or `SPEC-22 §5.8`). When a spec is revised, a cited section keeps its number
even if its content is condensed. Before renumbering anything, run:

```
grep -rhoE 'SPEC-[0-9]+ ?§ ?[0-9.]+' clinux-frontend/src clinuxflow-api/src clinuxflow-abdm-gateway/src | sort -u
```

## 2. Start here

1. **SPEC-01**: the system as built: repos, layers, data flow, storage, security boundaries.
2. **SPEC-05**: where data lives and why (free/paid/enterprise tiers, ABDM's role).
3. **SPEC-24**: how Facility, Provider, Affiliate and Patient data is modeled and validated.
4. **SPEC-13 + SPEC-25**: the workflow runtime (PlanDefinition → XState → persisted Task).
5. **PENDING-WORK.md**: what is open right now.

## 3. Status matrix

| Spec | Title | Status | One-line summary |
|---|---|---|---|
| 01 | Architecture overview | Current | The as-built system map. |
| 02 | Design-time compiler | Built | FHIR R4 dictionary from TypeScript types → YAML → Questionnaire. |
| 03 | Runtime scribe | Superseded | Constrained local decoding was the plan; a Workers AI call is what runs. See BLOG-07 for the forward design. |
| 04 | Clinical workflow | Superseded | HAPI `$extract` plan replaced by local SDC extraction + Composition assembly (SPEC-13). |
| 05 | Data tier & ABDM boundary | Current, partially built | Free = local/LAN, paid = cloud D1, enterprise = nano-DC (not built). |
| 06 | Cübo agentic harness | Partially built | Wikidata tagging built; multi-intent, dispatch layer, group chat not built. |
| 07 | SNOMED clinical chat | Design | Role-scoped SNOMED subsets, CDS rules, nano-DC models. Nothing built. |
| 08 | Cübo build sequence | Partially built | Phase 1 (Wikidata) done; phases 2–9 open. |
| 09 | ABDM-anchored onboarding | Superseded in mechanism | Principles stand; `abdmSchema.js`/`AbdmFieldForm.vue` replaced by SPEC-24's hosts. DigiLocker export built. |
| 10 | e-Sushrut assessment | Reference | Competitive capability matrix. |
| 11 | ABDM M1–M4 alignment | Current strategy | HFR/HPR are prerequisites; M1 light, M2/M3 deferred, M4 (NHCX) the next investment. |
| 12 | Rooms & bounded context | Partially superseded | Bounded-context and modal-resolution principles stand; room sequencing moved to SPEC-13/23. |
| 13 | FHIR workflow, documents, conformance | Partially built | Extraction hardened, PlanDefinition runtime and Composition assembly built; attestation and CapabilityStatement not. |
| 14 | HFSM runtime | Current (runtime), retired (UI slice) | XState + RxJS chosen; the chat-first Hospital slice was deleted. |
| 15 | Cübo unified surface | Partially built | THREE_PANE layout built; markdown rendering, URL pop-out, NLP retirement not. |
| 16 | Notebooks & Task-primary navigation | Design | Blocked on a Task-to-thread binding; Task persistence itself now exists (SPEC-25). |
| 17 | TanStack AI conversation layer | Design | Not adopted. The single Workers AI call is still inline. |
| 18 | PlanDefinition authoring via YAML | Built | Nested groups, condition types, system-flows library. |
| 19 | Local-first & federated modes | Partially built | IndexedDB backend for 5 collections; federation actor and conflict sandbox not built. |
| 20 | Get Started journey in Cübo | Built | Register/Login/Forgot/Change/Logout as a tracked PlanDefinition. |
| 21 | Six-dimension state map | Reference, §5 built | Parallel-not-nested composition; role-based next action built. |
| 22 | Four foundational decisions | Mixed | D1 → SPEC-25 (worklist not built); D2 built; D3 partly retired; D4 built as THREE_PANE. |
| 23 | Speciality Room & fixed anchors | Design (+ onboarding build notes) | Specialty rooms as data; "state machines only for clinical journeys". |
| 24 | StructureDefinition, GraphDefinition, AdaptiveSectionNav | Built | Seven profiles, two graphs, conformance validator, next-best-action, shared nav shell. |
| 25 | Federated Task persistence | Built | IndexedDB + D1 mirror, append-only audit, single-writer locks. |
| 26 | Facility join-token linking | Built | Staff/affiliate/organization linking via tokens, approved in Cübo chat. |

## 4. System map in one paragraph

ClinuxFlow is a Vue 3 single-page app (`clinux-frontend`) that stores every record locally first
(TanStack DB collections on localStorage or IndexedDB), optionally mirrors to a LAN server (a
Tauri app, `src-tauri/shared_server.rs`), and, for paid clinics, to a Cloudflare Workers API
(`clinuxflow-api`, Hono + D1 + KV + Durable Objects + Queues). Forms are authored in YAML,
compiled against a FHIR R4 path dictionary into `Questionnaire`s, captured as
`QuestionnaireResponse`s (LHC-Forms for clinical rooms, hand-authored Vue hosts for registration),
extracted into FHIR resources, validated against local StructureDefinitions, and assembled into
FHIR document Bundles. All ABDM traffic (ABHA, HPR, HFR) goes through a separate Worker
(`clinuxflow-abdm-gateway`), the only component holding ABDM credentials. Cübo is the
conversational shell over all of it.

## 5. Glossary

| Term | Meaning here |
|---|---|
| **Room** | A bounded capture context with its own Questionnaire (Front Desk, Consultation, Checkout; Facility/Provider/Patient for registration). |
| **Control layer** | Identity and roster: accounts, facilities, practitioners, affiliates, conformance. Stateless validation, no workflow actor. |
| **Runtime layer** | Clinical and session workflows: encounters, Task/PlanDefinition actors, locks, chat, video. |
| **System form / system flow** | A YAML-authored Questionnaire / PlanDefinition shipped with the app and seeded into every install. |
| **Host** | A hand-authored Vue capture component (e.g. `FacilityBasicsHost.vue`) that writes the same QuestionnaireResponse shape LForms would. |
| **Next best action** | A suggestion computed from either a GraphDefinition walk (control) or a PlanDefinition actor's ready set (runtime). |
| **Free / paid / enterprise** | `clinics.tier` `free` or `paid`; enterprise is paid plus a provisioned nano-DC (not built). |

## 6. Writing new specs

- Put the status block first and keep it accurate. A spec whose status is wrong is worse than no spec.
- Describe the system in the present tense. Put dated decision history in a short final section, not inline.
- Name files and routes. Cite line numbers only when necessary; they rot fastest.
- When a later spec overrides an earlier one, update the earlier one's status block in the same change.
