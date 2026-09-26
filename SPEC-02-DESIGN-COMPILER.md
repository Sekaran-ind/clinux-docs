# SPEC-02: Design-Time FHIR Dictionary and YAML → Questionnaire Compiler

| | |
|---|---|
| **Status** | Built |
| **Last reviewed** | 2026-09-26 |
| **Code** | `clinuxflow-api/tools/dictionary-builder.js`, `tools/build-graphs-bundle.js`, `tools/build-system-forms.js`, `tools/build-system-flows.js`, `tools/clinixflow.config.json`, `src/lib/shared/yaml-to-questionnaire.js`, `data/form-schematics.schema.json`, `data/graphs/*.graph.json`, `data/graphs.bundle.json` |
| **Routes** | `POST /api/workflow/compile`, `GET /api/workflow/default-blueprint`, `POST /api/workflow/save-to-library`, `GET /api/workflow/system-forms`, `GET /api/workflow/system-flows` |
| **Related** | SPEC-13 §5 (extraction), SPEC-18 (nested groups, PlanDefinition authoring), SPEC-24 (StructureDefinitions) |

## 1. Purpose

Let a clinic, or a developer, describe a form in short YAML that names real FHIR R4 paths, and
reject anything that is not a valid path **before** a form ever reaches a clinician. The compiled
output is a standard FHIR `Questionnaire` whose items carry enough metadata (`definition`) for
definition-based SDC extraction (SPEC-13 §5).

The core principle is that the set of allowed paths is **generated from real FHIR type
definitions, never hand-curated**, so the compiler cannot drift from the standard.

## 2. The dictionary

`tools/dictionary-builder.js` uses `ts-morph` to walk the vendored `@smile-cdr/fhirts` FHIR R4
TypeScript classes recursively, for every resource listed in
`tools/clinixflow.config.json → validation.allowedResources` (31 today: Patient, Observation,
Encounter, Condition, MedicationRequest, Organization, Location, Practitioner, PractitionerRole,
Coverage, Claim, Invoice, Account, DiagnosticReport, ServiceRequest, Specimen, Appointment,
Schedule, Slot, Medication, MedicationDispense, SupplyDelivery, PaymentNotice, Device,
DeviceMetric, Consent, Contract, HealthcareService, EpisodeOfCare, PlanDefinition,
OrganizationAffiliation).

Outputs:
- `data/graphs/<resource>.graph.json`: every reachable dotted path with its primitive type (and
  enum values where the type is a code), one shard per resource.
- `data/graphs.bundle.json`: all shards bundled for import into the Worker (no filesystem at
  request time).
- `data/form-schematics.schema.json`: the JSON Schema for YAML authoring, including the
  recursive `field` definition (a leaf, or a `type: group` with its own `fields`).

Adding a resource means adding it to the config and running `npm run build:kernel`. `Task` and
`ValueSet` are not in the list yet (SPEC-18 §9).

## 3. The compiler

`compileYamlToQuestionnaire` in `yaml-to-questionnaire.js` is the only compiler. The build tools
import the same function, so the build-time and request-time paths cannot diverge.

1. **Structural check**: Ajv (with `ajv-errors`) against `form-schematics.schema.json`.
2. **Semantic check**: every `field.path` must exist in its resource's graph shard. Unknown
   paths fail with the offending path named.
3. **Emit**: each `composition:` block becomes a group item; each field becomes an item typed
   from its `uiComponent`:

| `uiComponent` | FHIR item type | Notes |
|---|---|---|
| `TextInput` | `string` | |
| `NumericInput` | `decimal` | |
| `Checkbox` | `boolean` | |
| `DatePicker` | `date` | |
| `Dropdown` | `choice` | `answerOption` from `choices`, or the enum from the graph |
| `MultiSelect` | `choice`, `repeats: true` | every selection is extracted, not just the first |
| `Autocomplete` | `open-choice` | `answerValueSet` + `terminology-server` extension (`field.terminologyServerUrl`, defaulting to NLM Clinical Tables) |
| `Hidden` | as typed | fixed `initial` from `defaultValue` |
| `type: group` | `group` | nests to any depth; `repeats`, group-level `skipLogic` → `enableWhen` |

Every leaf item gets `definition = http://hl7.org/<ResourceType>#<path>` and, where set, an
`extension.url` for FHIR extensions (the 18 ABDM fields re-homed in SPEC-23's build).

## 4. Known deviations from the SDC standard

- **`definition` is not canonical.** SDC expects
  `http://hl7.org/fhir/StructureDefinition/<Type>#<Type>.<path>`. The local extractor only reads
  the fragment after `#`, so this works inside ClinuxFlow, but a third-party `$extract`
  implementation would reject it. Fixing it is a one-line change in the compiler plus a rebuild of
  `system-forms-library.json`. It is worth doing before any external FHIR server integration.
- **No `itemExtractionContext`.** Resource grouping is inferred from path roots and group
  structure (SPEC-13 §5.3) rather than declared with the SDC extension.

## 5. Where compiled forms live

| Catalog | Source | Storage |
|---|---|---|
| System forms | `tools/system-forms/*.yaml` (provider composition, patient profile, encounter composition, join request) | Precompiled into `data/system-forms-library.json`, seeded into every client's `formsLibrary` collection |
| System flows | `samples/hospital-setup-workflow-v1.yaml` + a worked-example response | Precompiled into `data/system-flows-library.json` (SPEC-18, SPEC-22 §5.10) |
| Clinic-authored forms | Designer (`/designer`) | The client's `formsLibrary` collection; `save-to-library` also writes KV (`FORMS_LIBRARY`) |

## 6. History

The original version of this spec (2025) called for exactly this dictionary-and-validator split,
and that design held. Two later additions changed its capability: recursive nested groups
(SPEC-18 §7 step 3) and configurable terminology servers (SPEC-18 §7 step 4). The early
assumption that a HAPI server would run `$extract` against the `definition` URLs was dropped for
local extraction (SPEC-13 §5), which is why the non-canonical URL form never caused a failure.

## 7. Open items

- Make `definition` SDC-canonical (§4).
- Add `Task` and `ValueSet` to `allowedResources` when a real consumer needs them.
- Consider emitting `itemExtractionContext` so a standard SDC engine could extract the same forms.
