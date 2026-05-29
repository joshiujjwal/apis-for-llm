# apis-for-llm — Task Breakdown

## How to Use This File

Workflow per task:
1. Write tests FIRST (red phase) — no production code until a failing test exists
2. Implement until tests pass (green phase)
3. Manually review the diff — read every line you changed
4. Commit with a descriptive message referencing the TODO item
5. Update `CLAUDE.md` / `AGENTS.md` if you discovered something non-obvious (compound loop)
6. Gate: all tests green + manual review before moving to next phase

---

## Phase 0: Foundation ⬜

- [ ] Init `package.json` with TypeScript, Fastify, Zod, Vitest, ESLint, Prettier
- [ ] `tsconfig.json` — strict mode, `src/` → `dist/`, path aliases (`@/` → `src/`)
- [ ] `.env.example` with all provider API key slots
- [ ] Vitest config + first smoke test (`src/index.ts` exports something)
- [ ] ESLint + Prettier config (no `any`, no unused vars)
- [ ] GitHub Actions CI: install → lint → test → build on every push
- [ ] Review and update all AI config files (CLAUDE.md, AGENTS.md, copilot-instructions.md)

**Gate:** CI passes on first push, smoke test green ✅

---

## Phase 1: Provider Adapters ⬜

Each adapter must implement the `LLMProvider` interface.

- [ ] Define `LLMProvider` interface in `src/providers/types.ts`
  - `chat(request: ChatRequest): Promise<ChatResponse>`
  - `stream(request: ChatRequest): AsyncIterable<ChatChunk>`
  - `countTokens(messages: Message[]): number`
  - `modelList(): Model[]`
- [ ] Write failing tests for the interface shape (red)
- [ ] OpenAI adapter (`src/providers/openai.ts`) — chat + stream
- [ ] Anthropic adapter (`src/providers/anthropic.ts`) — chat + stream
- [ ] Gemini adapter (`src/providers/gemini.ts`) — chat + stream
- [ ] Cohere adapter (`src/providers/cohere.ts`) — chat only (stretch)
- [ ] Provider registry (`src/providers/registry.ts`) — map name → adapter instance
- [ ] Unit tests: each adapter with mocked HTTP (MSW or nock)
- [ ] Integration test: real provider calls behind `INTEGRATION=true` flag

**Gate:** All unit tests pass, at least OpenAI + Anthropic adapters functional ✅

---

## Phase 2: Unified Request Format ⬜

- [ ] Zod schema for `ChatRequest` (model, messages, temperature, max_tokens, stream, …)
- [ ] Zod schema for `ChatResponse` and `ChatChunk`
- [ ] `normalizeRequest(raw)` — validate + strip unknown fields
- [ ] `denormalizeResponse(providerRes, provider)` — map provider-specific fields to unified shape
- [ ] Tests: round-trip for each provider's request/response format
- [ ] Streaming: unified `text/event-stream` SSE output regardless of provider

**Gate:** A single `ChatRequest` payload routes correctly to any adapter and returns `ChatResponse` ✅

---

## Phase 3: Routing Engine ⬜

- [ ] `RoutingStrategy` interface (`src/router/strategy.ts`)
- [ ] `SimpleStrategy` — explicit `model` field routes to correct provider
- [ ] `RoundRobinStrategy` — distribute across a pool of providers
- [ ] `CostOptimizedStrategy` — pick cheapest provider for given token estimate
- [ ] `FallbackChainStrategy` — try providers in order, next on failure
- [ ] Router composes strategy + provider registry
- [ ] Tests: each strategy with deterministic mocks

**Gate:** All 4 strategies tested and correct ✅

---

## Phase 4: Middleware Stack ⬜

- [ ] API key auth middleware — validate `Authorization: Bearer <key>` header
- [ ] Rate limiter middleware — per-key limit (in-memory sliding window)
- [ ] Request logger — structured JSON (provider, model, latency, tokens)
- [ ] Error handler — map provider errors to unified HTTP error codes
- [ ] Zod request validation middleware (400 on bad schema)
- [ ] Tests: middleware in isolation and integrated

**Gate:** Invalid keys get 401, rate-exceeded gets 429, malformed requests get 400 ✅

---

## Phase 5: Cost & Token Tracking ⬜

- [ ] Pricing table (`src/tracking/pricing.ts`) — per-model input/output cost per 1K tokens
- [ ] `TokenCounter` — wraps provider `countTokens`, stores per-request data
- [ ] `CostAccumulator` — aggregate cost per API key, per model, per day
- [ ] SQLite schema: `requests` table (key, provider, model, tokens_in, tokens_out, cost_usd, ts)
- [ ] `GET /v1/usage` endpoint — returns aggregated stats for the authenticated key
- [ ] Tests: pricing math for known models, accumulator aggregation

**Gate:** After 10 test requests, `/v1/usage` returns correct token and cost totals ✅

---

## Phase 6: REST API Surface ⬜

- [ ] `POST /v1/chat/completions` — main endpoint (OpenAI-compatible format)
- [ ] `GET /v1/models` — list available models across all configured providers
- [ ] `GET /v1/health` — liveness + provider connectivity check
- [ ] `GET /v1/usage` — cost/token stats (Phase 5)
- [ ] OpenAPI spec generated from Zod schemas (zod-to-openapi or similar)
- [ ] Integration tests: full request → response flow per endpoint

**Gate:** All endpoints return correct shape, OpenAPI spec validates ✅

---

## Phase 7: TypeScript SDK ⬜

- [ ] `src/sdk/client.ts` — `LLMGatewayClient` class
  - `chat(request)` — typed wrapper over `/v1/chat/completions`
  - `stream(request)` — returns `AsyncIterable<ChatChunk>`
  - `models()` — typed wrapper over `/v1/models`
  - `usage()` — typed wrapper over `/v1/usage`
- [ ] Auth via constructor: `new LLMGatewayClient({ baseUrl, apiKey })`
- [ ] SDK tests against a local test server
- [ ] Export as `npm` package (sub-package or workspace)

**Gate:** SDK can be imported and used to complete a chat round-trip in a test ✅

---

## Phase 8: Polish & Harden ⬜

- [ ] Retry logic with exponential backoff on 429/5xx from providers
- [ ] Timeout config per provider
- [ ] Structured error responses (`{ error: { code, message, provider } }`)
- [ ] `.env` validation at startup (fail fast if required keys missing)
- [ ] Load test: 100 concurrent requests, measure P99 latency
- [ ] Security audit: no key leakage in logs, no raw provider errors exposed
- [ ] Update README with final commands and examples

---

## Phase 9: Ship ⬜

- [ ] Docker image (`Dockerfile` + `.dockerignore`)
- [ ] `docker-compose.yml` with optional Redis for rate limiting
- [ ] Deploy guide in `docs/deploy.md`
- [ ] Changelog (`CHANGELOG.md`)
- [ ] Tag `v0.1.0`

---

## Parking Lot 🅿️

- WebSocket support for streaming
- Admin UI (React dashboard for cost tracking)
- Per-provider model capability matrix (vision, tools, etc.)
- Prompt caching layer
- Multi-modal support (image input normalization)
- Plugin system for custom providers

---

## Lessons Learned 📝

_Update this section as you discover non-obvious things._

- 
