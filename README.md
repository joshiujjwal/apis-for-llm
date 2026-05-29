# apis-for-llm

> 🚧 **Early Development**

A unified API gateway and abstraction layer that sits in front of multiple LLM providers — OpenAI, Anthropic, Gemini, Cohere, and more. Apps swap models without code changes.

## What It Does

- **Single REST endpoint** — send one request format, route to any LLM provider
- **SDK clients** — typed TypeScript client for Node.js and browser
- **Model routing** — round-robin, cost-optimized, latency-optimized, or rule-based
- **Rate limiting** — per-key, per-model, and per-provider throttling
- **Cost tracking** — token counting + pricing per provider, aggregated dashboards
- **Streaming** — unified SSE stream across all providers
- **Fallback chains** — auto-retry on failure with next provider in chain

## Tech Stack

| Layer | Choice |
|---|---|
| Runtime | Node.js 20 LTS |
| Language | TypeScript 5.x |
| HTTP Framework | Fastify |
| Validation | Zod |
| Testing | Vitest |
| DB (cost tracking) | SQLite via better-sqlite3 |
| Rate limiting | in-memory + Redis (optional) |
| SDK codegen | Handlebars templates |

## Getting Started

```bash
# Clone
git clone git@github.com:joshiujjwal/apis-for-llm.git
cd apis-for-llm

# Install
npm install

# Configure providers
cp .env.example .env
# Edit .env with your API keys

# Dev server (hot reload)
npm run dev

# Run tests
npm test

# Build
npm run build
```

## Project Structure

```
src/
  providers/          # Adapters per LLM provider (OpenAI, Anthropic, Gemini, …)
  router/             # Request routing logic (strategy pattern)
  middleware/         # Auth, rate limiting, logging, error handling
  tracking/           # Cost & token tracking, aggregation
  sdk/                # Generated TypeScript SDK client
tests/
  unit/               # Per-module unit tests (mirrors src/)
  integration/        # End-to-end API flow tests with mocked providers
docs/
  spec.md             # Feature specification
  adr/                # Architecture Decision Records
.github/
  copilot-instructions.md
```

## Contributing

1. **Write tests first** — red before green, always
2. Run `npm test` before any commit; zero test regressions allowed
3. PRs must include evidence of manual testing (curl output or screenshot)
4. Keep PRs small and focused — one concern per PR
5. Update `CLAUDE.md` or `AGENTS.md` when you learn something non-obvious
