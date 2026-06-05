---
name: typescript
description: Enforce TypeScript strict type-checking. Use this skill when editing .ts or .tsx files, configuring tsconfig.json, or when the user mentions TypeScript, type, any, unknown, strict mode, type assertion, generics, namespace, enum, ts-ignore, ts-expect-error, type guard, PropType. Forbids any, as any, // @ts-ignore, namespace, non-const enum, untyped catch. Forces strict tsconfig, unknown over any, type inference, ESM imports, const + as const over enum.
paths:
  - "**/*.ts"
  - "**/*.tsx"
allowed-tools:
  - Read
  - Grep
---

This skill enforces TypeScript strict type-checking. The rule: trust the type system, no escape hatches.

Apply only when the project uses `typescript ^5.0` or higher. If `package.json` pins an older major, **STOP** and ask the user.

## tsconfig requirements

```json
{
  "compilerOptions": {
    "strict": true,
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

`strict: true` enables `noImplicitAny` / `strictNullChecks` / `strictFunctionTypes` / `strictBindCallApply` / `strictPropertyInitialization` / `noImplicitThis` / `alwaysStrict` / `useUnknownInCatchVariables` / `strictBuiltinIteratorReturn`. Add `noUncheckedIndexedAccess` / `exactOptionalPropertyTypes` / `noImplicitOverride` / `noPropertyAccessFromIndexSignature` to harden further.

## Forbidden patterns

- `any` type (explicit or via inference). Use `unknown` and narrow with type guards.
- `as any` / `as unknown as Foo` chained casts. Either fix the upstream type or write a proper type guard.
- `// @ts-ignore`. Use `// @ts-expect-error <reason>` so the suppression breaks if the underlying issue is fixed.
- `namespace Foo { ... }`. Use ES module imports/exports.
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
```

Full tsconfig reference: https://www.typescriptlang.org/tsconfig
