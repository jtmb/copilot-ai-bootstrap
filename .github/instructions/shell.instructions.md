---
description: "Use when writing shell scripts. Covers safety flags, quoting, error handling, temporary files, and portability."
applyTo: "**/*.{sh,bash}"
---

# Shell Script Conventions

## Safety Flags — Mandatory

Every shell script MUST start with safety flags. No exceptions.

```bash
#!/usr/bin/env bash
set -euo pipefail
```

- **`set -e`**: Exit immediately on any command failure. Without it, a script silently continues past errors, producing garbage output or corrupting state.
- **`set -u`**: Treat unset variables as errors. Catches typos like `$BUILD_DIR` when you meant `$BUILD_DIR`.
- **`set -o pipefail`**: A pipeline fails if ANY command in it fails, not just the last one. Without it, `grep pattern file.txt | sort` succeeds even if `file.txt` doesn't exist.

If you must tolerate a failing command, be explicit:

```bash
# Expected to fail when no processes match
if ! pgrep -f "my-daemon" > /dev/null; then
    echo "Daemon not running — starting it"
fi
```

## Quoting — Mandatory

Always quote variable expansions unless you have a specific reason not to.

```bash
# Bad — breaks on filenames with spaces, globs, or empty values
rm -rf $TEMP_DIR
if [ $name = "admin" ]; then

# Good
rm -rf "$TEMP_DIR"
if [ "$name" = "admin" ]; then
```

- **Double-quote `"$var"`**: Prevents word splitting and glob expansion
- **Brace-delimit**: `"${var}_suffix"` when concatenating with text
- **Use `$(())` for arithmetic**: `$(( count + 1 ))` not `$count + 1`
- **Use `[[ ]]` for tests**: Safer than `[ ]` — handles empty strings, supports regex, no word splitting

## Error Handling

Every script that does anything destructive or important must handle errors.

```bash
# Trap errors with context
trap 'echo "Error on line $LINENO"' ERR

# Cleanup on exit (success or failure)
cleanup() {
    rm -rf "$TEMP_DIR"
    docker stop "$CONTAINER_ID" 2>/dev/null || true
}
trap cleanup EXIT
```

- Use `trap cleanup EXIT` for temporary files, background processes, docker containers
- Functions that can fail should return non-zero and let the caller decide
- `|| true` for commands you expect might fail in cleanup
- Never `rm -rf "$VAR"` without validating `$VAR` is set and non-empty first

## Temporary Files

Use `mktemp`, never hardcode paths in `/tmp/`.

```bash
# Bad — predictable, race-condition prone
TMPFILE=/tmp/my-script-output.txt

# Good — unique, secure
TMPFILE=$(mktemp) || exit 1
trap 'rm -f "$TMPFILE"' EXIT
```

- `mktemp -d` for temporary directories
- Always `trap` cleanup of temp files
- Never use `$$` for temp file names — predictable and insecure

## Portability

Write POSIX-compliant shell unless you know bash is guaranteed.

```bash
# Bash-only — fails on dash/sh
if [[ "$a" == "$b" ]]; then ... fi

# POSIX — works everywhere
if [ "$a" = "$b" ]; then ... fi
```

- **Use `#!/usr/bin/env bash`** if you need bash features (arrays, `[[ ]]`, `${var/pattern/replace}`)
- **Use `#!/bin/sh`** and stick to POSIX for maximum portability
- **No `echo -n`**: use `printf` for portable output formatting
- **Use `$()` over backticks**: `$(command)` nests, backticks don't
- **No `function` keyword**: `funcname() { ... }` is POSIX; `function funcname { ... }` is bash-only

## Patterns

```bash
# Reading lines safely — handles trailing newlines, special chars
while IFS= read -r line; do
    echo "Got: $line"
done < "$input_file"

# Reading from command output — process substitution
while IFS= read -r line; do
    echo "Got: $line"
done < <(some_command)

# Default values
OUTPUT_DIR="${1:-./output}"           # Use first arg, default to ./output
TIMEOUT="${TIMEOUT:-30}"              # Use env var, default to 30
```

- Use `IFS=` with `read` to preserve leading/trailing whitespace
- Use `-r` with `read` to prevent backslash interpretation
- `: "${VAR:=default}"` sets default if unset OR empty; `"${VAR:-default}"` only substitutes

## No Secrets

```bash
# Bad — password in command line, visible in ps
curl -u "admin:$PASSWORD" https://api.example.com

# Good — read from file or environment, use netrc or stdin
curl --netrc-file /secrets/netrc https://api.example.com
```

- Never pass secrets as command-line arguments — visible in `ps aux`
- Use environment variables, files with restricted permissions, or secret managers
- `chmod 600` on any file containing secrets

## Script Organization

```bash
#!/usr/bin/env bash
set -euo pipefail

# ── Defaults ──
VAR="${ENV_VAR:-default}"

# ── Usage ──
usage() { cat <<EOF
Usage: $(basename "$0") [OPTIONS] ARG
Description here.
EOF
}

# ── Parse args ──
while [[ $# -gt 0 ]]; do
    case "$1" in
        --flag) FLAG=true; shift ;;
        --help) usage; exit 0 ;;
        *) break ;;
    esac
done

# ── Validation ──
if [[ $# -lt 1 ]]; then
    echo "Error: required argument missing" >&2
    usage
    exit 1
fi

# ── Main ──
main() {
    # Logic here
}
main "$@"
```

- `usage()` function for every script with arguments
- Errors to stderr: `echo "..." >&2`
- `main "$@"` pattern keeps global scope clean
