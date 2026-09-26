# SPEC-08: Cübo Harness, SNOMED and Enterprise: Build Sequence

| | |
|---|---|
| **Status** | Partially built. Phase 1 done except its onboarding hook and content batch; phases 2–9 not started. |
| **Last reviewed** | 2026-09-26 |
| **Related** | SPEC-05 §6.3, SPEC-06, SPEC-07 |

## 1. Purpose

One build order across SPEC-06 (harness), SPEC-07 (SNOMED) and the enterprise tier, with the
dependencies and the test checkpoint for each phase.

**Testing posture.** The product is pre-pilot: no production users and no accumulated real data.
Phases may rework schemas outright instead of migrating them. Every checkpoint still requires
live verification that new behavior is correct; it does not require proving that old behavior is
byte-identical. Revisit this once a pilot clinic is live.

## 2. Phases

Numbered phases are the critical path. ∥ marks phases that can run in parallel once their own
prerequisite is met.

### Phase 0: unblockers (all resolved)
- SNOMED licensing: via NRCeS for India (SPEC-07).
- Starting specialty list: General Medicine + 11 (SPEC-07).
- `node-nlp` multi-label shape: softmax-normalized, so the mechanism is clause splitting
  (SPEC-06 §5). This also exposed the out-of-scope misclassification problem.
- Group chat transport: full P2P mesh (SPEC-06 §4).
- Still open: the Wikidata disambiguation UX in Designer.

### Phase 1: Wikidata design-time and onboarding tagging (SPEC-06 §6)
- **Done**: backend routes, KV cache, 429 handling, Designer's "Suggest from Wikidata". A
  cache-miss search measured 619 ms and the repeat call 3 ms.
- **Regressed**: the StaffOnboarding specialty tagging hook (lost in the SPEC-24 rebuild).
- **Not run**: the content pre-build batch (tag common field vocabulary for each starting
  specialty so the cache isn't empty on day one).
- Checkpoint still owed: show that tagging measurably improves a paraphrase `formSlotEngine.js`
  previously missed.

### Phase 2: multi-intent (SPEC-06 §5)
Clause splitting over the existing classifier, plus negative examples or a margin rule.
Checkpoint: "generate the SOAP note and switch to cardiology" fires both actions in live Cübo; an
unrelated sentence no longer scores as a trained intent.

### Phase 3: dispatch registry (SPEC-06 §3), after phase 2
Replace `Cubo.vue`'s fixed chain with registered actions, role-aware. Converge with SPEC-26 §12's
Task-driven chat actions rather than building two registries. Checkpoint: every existing
single-action behavior still works; one multi-intent message triggers several actions in order.

### Phase 4: SNOMED Part B (SPEC-07), after phase 1
Pre-built subsets per role, interim keyword matching with Wikidata synonyms, chips, tooltips,
safety banner, local rules table. Checkpoint: curated phrase → concept fixtures per role; role
scoping actually restricts results; the drug-versus-allergy banner fires across threads; bundle
size measured.

### Phase 5 ∥: group chat (SPEC-06 §4)
N-socket signaling, a mesh of peer connections. Checkpoint: three or more real browser contexts
all receive all messages; join and leave work; nothing is stored server-side.

### Phase 6: unified inbox, after phases 3 and 5
Merge AI, one-to-one and group threads into Cübo's thread list; retire `TeamChat.vue`.
Checkpoint: re-verify the slot-fill highlight, pacer and slash shortcuts, which have broken
before; encounter-scoped routing still works.

### Phase 7 ∥: dentistry and optometry modules, after phase 4
Odontogram component, optometry YAML block, severity-triggered routing through the existing
assignment path.

### Phase 8 ∥: enterprise plumbing (SPEC-05 §6.3)
`clinics.nano_dc_endpoint`, `requireNanoDcAccess()`, admin provisioning. Checkpoint mirrors the
`requirePaidTier()` tests: free denied, paid without endpoint denied, paid with endpoint allowed.

### Phase 9: enterprise features, after phase 8 and a real Cloudflare Tunnel
Sentence-transformer layer (SPEC-06 §7) and the nano-DC gateway (SPEC-07 Part C). Checkpoints:
mount stability with the embedding model; free and paid clinics keep the interim paths; each
model reachable through the real tunnel; a concurrent-load test for shared mode; one end-to-end
case (message → MedGemma audit → banner in the UI).

## 3. Summary

| Phase | Depends on | Parallel with | State |
|---|---|---|---|
| 0 Unblockers | — | all | Resolved (one UX item open) |
| 1 Wikidata tagging | 0 | — | Mostly done |
| 2 Multi-intent | 0 | — | Not started |
| 3 Dispatch registry | 2 | — | Not started |
| 4 SNOMED Part B | 1 | 5 | Not started |
| 5 Group chat | — | 2–4 | Not started |
| 6 Unified inbox | 3, 5 | — | Not started |
| 7 Dentistry/optometry | 4 | 5–6 | Not started |
| 8 Enterprise plumbing | — | any | Not started |
| 9 Embeddings + nano-DC gateway | 8 | — | Not started |
