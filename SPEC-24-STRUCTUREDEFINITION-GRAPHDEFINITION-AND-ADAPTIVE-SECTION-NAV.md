# Specification 24: Real StructureDefinitions + GraphDefinition for Facility/Provider/Affiliate/Patient, and a Shared Adaptive Section-Nav Replacing CustomFormHost

## 1. Objective

Two real gaps named in conversation, not yet built: (1) onboarding's "done"/"required"/"policy" logic is scattered ad-hoc JS (`getAnswer(...,'hospital_name')`, `required: true` flags in YAML) instead of one real, standard FHIR conformance artifact; retiring PlanDefinition-tracking for the 3 onboarding entities (SPEC-23) removed the one thing that gave any sense of sequence, and nothing replaced it. (2) `CustomFormHost.vue`/`FormField.vue` — built to fix LForms' cramped rendering — turned out to be the same mistake in different clothes: a second generic, schema-iterating auto-form-renderer. *"CustomFormHost... is a duplicate of lhcforms... with Profiles can we build a tabbed interface like Team modal window we have"* (explicit instruction) is the real fix: hand-authored panels, like `TeamSettingsModal.vue` already does, not another generic engine.

## 2. Five StructureDefinitions, not four — a real modeling correction found while designing this

"Affiliate" turned out to name two different FHIR shapes, confirmed by reading `TeamSettingsModal.vue`'s already-shipped "Affiliates" tab: it links an *individual visiting practitioner's own account* (`practitionerEmail` + `role`), not an organization-to-organization relationship. That's real, already-live, and staying. What SPEC-23 discussed separately — "administrative steps... some of these may become shared service offered by affiliates" (imaging, laboratories, billing partners) — is an *organization*-level relationship, unbuilt. Same English word, two FHIR resources. Both in scope, per explicit instruction ("Affiliate means both Affiliate Practitioner and Affiliate Organization").

| Entity | FHIR resource(s) | Real profile name |
|---|---|---|
| Facility | `Organization` | `ClinuxFlowFacility` |
| Provider | `Practitioner` (the person) + `PractitionerRole` (their role/specialty at a specific facility) | `ClinuxFlowProvider`, `ClinuxFlowProviderRole` |
| Affiliate Practitioner | `PractitionerRole` (a visiting practitioner's role at a facility that isn't their home facility) | `ClinuxFlowAffiliatePractitionerRole` |
| Affiliate Organization | `OrganizationAffiliation` (`organization` + `participatingOrganization` + `code` + `specialty` + `healthcareService`) — a real FHIR R4 resource built for exactly "org A provides service X to org B" | `ClinuxFlowAffiliateOrganization` |
| Patient | `Patient` | `ClinuxFlowPatient` — unchanged from SPEC-21 §6's own resolution (no login, DigiLocker-only) |

Splitting Provider into `Practitioner`+`PractitionerRole` (today's YAML conflates these onto `Practitioner.extension`) is a real, deliberate upgrade, not scope creep — it's what lets one Practitioner hold roles at more than one facility later, which the Affiliate Practitioner concept already requires structurally (a visiting practitioner's affiliate role has to be a *different* `PractitionerRole` instance than their home-facility role, referencing the same `Practitioner`).

**What a real StructureDefinition buys over YAML's `required: true` flags**: standard cardinality (0..1/1..1/0..*), value-set bindings, and — the concrete, previously-flagged gap this closes — **slicing**. `Practitioner.telecom` sliced by `system` (`phone` vs `email`) is the real, standard FHIR answer to the `ContactPoint.system` tagging gap named as still-open in SPEC-23 §"Still genuinely open." No more inferring phone-vs-email from array position.

**extension.url convention carries forward unchanged**: the 18 ABDM-specific fields already re-homed onto real, distinctly-URLed extensions (SPEC-23's own build) become each Profile's own declared extension slices, not a second parallel list.

## 3. Sequence and "next best action" — GraphDefinition alone can't do this; here's the honest split

Real FHIR `GraphDefinition` describes reference cardinalities between resource *types*, for `$graph` fetch/assemble — it has no concept of time, state, or "before/after." It cannot express "classify facility type before assigning LGD codes" on its own; that would be forcing a resource meant for one job to do a different one, the same mistake `CustomFormHost` was.

Two real, distinct sources of sequence, used for what each is actually good at — **neither is a runtime actor, both are pure, stateless, recomputed-on-demand functions**, so "no PlanDefinition/workflow for these entities" (SPEC-23) stays true:

- **Structural dependency** (a `Location` can't meaningfully exist before its `Organization` does) — implied for free by GraphDefinition's own reference topology (`link[].path`/`target[].type`/`min`/`max`). Real, standards-pure, no invention needed.
- **Business-rule sequencing** (the ABDM sub-chain; "subtype requires type first") — not implied by any reference. Encoded as StructureDefinition **invariants** (FHIRPath `constraint` elements) — "done" is "passes validation," evaluated fresh, never a persisted status.

**Next best action** = walk the GraphDefinition; for every link whose source is present-and-Profile-valid but whose target isn't, surface it as a candidate. A small, pure function (`src/lib/next-best-action.js`, clinuxflow-api) taking `(graphDefinition, currentResourceBundle, validationResults)` → an ordered list of `{resourceType, reason}`. No actor, no snapshot, no XState.

## 4. Knowledge graph / self-learning — precise about which layer

`GraphDefinition` is a *schema*-level artifact (node types, edge types) — a legitimate, reusable foundation, not itself a knowledge graph in the ML sense. The actual knowledge lives in the *instance* graph (real persisted `Organization`/`Practitioner`/`OrganizationAffiliation` resources — gated on real storage existing, still explicitly deferred) plus what's already seeded: Wikidata QID tagging on staff specialties (SPEC-06/08) is a real, already-built anchor into an external KG. This spec's schema work is compatible groundwork for that later effort, not a substitute for it and not blocking on it.

## 5. `AdaptiveSectionNav` — one shared responsive shell, not per-surface reinvention

*"Pops ups are not good for mobile devices... tab, accordion such grouping can be space occupying in mobile... the right three dot context menu can hold the options... this is what I wanted for Cubo as well... approach should be common across the application"* (explicit instruction). Confirmed live, not assumed: Cübo already has its own separate, ad-hoc version of this exact problem — `threePaneMobileView` (a mobile-only sub-tab strip that only exists inside 3-pane mode) plus entirely different hand-rolled toggle buttons (`toggleThreadView()`/`toggleProfileView()`) for FAB/MODAL_DOCK mode. Two bespoke mobile adaptations for the same underlying need — real evidence the unification is worth doing, not a hypothetical.

Built on **Reka UI** (`reka-ui`, already a dependency, already used once in `RegisterForm.vue`'s `RadioGroupRoot`) — confirmed real, available exports: `TabsRoot`, `AccordionRoot`, `DropdownMenuRoot`/`Trigger`/`Content`/`Item`. One component, `src/components/AdaptiveSectionNav.vue`:

- Props: `sections: [{id, label, icon}]`, `mode: 'tabs' | 'sidebar' | 'accordion' | 'panes'` (a hint for viewports at/above the breakpoint — `panes` meaning "show simultaneously," the other three meaning "one at a time, switchable"; `panes` is Cübo's own 3-pane case, not needed by the entity editor).
- Breakpoint: the app's existing `md:` (768px) Tailwind breakpoint — reused, not reinvented.
- Above the breakpoint: renders `mode` via the matching Reka primitive (`sidebar` is a plain vertical button list, not `NavigationMenuRoot` — that primitive is built for hover-flyout mega-menus, the wrong fit for a static in-panel list).
- Below the breakpoint: collapses to a single content area plus a "⋮" `DropdownMenuTrigger` listing `sections`; picking one shows just that section.
- Owns navigation chrome only, via a scoped slot per section — content stays hand-authored by the caller, exactly `TeamSettingsModal.vue`'s style. This is the actual fix for "CustomFormHost is a duplicate of lhcforms": the part that's shared and reusable is the *chrome*, never the field rendering.
- User's chosen `mode` persisted per-component-instance via localStorage, defaulting to `tabs`.

**Sequencing, not simultaneous scope**: build and prove `AdaptiveSectionNav` on the entity editor first (needed immediately, lower risk — fresh code). Retrofitting Cübo's own `threePaneMobileView`/toggle-button mechanism onto it is real, separate risk against a large, currently-working, tested component — an explicit phase 2 of this same spec, not bundled into the same pass.

## 6. Retiring `CustomFormHost.vue`/`FormField.vue`

Deleted once every consumer (`Onboarding.vue`, `StaffOnboarding.vue`, and the new Affiliate/Patient editors) is migrated to hand-authored panels inside `AdaptiveSectionNav`. What stays unchanged, deliberately: each panel still serializes its named, hand-coded fields into the same real `QuestionnaireResponse` item shape at save time (`{linkId, item:[{linkId, answer:[...]}]}`) — so `local-extractor.js`, `composition-assembler.js`, and `mergeGroupResponseItem`/`appendGroupResponseItem` all keep working completely unchanged. Only the render layer changes; the capture-to-storage backbone doesn't get torn out.

## 7. Build sequencing (this pass)

1. StructureDefinition + GraphDefinition JSON (clinuxflow-api) — the five profiles, one graph.
2. A conformance validator (cardinality/required-ness against a StructureDefinition) + tests.
3. The next-best-action function (§3) + tests.
4. `AdaptiveSectionNav.vue` (clinux-frontend) + tests for its own mode/breakpoint logic.
5. One full reference rebuild — Facility (`Organization`/`Onboarding.vue`) — proving the whole chain end to end, live-verified.
6. Provider, Affiliate Practitioner, Affiliate Organization, Patient follow the same now-proven pattern (separate passes).
7. Delete `CustomFormHost.vue`/`FormField.vue` once nothing references them.
8. Cübo's own `AdaptiveSectionNav` retrofit — phase 2, its own pass.

## 8. Relationship to existing specs

- `docs/SPEC-23-SPECIALITY-ROOM-AND-FIXED-ORCHESTRATION-ANCHORS.md` — this spec is the direct answer to its closing "Profile and Graph approach" pointer, and to its own "no plan definition or workflow" correction (§3 here keeps that true).
- `docs/SPEC-13-FHIR-WORKFLOW-DOCUMENTS-AND-CONFORMANCE.md` — `local-extractor.js`/`composition-assembler.js` stay exactly as built; this spec adds a conformance layer on top, not a replacement.
- `docs/SPEC-09-ABDM-ANCHORED-ONBOARDING-REBUILD.md` — the 18 ABDM extension fields it (via SPEC-23) established carry forward unchanged into each Profile's own extension slices.
