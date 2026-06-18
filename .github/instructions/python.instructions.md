---
description: "Use when working with Python files. Covers type hints, docstrings, secure coding, testing & QA, naming conventions, formatting, and project structure conventions."
applyTo: "**/*.py"
---

# Python Conventions

## Build & Test Commands

- **Test**: `pytest` (with `-v` for verbose, `-x` to stop on first failure)
- **Lint & format**: `ruff check . && ruff format --check .`
- **Type check**: `mypy .` (strict mode where practical)
- **Run**: `python -m {module}` or project-configured entry point

## Type Hints — Required

Every function signature and public method MUST have type annotations. Use `mypy` to validate.

```python
# Good
def calculate_total(items: list[Item], tax_rate: float = 0.08) -> float:
    ...

# Bad — no type hints
def calculate_total(items, tax_rate=0.08):
    ...
```

- Use `|` for unions (Python 3.10+): `str | None` instead of `Optional[str]`
- Use built-in generics: `list[User]` instead of `List[User]`
- Use `typing.Protocol` for structural subtyping
- Use `typing.TypedDict` for typed dictionaries

## Docstrings — Mandatory

Use **Google-style** docstrings for all public functions, classes, and methods.

```python
def connect(host: str, port: int, timeout: float = 30.0) -> Connection:
    """Open a connection to the target host.

    Args:
        host: The hostname or IP address to connect to.
        port: The TCP port number.
        timeout: Connection timeout in seconds. Defaults to 30.

    Returns:
        An authenticated Connection object.

    Raises:
        ConnectionError: If the host is unreachable or authentication fails.
        ValueError: If port is not in range 1-65535.
    """
```

- Describe **why**, not just **what**
- Always document raised exceptions
- Keep up to date — stale docstrings are worse than none

## Project Structure

```
project/
├── src/                    # Source code
│   ├── __init__.py
│   ├── main.py             # Entry point
│   ├── models/             # Data models
│   ├── services/           # Business logic
│   └── utils/              # Shared utilities
├── tests/                  # Test suite
│   ├── __init__.py
│   ├── conftest.py         # Shared fixtures
│   └── test_*.py           # Test modules
├── pyproject.toml          # Project config & dependencies
└── README.md
```

- Use `src/` layout (not flat) for proper import isolation
- Every package needs `__init__.py`
- Use `pyproject.toml` for all configuration (setuptools, ruff, mypy, pytest)

## Testing

- Use `pytest` with descriptive test names: `test_when_input_is_empty_returns_default`
- Prefer fixtures over setup/teardown methods
- Use `pytest.mark.parametrize` for table-driven tests
- Mock external services; don't hit real APIs in unit tests
- Aim for one assertion per test where practical

## Code Quality

- Run `ruff` for linting and formatting — it replaces `black`, `isort`, `flake8`, and more
- No commented-out code in commits
- Use `pathlib` instead of `os.path`
- Use f-strings, not `.format()` or `%` formatting
- Prefer `dataclasses` or `Pydantic` models over plain dicts

## Imports

- Standard library first, then third-party, then local — each group separated by a blank line
- No wildcard imports (`from module import *`)
- No circular imports — restructure if you encounter them

## Secure Coding

- **No secrets in code.** Use `os.environ.get()` or a secrets manager. Never commit `.env` files — add them to `.gitignore`. Use `python-dotenv` for local development only.
- **Parameterized queries only.** Never use f-strings or `%` formatting in SQL. Use SQLAlchemy parameterized queries, or for raw SQL use `cursor.execute("SELECT * FROM users WHERE id = %s", (user_id,))`.
- **No `shell=True` in subprocess.** Use `subprocess.run()` with a list of arguments, not a single string with `shell=True`. If you must shell out, use `shlex.quote()` on every argument.
- **Validate and sanitize all input.** Use Pydantic models for API input validation. Reject unknown fields. Set `extra = "forbid"` on Pydantic models exposed to external input.
- **Dependency hygiene.** Run `pip-audit` regularly. Pin dependencies with hashes in `requirements.txt` or use a lockfile (`poetry.lock`, `Pipfile.lock`). Don't install packages from URLs without verifying checksums.
- **Use `secrets` module for cryptography.** Never use `random` for tokens, passwords, or session IDs. Use `secrets.token_hex()` or `secrets.token_urlsafe()`.
- **Set secure defaults.** `DEBUG=False` in production. HTTPS enforced. Secure cookie flags (`HttpOnly`, `SameSite=Lax`, `Secure`). Use `bcrypt` or `argon2` for password hashing — never `hashlib` directly.

## Testing & QA

- **Coverage thresholds:** Set minimum coverage in `pyproject.toml` (`[tool.coverage.report] fail_under = 80`). Don't just aim for coverage — test behavior, not implementation.
- **Property-based testing:** Use `hypothesis` for functions with complex input spaces (parsers, validators, serializers). It finds edge cases you won't think of.
- **Test isolation:** Every test must be independent. No shared mutable state between tests. Use `conftest.py` fixtures with `scope="function"` (the default).
- **CI pipeline:** `ruff check` → `mypy` → `pytest` → `pip-audit` — in that order. Fail fast on lint/type errors before running tests.
- **Snapshot testing:** Use `syrupy` or `pytest-snapshot` for API response shapes and complex data structures that should be stable.
- **Mutation testing (optional, for critical paths):** Use `mutmut` or `cosmic-ray` on security-sensitive and financial code to verify test quality.

## Naming Conventions

- **Files:** snake_case (`user_service.py`, `test_auth.py`)
- **Classes:** PascalCase (`UserService`, `ApiClient`)
- **Functions/variables:** snake_case (`get_user_by_id`, `max_retries`)
- **Constants:** UPPER_CASE (`MAX_CONNECTIONS`, `DEFAULT_TIMEOUT`)
- **Private members:** `_` prefix (`_internal_method`, `_cache`)
- **"Magic" attributes:** `__` prefix for Python dunder methods only (`__init__`, `__str__`); avoid double-underscore name mangling
- **Boolean variables:** `is_` or `has_` prefix (`is_active`, `has_permission`)
- **Packages/modules:** short, lowercase, no underscores unless necessary for clarity
