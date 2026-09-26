# SPEC-06: Cübo Agentic Harness: Unified Chat, Multi-Intent Understanding, Semantic NLP

| | |
|---|---|
| **Status** | Partially built. §6 (Wikidata tagging) is built for the Designer call site; everything else is design. |
| **Last reviewed** | 2026-09-26 |
| **Code** | `clinux-frontend/src/components/Cubo.vue`, `src/nlp/{intents,formSlotEngine,fieldShortcuts,formFieldHarvester}.js`, `src/data/p2pChat.js`, `src/data/collections/{chatThreads,userChats}.js`, `clinuxflow-api/src/lib/control/wikidataTagging.js`, `src/durable-objects/ChatSignalingRoom.js` |
| **Related** | SPEC-07 (SNOMED), SPEC-08 (build order), SPEC-12 (bounded context), SPEC-15 (surface), SPEC-17 (LLM layer) |

## 1. Goal

Turn Cübo from a chain of single-purpose handlers into a harness that understands more than one
intent per message, knows the user's role and context, dispatches registered actions, and
holds AI chat, one-to-one chat and group chat in one inbox.

## 2. Current state

Cübo's send path is a fixed chain. The first branch that matches handles the message:

1. `classifyIntent` (`nlp/intents.js`): `node-nlp`, two trained intents (`generate_soap`,
   `switch_room`), single best intent above a 0.6 threshold, otherwise `general_chat`.
2. `recognizeSlots` / `applyFills` (`nlp/formSlotEngine.js`): `@nlpjs` NER trained on field
   labels and keywords harvested from every system form. Keyword-based, so colliding trigger text
   resolves to whichever field registered last.
3. `matchShortcuts` (`nlp/fieldShortcuts.js`): deterministic `/shortcut value` for exact fields.
4. Otherwise, on the paid tier, `test-scribe` (Workers AI).

Conversation stores:
- `chatThreads.js`: AI-assistant threads, one per category (`general`, `encounter`,
  `front-desk`, `billing`, `patient-directory`, `abdm-facility`, `abdm-patient`, `ai-engine`) or
  per encounter. Messages can carry a Vue component (rich content), audit rows, or navigation
  suggestions.
- `userChats.js` + `p2pChat.js`: one-to-one WebRTC chat, now surfaced inside Cübo's left pane
  (Contacts) in THREE_PANE mode (SPEC-22 §5.3). `TeamChat.vue` still exists separately.
- No group chat. No multi-intent output. No embedding-based matching anywhere.

## 3. Target: an agentic harness

- **Multiple intents per message** are dispatched together (§5).
- **Role and context awareness**: the account role, the active room, and the active Task feed what
  the harness considers relevant.
- **A registry of actions** (generate SOAP, switch room, fill a slot, message a colleague, open a
  record, approve a join request) chosen by understood intent, rather than each being its own code
  path in `Cubo.vue`. SPEC-26 §12 names the same need from the P2P side: chat-triggered actions
  should dispatch off a Task specification, not a hard-coded `v-if` per feature.
- **Multi-step requests** ("assign this patient to Dr. Singh and let her know").

This layer sits on top of the existing handlers, not in place of them.

## 4. Unified inbox

One thread list for AI chat, one-to-one chat and group chat. Group chat will be a **full P2P
mesh** (every participant connects to every other), not a relay or SFU: clinic groups are small
(2–8 people) and the chat feature's founding rule is that no server stores message content.
`ChatSignalingRoom` currently rejects a third socket (`sockets.size >= 2`); group chat needs
N-socket signaling and one `RTCPeerConnection` per peer.

## 5. Multi-intent classification (interim, no new dependency)

Verified against `node-nlp@5.0.0-alpha.5`: `process()` returns a `classifications` array, but
the scores are softmax-normalized (they sum to 1), so thresholding each intent independently does
not work. The plan is to split an utterance at conjunctions and clause boundaries and classify
each clause with the existing single-intent classifier, then union the results. Acceptance case:
"generate the SOAP note and switch to cardiology" fires both actions.

The same check found a separate weakness: with two narrow intents and no "none" class, an
unrelated input ("what is the weather") scored `generate_soap` at 1.0. Before multi-intent ships,
add negative examples or a top-two-margin rule.

The enterprise tier would later replace this with exemplar-embedding similarity (§7).

## 6. Wikidata semantic tagging (design time and onboarding)

Deliberately narrow: used when a form is authored or a specialty is chosen, never in runtime chat
classification. Wikidata is general knowledge, not a clinical terminology, so a human always
confirms the match.

**Built**:
- `WikidataTagging` behind `GET /api/nlp/wikidata-search` and `GET /api/nlp/wikidata-concept`,
  `requireUser()` only. The backend sets a compliant `User-Agent` (browsers cannot), caches in KV
  (`WIKIDATA_CACHE`) to stay inside Wikidata's per-client query budget, and honors
  `429`/`Retry-After`. Search and confirm are separate calls, so nothing auto-picks a match.
- **Designer**: "Suggest from Wikidata" next to a field's keyword input appends confirmed aliases
  to that field's harvested keywords, which improves `formSlotEngine.js` matching at no runtime
  cost.

**Regressed**: the onboarding call site ("Tag your specialty" on StaffOnboarding, which stored
`staff_specialty_wikidata_qid`) disappeared when StaffOnboarding was rebuilt on SPEC-24's hosts.
The backend still works; the UI hook is gone. Re-add it to `ProviderBasicsHost.vue`'s specialty
field if role-aware behavior (§3) is going to read it.

## 7. Sentence-transformer semantic layer (enterprise tier, last)

Moved to the enterprise tier alongside SPEC-07 Part C. Cübo mounts on every page, and a previous
NLP dependency with a fragile CommonJS graph once nearly stopped the app from mounting, so a
client-side embedding model is opt-in only. When built: a compact model (for example
`all-MiniLM-L6-v2` via Transformers.js), client-side by default, Workers AI embeddings as a
fallback. Both slot matching and intent matching would use it.

## 8. How the pieces compose

```
DESIGN TIME / ONBOARDING                 RUNTIME (every message)
field label or specialty chosen          user message
        │                                      │
Wikidata lookup (cached, confirmed)      clause split → single-intent classifier (§5)
        │                                      │
aliases added to field keywords          harness dispatch (§3) ──► registered actions
QID stored on the profile                      │
        └──► better NER slot matching ◄────────┘

ENTERPRISE: embeddings replace the keyword paths, same consumers underneath.
ALL CONVERSATIONS: one thread list (§4); the harness can act inside any of them.
```

## 9. Build order

See SPEC-08. In short: Wikidata tagging (done, with the onboarding hook to restore) → multi-intent
→ dispatch registry → group chat (independent) → unified inbox → enterprise embeddings.

## 10. Open questions

- Negative examples or margin rule for out-of-scope input (§5), required before multi-intent ships.
- Disambiguation UX when Wikidata offers several plausible concepts for a clinical term.
- Exactly what "role and preference awareness" reads beyond the account role and virtual room.
- Whether the dispatch registry (§3) and SPEC-26 §12's Task-driven chat actions should be one
  mechanism. They almost certainly should.
