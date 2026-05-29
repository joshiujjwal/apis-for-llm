# Feature Specification — apis-for-llm

## Overview

**Problem:** Every LLM provider has a different API shape, authentication scheme, streaming protocol, and error format. Apps that want to use multiple providers (or swap models) must implement N adapters and manage N different SDKs.

**Solution:** A self-hosted API gateway that exposes a single OpenAI-compatible REST interface and routes requests to the configured provider/model behind the scenes. Apps talk to one URL with one API key format, and the gateway handles translation, routing, rate limiting, and cost tracking.

**Non-goals:** This is not a prompt management tool, a fine-tuning platform, or a hosted SaaS product (yet).

---

## Functional Requirements

### Provider Adapters
- [ ] Support OpenAI (GPT-4o, GPT-4-turbo, GPT-3.5-turbo, o1, o3)
- [ ] Support Anthropic (Claude 3.5 Sonnet, Claude 3 Haiku, Claude 3 Opus)
- [ ] Support Google Gemini (Gemini 1.5 Pro, Gemini 1.5 Flash, Gemini 2.0 Flash)
- [ ] Support Cohere (Command R+) — stretch goal
- [ ] Each adapter translates request/response to/from unified format
- [ ] Each adapter supports streaming via SSE

### Request Routing
- [ ] Route by explicit `model` field (e.g. `gpt-4o` → OpenAI, `claude-3-5-sonnet` → Anthropic)
- [ ] Configurable fallback chain (try provider A, then B on failure)
- [ ] Round-robin distribution across a pool of providers for the same capability
- [ ] Cost-optimized routing: select cheapest model that meets capability requirements

### Unified API Surface
- [ ] `POST /v1/chat/completions` — OpenAI-compatible request/response shape
- [ ] `GET /v1/models` — list all models available via configured providers
- [ ] `GET /v1/health` — liveness check + per-provider connectivity ping
- [ ] `GET /v1/usage` — token + cost aggregates for the authenticated key
- [ ] Streaming via `text/event-stream` with `data: <json>` chunks
- [ ] Final SSE chunk is `data: [DONE]`

### Authentication & Rate Limiting
- [ ] Gateway-level API keys (`Authorization: Bearer gw-<key>`)
- [ ] Per-key rate limit (requests/minute, configurable)
- [ ] Per-provider rate limit passthrough (propagate 429 from upstream)
- [ ] Key management via environment variables (v0.1) or SQLite table (v0.2)

### Cost & Token Tracking
- [ ] Count tokens per request (using provider's token counter or tiktoken)
- [ ] Store input tokens, output tokens, model, provider, key, timestamp
- [ ] Maintain per-model pricing table (USD per 1K tokens input/output)
- [ ] Expose aggregated stats per key: total requests, total tokens, total cost

## Non-Functional Requirements

- [ ] P99 latency overhead < 20ms (gateway processing only, excluding provider RTT)
- [ ] Handles 100 concurrent requests without queuing
- [ ] Zero provider API keys stored in source code or logs
- [ ] All errors return structured JSON `{ error: { code, message } }`
- [ ] Works offline for tests (all provider calls mockable)
- [ ] Single binary deployment via Docker

---

## Data Model

### `ChatRequest` (unified input)

```typescript
{
  model: string;                    // e.g. "gpt-4o", "claude-3-5-sonnet-20241022"
  messages: Message[];              // { role: "user"|"assistant"|"system", content: string }
  temperature?: number;             // 0–2, default 1
  max_tokens?: number;
  stream?: boolean;                 // default false
  top_p?: number;
  stop?: string | string[];
  metadata?: Record<string, string>; // passed through to tracking
}
```

### `ChatResponse` (unified output)

```typescript
{
  id: string;
  object: "chat.completion";
  created: number;                  // unix timestamp
  model: string;                    // actual model used (may differ from requested)
  provider: string;                 // "openai" | "anthropic" | "gemini"
  choices: [{
    index: 0;
    message: { role: "assistant"; content: string };
    finish_reason: "stop" | "length" | "content_filter";
  }];
  usage: {
    prompt_tokens: number;
    completion_tokens: number;
    total_tokens: number;
    cost_usd: number;
  };
}
```

### `requests` table (SQLite)

| column | type | notes |
|---|---|---|
| id | TEXT PK | UUID |
| api_key_hash | TEXT | SHA-256 of key |
| provider | TEXT | "openai" / "anthropic" / "gemini" |
| model | TEXT | exact model string |
| tokens_in | INTEGER | prompt tokens |
| tokens_out | INTEGER | completion tokens |
| cost_usd | REAL | calculated from pricing table |
| latency_ms | INTEGER | total gateway latency |
| status | INTEGER | HTTP status returned to client |
| created_at | TEXT | ISO 8601 |

---

## API Interface

### `POST /v1/chat/completions`

**Request headers:**
```
Authorization: Bearer gw-<key>
Content-Type: application/json
```

**Request body:** `ChatRequest` (see above)

**Response (non-streaming):** `ChatResponse` JSON

**Response (streaming):** `Content-Type: text/event-stream`
```
data: {"id":"...","choices":[{"delta":{"content":"Hello"}}]}

data: {"id":"...","choices":[{"delta":{"content":" world"}}]}

data: [DONE]
```

**Errors:**
| Code | Meaning |
|---|---|
| 400 | Invalid request schema |
| 401 | Missing or invalid gateway API key |
| 429 | Rate limit exceeded |
| 502 | Provider returned an error |
| 503 | No healthy providers available |

---

## Test Plan

### Unit Tests
- [ ] Each provider adapter: request normalization, response denormalization
- [ ] Each routing strategy with mock provider pool
- [ ] Pricing table: spot-check known models (gpt-4o: $5/$15 per 1M)
- [ ] Cost accumulator: correct aggregation across multiple requests
- [ ] Rate limiter: allows up to limit, blocks at limit+1, resets after window

### Integration Tests
- [ ] Full request flow: client → gateway → mock provider → client
- [ ] Streaming: SSE chunks arrive in order, stream terminates with `[DONE]`
- [ ] Fallback chain: first provider 500s, second provider responds correctly
- [ ] Auth: missing key returns 401, valid key returns 200
- [ ] `/v1/usage` returns correct totals after N requests

### Edge Cases
- [ ] Empty message array → 400
- [ ] Unsupported model string → 400 with `available_models` hint
- [ ] Provider returns truncated response (`finish_reason: "length"`)
- [ ] Provider rate-limits gateway (`429` from upstream) → propagate correctly
- [ ] Very long prompt (100K tokens) — doesn't crash, returns appropriate error if over model limit

---

## Open Questions

1. **Key management v0.2**: SQLite-backed key table vs. static env vars — when to add?
2. **Semantic routing**: Route by task type ("summarize" → cheapest model, "code" → best model) — worth adding to Phase 3?
3. **Caching**: Should identical prompts be cached to reduce cost? What's the TTL strategy?
4. **Multi-modal**: How early should image/audio input be part of the `ChatRequest` schema?
5. **Provider-specific params**: Some models have unique params (e.g. Anthropic `thinking`, OpenAI `response_format`). Pass-through or normalize?
