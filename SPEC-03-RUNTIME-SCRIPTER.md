# SPEC-03: Runtime Inference Constraint and State Patch Engine

| | |
|---|---|
| **Status** | Superseded. The design was never wired into the product. Forward design: BLOG-07 (grammar-constrained extraction) and SPEC-17 (conversation layer). |
| **Last reviewed** | 2026-09-26 |
| **Code** | `clinuxflow-api/src/lib/runtime/runtime-scribe-engine.js` (unused), `src/routes/runtime.js` (`test-scribe`, `test-scribe-mock`), `clinux-frontend/src/nlp/*` |

## 1. The original idea

Turn a clinical conversation into form answers with a local model whose output is **constrained
by the compiled form**: derive a mask from the active Questionnaire's fields, force the model to
emit a flat JSON object keyed by those fields, then patch the values into the live form, leaving
hidden administrative fields alone and re-running dependent calculations (e.g. an abnormal-value
flag) afterwards.

The idea is still right. Constraining a model's output to the shape of the form it is filling is
the single most effective guard against fabricated structure, and it is the thread BLOG-07 picks
up with GBNF grammars.

## 2. What exists instead

| Piece | Reality |
|---|---|
| `RuntimeScribeEngine` | Exists in `src/lib/runtime/runtime-scribe-engine.js`. Nothing imports it. |
| `POST /api/workflow/test-scribe` | The live path. `requireUser()` + `requirePaidTier()`. One inline call to Workers AI `@cf/meta/llama-3.3-70b-instruct-fp8-fast` with the active fields in the prompt; the route does its own field matching. Output is prompted, not grammar-constrained. |
| `POST /api/workflow/test-scribe-mock` | Offline stand-in with the same response shape. |
| Client-side matching | `formSlotEngine.js` (NER over field labels) and `fieldShortcuts.js` (`/shortcut value`). Matches are shown as highlighted suggestions (`slotFillHighlights` store) before they land in the form. |
| State patching | `LhcFormHost.vue` re-renders with the updated record. There is no separate patch engine and no dependent-field recalculation beyond what LHC-Forms' own `calculatedExpression` does. |

## 3. Why the original plan stalled

- The product moved to Cloudflare Workers, where there is no local Ollama or llama.cpp to
  constrain; Workers AI's hosted models don't take custom grammars.
- The first real work was registration and ABDM, not ambient documentation.
- Owned-hardware inference (the nano data centre, SPEC-07 Part C) was deferred to an enterprise
  tier.

## 4. What still carries forward

1. **Output shape comes from the form.** Whatever model runs, its schema is generated from the
   compiled Questionnaire, never hand-written per prompt.
2. **Suggest, then confirm.** Model output is a proposal. Coded fields in particular must resolve
   through an explicit picker (SPEC-12 §4.3), never raw model text.
3. **Provenance.** Every value a model proposes should record that it came from a model, which
   model, and who accepted it. This is not built.

## 5. Open items

- Delete `runtime-scribe-engine.js`, or revive it as the llama.cpp adapter behind the nano-DC
  gateway (SPEC-17 §9 asks the same question).
- Decide where grammar generation lives: Questionnaire → GBNF at compile time (clinuxflow-api) or
  at request time in the nano-DC gateway. See BLOG-07.
