# SPEC-18: PlanDefinition Authoring via the YAML → Questionnaire → Extraction Pipeline

| | |
|---|---|
| **Status** | Built for PlanDefinition (§7 steps 1–4). ValueSet authoring (§6, step 5) and order sets (step 6) not built. The constrained condition field of step 4 was later relaxed to free text (SPEC-22 §5.12). |
| **Last reviewed** | 2026-09-26 |
| **Code** | `clinuxflow-api/samples/{workflow-definition-v1.draft,hospital-setup-workflow-v1}.yaml`, `src/lib/runtime/hospital-setup-workflow-response.js`, `tools/build-system-flows.js`, `data/system-flows-library.json`, `scripts/build-condition-types.js`, `data/condition-types.json`, `GET /api/valuesets/plandefinition-condition-types`, `clinux-frontend/src/data/collections/flowsLibrary.js`, `src/pages/Designer.vue` (Rooms) |
| **Related** | SPEC-02, SPEC-13 §2, SPEC-16, SPEC-22 §5.10–§5.12, SPEC-23 §3.3 |

## 1. Purpose

Author `PlanDefinition`s with the pipeline that already exists for forms, instead of a new tool.

## 2. Mechanism

1. Write a workflow-definition YAML with the usual `composition:` syntax, targeting
   `resourceType: PlanDefinition`.
2. Compile it with the unchanged `yaml-to-questionnaire.js`. The result is an **authoring form**: a
   Questionnaire whose items describe steps (action id and title, nested `relatedAction`, nested
   `condition`).
3. Fill it in. The response describes one concrete plan.
4. Extract with `local-extractor.js`, which buckets answers by the resource type in each item's
   `definition`. Out comes a real `PlanDefinition`.

The extractor is resource-type-agnostic, so the same route works for any resource the dictionary
knows.

## 3. Why this rather than a diagram tool

- **Stately (visual statechart editor)**: rejected as the authoring surface. The YAML route needs
  no new dependency, keeps one authoring paradigm, and validates at compile time. The honest cost
  is that a form is a worse view of a graph than a diagram, so a visualizer should be an optional
  review layer over compiled output.
- **Inferring sequence from data-capture YAML**: impossible, because a room's own form has no
  reason to know other rooms exist.

## 4. Scope boundary

This produces the Definition tier only. Running it is SPEC-13 §2's runtime.

## 5. Coordination walk-through

Checked end to end on paper: `Task.owner` and `Task.partOf` give each person their own slice of
one plan instance; the handoff signal can ride the P2P chat channel; eligibility for ownership
comes from the roster (`PractitionerRole`/`OrganizationAffiliation` `active` and `period`).
Caveat, still true: persisted Tasks exist (SPEC-25), but ownership, owner notification and
roster-driven assignment are not built.

## 6. Reference lists are ValueSets, not order sets

"Author LGD codes and similar lists the same way" is right about the mechanism and wrong about the
resource. A list of valid codes is a `ValueSet`. An order set (`PlanDefinition.type = order-set`)
is a reusable bundle of clinical actions, for example a cardiology consultation's typical steps.
The same pipeline can produce all three: PlanDefinition (sequencing), ValueSet (reference lists)
and order-set PlanDefinitions (specialty bundles).

LGD codes ended up sourced live from the ABDM gateway rather than authored (SPEC-12 §3, defect 4),
so no LGD ValueSet was needed. The HFR master lists that are static
(`data/hfr-master-valuesets.json`, served at `GET /api/valuesets/hfr-master/:type`) were generated
by a script (`build-hfr-master-valuesets.js`), not through this pipeline.

## 7. Build record

1. **Dictionary covers PlanDefinition.** Added to `allowedResources` and rebuilt (a 676-node
   shard). The draft's `relatedAction.targetId` was wrong; the R4 property is `actionId`.
2. **Five-room draft authored** (`workflow-definition-v1.draft.yaml`). Extracting it exposed that
   two independently repeating sibling groups (Rooms and Sequencing) could not share one array:
   `relatedAction` from one landed inside another room's `action[0]`.
3. **Real nested groups, fixed at the source.** LHC-Forms and FHIR already nest groups to any
   depth; the YAML compiler and schema did not. Now a field can be `type: group` with its own
   `fields` and `repeats`, compiles to a nested group with its own `definition`, and group-level
   `skipLogic` becomes `enableWhen`. The extractor walks a stack of group frames. `relatedAction`
   now nests inside its `action`. All system forms still compile unchanged.
4. **Constrained `condition` field.** A local `CodeSystem` of condition types (`specialty-equals`,
   `provider-actively-affiliated`, later `role-equals`), mirroring how `specialities.json` is a
   real `ConceptMap` to SNOMED. It is compiled with draft FHIRPath templates into
   `data/condition-types.json` and served as a pre-expanded ValueSet. The field captured
   `condition.expression.reference` through an Autocomplete. This step also fixed the compiler's
   hard-coded terminology server (`field.terminologyServerUrl`).
   **Later relaxed** (SPEC-22 §5.12): both sample YAMLs now capture
   `condition.expression.expression` as plain text, because the Autocomplete made authoring hard
   and nothing evaluates conditions yet. Revisit when a condition-evaluating runtime exists: free
   FHIRPath text should not reach a real evaluator unvalidated.
5. ValueSet authoring through the pipeline: not built (no current need, see §6).
6. First order-set PlanDefinition (cardiology bundle): not built. It now belongs to SPEC-23's
   Speciality Rooms.
7. ~~Runtime~~: built in SPEC-13.

**The system-flows library** (SPEC-22 §5.10) connected the two halves:
`hospital-setup-workflow-v1.yaml` plus a worked-example response
(`hospital-setup-workflow-response.js`) compile and extract at build time into
`system-flows-library.json`, served at `GET /api/workflow/system-flows` and seeded into the
client's `flowsLibrary`. The extracted plan matched the hand-authored JS plan action for action.
Designer's Rooms view compiles, fills and extracts room plans interactively
(`POST /api/workflow/extract`). Since SPEC-22 §5.14 nothing runs the hospital-setup plan;
`ROOM_RUNTIME_WIRED` is empty.

## 8. Related specs

SPEC-16 §6 (authoring gap closed here), SPEC-13 §2 and §4 (the resource model), SPEC-12 §4.3
(coded fields), SPEC-22 §5.10–§5.12 (library and Designer).

## 9. Open items

- `condition` is authored but never evaluated; the FHIRPath templates were never run through a
  real engine; the specialty parameter substitution isn't wired.
- Add `Task` to the dictionary only when something authors Tasks through YAML.
- `action.definitionCanonical` composition (SPEC-23 §3.3) needs compiler, extractor and runtime
  support.
