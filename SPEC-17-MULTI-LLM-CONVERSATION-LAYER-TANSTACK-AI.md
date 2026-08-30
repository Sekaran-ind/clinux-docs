# Specification 17: Multi-LLM Conversation Layer — TanStack AI

## 1. Objective

Adopt TanStack AI as Cübo's LLM-conversation layer — the parts of Cübo that actually call a model (narrative generation, free-text disambiguation, SOAP drafting) — distinct from and layered under the HFSM (`docs/SPEC-14-HFSM-RUNTIME-AND-CHAT-FIRST-CAPTURE.md`), which continues to own deterministic domain-workflow state. Verified this session against current release information rather than training-data recall, given how fast this specific project is moving (RC five days old at time of writing).

## 2. What TanStack AI is — verified this session

A type-safe, provider-agnostic TypeScript SDK for streaming chat, tool calling, agents, structured outputs, and multimodal generation, built as composable pieces (adapters, tools, agent loops, transport, UI) rather than a monolith. Isomorphic tools defined once with `.server()`/`.client()` implementations. Tool-approval flows for human-in-the-loop. Agentic cycle management for multi-step, autonomous tasks. Release timeline: Alpha Dec 4, 2025 → Beta June 9, 2026 ("every modality, hardened protocol, middleware, orchestration, host-side MCP, 265 E2E tests across 10 providers") → RC ~Aug 22, 2026 ("architecture locked in, 24 providers, AG-UI, media generation, MCP, sandboxes, persistence"). Not yet 1.0/stable.

Sources: [TanStack AI overview](https://tanstack.com/ai/latest/docs/getting-started/overview), [Agent Skills docs](https://tanstack.com/ai/latest/docs/getting-started/agent-skills), [TanStack AI Beta announcement](https://tanstack.com/blog/tanstack-ai-beta), [TanStack AI RC announcement](https://tanstack.com/blog/tanstack-ai-rc), [TanStack AI vs Vercel AI SDK](https://tanstack.com/ai/latest/docs/comparison/vercel-ai-sdk), [Better Stack overview](https://betterstack.com/community/guides/ai/tanstack-ai/).

## 3. Why this fits, concretely — not ecosystem convenience alone

- **Ecosystem continuity**: `@tanstack/db`, `@tanstack/vue-db`, `@tanstack/pacer`, `@tanstack/vue-query` are already dependencies of `clinux-frontend`.
- **Provider-agnostic adapters** match a multi-backend LLM future ClinuxFlow is already committed to, not a hypothetical: `docs/SPEC-07-SNOMED-CLINICAL-CHAT.md` Part C's nano-DC gateway plans routing to MedGemma/MedCAT over a Cloudflare Tunnel, alongside the existing Workers AI call (§4).
- **Tool-approval flows** are close to a direct implementation vehicle for `docs/SPEC-12-ROOMS-AND-BOUNDED-CONTEXT-SLOT-FILLING.md` §4.3's rule: coded-value fields never accept raw model output without an explicit human confirm step. This isn't a generic feature that happens to sound relevant — it's close to the exact mechanism that rule needs.

## 4. What it replaces — verified this session, not assumed

The current live LLM call is a single hardcoded inline binding, not a mature abstraction layer this would be displacing:

- `clinuxflow-api/src/index.js:1015` — `POST /api/workflow/test-scribe`, gated by `requireUser()` + `requirePaidTier()`.
- `clinuxflow-api/src/index.js:1035` — `c.env.AI.run('@cf/meta/llama-3.3-70b-instruct-fp8-fast', {...})`. One provider, one hardcoded model, inline (non-reusable) field-matching logic in the route handler itself.
- A separate, more structured engine exists — `clinuxflow-api/src/lib/runtime-scribe-engine.js` (`RuntimeScribeEngine`, SPEC-03) — but its own header comment states plainly: "this class is not currently imported by server.js." Dead/standalone code, not the live path; `/api/workflow/test-scribe` and its offline stand-in `/api/workflow/test-scribe-mock` implement their own inline logic instead of using it.

**TanStack AI upgrades a minimal, non-abstracted single call site — it does not displace a working, mature system.** This resolves the open question from the prior session's discussion.

## 5. Relationship to the HFSM — layering, not competing

XState (SPEC-14) owns "what room/state are we in, what's currently in bounded context" — deterministic, not model-driven. TanStack AI owns the model-conversation mechanics within whatever the HFSM currently permits: its exposed tool-set on any given turn must be scoped to the active bounded context (SPEC-12 §4.2), not a fixed global surface. The HFSM constrains; TanStack AI executes inside that constraint. This is the same layering discipline already applied to RxJS vs. XState (SPEC-14 §2) — event/model plumbing under a deterministic state authority, not two authorities competing.

## 6. Caution, held to the same standard as every other dependency in this series

RC, not 1.0, five days old at time of writing. "Architecture locked in" per TanStack's own release notes is real de-risking, but `docs/SPEC-06-CUBO-AGENTIC-HARNESS.md` §7 already documents a live near-miss in this exact app — a fragile dependency on Cübo's critical path nearly broke its ability to mount. Same bundle-size/mount-stability checkpoint applies here, not skipped because the org is already trusted elsewhere in this codebase.

## 7. Build order

1. Bundle-size/mount-stability checkpoint (§6) — before anything else lands on Cübo's critical path.
2. Replace `index.js:1035`'s inline `test-scribe` call with a TanStack AI adapter wrapping the same Workers AI binding — smallest possible first step, no behavior change, proves the integration works before anything depends on it.
3. Add a second provider adapter (even a mock/local one) to prove the provider-agnostic claim is real in this codebase, not just in TanStack's own docs.
4. Wire tool-approval flows for SPEC-12 §4.3's coded-value confirm requirement — the concrete first real use of the agent-loop/tool-calling capability, not a generic demo.
5. Scope the exposed tool-set per active HFSM bounded context (§5) — required before any broader multi-step agentic behavior is allowed to run unconstrained.

## 8. Relationship to existing specs

- `docs/SPEC-06-CUBO-AGENTIC-HARNESS.md` §7 — the bundle-size caution this spec's §6 is held to.
- `docs/SPEC-12-ROOMS-AND-BOUNDED-CONTEXT-SLOT-FILLING.md` §4.3 — the concrete first use case for tool-approval flows.
- `docs/SPEC-14-HFSM-RUNTIME-AND-CHAT-FIRST-CAPTURE.md` §2 — the parallel technology-decision precedent (RxJS/XState) this spec follows the same discipline as.
- `docs/SPEC-07-SNOMED-CLINICAL-CHAT.md` Part C — the multi-backend future this spec's provider-agnostic choice serves.

## 9. Open items

- Whether `RuntimeScribeEngine` (SPEC-03, currently unwired) should be retired now that TanStack AI covers its intended role, or kept for the standalone (non-Workers) use case its own comment describes.
- Exact mechanism connecting XState's current bounded context to TanStack AI's tool availability (§5) — not yet designed, just required.
- Whether to wait for TanStack AI's 1.0/stable release before wiring it into any paid/production route, or proceed on RC with §6's checkpoint as the gate.
