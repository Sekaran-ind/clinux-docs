# Specification 15: Cübo as the Unified Interaction Surface

## 1. Objective

Consolidate the interaction-surface design decided across the discussion following SPEC-14: Cübo hosts guided-flow capture directly rather than a standalone page, a three-pane layout, URL-based navigation for dense rich components, ChatGPT/Claude/Gemini-style rendering, section-grouped cards, and retirement of NLP cross-field slot-matching. Nothing here is built yet except SPEC-14's shipped slice, which §2 identifies as not matching this design.

## 2. Correction: SPEC-14's built slice doesn't match this design

`clinux-frontend/src/pages/HospitalOnboardingChat.vue` (SPEC-14 §7) is a standalone page with its own bespoke chat-bubble UI — not Cübo. It doesn't appear in Cübo's thread list, doesn't share `Cubo.vue`'s dispatch machinery, and a user can't redirect it mid-flow the way they could a real Cübo conversation. §3-§7 below describe what it should be rebuilt against; §8 sequences that rework as the first build step, not an afterthought.

## 3. Three-pane layout

- **Left** — the Task/Workflow navigator. Formalized in `docs/SPEC-16-NOTEBOOKS-AND-TASK-PRIMARY-NAVIGATION.md` — doubles as journey map and thread list (one structure, not two separate widgets).
- **Middle** — Cübo itself, the conversation.
- **Right** — structured data (LForms or otherwise) reflecting the current HFSM state's capturable/captured fields.

## 4. Dense content pops out via URL navigation, not inline embedding

When the right pane's content is too dense or wide for panel width (a full AG Grid, a large table), it navigates to a dedicated route rather than embedding the heavy component repeatedly inside the pane or the chat history — each navigation mounts/unmounts once instead of stacking instances inside a scrolling log. A back affordance returns to the conversation, **resynced to the thread's current state**, not a stale snapshot from when navigation happened — the HFSM may have advanced while the user was on the child page. This also keeps rooms real, addressable, resumable pages rather than trading that away for chat-first navigation, consistent with the principle held since the start of this design line.

## 5. Rendering: markdown narrative + cards, ChatGPT/Claude/Gemini shape

Single input control, feature buttons (document upload — reusing the existing document pipeline already in Front Desk/Consultation Desk's Documents tab, not a new upload path). Cübo's own responses render as sanitized markdown, not plain text. Structured data renders as cards interleaved with the narrative, not as a separate concept bolted alongside chat.

## 6. Card granularity: YAML section groups, not one field at a time

`abdmSchema.js`'s `section` field (distinct from `groupLinkId`'s FHIR-write grouping) already declares the right chunking boundary, and `groupFieldsBySection()` already exists — it's what `hospitalRegistrationMachine.js`'s parent states are already built on (SPEC-14 §7). The change is in consumption, not structure: render one card per *section* (parent state), all its fields together, instead of one card per leaf field state as the current built slice does.

## 7. Retire NLP cross-field slot-matching; keep intent classification

`clinux-frontend/src/nlp/formSlotEngine.js`'s NER-based cross-field matching exists to solve "which of many simultaneously-visible fields does this text belong to" — collision-prone by its own documented design (`docs/SPEC-06-CUBO-AGENTIC-HARNESS.md` §2). Under bounded context + one-active-state (SPEC-12 §4.2), that ambiguity mostly stops existing — the current field is already known; free text just answers it. Retire `formSlotEngine.js`'s matching role. `clinux-frontend/src/nlp/intents.js`'s intent classification (recognizing a message as a command — "switch me to Consultation," "go back" — rather than an answer) is a different job and stays; it matters more, not less, once Cübo is the sole interaction surface.

## 8. Build order

1. Rework `HospitalOnboardingChat.vue` into an actual Cübo-hosted thread — corrects §2's gap, first because everything else assumes it.
2. Build the three-pane shell around the reworked flow.
3. Wire URL pop-out for the right pane's dense-content case.
4. Switch card granularity from field-level to section-level.
5. Retire `formSlotEngine.js`'s matching; confirm `intents.js` still correctly handles mid-flow redirects under the new model.

## 9. Relationship to existing specs

- `docs/SPEC-12-ROOMS-AND-BOUNDED-CONTEXT-SLOT-FILLING.md` — §4.3's modal-resolution primitive is what §5/§6 here render as cards.
- `docs/SPEC-13-FHIR-WORKFLOW-DOCUMENTS-AND-CONFORMANCE.md` §3 — the Cübo thread model this spec's left pane sits on top of, further revised by SPEC-16.
- `docs/SPEC-14-HFSM-RUNTIME-AND-CHAT-FIRST-CAPTURE.md` — this spec reworks the UI *consumption* of that machine, not the machine itself; `hospitalRegistrationMachine.js`'s hierarchy is unchanged.

## 10. Open items

- Exact `Cubo.vue` integration point for hosting an HFSM-driven thread — not yet designed at the component level.
- Whether the right pane extends the existing 2-pane mobile toggle precedent (`clinux-mobile-responsive-chat-forms-toggle` memory note) directly to a 3-way toggle, or needs its own collapse behavior.
- Markdown sanitization approach for Cübo's rendered narrative — not yet chosen.
