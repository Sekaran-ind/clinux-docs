# SPEC-17: Multi-LLM Conversation Layer (TanStack AI)

| | |
|---|---|
| **Status** | Design. Not adopted; no `@tanstack/ai` dependency in either repo. |
| **Last reviewed** | 2026-09-26 (release facts are as of the original 2026-08 research and need re-checking before adoption) |
| **Code today** | `clinuxflow-api/src/routes/runtime.js` (`POST /api/workflow/test-scribe`, one inline `c.env.AI.run('@cf/meta/llama-3.3-70b-instruct-fp8-fast', ...)`) |
| **Related** | SPEC-03, SPEC-07 Part C, SPEC-12 §4.3, SPEC-14, BLOG-07 |

## 1. Purpose

Adopt a provider-agnostic LLM conversation layer for the parts of Cübo that call a model (SOAP
drafting, free-text disambiguation, narrative), layered under the deterministic workflow runtime.

## 2. What TanStack AI is

A type-safe, provider-agnostic TypeScript SDK for streaming chat, tool calling, agent loops,
structured output and multimodal generation, built as composable adapters rather than a
monolith. It supports isomorphic tools (`.server()`/`.client()`) and tool-approval flows for
human-in-the-loop. At the time of research: alpha December 2025, beta June 2026, release
candidate around August 2026, not yet 1.0.

## 3. Why it fits

- The app already uses `@tanstack/db`, `@tanstack/vue-db`, `@tanstack/pacer` and
  `@tanstack/vue-query`.
- Provider-agnostic adapters match the planned mix: Workers AI now, nano-DC models (MedGemma and
  others) behind a Cloudflare Tunnel later.
- **Tool approval** is close to exactly what SPEC-12 §4.3 needs: model output for a coded field
  is a proposal a human must confirm.

## 4. What it would replace

A single inline call, not a mature abstraction. `test-scribe` builds its prompt from the active
fields and does its own field matching in the route handler. `RuntimeScribeEngine` exists but is
unused (SPEC-03 §2).

## 5. Layering

XState owns which room and state are active and what is in bounded context. The conversation
layer works only inside that: the tools it exposes on a turn are scoped to the active context,
never a fixed global set.

## 6. Caution

Pre-1.0, on the component that mounts on every page. The same mount-stability checkpoint as
SPEC-06 §7 applies.

## 7. Build order

1. Bundle and mount checkpoint.
2. Wrap the existing Workers AI call in an adapter, with no behavior change.
3. Add a second adapter (even a local mock) to prove provider independence in this codebase.
4. Use tool approval for coded-value confirmation.
5. Scope tools per active context before any multi-step agent behavior runs.

## 8. Related specs

SPEC-06 §7 (mount caution), SPEC-12 §4.3 (first real use), SPEC-14 §2 (same dependency
discipline), SPEC-07 Part C (the multi-backend future).

## 9. Open items

- Retire `RuntimeScribeEngine`, or revive it as the llama.cpp path in the nano-DC gateway.
- The mechanism connecting XState's context to the tool set.
- Wait for 1.0, or proceed on the release candidate behind the checkpoint.
- Re-check the library's current release status before starting.
