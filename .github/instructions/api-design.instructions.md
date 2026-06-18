---
description: "Use when designing or implementing REST/HTTP APIs. Covers status codes, error response shapes, versioning, auth, pagination, rate limiting, and idempotency."
applyTo: "**/{routes,handlers,api,controllers,endpoints}/**/*.{ts,js,py,go,rs,java,rb}"
---

# API Design Conventions

## HTTP Status Codes — Be Precise

Return the most specific status code available. Don't default to 200 or 500.

```text
2xx — Success
  200 OK            — Standard success (GET, PATCH)
  201 Created       — Resource created (POST). MUST include Location header
  202 Accepted      — Async processing started. Return status endpoint
  204 No Content    — Success, no body (DELETE)

3xx — Redirection
  301 Moved Permanently — Resource has a new permanent URL
  304 Not Modified  — Cached response still valid (ETag/If-None-Match)

4xx — Client Error
  400 Bad Request   — Malformed input (validation errors)
  401 Unauthorized  — Missing or invalid credentials
  403 Forbidden     — Authenticated but not authorized
  404 Not Found     — Resource doesn't exist
  409 Conflict      — Resource state conflict (duplicate, version mismatch)
  422 Unprocessable — Semantic validation failure (well-formed but wrong)
  429 Too Many Requests — Rate limit exceeded

5xx — Server Error
  500 Internal Error — Unexpected failure (bug). Never return by default
  502 Bad Gateway   — Upstream returned invalid response
  503 Unavailable   — Temporarily down (maintenance, overload)
  504 Gateway Timeout — Upstream didn't respond in time
```

- **Never return 500 by catching and swallowing.** 500 means "bug." Log the error and let it surface.
- **Never return 200 with an error message.** `{ "error": "not found" }` with 200 breaks every HTTP client, cache, and monitoring system.
- **401 vs 403**: 401 = "who are you?" (missing auth). 403 = "I know who you are, but no."

## Error Response Shape — Standardize

Every error response MUST use the same structure.

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Human-readable description of what went wrong",
    "details": [
      {
        "field": "email",
        "reason": "must be a valid email address",
        "value": "not-an-email"
      }
    ],
    "requestId": "req_a1b2c3d4"
  }
}
```

- **`code`**: machine-readable, stable, uppercase with underscores (`INSUFFICIENT_FUNDS`, not `insufficientFunds` or `error 12`)
- **`message`**: human-readable, safe to show in UI, never includes stack traces or internal paths
- **`details`**: array of field-level errors for validation failures. Empty or omitted for non-validation errors
- **`requestId`**: correlation ID for debugging — returned in response headers too
- **Never leak internals**: No SQL errors, stack traces, file paths, or framework names in error responses

## Versioning

APIs MUST be versioned. Pick one strategy and stick to it.

```text
# URL path versioning (most common, simplest to cache/rout)
GET /api/v1/users

# Header versioning (cleaner URLs, harder to test in browser)
GET /api/users
Accept: application/vnd.myapp.v2+json
```

- **URL path versioning** is the safe default. Easier to route, cache, document, and test.
- **Major version only**: `v1`, `v2`, not `v2.1.3`. Fine-grained changes use feature flags or backwards-compatible additions.
- **Never break a published version.** New field additions are fine on existing versions — but don't change field types, remove fields, or change semantics.
- Deprecation: set `Sunset` and `Deprecation` headers. Announce with at least one major version overlap.

## Authentication & Authorization

Every API endpoint must authenticate unless explicitly public.

```text
# Authentication: verify identity
Authorization: Bearer <token>

# Authorization: verify permissions
- Check permissions AFTER authentication
- Return 401 for missing/invalid credentials
- Return 403 for valid credentials but insufficient permissions
- Never differentiate 401 and 403 based on resource existence (prevents enumeration)
```

- Auth middleware runs before any handler logic
- Rate limiting runs before auth (prevent brute force)
- Auth tokens: short-lived access (15 min) + longer refresh (7 days). Rotate refresh tokens.
- API keys for service-to-service: use random 256-bit keys, not predictable patterns

## Pagination — Mandatory

Every list endpoint MUST paginate. Never return unbounded results.

```json
{
  "data": [...],
  "pagination": {
    "cursor": "eyJsYXN0X2lkIjogNDJ9",
    "hasMore": true,
    "total": 1543
  }
}
```

- **Cursor-based**: preferred for large datasets (consistent under inserts/deletes). Return an opaque cursor, client passes `?cursor=<cursor>`.
- **Offset-based**: simpler for small-to-medium datasets, UIs with numbered pages. Return `?offset=0&limit=20`.
- **Default limit**: 20-50 items. Max limit: 100. Reject requests above max with 400.
- **Always return `hasMore` or `total`** so clients know when to stop

## Rate Limiting

Every API needs rate limits. Document them.

```text
# Response headers
X-RateLimit-Limit: 1000
X-RateLimit-Remaining: 987
X-RateLimit-Reset: 1687132800
X-RateLimit-Used: 13
```

- **Per-user**: prevent one client from degrading service for others
- **Per-IP**: fallback for unauthenticated requests
- **Grace period**: return 429 + `Retry-After` header before hard-cutting off
- **Burst vs sustained**: allow short bursts above baseline rate, throttle sustained excess
- Authenticated endpoints: 429 returns a `Retry-After` seconds header

## Idempotency

`PUT` and `DELETE` must be idempotent. `POST` should use idempotency keys for payment/creation operations.

```text
# Client sends
POST /api/v1/charges
Idempotency-Key: 8f7b3c2a-9e4d-11ed-a1eb-0242ac120002

# Server behavior
- First request with key: process, store response, return 201
- Repeat with same key: return stored response (same status code and body), don't re-process
- Different body with same key: return 422
```

- Idempotency keys stored for at least 24 hours
- Keys are UUIDs generated by the client — never use timestamps or sequential IDs
- `GET` is always idempotent by HTTP spec. `PUT`/`DELETE` should be.

## Request/Response Conventions

- **JSON only**: `Content-Type: application/json`. Reject other content types with 415.
- **ISO 8601 timestamps**: `"2026-06-18T14:30:00Z"`. Always UTC. Never ambiguous formats.
- **Snake_case or camelCase**: pick one and enforce it everywhere. Don't mix.
- **No envelope for single resources**: `GET /users/42` → `{ "id": 42, "name": "..." }` not `{ "data": { "id": 42, ... } }`
- **Envelope for collections**: `GET /users` → `{ "data": [...], "pagination": {...} }`
- **Boolean fields**: `isActive`, `hasSubscription`. Never `active: 1` or `active: "Y"`.
- **Empty arrays over null**: `"items": []` not `"items": null` — avoids null-check bugs in every client

## HTTP Methods — Use Correctly

```text
GET     /users          — List users
GET     /users/42       — Get user 42
POST    /users          — Create a user
PUT     /users/42       — Replace user 42 (full replacement)
PATCH   /users/42       — Update user 42 (partial update)
DELETE  /users/42       — Delete user 42
```

- **Never use GET for mutations.** GET must be safe and idempotent. Caches, crawlers, and browsers assume it is.
- **POST for creation**, return 201 + Location header + created resource body
- **PUT for full replacement**, PATCH for partial updates. If you only support one, use PATCH.
- **No verbs in URLs**: `POST /users` not `POST /create-user`. The HTTP method is the verb.

## Documentation

Every API needs an OpenAPI spec (`openapi.yaml` or `openapi.json`). Not optional.

- Document every endpoint, every status code it can return, every field
- Examples for all request/response bodies
- Keep the spec in the repo, generate docs from it — spec is source of truth, not the docs page
