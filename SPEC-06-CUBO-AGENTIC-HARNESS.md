# Specification 06: Cübo Agentic Harness — Unified Chat, Multi-Intent Understanding & Semantic NLP

## 1. Objective

Evolve Cübo from a reactive, single-intent NLP assistant into an agentic harness: a personal assistant aware of a user's role/preferences, able to recognize multiple intents in one utterance, and able to act across a unified inbox that holds 1:1 chat, group chat, and AI chat in one place — the same way WhatsApp treats all three as one conversation list, not three separate apps. This spec is the reference point for that work; nothing here is built yet except where explicitly marked.

## 2. Current state (grounded in the real code, not the aspirational product-blog version)

Cübo today is a dispatch chain of independent, narrow, rule-based systems, each real and working, none semantic:

- **`nlp/intents.js`** — `node-nlp`'s `NlpManager`, a small hand-trained vocabulary (`generate_soap`, `switch_room`), single-best-intent (argmax), threshold-gated, falls back to `general_chat`. Genuinely single-label: one utterance → one intent, ever.
- **`nlp/formSlotEngine.js`** — `@nlpjs/nlp`'s NER, trained on harvested field label/keyword/description text across every system form. Token/keyword-based, not semantic — collision-prone by its own documented design ("whichever field was registered last under a colliding trigger text wins").
- **`nlp/fieldShortcuts.js`** — a deterministic `/shortcut` escape hatch precisely because the NER matching above is ambiguous; bypasses NLP matching entirely via guaranteed-unique keys.
- **`chatThreads.js`** — Cübo's own AI-assistant conversation history, one thread per category/encounter. This is the "AI chat" leg of the eventual unified inbox.
- **`userChats.js` / `TeamChat.vue` / `p2pChat.js`** — true P2P 1:1 user-to-user chat, shipped this session, currently a *separate* panel from Cübo's own window (a deliberate scope decision at the time, to avoid destabilizing Cübo's existing logic). This is the "1:1 chat" leg.
- **No group chat exists yet.** No multi-intent output exists yet. No semantic/embedding-based matching exists anywhere in the NLP stack — everything above is token/keyword/Naive-Bayes-style, not dense-vector similarity.

## 3. Target: Cübo as an agentic harness

The distinction that matters: today, Cubo.vue's send path is a hardcoded chain — try `classifyIntent`, else try slot-fill, else try a shortcut, else fall through to the LLM scribe call — one branch fires per message, chosen by whichever check happens to match first. An agentic harness instead means:

- **Multiple recognized intents per utterance** get dispatched together, not just the single best-scoring one (§5).
- **Role/preference awareness** — the existing "virtual room" persona (speciality/role selection) becomes an actual input to what the harness considers available/relevant, not just a display label.
- **A registered set of actions** (generate SOAP, switch room, fill a slot, message a colleague, — and new ones as they're added) that the harness selects among based on understood intent(s), rather than each one being its own isolated code path in Cubo.vue.
- **Multi-step sequencing** becomes possible ("assign this patient to Dr. Singh and let her know") once intents are no longer mutually exclusive.

This is an orchestration layer to add on top of the existing action handlers, not a rewrite of them — `generate_soap`, `switch_room`, slot-fill application, and shortcut resolution all keep their current implementations; the harness's job is deciding *which of them fire, together, for a given message*.

## 4. Unified inbox: 1:1 chat, group chat, AI chat in one place

Target: Cübo's own thread list becomes the single home for every conversation type, matching the WhatsApp mental model directly:

- **AI chat** — today's `chatThreads.js`, unchanged in substance.
- **1:1 chat** — today's `userChats.js`/`p2pChat.js`, currently surfaced in the separate `TeamChat.vue` panel; this unification is what actually merges it into Cübo's own window rather than a sibling panel.
- **Group chat** — does not exist yet. **Resolved (Phase 0): full P2P mesh**, not a hub/SFU. A hub/SFU would mean messages pass through a central relay — a real departure from "nothing stored in the interim," the principle this whole chat feature was explicitly built around from the start, not a detail to trade away for scale this app doesn't need: clinic groups (a shift-handover thread, one patient's care team, a department) are inherently small, 2–8 people, well within what a full mesh handles fine. `ChatSignalingRoom.js` today hard-assumes exactly 2 sockets per room; group chat needs it extended to N sockets with broadcast relay, and each client maintaining one `RTCPeerConnection` per other participant. Explicitly not aiming for large-broadcast-style groups — that was never the goal here.

## 5. Multi-intent classification — interim mechanism, no new dependency

**Phase 0 verified this live against the actual pinned `node-nlp@5.0.0-alpha.5`**, training the exact same two-intent shape `intents.js` already uses. Two findings, one confirming the plan, one overturning the original mechanism:

- **Confirmed**: `NlpManager.process()` does expose a full `classifications` array (`[{intent, score}, ...]`) alongside the single best match `intents.js` already reads — the data needed for multi-intent is genuinely there, no new dependency required.
- **Overturned**: those scores are **softmax-normalized — they sum to 1 across all trained intents** (verified: 0.6465 + 0.3535 = 1.0 exactly on a mixed-intent input). That's a mutually-exclusive multi-*class* distribution, not independent multi-*label* scores — thresholding each one independently doesn't work as originally planned, because every intent's score is mechanically suppressed by however many *other* intents exist, unrelated to whether that intent is actually present.

**Revised mechanism**: split the utterance on conjunctions/clause boundaries (e.g. "and", punctuation) and classify each clause independently through the *same* existing single-intent classifier, then union the results. "Generate the soap note **and** switch to cardiology" splits into two clauses, each confidently and independently classified (no softmax competition between them), rather than trying to coax genuine multi-label behavior out of a classifier that isn't built for it. Reuses the exact single-intent behavior already working today, per clause.

**A second, separate finding worth flagging, not just noted in passing**: the same live check showed a completely unrelated input ("what is the weather") still returned `generate_soap` at `score: 1` — with only two narrow trained intents and no explicit "none of the above" class, the classifier doesn't discriminate out-of-scope input at all, confidently. This is a pre-existing weakness in *today's* single-intent classifier too (not introduced by this change), just newly surfaced by actually testing it rather than assuming the threshold check was sufficient. Worth addressing (broader/more diverse training data, explicit negative examples, or a top-2-margin check instead of a bare threshold) independent of the multi-intent work.

"Generate the SOAP note and switch to cardiology" recognizing both intents is still the concrete acceptance case — met by clause-splitting the existing classifier's output now, not raw multi-label thresholding.

**Future upgrade (Enterprise tier, §7)**: once the sentence-transformer lands, multi-intent moves to per-intent exemplar-embedding similarity — genuinely independent scores, no clause-splitting heuristics needed, and handles paraphrasing the keyword-trained classifier can't ("wrap up the note" vs. the exact-trained "generate the soap note"). The interim mechanism isn't thrown away — it's what free/Cloud tier keeps running permanently; Enterprise tier gets the richer version on top.

## 6. Wikidata-based semantic tagging — design-time & onboarding (the real interim semantic layer)

Scoped deliberately narrow: **design-time and onboarding-time only, never runtime chat classification.** This is what actually improves semantic coverage in the interim, without the bundle-size/reliability risk a client-side embedding model would add to a component already once nearly broken by an NLP dependency.

**Two call sites:**
- **Forms Designer / YAML authoring** — when a form author defines a field label (e.g. "Chief Complaint," "Systolic BP"), query Wikidata for matching concepts, let the author confirm the right one (disambiguation matters — Wikidata is general-knowledge, not a clinical terminology, so a "closest match" can be wrong or absent for clinically-precise terms). Once confirmed, the concept's aliases/synonyms get appended to that field's harvested `keywords` (`formFieldHarvester.js`'s existing output) — directly improving `formSlotEngine.js`'s NER matching today, with zero runtime cost, since this only ever runs once, at authoring time.
- **Onboarding** — role/specialty selection queries the same lookup, storing the matched QID + related concepts on the account/profile. This is the concrete "how will this be used in the main flow" resolution for the staff-specialization Wikidata tagging work paused earlier — it feeds the agentic harness's role/preference-awareness (§3) and the field-tagging above, not a standalone feature.

**Mechanism — backend proxy + cache, not a direct client-side call, for a concrete reason, not just convention**: Wikidata's SPARQL endpoint (`query.wikidata.org`) is public and CORS-enabled, so a direct browser call is technically possible — but its usage policy requires a compliant, app-identifying `User-Agent` header, which browsers refuse to let JS override (a forbidden header). A backend module in clinuxflow-api can set a real `User-Agent` and is the only way to comply. It also turns per-user query volume into one shared, cacheable budget: Wikidata's real rate limit is 60s of query time per 60s **per client (IP+User-Agent)** — since every ClinuxFlow clinic would otherwise share Cloudflare's egress identity, caching common terms (same principle as the ABDM master-data cache) isn't optional polish, it's what keeps the app under that shared budget as usage grows.

- `GET /api/nlp/wikidata-lookup?term=...` — checks a cache (KV or D1) first; on miss, queries Wikidata (SPARQL for hierarchy/related-concept traversal, or the simpler `wbsearchentities` search API where a plain closest-match lookup is all that's needed) with a real `User-Agent`; stores `{qid, label, description, aliases, fetchedAt}`.
- Respects `429`/`Retry-After` from Wikidata rather than retrying blindly.

## 7. Sentence-transformer semantic layer — moved to Enterprise tier

**Revised**: no longer Phase 1 / foundational. Sequenced with SPEC-07 Part C as one of the last features, gated the same way (Enterprise tier — `docs/SPEC-05-DATA-TIER-AND-ABDM-BOUNDARY.md` §6). Reasoning: it was already the spec's own flagged risk (Cübo sits on every page's critical path, and a previous NLP dependency's fragile CJS graph nearly broke the app's ability to mount at all) — deferring it to an opt-in tier means free/Cloud-tier Cübo never carries that risk, while §5's interim mechanism and §6's Wikidata tagging already deliver most of the practical semantic value without it.

**Where it runs, once built:**
- **Client-side (still the default within Enterprise tier)** — a compact model (e.g. `all-MiniLM-L6-v2`) via Transformers.js/ONNX. Even as an Enterprise feature, no reason to prefer a server round-trip over running it locally on the enterprise user's own device.
- **Server-side** — only if the client-side model proves insufficient; Workers AI already has embedding models available via the same binding `test-scribe` uses.

Both `formSlotEngine.js`'s field-matching and `intents.js`'s intent-matching would consume the same embedding primitive once built, same as originally planned — just later, and behind a gate.

## 8. How the pieces compose

```
  DESIGN TIME / ONBOARDING (§6, Wikidata)          RUNTIME (every message, free/Cloud tier)
  ────────────────────────────────────             ──────────────────────────────────────
  field label / role picked                         user message
       │                                                  │
       ▼                                                  ▼
  Wikidata lookup (cached)                          node-nlp classification (§5)
       │                                                  │
       ▼                                                  ├──► per-intent threshold scoring (multi-intent, no argmax)
  synonyms appended to field keywords,                     │         │
  QID stored on profile                                    │         ▼
       │                                                    │   [ agentic harness dispatch (§3) ] ──► generate_soap / switch_room / message-a-colleague / ...
       └──► improves NER slot-fill matching                │
            immediately, no runtime cost                    └──► NER slot-fill matching (enriched by §6's synonyms)

  ENTERPRISE TIER ONLY (§7, sentence-transformer) — replaces the node-nlp/NER paths above with
  embedding similarity once built, same dispatch/slot-fill consumers underneath.

  All conversation surfaces (AI chat, 1:1 chat, group chat) ──► one unified thread list (§4),
  with the harness able to act inside any of them, not just the AI-chat leg.
```

## 9. Phased build order

1. **Wikidata design-time/onboarding tagging (§6)** — foundational now, free/Cloud-tier-safe, no bundle-size risk. Build first.
2. **Multi-intent classification, interim mechanism (§5)** — node-nlp's own per-label scoring; depends on (1) only for better keyword coverage, not a hard blocker.
3. **Agentic harness dispatch layer (§3)** — depends on (2) plus the existing action handlers; role/preference-awareness reads from onboarding's Wikidata-tagged profile (§6).
4. **Group chat** — extends `ChatSignalingRoom.js`/`p2pChat.js`; can proceed in parallel with (1)–(3) once the mesh-vs-hub decision (§4) is made.
5. **Unified inbox UI** — merges `chatThreads.js` + `userChats.js` + group threads; sequenced after group chat (4) exists.
6. **Sentence-transformer (§7) + SNOMED Part B's semantic layer — Enterprise tier, last**, alongside SPEC-07 Part C. Full detail in `docs/SPEC-08-CUBO-BUILD-SEQUENCE.md`.

## 10. Open design questions

- ~~Group chat transport~~ — **resolved (§4)**: full P2P mesh, scoped to small clinic-sized groups.
- ~~`node-nlp`'s multi-label result shape~~ — **verified live (§5)**: `classifications` array exists but is softmax-normalized, not independent scores; mechanism revised to clause-splitting instead of per-intent thresholding.
- **New, surfaced by that same live check**: the classifier shows no discrimination on out-of-scope input with only 2 narrow trained intents (confidently misclassifies unrelated text) — needs addressing (broader training data, explicit negative examples, or a top-2-margin check) before Phase 2 ships, not deferred as a known-acceptable gap.
- Wikidata concept-matching quality for clinically-precise terms — a real, expected limitation (§6), not something to discover live; needs a disambiguation UX for the Designer flow specifically, not silent best-guessing.
- What "role/preference awareness" concretely reads from beyond the virtual-room persona and §6's onboarding tag — not yet fully scoped.
