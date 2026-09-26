# State Machines in Clinical Software: Where They Help and Where They Hurt

*Building Clinical Software That Deserves Trust, part 9*

**For:** engineers and architects deciding how to model workflows in a clinical application.

---

Clinical work is full of sequences: register, triage, consult, bill. Staff hand work to each other.
Some steps can't start before others finish. It's natural to reach for a workflow engine or a
statechart library, and for the right problems they are excellent. We also applied them to a
problem they were wrong for, and removing them made the product better. This post is about telling
the two apart.

## Where state machines earn their keep

A state machine is worth its cost when the work has:
- **Real ordering constraints**: checkout can't close an encounter that was never consulted.
- **Several actors handing off**: reception, nurse, doctor, billing.
- **Interruption and resumption**: a consultation paused for a lab result, resumed an hour later on
  another device.
- **An audit requirement on the process itself**: who moved this visit from triage to consult, and
  when.

Outpatient visits, procedures, referrals and long-running care programmes all fit. So do small
session flows such as sign-up and account recovery, where "what can the user do now?" really
depends on what already happened.

## Where they hurt

Registration looked like a workflow too: facility basics, then locations, staff, services, hours,
consents, then the ABDM registry steps. We modeled it as a tracked plan with a checklist, persisted
progress and an audit trail. Problems followed:

- **Two sources of truth.** The checklist said "Staff: done"; the data said whether staff existed.
  They could disagree, and when they did, the checklist was wrong.
- **Migration pain.** Adding steps to the plan stranded every saved progress state in an
  incompatible shape, and every step showed as locked.
- **False sequence.** Most of those steps don't depend on each other at all. The machine imposed an
  order the domain didn't have.

For registration, "done" isn't a state that something moves into; it's a fact about the data: does
this facility record pass validation? That question is best answered by a pure function run on
demand. We replaced the workflow with FHIR profiles (what a valid facility, practitioner or patient
looks like) plus a graph of how those resources reference each other, and a small function that
walks the graph and says what's missing next. No persisted state, no migrations, no disagreement.

Our rule since then: **state machines for clinical journeys; stateless validation for master and
roster data.**

## Three shapes, not one

| Shape | Example | Lifetime | Model |
|---|---|---|---|
| Roster lifecycle | A facility, a practitioner's affiliation | Perpetual | Mostly stateless validation, plus discrete events such as a join approval |
| Pipeline | One outpatient visit | Bounded: starts and ends | A state machine per visit |
| Episode | A pregnancy, a rehab programme | Long-running, spanning many visits | A grouping of pipelines under a care plan |

Forcing all three into one engine is where designs go wrong.

## Standards first: define in FHIR, execute in code

We don't hand-write a machine per workflow. The definition is a FHIR `PlanDefinition`: actions,
with `relatedAction` expressing "this starts after that ends". A small compiler turns it into an
XState statechart: one parallel region per action, moving through `pending → ready → active →
done`, with dependencies as guards. A running action corresponds to a FHIR `Task`, and what it
captured is a `QuestionnaireResponse`. Definition, request and event: FHIR's workflow pattern,
with the statechart as the execution engine underneath.

Plans are authored in the same YAML pipeline as forms, compiled and extracted into real
PlanDefinitions. The same compiled plan can run in the app and be read by any FHIR tool.

## Ten rules we learned the hard way

**1. Don't encode false dependencies.** If six steps can be done in any order, they should all
become available at once. A linear chain the domain doesn't require is friction the user feels and
a lie the audit trail records.

**2. Decide explicitly whether an action can happen again.** In XState a `final` state ignores all
further events, silently. Our "log in" action was final, so the second login in a session did
nothing: no request, no error. Login, logout and password change must be repeatable. Even
"register" had to be: `done` belongs to the action, not to an email address, so registering a
second account in one session was silently dropped. Make repeatability an explicit flag per
action.

**3. The machine holds status, never data.** Machine context has status and error messages. Records
live in the data store. A machine holding a copy of a record will eventually hold a stale one.

**4. One dispatcher.** UI components never send events to machines directly; one runtime owns every
running machine and routes events. Each UI instance reacts only to transitions it caused. We had the
same form mounted twice, and one instance's success triggered the other's navigation.

**5. Subscribe before start, or replay.** A listener attached after a machine starts doesn't get the
initial state. Ours showed "undefined" until the next change.

**6. Version persisted state.** Before restoring a saved machine, check that its shape still matches
the current definition. If not, start fresh and say so. The clinical data lives elsewhere and is
unaffected; only progress resets.

**7. Keep an append-only audit of transitions, and show it.** Every transition (and every blocked
attempt) is appended locally and mirrored to the server. We also show them as small lines in the
user's conversation view; users trust a flow more when they can see why a step became available.

**8. One writer per running instance.** A machine's saved state is one interdependent tree and
can't be safely merged. Use a time-limited lock: one device drives, others watch, and a crashed
device's lock expires. Pair it with the append-only log so a bypassed lock is still reconstructable.

**9. Map to FHIR Task status honestly.** Our four internal states map to `draft`, `requested`,
`in-progress` and `completed`. FHIR has eight more, and the missing ones matter clinically:
`on-hold` (waiting for labs looks different from being worked on), `failed` (a failed step that is
silently retried leaves no audit trace), `accepted`/`rejected` (claiming work when several people
could take it). Know which ones you don't model, and add them before you need them in an audit.

**10. Trigger across workflows explicitly.** When finishing one workflow should start another (after
login, suggest the right onboarding journey for this role), use an explicit "on this action done"
hook. Watching another machine's internal state couples two things that should only share an event.

## A note on conditions

`PlanDefinition` lets an action be conditional ("include this step only for cardiology"), expressed
in FHIRPath. It's tempting to let authors type expressions freely. Don't let free-text expressions
reach a runtime that evaluates them: offer a governed list of condition types with parameters, and
validate them. We relaxed our own authoring to free text while nothing evaluated conditions; the
moment something does, that decision has to be reversed.

## Where ClinuxFlow is today

Built: the PlanDefinition-to-statechart compiler and runtime, with repeatable actions, invoked side
effects, cross-plan triggers, version checks, an append-only audit and single-writer locks, persisted
locally and mirrored to the cloud. The sign-up and account journey runs on it. Registration uses
stateless validation instead.

Not built yet: the outpatient visit as a PlanDefinition with a Task per station (it currently runs
on simpler status fields and stage locks), evaluation of conditions, the missing Task states, and
episode-of-care grouping. The visit comes first; it is the workflow these rules were written for.
