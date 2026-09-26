# SPEC-20: Journey Workbench Pattern, Applied to the Unauthenticated Cübo Entry

| | |
|---|---|
| **Status** | Built. The entry journey runs as a tracked PlanDefinition in Cübo. Its presentation moved from inline chat forms (§6) to THREE_PANE's Next Action and Content tabs (SPEC-22 §5.1, §5.7). |
| **Last reviewed** | 2026-09-26 |
| **Code** | `clinux-frontend/src/workflow/entryPlanDefinition.js`, `src/stores/entryWorkflow.js`, `src/components/auth/*`, `src/components/auth/entryFormRegistry.js`, `src/components/Cubo.vue`, `clinuxflow-api/src/routes/auth.js`, `migrations/0009_add_security_question.sql` |
| **Related** | SPEC-13 §2, SPEC-15, SPEC-21 §5, SPEC-22, SPEC-25 |

## 1. Purpose

Take a reference pattern (a single-file "journey workbench" demo) and apply it to the smallest real
target: signing up, signing in and recovering an account from inside Cübo, driven by a real
PlanDefinition.

## 2. What the reference pattern shows

- **Journey picker**: one card per journey with metadata and a version selector. ClinuxFlow's
  equivalent is the rooms landing in Designer and Cübo's Next Action list.
- **A real numbered stepper**, grouped by phase and tagged with the standard resource behind each
  step, plus a completion bar. This only works when a real Task graph sits behind it, which is why
  surfaces without one (the old AI Engine sandbox) had to stay flat lists.
- **Inline capture** in the conversational flow, and a clearly separated **copilot** card that
  suggests but never commits ("converts intent into a candidate, never commits it for you").
- **Right-pane tabs**: entity view with a live JSON view, **Observe** (per-stage completeness chart
  and event counts), **Audit** (transitions, copilot interactions and blocked attempts logged
  separately).
- **A data-scope disclosure in the UI itself** ("this session is stored only in this browser…").
  Worth copying: every guided journey should say where its data goes.
- **Conditional repeating groups with the rule shown in plain language** next to "add another".

## 3. First application: the entry journey

In an unauthenticated Cübo session the General thread hosts Register, Log In, Forgot Password and
Change Password (plus Log Out once signed in), as a state machine.

## 4. Decisions

1. **Additive.** Index.vue's own register and login modals stay and render the same form
   components.
2. **Forgot password by security question**, with no email infrastructure. The answer is hashed
   like a password (PBKDF2), normalized by trimming and lower-casing. It works for a solo admin with
   no one to reset them.
3. **Audit now, Observe later.** The runtime's audit log is shown; the completeness chart is not
   built.
4. **A separate entry plan.** The test plan (`AUTH_PLAN_DEFINITION`) gates login on register, which
   is right for testing sequencing and wrong for real users. `ENTRY_PLAN_DEFINITION` leaves all
   actions ungated. Change password's real gate is the server's `requireUser()`.

## 5. Built

**Backend**: `accounts.security_question`/`security_answer_hash`; `PATCH /api/auth/security-question`,
`POST /api/auth/forgot-password/question`, `POST /api/auth/forgot-password/reset`; plus the existing
register, login and change-password routes.

**Runtime**: `ENTRY_PLAN_DEFINITION` (`unauth-entry-v1`) with actions `register`, `login`,
`forgot_password`, `change_password` (all `repeatable`) and `logout` (`repeatable`,
`requiresAuth`). `buildEntryServices` wires each to a real call. `stores/entryWorkflow.js` is the
app-wide Pinia home: statuses, errors, `activeEntryAction`, `onRoleKnown`, and a single `logout()`
used everywhere. Snapshots and audit entries persist through SPEC-25 (durable key
`unauth-entry-v1:<accountId>`; anonymous transitions stay local).

**UI**: host-agnostic forms (`RegisterForm`, `LoginForm`, `ForgotPasswordForm`,
`ChangePasswordForm`, `SecurityQuestionForm`) used by Index.vue's modals and by Cübo.
`JoinTokenRedeemForm` is deliberately separate and untracked (SPEC-26).

## 6. Hosted in Cübo's General thread, not a page

The first version was a standalone `/get-started` page, which contradicted SPEC-15 §1; it was
deleted and rebuilt inside Cübo's existing `general` category. Messages can carry a Vue component,
so forms first rendered inline in the chat. **That inline presentation was later replaced**
(SPEC-22 §5.1): in THREE_PANE, the Next Action tab lists the ready entry actions and the Content tab
shows the chosen form, while the chat narrates. Pre-auth, the entry actions are primary; post-auth,
role-based journey links lead and the entry actions appear below them.

Bugs found live and fixed:
- **Multi-instance hijack.** The same form mounted twice (hidden modal plus Cübo) meant a success in
  one fired the other's handler and navigated away. Each form now reacts only to transitions it
  started.
- **Late subscriber.** Subscribing after `actor.start()` missed the initial snapshot, so statuses
  stayed `undefined`. Fixed by an explicit refresh after subscribing.

### 6.1 Completed forms collapse
Each rich message records its own `done` state and shows a chip ("✓ Registered", "✓ Skipped")
instead of a live, resubmittable form. This is per message, not read from global status, because
actions repeat. Thread routing between Cübo instances was checked: the data is shared and
consistent; only the default thread differs by page, by design until SPEC-16.

### 6.2 `done` was terminal, blocking a second login
Every action's `done` was an XState `final` state, so a second login in one session was silently
dropped. Fixed with the opt-in `action.repeatable` flag. Register was later made repeatable too: `done`
belongs to the action, not the email, so signing out and registering a different account in the
same session was silently dropped (reproduced live). The server's email-uniqueness check remains
the real guard.

## 7. Related specs

SPEC-15 (Cübo hosts guided flows), SPEC-16 (the status is real Task state; the numbered stepper
awaits notebooks), SPEC-19 §12 (why the old sandbox stayed a flat list), SPEC-22 §5.1–§5.9 (the
presentation's later evolution).
