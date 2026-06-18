---
description: "Use when writing standalone TypeScript (non-Next.js). Covers tsconfig, module systems, type safety, async patterns, and Node.js conventions."
applyTo: "**/*.{ts,tsx}"
---

# TypeScript Conventions

## TypeScript Configuration

Your `tsconfig.json` must be strict.

```json
{
  "compilerOptions": {
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "noImplicitReturns": true,
    "noFallthroughCasesInSwitch": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "esModuleInterop": true,
    "isolatedModules": true,
    "forceConsistentCasingInFileNames": true,
    "skipLibCheck": true
  }
}
```

- **`strict: true`**: enables all strict type-checking flags. Not optional — add it to every project.
- **`noUncheckedIndexedAccess`**: `obj[key]` returns `T | undefined`. Catches the most common runtime error.
- **`noUnusedLocals`/`noUnusedParameters`**: dead code is a bug. Use `_` prefix for intentionally unused params.
- **`skipLibCheck: true`**: don't type-check `node_modules`. Faster builds, fewer spurious errors.

## Type Safety — Mandatory

Never use `any` except at API boundaries with explicit justification.

```typescript
// Bad — any infects everything it touches
function process(data: any): any {
    return data.value;
}

// Good — use unknown and narrow
function process(data: unknown): string {
    if (typeof data === "object" && data !== null && "value" in data) {
        return String((data as { value: unknown }).value);
    }
    throw new Error("Invalid data");
}
```

- **Use `unknown` over `any`**: forces you to narrow the type before use
- **Use type predicates** for runtime validation:

```typescript
function isUser(obj: unknown): obj is User {
    return typeof obj === "object" && obj !== null && "id" in obj && "email" in obj;
}
```

- **Use `as` casts sparingly**: each cast is an assertion you're betting correctness on
- **Use branded types** for nominal typing when needed:

```typescript
type UserId = string & { readonly __brand: "UserId" };
function createUserId(id: string): UserId { return id as UserId; }
```

## Error Handling

Use typed errors, not string matching.

```typescript
class AppError extends Error {
    constructor(
        message: string,
        public readonly code: string,
        public readonly statusCode: number,
        public readonly details?: unknown
    ) {
        super(message);
        this.name = "AppError";
    }
}

// Catch and handle by type, not message text
try {
    await doSomething();
} catch (err) {
    if (err instanceof AppError) {
        return { error: err.message, code: err.code };
    }
    throw err; // Re-throw unexpected errors
}
```

- **Never `catch (e)` without re-throwing or handling**: swallowed errors are debugging nightmares
- **Use `instanceof` checks**, never check `err.message.includes("timeout")`
- **Don't `throw` string literals**: always `throw new Error()`
- **Async functions**: always `await` or return. Floating promises lose errors silently.

## Async Patterns

```typescript
// Bad — sequential when parallel is possible
const user = await fetchUser(id);
const posts = await fetchPosts(id);  // Waits for user to finish

// Good — parallel
const [user, posts] = await Promise.all([
    fetchUser(id),
    fetchPosts(id),
]);

// Good — timeout wrapper
async function withTimeout<T>(promise: Promise<T>, ms: number): Promise<T> {
    const timeout = new Promise<never>((_, reject) =>
        setTimeout(() => reject(new Error("Timeout")), ms)
    );
    return Promise.race([promise, timeout]);
}
```

- **`Promise.all` for independent operations**: don't sequence what can run in parallel
- **`Promise.allSettled` when partial failure is OK**: returns results + errors, doesn't short-circuit
- **Always add timeouts to external calls**: network requests, DB queries, file reads
- **Don't mix `async/await` with `.then()` chains**: pick one style per function

## Module System

- **Use ES modules**: `"type": "module"` in `package.json`. Use `import`/`export`, not `require`.
- **Avoid barrel files** (`index.ts` that re-exports everything): they cause circular dependencies and slow bundlers. Import directly from the file.
- **One export per file is fine**: you don't need `index.ts` to have multiple exports. Explicit imports are more maintainable.
- **Use path aliases sparingly**: `@/components/Button` is convenient but breaks if you move files. Relative imports `../../components/Button` are refactor-safe.

## Node.js Conventions

- **Use `node:` prefix for built-in modules**: `import fs from "node:fs"` not `import fs from "fs"`. Makes it clear it's a built-in, not an npm package.
- **Prefer `fs/promises` over `fs` callbacks**: `await readFile(path, "utf-8")` not nested callbacks
- **Use `AbortController` for cancellable operations**: `fetch(url, { signal })`, `setTimeout` wrappers
- **Handle uncaught exceptions**:

```typescript
process.on("uncaughtException", (err) => {
    logger.fatal("Uncaught exception", { err });
    process.exit(1);
});

process.on("unhandledRejection", (reason) => {
    logger.fatal("Unhandled rejection", { reason });
    process.exit(1);
});
```

## Testing

- **Use `vitest` or `jest`** with `@swc/jest` or `ts-jest` for fast TypeScript test execution
- **Mock at the boundary**: mock HTTP calls, DB queries, file I/O. Don't mock your own types.
- **Use `zod` schemas in tests**: validate API responses match the expected shape, not just individual fields:

```typescript
const UserSchema = z.object({
    id: z.string().uuid(),
    email: z.string().email(),
    createdAt: z.string().datetime(),
});
const user = await api.getUser("42");
expect(() => UserSchema.parse(user)).not.toThrow();
```

## Styling & Organization

- **File naming**: kebab-case (`user-service.ts`, `auth-middleware.ts`)
- **Type files**: co-located with the code they type (`user.ts` has `User`, `CreateUserInput`). Use `types.ts` only for shared types used across the module.
- **Constants**: UPPER_CASE for primitive constants (`MAX_RETRIES`), PascalCase for object constants (`DefaultConfig`)
- **Enums**: prefer `as const` objects over TypeScript enums:

```typescript
// Better — simpler JS output, easier to iterate
const Status = {
    Active: "active",
    Inactive: "inactive",
} as const;
type Status = (typeof Status)[keyof typeof Status];
```
