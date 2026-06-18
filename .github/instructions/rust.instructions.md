---
description: "Use when working with Rust files. Covers formatting, clippy lints, testing, secure coding, naming conventions, project layout, error handling, and ownership patterns."
applyTo: "**/*.rs"
---

# Rust Conventions

## Build & Test Commands

- **Build**: `cargo build`
- **Release build**: `cargo build --release`
- **Test**: `cargo test`
- **Test with output**: `cargo test -- --nocapture`
- **Lint**: `cargo clippy -- -D warnings` (treat warnings as errors)
- **Format**: `cargo fmt -- --check` (CI) or `cargo fmt` (apply)
- **Docs**: `cargo doc --open`

## Documentation — Mandatory

Every public item MUST have a `///` doc comment with examples where helpful.

```rust
/// Creates a new connection pool for the given database URL.
///
/// The pool starts with `min` connections and grows up to `max` under load.
/// Connections are validated with a ping before being handed out.
///
/// # Examples
/// ```
/// let pool = connect("postgres://localhost/db", 5, 20)?;
/// ```
///
/// # Errors
/// Returns `ConnectError` if the database is unreachable.
pub fn connect(url: &str, min: u32, max: u32) -> Result<Pool, ConnectError> {
```

- Module-level docs with `//!` at the top of `lib.rs` or `mod.rs`
- Examples in doc comments are compiled and tested — keep them working
- `#[must_use]` on functions where ignoring the return value is a bug

## Error Handling

- Use `Result<T, E>` for recoverable errors — never `unwrap()` in library code
- Use `thiserror` for library error types, `anyhow` for application code
- Implement `std::fmt::Display` and `std::error::Error` for custom errors
- Use the `?` operator for propagation
- Context on errors:

```rust
use anyhow::Context;
let config = read_file("config.toml")
    .with_context(|| "failed to read config")?;
```

- Reserve `panic!` for unrecoverable states (invariants, not expected failures)

## Project Layout

```
project/
├── Cargo.toml
├── Cargo.lock
├── src/
│   ├── main.rs             # Binary entry point
│   ├── lib.rs              # Library root (if library)
│   ├── models/
│   ├── services/
│   └── utils/
├── tests/                  # Integration tests
│   └── integration_test.rs
├── benches/                # Benchmarks
├── examples/               # Example binaries
└── rustfmt.toml            # Formatting config
```

- Use workspaces for multi-crate projects: `[workspace]` in root `Cargo.toml`
- Integration tests go in `tests/`, not `src/`
- Use `rustfmt.toml` for team formatting standards

## Ownership & Borrowing

- Prefer references (`&T`, `&mut T`) over cloning unless ownership is required
- Use `Cow<'_, T>` for copy-on-write patterns
- Derive `Clone` only when cloning is semantically correct, not just convenient
- Use `Arc<Mutex<T>>` for shared mutable state across threads; consider `tokio::sync` for async contexts
- Prefer `&str` over `&String` in function parameters

## Testing

- Unit tests go inline in the same file, in a `#[cfg(test)] mod tests { ... }` block
- Integration tests go in `tests/` directory
- Use descriptive test names: `test_when_queue_is_full_returns_error`
- Prefer `assert_eq!` over `assert!` with `==` for better error messages
- Use `rstest` crate for parameterized/fixture-based tests when needed

## Clippy & Formatting

- All code MUST pass `cargo clippy -- -D warnings` — no exceptions
- All code MUST be formatted with `cargo fmt` (use `rustfmt.toml` for custom rules)
- Enable additional clippy lints in `Cargo.toml`:

```toml
[lints.clippy]
pedantic = "warn"
unwrap_used = "warn"
expect_used = "warn"
```

## General Practices

- Prefer `enum` over `bool` for function parameters — `set_mode(Mode::ReadOnly)` not `set_readonly(true)`
- Use the type system to make invalid states unrepresentable
- Derive common traits: `Debug`, `Clone`, `PartialEq`, `Eq`, `Hash`, `Serialize`, `Deserialize`
- Use `tracing` crate for structured logging (not `println!` or `log` crate directly)
- Async: prefer `tokio` runtime; use `async_trait` for async trait methods

## Secure Coding

- **No secrets in code.** Use `std::env::var()` or a secrets manager. Use `dotenvy` for local development `.env` files — never commit `.env`.
- **Justify every `unsafe` block.** Every `unsafe` must have a `// SAFETY:` comment explaining why the invariants are upheld. If you can't write that comment, don't use `unsafe`. Unsafe blocks should be as small as possible.
- **No `unwrap()` or `expect()` in library code.** Libraries should propagate errors via `Result`. Application binaries may use `expect()` for unrecoverable startup failures only.
- **Input validation.** Validate all external input — HTTP bodies, CLI args, file contents — before processing. Use `validator` crate for derive-based validation on structs.
- **Dependency auditing.** Run `cargo audit` in CI. Pin dependencies with `Cargo.lock` (committed for binaries, in `.gitignore` for libraries). Review `cargo-deny` for license compliance and duplicate dependencies.
- **Index bounds checking.** Use `.get()` instead of `[]` when the index is not guaranteed valid. Prefer iterators over manual indexing — they're both safer and faster.
- **Sensitive data zeroing.** Use `zeroize` crate to clear secrets from memory after use. Use `secrecy` crate for wrapping secrets (it implements `Debug` without leaking the value).

## Testing & QA

- **Doctests run in CI.** Every public function with a doc example compiles and runs as a test. Keep examples working — they are the first line of documentation.
- **Fuzz testing:** Add `cargo-fuzz` targets for parsers, decoders, and any function that accepts raw bytes or strings. Run fuzzers periodically in CI.
- **Benchmarks:** Use `criterion` crate for benchmarking. Run benchmarks before and after performance changes. Document benchmark results in commit messages for significant changes.
- **Coverage:** Use `cargo-tarpaulin` or `cargo-llvm-cov` for coverage reports. Enforce a minimum threshold (suggest 70%) in CI.
- **Miri (undefined behavior detection):** Run `cargo miri test` on `unsafe` code paths to catch undefined behavior and subtle bugs.
- **Test organization:** Unit tests in `#[cfg(test)] mod tests` blocks within the source file. Integration tests in `tests/`. Use `common/mod.rs` in `tests/` for shared integration test helpers.

## Naming Conventions

- **Files:** snake_case (`user_service.rs`, `auth_tests.rs`)
- **Types (structs, enums, traits, type aliases):** PascalCase (`UserService`, `ConnectionPool`, `IntoIterator`)
- **Functions/variables:** snake_case (`get_user_by_id`, `max_retries`)
- **Consts/statics:** SCREAMING_SNAKE_CASE (`MAX_CONNECTIONS`, `DEFAULT_TIMEOUT`)
- **Macros:** snake_case or PascalCase (follow standard library convention for the macro name)
- **Crate names:** snake_case, preferably single word (`serde`, `reqwest`). Don't use `rust-` or `rs-` prefixes.
- **Feature flags:** snake_case in `Cargo.toml` (`[features] enable-tls = []`)
- **Constructor convention:** `new()` for the simplest constructor, `with_*()` for builder methods. If fallible, use a `build()` method that returns `Result`.
- **Avoid redundant prefixes.** In module `auth`, the type is `Token` not `AuthToken` — callers use `auth::Token`. In module `models`, it's `User` not `UserModel`.
