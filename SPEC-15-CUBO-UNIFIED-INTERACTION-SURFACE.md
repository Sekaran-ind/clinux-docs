# SPEC-15: Cübo as the Unified Interaction Surface

| | |
|---|---|
| **Status** | Partially built. The three-pane layout (§3) and in-Cübo hosting of guided flows exist. Markdown rendering (§5), section-level cards (§6), URL pop-out (§4) and retiring slot matching (§7) are not built. |
| **Last reviewed** | 2026-09-26 |
| **Code** | `clinux-frontend/src/components/Cubo.vue`, `src/components/cubo/{CuboProfilePanel,CuboContactConversation}.vue`, `src/stores/cubo.js`, `src/components/cubo.css`, route `/ai-engine` |
| **Related** | SPEC-16 (left pane model), SPEC-20 (entry journey), SPEC-22 §5.1–§5.3 (how the shell was built), SPEC-24 §5 (shared nav) |

## 1. Purpose

Cübo hosts guided capture directly, not in standalone pages; a three-pane layout; dense content
opens as real pages; responses read like a modern assistant (narrative plus cards); and slot
matching gives way to bounded context.

## 2. Correction that started this spec

The first chat-first flow (`HospitalOnboardingChat.vue`, SPEC-14 §7) was a standalone page with
its own chat UI, not a Cübo thread. That flow has since been deleted, and the entry journey was
built inside Cübo from the start (SPEC-20).

## 3. Three-pane layout (built)

Cübo has four layouts (`stores/cubo.js`): `FAB`, `EXPANDED`, `MODAL_DOCK`, and `THREE_PANE`.
`THREE_PANE` is what the `/ai-engine` route renders (`Cubo` mounted directly with
`forceLayout: 'THREE_PANE'`), and any other layout can switch into it.

- **Left**: an accordion of Contacts (team and linked affiliates, plus pending join requests),
  Contact Groups (placeholder), Threads, and Notebooks (placeholder, SPEC-16).
- **Middle**: the conversation (the unchanged header, body and composer), or a contact's P2P
  conversation when a contact is selected. The composer is pinned to the viewport bottom.
- **Right**: tabs **Next Action** (the entry journey's ready actions and role-based journey
  links), **Content** (the selected entry form), **Profile** (specialty and role picker).
- **Below 768px**: one pane at a time with a Threads / Chat / Content tab strip.

The workflow audit log appears in the chat timeline as compact system rows.

## 4. Dense content opens as a page (not built as a mechanism)

When the right pane's content is too wide (a grid, a large form), navigate to a real route rather
than embedding heavy components in the pane or the chat log. Returning must resync to the
thread's current state, not a stale snapshot. In practice this happens informally today:
registration journeys are page links (`/onboarding`, `/practitioner-home`), and only the small
entry forms mount in the Content tab (SPEC-22 §5.14).

## 5. Rendering (not built)

A single input with feature buttons (document upload reusing the existing documents pipeline);
assistant responses as sanitized markdown; structured data as cards interleaved with the
narrative. Messages already support embedded components (`component`/`componentProps`),
navigation suggestions and audit rows. Markdown is not rendered, and no sanitizer has been chosen.

## 6. Card granularity: sections, not single fields (not built)

Render one card per section (all its fields together) rather than one per field. The section
boundary is the Questionnaire group.

## 7. Retire cross-field slot matching, keep intent classification (not done)

`formSlotEngine.js` exists to guess which of many visible fields a sentence belongs to, and it
collides by design. Under bounded context the active field is known, so that guess becomes
unnecessary. Intent classification ("switch me to Consultation", "go back") stays and matters more.
Both still run today.

## 8. Build order

1. ~~Host guided flows in Cübo~~: done for the entry journey.
2. ~~Three-pane shell~~: done, including mobile collapse.
3. Markdown rendering with a sanitizer.
4. Section-level cards.
5. Formal pop-out and return-with-resync.
6. Retire slot matching once bounded context exists (SPEC-12 §4.2).
7. Retrofit Cübo's own mobile tab strip onto `AdaptiveSectionNav` (SPEC-24 §7 step 8).

## 9. Related specs

SPEC-12 §4.3 (what cards render), SPEC-13 §3 and SPEC-16 (the left-pane model), SPEC-14 (the
machine this surface consumes).

## 10. Open items

- Markdown sanitizer choice.
- `THREE_PANE`'s mobile strip and `AdaptiveSectionNav` are two mechanisms for one need.
- Contact Groups have no data model; they depend on group chat (SPEC-06 §4).
