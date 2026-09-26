# Designing a Conversational Workspace for Clinicians: Lessons from Cübo

*Building Clinical Software That Deserves Trust, part 6*

**For:** product managers, designers and clinical leads considering a chat-style interface in a
clinical product.

---

Chat is the interface of the moment, and clinical software is being redrawn around it. Some of
that is right: a conversation is a good way to ask "what should I do next?" or to walk a
first-time user through something unfamiliar. Some of it is wrong: nobody wants to register forty
walk-in patients by chatting with a bot.

Cübo is ClinuxFlow's conversational workspace. It went through several designs, and most of what we
learned came from reversing our own decisions. This post collects those lessons. Earlier drafts
described Cübo as a multimodal voice, imaging and document AI routed to local models; that is a
future direction (parts 5 and 7). What exists today is a workspace, and the lessons are about the
workspace.

## Where chat helps and where it hurts

| Chat helps | Chat hurts |
|---|---|
| Infrequent, guided tasks (signing up, recovering an account) | High-volume, repetitive entry (front-desk registration in a rush) |
| "What should I do next?" | Scanning a list, comparing records, reading a chart |
| Coordinating with colleagues | Anything a clerk does 200 times a day |
| Explaining what just happened | Dense forms with dozens of fields |

The design question isn't "chat or forms?" It's how the two share a screen.

## Lesson 1: chat hosts the work; it doesn't replace the pages

Our first chat-first registration flow was a separate page with its own chat interface. It
couldn't see other conversations, couldn't be interrupted, and duplicated the real form. We
deleted it. Cübo now has a three-pane layout:

- **Left**: context. Colleagues and pending requests, conversation threads, and (planned)
  notebooks grouped by patient visit or facility.
- **Middle**: the conversation.
- **Right**: structured content, in tabs: the next actions, the current form, and profile
  settings.

On a phone the three panes collapse into one with a tab strip. Every real working screen (Front
Desk, Consultation, Checkout, registration) remains a real page with its own address, which you
can bookmark, resume and print. Cübo is a way in, not a replacement.

## Lesson 2: "what next" should come from state, not from a model

The most useful thing Cübo does is show the next sensible actions. We compute them from the
workflow's actual state, not by asking a language model to guess. The entry journey (register,
log in, recover a password, change it, log out) runs as a real state machine defined as a FHIR
`PlanDefinition`. The Next Action tab lists the actions that are ready right now, plus the
journeys appropriate to the user's role. For a facility manager that is "Register Your Facility";
for a health professional it is "Add My Details".

Getting precedence right took three attempts:
- First, the ready entry actions stayed visible after login and crowded out the role-specific
  suggestions, even though they were correctly "ready" in the state machine.
- Then we hid them entirely after login, which was too blunt: changing a password is always
  relevant.
- Now role-specific journeys lead, and always-available actions follow under "Also available".
  Which list an action belongs in is declared on the action itself (`roles`, `requiresAuth`), not
  in a second hand-maintained table.

## Lesson 3: never navigate silently

The assistant may suggest going somewhere; the user decides. Navigation suggestions are buttons
in the conversation, not automatic redirects. We also draw a firm line between two kinds of
suggestion:
- **Page journeys** (registering a facility, completing a professional profile) open the real,
  full-width page.
- **Small tracked forms** (log in, change password) open in the right pane.

Squeezing a registration form into a side panel was one of our mistakes: the forms were cramped
and the panel duplicated a page that already existed.

## Lesson 4: scope the assistant to what's on screen

Our early slot-filling matched a sentence against every field the page had registered and picked
the best keyword match. When two fields shared a trigger word, the last one registered won,
silently. The fix is **bounded context**: the assistant writes only to the section the user is
working in, and when input belongs elsewhere it says "that sounds like it belongs in X, switch?"
instead of guessing. It may read across the whole record so it isn't forgetful; it may write only
in scope. Where input is genuinely ambiguous (two candidate fields, or which of three doctors),
it shows the options and asks. We apply this rule field by field today; a single shared
component for it is still to be built.

## Lesson 5: put the audit trail in the conversation

Every state change (an action becoming ready, starting, completing, or being blocked) appears in
the conversation as a small system line alongside the messages. Users see why something became
available, and when something fails they can see where. It costs almost nothing, and it
changed how people trusted the flow.

## Lesson 6: forms in chat need a finished state

When forms first appeared in the conversation, a completed form stayed live and could be submitted
again, which looked like a bug and sometimes was one. Each form message now collapses to a small
receipt ("✓ Registered", "✓ Skipped") once done. The state belongs to that message, not to a
global status, because some actions legitimately repeat.

We also hit a subtle bug: the same form mounted twice (once in a hidden dialog, once in the chat),
and a success in one triggered the other's navigation. Each form instance now reacts only to
actions it started.

## Lesson 7: people and the assistant in one place

Staff chat is peer-to-peer and lives in the same workspace as the assistant: pick a colleague in
the left pane and the middle pane becomes that conversation. The server only relays the connection
handshake; message content never touches it. Video calls with a colleague start from the same
place.

The same place carries **approvals**. When someone redeems a facility's join link, the admin sees a
pending request in their contacts, opens a card showing the requester's details, and approves or
rejects it there. Offline delivery goes through a queue that carries only encrypted content.
Approvals in the place staff already look turned out to be more reliable than a separate admin
queue page nobody checks.

## Lesson 8: the assistant must never be what breaks the app

Cübo mounts on every page, so anything it depends on is on every page's critical path. An early
language-processing library with a fragile dependency graph once nearly stopped the whole app from
loading. Since then, every dependency Cübo adds gets a load-and-mount check, heavy models are
opt-in, and the app works fully with Cübo's AI features absent.

## Lesson 9: design for the actual front desk

- **Shared screens.** Session cards visible to a waiting room mask patient details.
- **Mixed languages.** Clinical conversation in India switches language mid-sentence. Any language
  feature has to be tested that way.
- **Phones first for many staff.** Every pane and form has to work at phone width.

## What we'd tell another team

1. Decide which tasks are conversational and which are dense, and don't force either into the other.
2. Compute "what next" from real state; use models to explain and draft, not to decide.
3. Make navigation an explicit user action.
4. Scope writes to the active context; ask when ambiguous.
5. Show state changes in the conversation.
6. Keep the assistant off the critical path.

## Where Cübo is today

Built: the three-pane workspace with mobile collapse; the state-driven entry journey and role-based
next actions; audit lines in the conversation; peer-to-peer staff chat and video; join-request
approvals; field suggestions from chat with explicit acceptance; SOAP drafting on the paid tier.

Not built: rendered markdown responses; group chat; notebooks (conversations organized by patient
visit or facility); understanding several commands in one message; a single assistant action
registry; and the voice, imaging and document features of earlier drafts.
