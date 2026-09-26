# SPEC-22: Four Foundational Decisions: Persisted Workflow, System Flows, Cübo State-Mirroring, Capture Surface

| | |
|---|---|
| **Status** | Mixed. D1 (persisted workflow) → built as SPEC-25; the personal worklist is not built. D2 (system flows) → built. D3 (Cübo mirrors app state) → logout reset built; the `administration` thread was built and later removed. D4 (capture surface) → built as the THREE_PANE shell. §5.4–§5.9's in-Cübo hospital-setup checklist was built and then **retired** (§5.14). |
| **Last reviewed** | 2026-09-26. Rewritten from a 300-line build log into outcomes; section numbers kept because code cites them. |
| **Code** | `clinux-frontend/src/components/Cubo.vue`, `src/stores/{entryWorkflow,cubo}.js`, `src/workflow/{entryPlanDefinition,onboardingJourneys,rooms}.js`, `src/data/collections/{flowsLibrary,roomSettings}.js`, `src/data/collections/formData.js` (group slicing and merging), `src/pages/Designer.vue`, `clinuxflow-api/tools/build-system-flows.js`, `POST /api/workflow/extract` |
| **Related** | SPEC-16, SPEC-18, SPEC-20, SPEC-21, SPEC-23, SPEC-25 |

## 1. The four decisions

1. Workflow, Request and Event resources are persisted and viewable like any other entity, and
   feed a personal worklist at login (with confirmation before navigating).
2. A `system-flow` YAML library parallel to system forms: steps, conditions, guards.
3. Cübo's active thread mirrors application state: General when signed out; the right thread for
   the context when signed in; a hard reset to General on logout.
4. Form capture through Cübo happens on a real capture surface, and navigation is always an
   explicit user action.

## 2. Decision 1: persisted workflow resources and a worklist

**Outcome**: persistence is built as SPEC-25 (a `ClinuxFlowTask` profile, IndexedDB snapshots and
an append-only audit log, a D1 mirror with locks). **Not built**: a queryable Task record per
action that a worklist can list (SPEC-25 persists actor snapshots and audit rows, not per-action
`Task` resources), and the worklist UI itself. The encounter worklists in Front Desk, Consultation
and Checkout are station queues, not personal Task lists.

## 3. Decision 2: the system-flow library

**Outcome: built** (see §5.10 and SPEC-18 §7). Along the way a real extractor bug was fixed: nested
repeating-group instance counters were keyed by `linkId` globally rather than per parent instance,
leaving holes in `relatedAction` arrays for any linear chain longer than two steps.

## 4. Decision 3: Cübo mirrors application state

- Signed out → General only: true.
- Encounter context → encounter thread: true (the `encounterId` prop).
- **Logout resets to General**: built (§5.9) through one `entryWorkflow.logout()` used everywhere.
- **Hospital context → an Administration thread**: built (§5.9), then removed with the checklist
  it served (§5.14). There is no `administration` category today.
- A general "auth stage × context → active thread" rule computed centrally, replacing per-page
  `category` props: not built; it belongs to SPEC-16.

## 5. Decision 4: the capture surface

The original wording was "all LHC-Forms capture in a drawer, never inline". It was reframed in §5.1:
the real requirement is a standing third pane whose presentation (column or toggle) depends on
width. SPEC-20's inline chat forms were superseded.

### 5.1 The THREE_PANE shell (current)
Cübo gained a fourth layout, `THREE_PANE`, rendered directly by the `/ai-engine` route (`Cubo` with
`forceLayout`, no wrapper page; the layout resets on leaving). Left: threads. Middle: the unchanged
conversation. Right: the selected entry form. Other layouts can switch into it. Index.vue's "Try
Guided Setup" opens it. Two first attempts were rejected by review (content on Index.vue; a wrapper
page), so this shape is the settled one.

### 5.2 Completing the three panes (current)
- Composer pinned to the viewport (a real bug: the page grew instead of the history scrolling).
- Right pane tabs: Next Action, Content, Profile (`CuboProfilePanel.vue` shared with the overlay).
- The workflow audit log is merged into the chat timeline as compact system rows.
- Mobile: one pane at a time with a three-way tab strip below 768px.

### 5.3 Contacts in the left pane (current)
`CuboContactConversation.vue` brings `TeamChat.vue`'s P2P conversation into Cübo; the left pane is
an accordion of Contacts, Contact Groups (placeholder), Threads and Notebooks (placeholder).
Contacts load reactively when auth changes (a real bug: they loaded once at mount, before sign-in).
`TeamChat.vue` still exists and is reachable from ClinicHome.

### 5.4 Hospital setup, group by group, in Cübo (retired, see §5.14)
Built: a real LHC-Forms render of one Provider-composition group at a time in the Content tab,
sequenced by a hospital-setup PlanDefinition. **Primitives from this pass that are still in use**:
`sliceRecordGroup` (LHC-Forms throws if handed record groups the sliced Questionnaire lacks) and
`mergeGroupResponseItem` in `formData.js`, which merges real FHIR answer types rather than
flattening them to strings. Onboarding.vue and the SPEC-24 hosts save through them.
`sliceQuestionnaireGroup` (`formsLibrary.js`) is still exported but has no caller. `HospitalOnboarding.vue`, `HospitalOnboardingChat.vue`
and `hospitalRegistrationMachine.js` were deleted here.
The Staff drawer moved to a sliced LHC-Forms render with `appendGroupResponseItem` (append a new
instance, never replace others). This closed a real field gap and let `AbdmFieldForm.vue` and
`abdmSchema.js` be deleted. (StaffOnboarding has since moved to SPEC-24 hosts.)

### 5.5 All ten groups, as a checklist (retired, see §5.14)
The flow grew to ten actions: the ABDM sub-chain kept strict order; the six operational groups
(Location, Staff, Services, Hours, Consent, Appointment) depended only on Hospital Details, so they
unlocked together as a menu rather than a false linear chain. `mergeGroupResponseItems` (plural)
replaces a repeating group's full instance set, including to zero. `sliceRecordGroup` was fixed to
return every instance. The principle carries forward: **don't encode false dependencies**.

### 5.6 Snapshot compatibility (current)
Growing a plan under the same id made old snapshots unrestorable (XState returned an undefined
state, so every step looked locked). `registerPlan()` now compares a snapshot's regions with the
plan's action ids and starts fresh on a mismatch. Workflow state resets; the captured data, which
lives in `formData`, is untouched.

### 5.7 Entry actions in Next Action, not the chat (current)
The entry actions moved from a strip above the chat to the Next Action tab, where role suggestions
already lived: one presentation for one concept. Choosing an action switches to Content.
Precedence: before sign-in, entry actions are primary; after sign-in, role journeys lead and
Forgot Password, Change Password, Register and Log Out follow under "Also available". Next Action is
the first and default tab.

### 5.8 Plan metadata replaces a hard-coded suggestion map (current, amended by §5.14)
`ROLE_SUGGESTIONS` was removed. Per-action metadata (`requiresAuth`, `roles`) on the plan decides
visibility, and `isVisible()` answers every "should this be offered" question. `logout` became a
tracked action, so sign-out anywhere produces the same audited transition. At the time,
`facility_registration` and `staff_registration` were plan actions completed from outside (a Pinia
watch on the checklist); §5.14 removed them from the plan.

### 5.9 Logout reset and the administration thread (logout current; thread retired)
One central `logout()` does sign-out, the tracked transition, and a reset to the default General
thread. The `administration` category was added for hospital setup and removed with it.

### 5.10 The system-flow loader (built; nothing runs the flow today)
`tools/build-system-flows.js` compiles `hospital-setup-workflow-v1.yaml` and extracts a worked-example
response (`hospital-setup-workflow-response.js`) into `data/system-flows-library.json`, served at
`GET /api/workflow/system-flows` and seeded into `flowsLibrary`. A test proves the extracted plan
equals the hand-authored one, the first test with many top-level sibling actions sharing one
parent. This was the first time a compiled PlanDefinition drove the real runtime; since §5.14, no
store consumes it.

### 5.11 The Room-Architect Designer (current, scope narrowed)
Designer's landing is **Rooms** (`ROOM_DEFINITIONS`: Facility, Provider, Patient, Encounter, Account).
Each room has a card grid scoped by `roomId`, a "Design & Compile Room" editor (compile the room's
workflow YAML → fill → extract through the new `POST /api/workflow/extract` → save a version to
`flowsLibrary`), "Forms Authoring" that links plan actions to existing cards, and a persisted
per-room role editor (`roomSettings.js`). `ROOM_RUNTIME_WIRED` is now empty (§5.14). SPEC-23 found
the five static rooms do not scale to one room per specialty.

### 5.12 Room-card parity, Cübo as FAB in Designer, simpler conditions (current)
Room cards got the same context menu as form cards (Designer/YAML view, versions). Designer shows
Cübo as a floating button instead of a fixed pane. The condition field became plain
`expression.expression` text (reversing SPEC-18 §7 step 4's constraint, acceptable only while
nothing evaluates conditions).

### 5.13 The room model doesn't scale (resolved by SPEC-23's design)
Every specialty will become a room, rooms share administrative sub-flows (onboarding, imaging,
labs, billing), some fulfilled by affiliates, and hand-writing workflow YAML per room is too hard.
See SPEC-23.

### 5.14 Registration leaves the state machine (current)
Two explicit instructions: registration uses real page routes, not a narrow Cübo pane; and
*"for the 3 onboarding journeys no plan definition or workflow is required; state machines will be
used only for clinical journeys."* Deleted: `hospitalSetupWorkflow.js`, `hospitalSetupPlanDefinition.js`,
their tests, the checklist UI and the `administration` category. `facility_registration` and
`staff_registration` left the entry plan and became plain role-gated links in
`workflow/onboardingJourneys.js`. **Rule since**: journey links are page navigations; only small
tracked entry forms mount in the Content tab.

## 6. Where each decision stands

| Decision | State |
|---|---|
| D1 persisted workflow + worklist | Persistence built (SPEC-25); per-action Task records and worklist not built |
| D2 system-flow library | Built; no live consumer since §5.14 |
| D3 Cübo mirrors state | Logout reset built; central state→thread rule not built (SPEC-16) |
| D4 capture surface | Built as THREE_PANE (§5.1–§5.3) |

## 7. Related specs

SPEC-16 §6 (D1 was its prerequisite), SPEC-18 (D2's compiler), SPEC-19 §9 (status mapping),
SPEC-20 §6 (superseded presentation), SPEC-21 (the auth × journey rule D3 implies), SPEC-23 (§5.13's
answer), SPEC-25 (D1's implementation).
