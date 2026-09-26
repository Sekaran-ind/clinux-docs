# SPEC-21: The Six-Dimension State Map: Parallel, Not Nested

| | |
|---|---|
| **Status** | Reference and decision record. §5 (role-based next action) is built. The tier lifecycle, unified storage state and journey routing (§4) are not. |
| **Last reviewed** | 2026-09-26 |
| **Code (for §5)** | `clinux-frontend/src/workflow/{planDefinitionRunner,workflowRuntime}.js` (`result_<actionId>`, `onActionDone`), `src/stores/entryWorkflow.js` (`onRoleKnown`), `src/workflow/onboardingJourneys.js`, `src/components/Cubo.vue` |
| **Related** | SPEC-05, SPEC-16, SPEC-19 §5, SPEC-20, SPEC-23 |

## 1. Purpose

Check a proposed "state machine map" (App Tier → Storage → Auth Stage → ABDM Status → Journey →
Actors, drawn as six nested boxes) against the real code, and recommend how the dimensions should
actually compose.

## 2. Each dimension against the code

| Dimension | Proposed values | Reality |
|---|---|---|
| Auth stage | Public / Authenticated | Real and tested: the entry plan (SPEC-20). |
| ABDM status | Unregistered / Registered | Partly real. HFR/HPR/ABHA registration works in the sandbox; "registered" must not be confused with ABDM M2/M3 certification, which does not exist. |
| App tier | Free / Trial / Paid / Suspend / Discontinue | Two of five: `clinics.tier IN ('free','paid')`. |
| Storage | Local / Server / Cloud / Private-DC | Separate mechanisms (local collections, LAN server, D1 mirrors; the nano-DC is design). Nothing lets the app ask "which storage mode am I in". |
| Journey | General / Onboarding / Consultation / Service Desk | Cübo's `category` field (`general`, `encounter`, `front-desk`, `billing`, `patient-directory`, `abdm-facility`, `abdm-patient`, `ai-engine`), chosen by each page's prop, not by state. |
| Actors | Hospital admin / Provider / Patient | Staff roles only. **Patient is not an actor** (see §6). |

## 3. Mostly parallel, not nested

- Auth → ABDM status: genuinely nested (no ABDM status before authentication).
- Auth → Journey: mostly nested, except `general`, which spans both.
- Journey → Actor: not nested. Role is a stable property of the session, not of the journey.
- Tier → Storage: a **constraint** (tier limits which storage modes are allowed), not containment.
- Tier as the outermost box: wrong. A suspended clinic's users are still signed in and mid-journey,
  just blocked from paid actions.

**Recommendation**: one parallel top-level machine, `tier × storage × authStage`, with ABDM status
and journey and actor as sub-states inside `authenticated`. This extends SPEC-19 §5's
`mode × connectivity × interaction` pattern. Not built.

## 4. Genuinely new work (not built)

- **Tier lifecycle**: trial expiry, suspension grace, what happens to a suspended clinic's data,
  and discontinue with export and delete. A schema and business logic, not new enum values.
- **A unified storage state**: a thin layer that answers "which mode am I in" over the existing
  mechanisms.
- **State-driven journey routing**: Cübo's active thread chosen from real Task state (SPEC-16 §5).

## 5. Role-based next action (built)

- `planDefinitionRunner.js` keeps an invoked service's result as `result_<actionId>`, read with
  `actionResult()`.
- `workflowRuntime.js` exposes `onActionDone(planId, actionId, cb)`, the cross-plan trigger
  SPEC-13 §9 lacked. It fires every time, including for repeatable actions.
- `entryWorkflow.js` exposes `onRoleKnown(cb)`, wired to both `register` and `login`, reading
  `role` from the real account returned by the API.
- Cübo, in the general/entry context, posts a navigation-suggestion message with the journeys for
  that role, taken from `ONBOARDING_JOURNEYS`: `hospital_admin` → Register Your Facility
  (`/onboarding`); `health_professional` → Add My Details (`/practitioner-home`);
  `admin_and_health_professional` → both. The same list drives the Next Action tab.
- `role-equals` was added to the condition-type catalog for the declarative side (SPEC-18 §7
  step 4). Nothing evaluates it.
- Found along the way: tests sharing the real `taskActorSnapshots` collection leaked a `done`
  snapshot between tests; fixed by clearing it in `beforeEach`.

The links are navigation suggestions, not tracked plans, consistent with SPEC-22 §5.14.

## 6. Refinement: the overlap diagram and the Patient decision

A second diagram replaced strict nesting with overlap (Public and Authenticated overlap exactly at
"onboard"), placed ABDM as a side structure, and added per-role journey stages and a storage
lifecycle.

- Overlap and ABDM placement: correct, kept.
- Per-role stages map to existing mechanisms: the hospital's `affiliates` and the provider's
  `linking` are one mechanism (SPEC-26's join tokens and `facility_affiliates`); the hospital's
  `activate` is publishing the clinic page. **The provider's `activate` is still undefined.**
- **Patient decision** (explicit): *"Patient will not have a login to this application. Patient
  can see his health records via DigiLocker, no features more than that."* Patient is a data
  subject staff act for, never an account. The only patient-facing touchpoint is the DigiLocker
  export (SPEC-09 §5).
- The storage lifecycle (Draft → Ready → Reviewed → Complete, with Retrieve and Publish between
  local and server) is real missing modeling. It should be merged into SPEC-19 §7's
  versioned-conflict design, not built as a second sync model.

## 7. Related specs

SPEC-19 §5 (the parallel-region pattern), SPEC-20 (the auth dimension's implementation), SPEC-16
§5 (journey routing), SPEC-05 and SPEC-07 Part C (the storage mechanisms).
