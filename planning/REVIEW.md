# Review of `planning/PLAN.md`

## Overall Assessment

`PLAN.md` is ambitious, detailed, and mostly implementation-ready. It does a strong job of defining the product vision, user journey, backend/frontend architecture, and delivery phases for an agentic workflow. The plan is especially strong in:

- clear UX goals and demo value,
- practical single-container deployment constraints,
- API-first thinking,
- explicit trade simulation behavior,
- and cross-agent coordination intent.

That said, several critical specification gaps and consistency issues remain that can cause rework during implementation if not resolved up front.

## What's Working Well

1. **Product clarity**
   The target experience ("AI trading workstation" with live data + execution) is crisp and compelling.

2. **Architecture pragmatism**
   Single FastAPI container serving static frontend + APIs + SSE is a sensible scope choice for a capstone.

3. **Market data abstraction**
   Simulator and external provider behind a common interface is the right separation.

4. **Agentic value demonstration**
   LLM auto-execution and watchlist mutation are well aligned with course goals.

5. **Operational simplicity**
   SQLite + lazy init + seeded defaults reduce onboarding friction significantly.

## High-Priority Gaps (Must Address)

1. **API response contracts are incomplete**
   Request paths and methods are defined, but concrete JSON response schemas are not consistently specified for each endpoint. This can break frontend/backend parallel work.

2. **SSE payload details are underspecified**
   Event shape, batching behavior (single ticker vs array), heartbeat strategy, and reconnect semantics are not fully pinned down.

3. **Backend application structure is not explicit enough**
   Missing concrete module boundaries and lifecycle details (startup tasks, shutdown cleanup, dependency wiring), which are important for multi-agent implementation.

4. **LLM implementation path is inconsistent**
   Plan references LiteLLM + OpenRouter + Cerebras-specific guidance, but dependency and fallback behavior are not fully settled. Pick one client path explicitly and lock it.

5. **Trade/watchlist edge cases need explicit rules**
   Undefined behavior for:
   - buying a ticker not currently in watchlist,
   - removing a watched ticker with open position,
   - partial failures in multi-trade LLM actions.

## Medium-Priority Issues

1. **Data retention details are thin**
   Sparkline arrays need a hard cap to avoid unbounded memory growth in long sessions.

2. **Portfolio history strategy conflicts with implementation cost**
   Periodic snapshots + live interpolation can be simplified; at minimum define exactly what is persisted and why.

3. **Schema carries early multi-user complexity**
   `user_id="default"` everywhere adds conceptual load without current value. Acceptable if future-proofing is mandatory, but otherwise simplify.

4. **Charting guidance needs technical correction**
   Some chart-library notes are mismatched (e.g., rendering model assumptions).

5. **Developer workflow section should be stronger**
   Local dev loop (frontend hot reload + backend reload + proxying) should be explicit to avoid setup drift.

## Suggested Decisions to Lock Now

1. **Define canonical API schemas** for all endpoints with examples.
2. **Specify exact SSE event contract** (event name, payload, cadence, heartbeat).
3. **Choose one LLM client stack** (`litellm` *or* OpenAI-compatible client via OpenRouter).
4. **Set chat history window** (e.g., last 20 messages).
5. **Set deterministic trade execution policy** for batched LLM actions (sequential best-effort with per-item status).
6. **Define watchlist-position interaction policy** (allow remove, block remove, or soft-remove with flags).
7. **Cap client sparkline points** (e.g., 200 per ticker rolling window).

## Implementation-Readiness Verdict

**Verdict: 8/10 — strong foundation, but not fully execution-safe yet.**

The plan is close to implementation-ready. After locking contracts and edge-case behavior above, it should support efficient parallel development with minimal rework.

## Recommended Next Step

Before coding starts, add a short "**Spec Lock Addendum**" section to `PLAN.md` that finalizes:

- endpoint response schemas,
- SSE payload contract,
- LLM client choice and fallback model,
- trade/watchlist edge-case rules,
- and data retention limits.

This should be the single source of truth for all agents.
