---
description: "Write high-quality documentation: READMEs, API docs, ADRs, and architecture decision records. Use after building features or when docs are stale."
argument-hint: "What to document (readme, api, adr, architecture)"
agent: "Explore"
---

# Write Project Documentation

You are writing documentation for this project. Good docs are the difference between a project that's used and one that's abandoned. They answer "what is this?", "how do I use it?", and "why was it built this way?"

## Before You Write

1. Read the project's existing docs to understand tone and structure
2. Read `docs/ARCHITECTURE.md` for structural context
3. Run the project (if applicable) — don't document what you haven't seen work
4. Check `.github/instructions/` for framework-specific conventions that should be referenced

## README.md

A good README answers these questions in order:

```markdown
# Project Name
One-line description of what this does and who it's for.

## Quick Start
The fastest path to a working setup. Goal: under 5 minutes.

## Usage
Common workflows with copy-pasteable examples.

## Configuration
Environment variables, config files, feature flags.

## Development
How to set up a dev environment, run tests, contribute.

## Architecture
High-level overview — link to docs/ARCHITECTURE.md for details.

## License
```
```

- **Write for someone who just found your repo**: they have 30 seconds to decide if it's relevant
- **Copy-pasteable examples**: every code block should be runnable as-is (or with minimal substitution)
- **No badges in the first screenful**: they push actual content below the fold
- **Keep it current**: outdated Quick Start is worse than no Quick Start

## API Documentation

For every endpoint:

```markdown
### GET /api/v1/users/:id

Returns a single user by ID.

**Path Parameters**
| Name | Type | Description |
|------|------|-------------|
| id   | UUID | User ID |

**Query Parameters**
| Name | Type | Default | Description |
|------|------|---------|-------------|
| include | string | — | Comma-separated relations to include (posts, settings) |

**Response (200)**
```json
{
  "id": "uuid",
  "email": "string",
  "name": "string",
  "createdAt": "ISO 8601 datetime"
}
```

**Errors**
| Code | Description |
|------|-------------|
| 404  | User not found |
| 401  | Missing authentication |
```

- **Document every status code the endpoint can return**
- **Show example request AND response** — curl command + JSON body
- **Note auth requirements, rate limits, and idempotency support**

## Architecture Decision Records (ADRs)

For significant technical decisions, create `docs/adr/NNNN-title.md`:

```markdown
# ADR-0001: Use PostgreSQL as Primary Database

**Status**: Accepted (2026-06-18)

**Context**: We need a relational database for the core data model...

**Decision**: Use PostgreSQL 16.

**Alternatives Considered**:
- MySQL 8: rejected due to weaker JSON support and less mature full-text search
- SQLite: rejected due to lack of concurrent write support at scale

**Consequences**:
- Positive: Strong type system, excellent JSON support, mature ecosystem
- Negative: Operational complexity higher than SQLite; team needs Postgres expertise
```

- **ADRs explain "why," not "what"** — the code already shows what
- **Every ADR lists rejected alternatives** with reasons
- **Status**: Proposed → Accepted → Deprecated → Superseded (with reference to the new ADR)
- **File naming**: `adr/NNNN-lowercase-title-with-hyphens.md`. Sequential numbering.

## Style Guide

- **Active voice**: "The handler returns a 404" not "A 404 is returned by the handler"
- **Present tense**: "Validates the input" not "Will validate the input"
- **You/your**: address the reader directly. "Set your API key" not "The API key should be set"
- **Oxford comma**: "installs dependencies, sets up the database, and starts the server"
- **Code spans for symbols**: `UserService`, `GET /api/users`, `config.timeout`
- **Code blocks for commands**: triple-backtick with language tag
- **One sentence per line** in markdown source: makes diffs readable
```

## Keep Docs Current

The most harmful docs are stale docs — they actively mislead. After writing documentation:
- Add a "Last updated" date if not auto-generated
- Link to the source of truth (OpenAPI spec → generated docs, not hand-written API docs)
- If a doc is outdated and you can't fix it now, add `<!-- OUTDATED: reason, 2026-06-18 -->` at the top
