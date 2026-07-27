---
name: typescript
description: Enforce TypeScript 7 strict type-checking. Use when editing .ts/.tsx or tsconfig.json, or when the user mentions TypeScript, any, unknown, strict mode, type assertion, generics, enum, namespace, baseUrl, or @ts-ignore. Strict is on by default in 7 and stays on, reaches for unknown plus narrowing where any would go, const objects in place of enum, ES modules in place of namespace, and fixes the type rather than reaching for as any or @ts-ignore.
paths:
  - "**/*.ts"
  - "**/*.tsx"
allowed-tools:
  - Read
  - Grep
---

> Targets TypeScript 7 · verified 2026-07 (latest 7.0.2, released 2026-07-08).

This skill enforces TypeScript strict type-checking. The rule: trust the type system, no escape hatches.

Apply only when the project uses `typescript ^7.0` or higher. If `package.json` pins 5.x or 6.x, **STOP** and ask the user — the upgrade path runs 5 → 6 → 7, and 7 turns everything 6 deprecated into a hard error.

## tsconfig

TypeScript 7 ships these on by default. Do not write them out, and do not turn them off:

`strict` (which brings `noImplicitAny` / `strictNullChecks` / `strictFunctionTypes` / `strictBindCallApply` / `strictPropertyInitialization` / `noImplicitThis` / `alwaysStrict` / `useUnknownInCatchVariables` / `strictBuiltinIteratorReturn`), `module` at `esnext`, `target` at the current stable ES version, `noUncheckedSideEffectImports`, and `stableTypeOrdering` (which cannot be disabled). `alwaysStrict` is forced true.

Two defaults that catch people on upgrade:

- **`rootDir` now defaults to `./`**, not an inferred common source directory. A project whose sources live in `src/` must set it explicitly, or `outDir` output lands in the wrong place.
- **`types` now defaults to `[]`.** Every `@types` package used to be pulled in automatically. List what you need, or set `["*"]` to get the old behaviour back.

Add these to harden past the defaults:

```json
{
  "compilerOptions": {
    "noUncheckedIndexedAccess": true,
    "exactOptionalPropertyTypes": true,
    "noImplicitOverride": true,
    "noPropertyAccessFromIndexSignature": true,
    "noFallthroughCasesInSwitch": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true
  }
}
```

## Removed in 7 — hard errors, not warnings

| Gone | Use instead |
|---|---|
| `target` at `es5`, plus `downlevelIteration` | ES2015 is the floor now; drop both lines |
| `baseUrl` | `paths`, resolved relative to the project root |
| `moduleResolution` at `node` / `node10` / `classic` | `nodenext` or `bundler` |
| `module` at `amd` / `umd` / `systemjs` / `none` | `esnext`, which is the default |
| `outFile` | a bundler |
| `esModuleInterop` or `allowSyntheticDefaultImports` set to false | cannot be disabled — delete the line |
| `module Foo { }` spelling of a namespace | ES modules. Ambient `declare module "pkg"` is still fully supported |
| `import x from "y" assert { ... }` | `with { ... }` |
| JSDoc `@enum` | `@typedef` on `(typeof YourEnum)[keyof typeof YourEnum]` |

## Forbidden patterns

- `any` type (explicit or via inference). Use `unknown` and narrow with type guards.
- `as any` / `as unknown as Foo` chained casts. Either fix the upstream type or write a proper type guard.
- `// @ts-ignore`. Use `// @ts-expect-error <reason>` so the suppression breaks if the underlying issue is fixed.
- `namespace Foo { ... }`. Use ES module imports/exports. The `module Foo { }` spelling is a hard error in 7.
- `enum Color { ... }`. Use `const Color = { Red: 'red', Blue: 'blue' } as const; type Color = typeof Color[keyof typeof Color]`.
- `function foo(x): void` (implicit any param). Annotate or rely on inference.
- `Function` type. Use specific `(...args: T[]) => R`.
- `Object` type. Use `Record<string, unknown>` or `object`.
- `try { ... } catch (e) { e.message }` without narrowing. With `useUnknownInCatchVariables`, `e` is `unknown`; check `e instanceof Error` first.
- Manual `interface` for exhaustive union states. Use discriminated unions: `type Result = { kind: 'ok'; value: T } | { kind: 'err'; error: E }`.
- Returning `Promise<any>` / `Promise<object>`. Type the return.
- Hand-written type guards when the project already uses zod / valibot / effect-schema. Use the schema library's `parse` / `safeParse`.
- Hand-written utility when Node standard library or already-installed deps cover it: `crypto.randomUUID()` (Node 14.17+), `structuredClone()` (Node 17+), `Promise.withResolvers()` (Node 22+); reach for `lodash` / `date-fns` if already in `package.json` instead of writing your own debounce / formatDate.
- Monkey-patching globals (`Array.prototype.foo = ...`, `String.prototype.bar = ...`). Use a utility module instead.
- Hand-written utility (`formatDate` / `debounce` / `cn` / `sleep` / type guard) when the project already exposes one under `src/utils/`, `src/lib/`, or similar. grep first; reuse if found.

## Inference > explicit annotation

Let TypeScript infer when the type is obvious from the right-hand side or return statement. Annotate at boundaries: function parameters, exported APIs, public class fields.

```ts
// preferred
const items = users.map(u => u.id)                // string[] inferred
function greet(name: string) { return `hi ${name}` }   // string return inferred

// over
const items: string[] = users.map((u: User): string => u.id)
function greet(name: string): string { ... }
```

## Catch blocks (useUnknownInCatchVariables)

```ts
try {
  await doWork()
} catch (e) {
  if (e instanceof Error) {
    log.error(e.message)
  } else {
    log.error('unknown error', { e })
  }
}
```

Do not write `catch (e: any)` or access `e.message` without narrowing.

## Index access (noUncheckedIndexedAccess)

`arr[0]` returns `T | undefined`. Always check or use optional chaining:

```ts
const first = arr[0]
if (first === undefined) return
// ... use first

// or
arr[0]?.name
```

## Const assertions for literal types

```ts
const ROLES = ['admin', 'editor', 'viewer'] as const
type Role = typeof ROLES[number]
```

Replaces enum, runs at zero cost, and is JSON-serializable.

## When you cannot follow the rules

If a third-party library has weak types and you must cast, **STOP** and report:

> Need to call [API] from [library] which returns [weakly typed thing]. Approve one of: (A) write a type guard `function isFoo(x: unknown): x is Foo`, (B) declare a `.d.ts` augmentation, (C) use `unknown` and narrow at use site, (D) `as Foo` with a TODO comment if no other option fits.

Do not silently scatter `as any` or `// @ts-ignore`.

## Verification (grep after every .ts change)

```bash
grep -rnE ':\s*any\b|<any>|as any' --include='*.ts' --include='*.tsx' .
grep -rnE '@ts-ignore' --include='*.ts' --include='*.tsx' .
grep -rnE '\bnamespace\s+\w+\s*\{' --include='*.ts' --include='*.tsx' .
grep -rnE '^\s*enum\s+\w+\s*\{' --include='*.ts' --include='*.tsx' .  # non-const enum
grep -rnE 'catch\s*\(\s*\w+\s*\)\s*\{[^}]*\.message' --include='*.ts' --include='*.tsx' .
grep -rnE '"(baseUrl|outFile|downlevelIteration)"' --include='tsconfig*.json' .   # removed in 7
grep -rnE '"(target|moduleResolution)"\s*:\s*"(es5|node|node10|classic)"' --include='tsconfig*.json' .
grep -rnE 'assert\s*\{' --include='*.ts' --include='*.tsx' .                       # import assertions -> with
```

Full tsconfig reference: https://www.typescriptlang.org/tsconfig
