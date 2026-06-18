# AGENTS.md — Mandatory Pre-Read

Before implementing, writing, editing, or deleting any code in this repository, you MUST:

1. Read this file for project conventions, architecture, common patterns, and documentation rules
2. Follow all rules and conventions documented here
3. After making changes, re-read relevant sections and update anything that is now wrong

**Do not skip this step.** AGENTS.md is the source of truth for how this codebase works. Treat it as a mandatory pre-read before any code work.

## Customizing for Your Project

This file contains **generic core rules** that apply to all projects regardless of language or framework. Framework-specific conventions (build commands, directory layouts, language idioms) live in `.github/instructions/{framework}.instructions.md` and are loaded automatically when editing matching files.

To customize:
- **Add a framework rule**: Create `.github/instructions/{name}.instructions.md` with an `applyTo` frontmatter field
- **Add a project-specific rule**: Use `.github/instructions/{name}.instructions.md` scoped to your source directories
- **Add a task template**: Create `.github/prompts/{name}.prompt.md` (invocable via `/` in VS Code chat)
- **Enforce deterministically**: Add `.github/hooks/{name}.json` for lifecycle-based guardrails

See `USAGE.md` for the full decision tree and step-by-step guides.


# Code Comments — Mandatory

Every function, class, non-obvious block, and exported symbol MUST have a human-readable comment explaining **why** it exists and **what** it does. Follow these rules:

- Write comments as if explaining to a new teammate — plain English, no jargon shortcuts.
- Focus on intent and edge cases, not restating the code.
- Keep comments up to date when logic changes. Stale comments are worse than no comments.
- JSDoc / TSDoc for public APIs; inline `//` for internal logic that isn't self-evident.
- Python: Google-style docstrings. Go: doc comments on exported symbols. Rust: `///` doc comments with examples where helpful.


# Docs — Always Keep Them In Sync

Every source change must update the corresponding docs in `docs/` at the repo root. Docs must be human-readable and structured so other LLMs can consume them.

1. Check which docs in `docs/` at the repo root cover the behavior you changed
2. Re-read those docs
3. Update anything that's now wrong. If no doc covers it, create one.

**Do not defer.** Apply doc updates in the same turn as the code change. Treat docs as part of the feature.


# Test Before You're Done

Never claim a change is complete until you have verified it. Before wrapping up:

1. **Lint & type-check** — run the project's linter and type checker. Fix every error.
2. **Build check** — run the project's build command. Ensure it succeeds with no warnings you introduced.
3. **Manual smoke test** — if the change touches UI or API behavior, run the dev server and hit the affected path at least once.
4. **Existing tests** — run the project's test suite. If existing tests break, fix them before declaring done.
5. **Add tests** — if you added new logic, add at least one test that would fail without your change.

If any step fails, fix it and re-run. Only say "done" when everything is green.


# Reusable Code — Mandatory

Don't repeat yourself. Every piece of logic must live in exactly one place.

- **Extract, don't duplicate.** If the same logic appears in two places, pull it into a shared utility, hook, or helper.
- **Shared components belong in a designated shared directory.** Don't inline reusable UI or logic in page-level or module-level code.
- **Shared utilities go in a designated utilities directory.** Date formatting, string helpers, API wrappers — anything used across multiple files lives here.
- **Prefer composition over inheritance.** Keep abstractions flat and composable.
- **Reference existing patterns.** Before creating a new utility, check if an equivalent already exists in the codebase.


# Secure Coding — Mandatory

Security is not optional. Every change must consider its security implications.

- **No secrets in code.** API keys, tokens, passwords, connection strings — never hardcoded. Use environment variables, a secrets manager, or the platform's secret store. If a secret appears in a code review, reject it.
- **Validate all input at trust boundaries.** API handlers, CLI arguments, form fields, file uploads — sanitize and validate before processing. Use framework-provided validation; don't roll your own.
- **Principle of least privilege.** Code should run with the minimum permissions needed. Database connections should use read-only when writes aren't required. Avoid `sudo`, `root`, or admin roles unless strictly necessary.
- **Never eval or exec untrusted input.** No `eval()`, `exec()`, `runtime.Compile()`, or equivalent on user-supplied data. If you think you need dynamic code execution, find another approach.
- **Keep dependencies audited.** Run the project's dependency scanner regularly (`npm audit`, `pip audit`, `cargo audit`, `govulncheck`). Don't introduce dependencies with known vulnerabilities.
- **Escape output at render time.** HTML, SQL, shell commands — use parameterized queries, template engines with auto-escaping, and `shlex.quote()`. Never concatenate user input into a command string or query.
- **Rate-limit and timeout external calls.** Any network request needs a timeout. Authentication endpoints need rate limiting. Assume every endpoint will be probed by malicious actors.


# Project Structure Best Practices

- **Group by feature, not by type (for mid-to-large projects).** `feature/auth/`, `feature/payments/` — not `controllers/auth.js` + `models/payment.js` spread across folders. For small projects, a flat structure is fine.
- **Co-locate tests with source code.** Tests live next to the code they test (`foo.test.ts` beside `foo.ts`) or in a parallel `tests/` directory at the project root. Never in a distant, disconnected location.
- **One concern per file.** A file should do one thing well. If a file exceeds ~300 lines, consider splitting it.
- **Avoid circular dependencies.** If module A imports B and B imports A, restructure. Extract shared logic into a third module C that both depend on.
- **`index` files are for re-exporting, not logic.** `index.ts` / `__init__.py` / `mod.rs` should export the public API — they should not contain business logic.
- **Configuration lives in one place.** Environment variables, config files, CLI flags — consolidate into a single config module. No `process.env` / `os.environ` calls scattered across the codebase.


# Git & Version Control — Mandatory

Commits are documentation. Every commit tells a story.

- **Atomic commits.** One logical change per commit. Don't bundle unrelated fixes, refactors, and features in the same commit.
- **Meaningful messages.** Conventional Commits format: `type(scope): description`. Types: `feat`, `fix`, `refactor`, `test`, `docs`, `chore`. Scope is optional but encouraged (`feat(auth): add OAuth2 login`).
- **Never commit generated files.** Build artifacts, compiled binaries, minified bundles, lock files from non-primary package managers — all in `.gitignore`.
- **Never commit secrets.** See Secure Coding above. If you accidentally commit a secret, rotate it immediately — removing from git history is not enough.
- **Branch naming:** `type/description` — `feat/user-auth`, `fix/login-timeout`, `refactor/db-layer`. No personal branches (`johns-branch`).
- **Rebase, don't merge** (when agreed by team). Clean linear history is easier to bisect and revert.
- **Pull request size:** Under 400 lines changed. Large PRs are harder to review and more likely to introduce bugs. Split big features into stacked PRs.


# Observability — Mandatory

If you can't observe it, you can't fix it. Every service must be debuggable in production.

- **Structured logging.** Use JSON-formatted logs. Every log line must include: timestamp, level, message, trace ID. Use the project's logging framework — never `console.log`, `print`, or `println` in production code.
- **Log levels appropriately.** `DEBUG`: developer details, `INFO`: key events (startup, shutdown, request received), `WARN`: recoverable problems (retry succeeded, fallback used), `ERROR`: needs human attention (retry exhausted, data loss).
- **Metrics over logs for patterns.** Don't log every request — emit a counter metric. Logs are for specific events; metrics are for trends.
- **Traces for distributed calls.** Every outgoing HTTP/RPC/database call must propagate a trace context. Use OpenTelemetry or the platform's tracing SDK.
- **Health check endpoint.** Every service needs a `/health` or equivalent that returns 200 when the service is operational. The health check must verify actual dependencies (database, queue) — not just "the process is running".
- **Graceful degradation.** If a dependency is down, return degraded responses or cached data — don't crash. Log the failure, emit a metric, keep serving what you can.


# Performance — Mandatory

Write correct code first, then make it fast. But never ship something you know is slow.

- **Measure before optimizing.** Profile first (`pprof`, `flamegraph`, Chrome DevTools, `perf`). Don't guess where the bottleneck is — you'll be wrong.
- **Don't prematurely optimize.** Readable, correct code that's 5% slower beats unreadable, buggy code that's fast. Optimize when you have evidence of a problem.
- **N+1 queries are a bug, not an optimization.** Use eager loading, batch queries, or data loaders. One query that returns 100 rows beats 100 queries that return 1 row.
- **Cache with intent.** Every cache entry needs a TTL, an invalidation strategy, and a fallback. Cache misses must be survivable.
- **Paginate all list endpoints.** Never return unbounded results. Default to 20-50 items per page. Return total count or a cursor for the next page.
- **Timeouts on every external call.** No call should block indefinitely. Set reasonable timeouts (HTTP: 30s, DB query: 10s, cache: 1s) and handle timeout errors gracefully.
- **Resource cleanup.** Close files, release connections, cancel timers. Use `defer`, `finally`, `context managers`, or `Drop` — whatever the language provides.


# Error Handling — Mandatory

How you handle errors defines your system's reliability. Never swallow them silently.

- **Never ignore an error.** Every error return value, exception, or rejected promise must be either handled or explicitly propagated. A `catch` block that does nothing is a bug.
- **Wrap with context.** When propagating an error, add what you were trying to do. "connection refused" is useless. "connecting to payment service at checkout: connection refused" is actionable.
- **Handle errors at a single level.** Either log it or return it — never both. Logging + rethrowing produces duplicate noise.
- **Use typed/categorized errors, not string matching.** Clients should branch on error codes or types, not on `error.message.includes("timeout")`.
- **Distinguish recoverable from non-recoverable.** Recoverable errors (timeout, rate limit, temporary failure): retry with backoff. Non-recoverable (validation, auth, not found): fail fast.
- **Never expose internals in error messages to users.** No stack traces, SQL errors, file paths, or framework names in API responses. Log those internally; return a clean error to the caller.
- **Resource cleanup on error paths.** Use `defer`, `finally`, context managers, or RAII — whatever the language provides — to release resources even when errors occur.
- **Crash-only for truly unrecoverable states.** If an invariant is violated (corrupt data, impossible state), crash loudly and let the orchestrator restart. Don't limp along in an unknown state.


# Configuration — Mandatory

Configuration must be explicit, centralized, and environment-aware.

- **One config module.** All environment variables, config files, CLI flags, and feature flags are read in exactly one place. No `process.env` / `os.environ` / `std::env::var` calls scattered across the codebase.
- **Validate at startup.** Validate all configuration values when the application starts — fail fast if required values are missing, have wrong types, or are out of range. Never discover a missing config value at runtime deep in a request handler.
- **Environment-specific config.** Use environment variables for deployment-specific values (URLs, credentials, feature flags). Use config files for structural settings (schema, timeouts, limits). Never mix them.
- **12-factor app config.** Store config in the environment, not in code. `ENVIRONMENT=production`, `DATABASE_URL=postgres://...`, `LOG_LEVEL=info`. No `if (env === "production")` hardcoded conditionals — use config values.
- **Secrets are not config.** API keys, passwords, tokens live in a secrets manager or environment variable injected by the platform. They are never in config files, committed to git, or defaulted in code.
- **Feature flags.** Use a feature flag system for gradual rollouts and operational toggles. Every flag needs an owner, an expiry date, and a plan for removal.
- **Sensible defaults.** Every config value has a safe default. A new developer should be able to run the project with zero configuration. Zero-config means zero-friction.
- **Document every config value.** What it does, valid values, default, and what happens when it's missing. The config module is its own documentation.


# Naming Conventions

Names are documentation. Choose them carefully and consistently.

- **Be descriptive.** `getUserById` not `get`. `calculateTax` not `doStuff`. The name should tell you what it does without reading the implementation.
- **No abbreviations.** `config` not `cfg`. `response` not `resp`. `number` not `num`. Exceptions: well-known acronyms (URL, HTML, JSON, ID) and language-specific conventions (Go allows short names in limited scope).
- **Consistent casing per language:**
  - TypeScript/JavaScript: PascalCase for components/types, camelCase for functions/variables, UPPER_CASE for constants, kebab-case for files.
  - Python: PascalCase for classes, snake_case for functions/variables, UPPER_CASE for constants.
  - Go: PascalCase for exported symbols, camelCase for unexported, lowercase single-word package names.
  - Rust: PascalCase for types, snake_case for functions/variables, SCREAMING_SNAKE_CASE for consts/statics.
- **Boolean variables read as a question.** `isLoading`, `hasError`, `canSubmit` — not `loading`, `error`, `submit`.
- **Avoid Hungarian notation and type prefixes.** No `strName`, `iCount`, `bEnabled`. The type system handles types; names should describe purpose.

