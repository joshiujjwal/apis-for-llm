# GitHub Copilot Instructions — apis-for-llm

## Project Context

This is a unified LLM API gateway written in TypeScript + Node.js (Fastify). It proxies requests to OpenAI, Anthropic, Gemini, and other LLM providers through a single OpenAI-compatible REST interface, with routing strategies, rate limiting, and cost tracking.

## Stack

- **Runtime**: Node.js 20 LTS
- **Language**: TypeScript 5.x (strict mode)
- **HTTP Framework**: Fastify
- **Validation**: Zod (all I/O boundaries)
- **Testing**: Vitest + MSW (HTTP mocking)
- **Database**: better-sqlite3 (synchronous SQLite)

## Coding Conventions

- Named exports only — no `export default`
- Zod schemas for every HTTP request body, response shape, and provider API response
- All provider HTTP calls must go through the adapter in `src/providers/` — never call provider APIs from middleware or routes directly
- Errors must be typed: define and throw `ProviderError`, `AuthError`, `RateLimitError` classes from `src/middleware/error-handler.ts`
- Streaming uses `AsyncIterable<ChatChunk>` — adapters yield chunks, router forwards them
- Pricing values are USD per **single token** (not per 1K) — document the unit in the pricing table

## Testing Conventions

- Every new function needs a test in `tests/unit/`
- Mock all LLM provider HTTP with MSW — no real network in unit tests
- Integration tests go in `tests/integration/` and require `INTEGRATION=true`
- Test names: `describe("ModuleName > functionName")` → `it("returns X when Y")`

## Boundaries — What NOT to Do

- Do not add `any` types — use `unknown` and narrow explicitly
- Do not remove or skip existing tests
- Do not refactor unrelated code unless asked
- Do not expose raw provider error messages to HTTP clients — always go through error-handler
- Do not add new dependencies without updating CLAUDE.md
- Do not hardcode API keys or model pricing — keys go in `.env`, pricing goes in `src/tracking/pricing.ts`
- Do not add business logic inside Fastify route handlers — routes call router, router calls adapters
