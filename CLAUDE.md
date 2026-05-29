# CLAUDE.md — apis-for-llm

## Commands

```bash
npm run dev          # Fastify dev server with hot reload (tsx watch)
npm run build        # tsc → dist/
npm start            # node dist/index.js
npm test             # vitest run (all tests)
npm run test:watch   # vitest watch mode
npm run test:int     # integration tests (requires INTEGRATION=true + real keys)
npm run lint         # eslint src/ tests/
npm run fmt          # prettier --write src/ tests/
npm run typecheck    # tsc --noEmit
```

## Directory Map

```
src/
  index.ts              Entry point — builds and starts Fastify server
  providers/
    types.ts            LLMProvider interface + ChatRequest/ChatResponse types
    registry.ts         Map of provider name → adapter instance
    openai.ts           OpenAI adapter
    anthropic.ts        Anthropic adapter
    gemini.ts           Gemini adapter
  router/
    index.ts            Router — wires strategy + registry
    strategy.ts         RoutingStrategy interface
    simple.ts           Route by model name
    round-robin.ts      Distribute across provider pool
    cost-optimized.ts   Pick cheapest provider
    fallback.ts         Try in order, next on failure
  middleware/
    auth.ts             API key validation (Fastify preHandler)
    rate-limit.ts       Sliding window rate limiter
    logger.ts           Structured request logger
    error-handler.ts    Unified error → HTTP response mapping
  tracking/
    pricing.ts          Per-model input/output cost table
    counter.ts          Token counting + cost calculation
    db.ts               SQLite setup (better-sqlite3)
    accumulator.ts      Aggregate stats per key/model/day
  sdk/
    client.ts           LLMGatewayClient class
tests/
  unit/                 Mirror of src/ — one file per module
  integration/          Full API flow tests with MSW-mocked providers
```

## Non-Obvious Conventions

- **Unified model strings**: Internal model IDs match provider-canonical strings exactly (`gpt-4o`, `claude-3-5-sonnet-20241022`, `gemini-1.5-pro`). Never invent aliases — map from the unified string to provider's own ID inside the adapter.
- **No `any`**: TypeScript strict mode is enforced. If you're casting to `any`, add a `// TODO: type this` comment and a test.
- **Provider errors**: Never expose raw provider error messages to clients. Map to `{ error: { code, message } }` in `error-handler.ts`.
- **Streaming**: All streaming uses `AsyncIterable<ChatChunk>`. Adapters must yield chunks then `return` — do not throw inside the generator unless the stream failed.
- **Pricing table**: Costs are in USD per **1 token** (not per 1K). Multiply by token count, no division needed. This avoids floating point division errors.
- **SQLite**: Use synchronous `better-sqlite3` — do not mix with async DB drivers. All DB calls happen in the tracking module only.
- **Test isolation**: Each test creates its own provider mock scope (MSW). Never share mock state between tests.

## Environment Variables

| Key | Required | Description |
|---|---|---|
| `OPENAI_API_KEY` | for OpenAI | sk-... |
| `ANTHROPIC_API_KEY` | for Anthropic | sk-ant-... |
| `GEMINI_API_KEY` | for Gemini | AIza... |
| `GATEWAY_API_KEYS` | yes | Comma-separated list of valid gateway keys |
| `PORT` | no | Default 3000 |
| `RATE_LIMIT_RPM` | no | Requests per minute per key, default 60 |
| `DB_PATH` | no | SQLite file path, default `./data/usage.db` |
| `LOG_LEVEL` | no | `info` / `debug` / `error`, default `info` |

Startup fails fast if `GATEWAY_API_KEYS` is missing.

## Workflow

1. **Read TODO.md** — find the next unchecked task in the current phase
2. **Run tests first** — `npm test` should be green before you write anything
3. **Write a failing test** — red phase. Commit message: `test: failing test for <thing>`
4. **Implement** — green phase. Commit message: `feat: implement <thing>`
5. **Lint + typecheck** — `npm run lint && npm run typecheck`
6. **Manual spot-check** — curl the endpoint or inspect the output
7. **Update this file** if you learned something non-obvious
8. **Check the gate** in TODO.md before starting the next phase
