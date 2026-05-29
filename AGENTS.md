# AGENTS.md — apis-for-llm

## Setup

```bash
# Install dependencies
npm install

# Copy env template and fill in keys
cp .env.example .env

# Verify setup
npm run typecheck && npm test
```

## Code Style (TypeScript / Node.js)

- **TypeScript strict mode** — no `any`, no `!` non-null assertions without a comment
- **Functional-first** — prefer pure functions; keep side effects in the edges (adapters, DB, HTTP handlers)
- **Named exports only** — no default exports (easier to refactor, better tree-shaking)
- **Zod for all I/O boundaries** — every external input (HTTP request, provider response, env vars) must be validated with Zod before use
- **File naming** — `kebab-case.ts` for all source files
- **No barrel files** — import from the specific module, not from `index.ts` re-exports
- **Error types** — use typed error classes (`class ProviderError extends Error`) not string throws
- **Async** — always `async/await`; no raw `.then()` chains except in SDK client for browser compat

## Testing

- **Framework**: Vitest
- **Red/Green TDD**: Write a failing test first. Do not write implementation without a red test.
- **Provider mocking**: Use MSW (`msw/node`) to mock all HTTP calls to LLM providers. Never make real network calls in unit tests.
- **Test file location**: `tests/unit/<module-name>.test.ts` mirrors `src/<module-name>.ts`
- **Coverage**: Run `npm run test -- --coverage`. New modules must not reduce overall coverage.
- **Integration tests**: Gated by `INTEGRATION=true` env var. These call real provider APIs. Do not run in CI unless you've confirmed keys are available.
- **Assertions**: Use `expect(value).toEqual(expected)` for objects, `toMatchObject` for partial shape checks

```bash
npm test                        # run all unit tests
npm run test:watch              # watch mode
INTEGRATION=true npm run test:int  # real provider calls
npm run test -- --coverage      # coverage report
```

## PR Instructions

- **One concern per PR** — routing fix and new adapter are two PRs, not one
- **Evidence required**: PR description must include one of:
  - curl output showing the endpoint works
  - test output snippet showing new tests pass
  - before/after comparison for bug fixes
- **Review AI descriptions**: When Copilot/Claude writes a PR description, read it carefully and correct anything wrong before merging
- **Zero regressions**: `npm test` must be green. If a pre-existing test breaks, fix it in the same PR with an explanation.
- **Commit format**: `<type>: <short description>` where type is `feat`, `fix`, `test`, `chore`, `docs`, `refactor`

## Key Architectural Rules

1. **Adapters are thin** — they translate format only; no business logic
2. **Router owns strategy** — routing decisions live in `src/router/`, not in adapters or middleware
3. **Tracking is a side effect** — never block the request path on DB writes; use `setImmediate` or fire-and-forget
4. **Middleware is composable** — each middleware does one thing; compose in `src/index.ts`
5. **SDK mirrors REST** — SDK method names map 1:1 to REST endpoints; no extra abstraction
