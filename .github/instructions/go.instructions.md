---
description: "Use when working with Go files. Covers formatting, testing, error handling, secure coding, naming conventions, project layout, and concurrency patterns."
applyTo: "**/*.go"
---

# Go Conventions

## Build & Test Commands

- **Build**: `go build ./...`
- **Test**: `go test ./...` (add `-v` for verbose, `-race` for race detection)
- **Test coverage**: `go test -coverprofile=coverage.out ./...`
- **Vet**: `go vet ./...`
- **Lint**: `golangci-lint run` (uses `.golangci.yml` config)
- **Format**: `gofmt -w .` and `goimports -w .`

## Code Comments — Go Style

Every exported symbol MUST have a doc comment starting with the symbol name.

```go
// NewClient creates an authenticated API client for the given endpoint.
// The client supports automatic retry with exponential backoff.
func NewClient(endpoint string, opts ...ClientOption) (*Client, error) {
```

- Package comments go in a `doc.go` file or above the `package` declaration
- Comments are complete sentences with proper punctuation
- Keep comments up to date — the compiler and `go vet` don't check comment accuracy

## Error Handling

- **Always check errors.** Never ignore an error return value.
- **Wrap with context** using `fmt.Errorf` with `%w`:

```go
if err != nil {
    return fmt.Errorf("loading config from %s: %w", path, err)
}
```

- Use `errors.Is()` and `errors.As()` for error inspection — never compare error strings
- Define sentinel errors with `var ErrX = errors.New("...")` in the package
- Only handle an error once — either log it or return it, not both

## Project Layout

```
project/
├── cmd/                    # Main applications (one per subdirectory)
│   └── server/
│       └── main.go
├── internal/               # Private packages (not importable externally)
│   ├── config/
│   ├── handler/
│   └── store/
├── pkg/                    # Public library packages
├── go.mod
├── go.sum
└── Makefile
```

- `cmd/` — one subdirectory per binary, each with a minimal `main.go`
- `internal/` — prevents external packages from importing
- Don't use `src/` — it's not a Go convention

## Testing

- **Table-driven tests** are the standard pattern:

```go
func TestFormat(t *testing.T) {
    tests := []struct {
        name string
        input string
        want  string
    }{
        {"empty", "", ""},
        {"simple", "hello", "HELLO"},
    }
    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            got := Format(tt.input)
            if got != tt.want {
                t.Errorf("Format(%q) = %q, want %q", tt.input, got, tt.want)
            }
        })
    }
}
```

- Use `t.Parallel()` for independent tests
- Use `t.Cleanup()` instead of `defer` in tests
- Use `testdata/` directories for test fixtures
- Integration tests use build tags: `//go:build integration`

## Concurrency

- Share memory by communicating — prefer channels over mutexes
- Use `context.Context` for cancellation and deadlines in all long-running operations
- Never start a goroutine without knowing when it will stop
- Use `sync.WaitGroup` or `errgroup.Group` to manage goroutine lifecycles
- The `errgroup` package (`golang.org/x/sync/errgroup`) is preferred when you need error propagation

## General Practices

- Zero-value initialization is preferred over constructors when sufficient
- Accept interfaces, return structs
- Package names are lowercase, single-word, no underscores
- Variable names: short for local scope (`i`, `c`), descriptive for package-level
- Don't use `panic` for expected errors — only for truly unrecoverable states

## Secure Coding

- **No secrets in code.** Use `os.Getenv()` or a secrets manager. Never commit `.env` files. Use `os.ReadFile()` for config files outside the repo.
- **Input validation at every boundary.** Validate all HTTP request bodies, URL params, and CLI args before use. Use struct tags with `validate` (e.g., `go-playground/validator`) for declarative validation.
- **SQL injection prevention.** Always use parameterized queries with `?` placeholders. Never concatenate user input into SQL strings. Use `database/sql` or an ORM that parameterizes.
- **Template injection prevention.** Use `html/template` (not `text/template`) when rendering HTML. `html/template` auto-escapes by context (HTML, JS, CSS, URL).
- **Use `crypto/rand` for secrets.** Never use `math/rand` for tokens, session IDs, or cryptographic purposes. Use `crypto/rand.Read()` or `encoding/base64`.
- **TLS verification enabled.** Don't set `InsecureSkipVerify: true` on TLS configs except in development. Use proper certificate validation in production.
- **Timeouts on all network calls.** Every `http.Client`, `grpc.Dial`, and `sql.DB` connection needs a timeout. A stuck goroutine leaks memory.

## Testing & QA

- **Fuzz testing:** Add fuzz tests (`func FuzzXxx(f *testing.F)`) for parsers, decoders, and input-handling functions. Run with `go test -fuzz=. -fuzztime=30s` in CI for new fuzz targets.
- **Benchmarks:** Every performance-sensitive function should have a benchmark (`func BenchmarkXxx(b *testing.B)`). Run with `go test -bench=. -benchmem` to catch regressions.
- **Coverage in CI:** Run `go test -coverprofile=coverage.out ./...` and enforce a minimum coverage threshold (suggest 70%). Use `go tool cover -html=coverage.out` to inspect gaps.
- **Race detection:** Always run `go test -race ./...` in CI. The race detector catches data races that are otherwise silent and catastrophic in production.
- **Test helpers:** Use `t.Helper()` at the top of test helper functions so failure line numbers point to the calling test, not the helper.

## Naming Conventions

- **Packages:** lowercase, single word, no underscores, no camelCase. `httpclient` not `http_client` or `httpClient`. Short and descriptive.
- **Files:** snake_case (`user_handler.go`, `auth_test.go`)
- **Exported symbols:** PascalCase (`NewClient`, `MaxRetries`)
- **Unexported symbols:** camelCase (`initConfig`, `defaultTimeout`)
- **Interfaces:** single-method interfaces suffixed with `-er` (`Reader`, `Writer`, `Closer`). Multi-method interfaces describe behavior (`FileSystem`, `RoundTripper`).
- **Acronyms:** all-uppercase or all-lowercase, consistently. `HTTPServer` or `httpServer`, never `HttpServer`. `URL` not `Url`.
- **Getters:** no `Get` prefix. `Count()` not `GetCount()`. Exception: when the getter involves I/O or computation, prefix is acceptable (`GetRemoteConfig()`).
- **Short names in limited scope:** `i` for loop index, `c` for context in a ~10-line function, `r` for an http.Request in a handler. Don't abbreviate at package level.
