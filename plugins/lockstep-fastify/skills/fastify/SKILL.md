---
name: fastify
description: Enforce Fastify v5 plugin/route conventions for Node/TypeScript. Use when editing .ts/.js/.mjs that import fastify or @fastify/*, or when the user mentions Fastify, plugin, route, schema validation, encapsulation, fastify-plugin/fp, or lifecycle hooks. Forbids manual JSON validation when schema works, missing fp wrapper for cross-scope decorators, and app-root hooks that should be plugin-scoped.
paths:
  - "**/*.ts"
  - "**/*.tsx"
  - "**/*.js"
  - "**/*.mjs"
allowed-tools:
  - Read
  - Grep
---

This skill enforces Fastify v5 plugin/route conventions for Node and TypeScript projects. The rule: every plugin is encapsulated by default; expose to parent scope only when needed; routes declare schemas, not manual validation.

Apply only when the file imports `fastify` or `@fastify/*`. If the project uses Express or Koa, **STOP** and ask the user before applying these rules.

## Plugin signature

```ts
import type { FastifyPluginAsync } from 'fastify'

export const userRoutes: FastifyPluginAsync = async (fastify, opts) => {
  fastify.get('/users/:id', { schema: getUserSchema }, async (req, reply) => {
    return { id: req.params.id }
  })
}
```

Use `async (fastify, opts) =>` over `(fastify, opts, done) =>`. Async return is the completion signal.

## fastify-plugin (fp) — when to wrap

Wrap with `fp` when:

- The plugin calls `fastify.decorate(...)` and parent or sibling scopes need access.
- The plugin calls `fastify.addHook(...)` that should propagate to the whole app.
- It registers a top-level shared resource (database connection, cache).

Do **not** wrap when:
- The plugin only registers routes (routes belong in their own encapsulated scope).
- The decorators are intentionally local to the subtree.

```ts
import fp from 'fastify-plugin'

export default fp(async (fastify, opts) => {
  fastify.decorate('db', await createDb(opts))
}, { name: 'db-plugin' })
```

## Route schemas — required

Every route declares JSON schema for `body`, `querystring`, `params`, and `response`. Fastify uses these for validation, serialization, and OpenAPI generation. Manual validation in handlers is forbidden when the schema can express the rule.

```ts
const createUserSchema = {
  body: {
    type: 'object',
    required: ['email'],
    properties: {
      email: { type: 'string', format: 'email' },
      name: { type: 'string', minLength: 1, maxLength: 100 }
    },
    additionalProperties: false
  },
  response: {
    201: {
      type: 'object',
      required: ['id', 'email'],
      properties: {
        id: { type: 'string' },
        email: { type: 'string' }
      }
    }
  }
} as const

fastify.post('/users', { schema: createUserSchema }, async (req, reply) => {
  reply.code(201)
  return { id: '...', email: req.body.email }
})
```

## Forbidden patterns

- `(fastify, opts, done) => { ... done() }` callback style. Use async.
- Manual validation in handler when JSON schema works.
- `fastify.addHook('onRequest', ...)` at app root for cross-cutting auth. Move to a plugin or use route-level `preHandler`.
- `fastify.decorate(...)` without `fp` wrapper, then trying to use the decoration in a sibling plugin. Encapsulation will isolate it.
- `app.use(...)` Express-style middleware. Native Fastify hooks/plugins preferred.
- `req.body as any` or skipping schema. Use TypeScript with `JSONSchema`-derived types or `@sinclair/typebox` / `zod` + `fastify-type-provider-zod`.
- Returning a status code via `return reply.code(400).send({ ... })` from an async handler. Either `reply.code(400)` then `return data`, or throw an `Error` with `.statusCode` set: `throw Object.assign(new Error('bad input'), { statusCode: 400 })`. (If `@fastify/sensible` is registered, `throw fastify.httpErrors.badRequest('...')` works too—but `httpErrors` is not core; it requires that plugin.)
- Registering plugins after `fastify.listen()`. Order: register all plugins → `await fastify.ready()` → `fastify.listen()`.
- Hand-rolled CORS / cookie / rate-limit / helmet / multipart / static / websocket. Use the official plugins: `@fastify/cors`, `@fastify/cookie`, `@fastify/rate-limit`, `@fastify/helmet`, `@fastify/multipart`, `@fastify/static`, `@fastify/websocket`.
- Hand-rolled runtime type guards when the route `schema` + type provider already covers it. Pick a type provider: `@fastify/type-provider-typebox` (official) or `fastify-type-provider-zod` (community, no `@fastify/` prefix).
- Writing a new plugin when the project already has one under `src/plugins/` or `plugins/` covering the same concern (cors, auth, db connection, etc.). grep first; reuse if found.

## Encapsulation in practice

```ts
const app = fastify()

await app.register(dbPlugin)                                     // wrapped in fp — app.db globally available
await app.register(authPlugin, { prefix: '/api' })               // decorates app.authenticate
await app.register(userRoutes, { prefix: '/api/users' })         // consumes app.db / app.authenticate

await app.listen({ port: 3000 })
```

`userRoutes` does not need `fp` because it only consumes parent decorators.

## Graceful shutdown

```ts
const close = async () => {
  await app.close()   // Fastify runs onClose hooks, drains connections
  process.exit(0)
}
process.on('SIGINT', close)
process.on('SIGTERM', close)
```

## When you need behavior outside Fastify's plugin system

If a third-party library does not provide a Fastify plugin, **STOP** and report:

> Need [library X]. Fastify plugin available: [@fastify/x or community fp wrapper]? If none, approve writing a thin fp wrapper or using `app.register(async (app) => { ... })` inline.

## Verification (run after edits that import fastify)

```bash
grep -rnE 'fastify\(\)' --include='*.ts' --include='*.js' .
grep -rnE 'function\s*\(fastify[^)]*,\s*opts[^)]*,\s*done\)' --include='*.ts' --include='*.js' .   # done-style callback
grep -rnE 'fastify\.decorate\(' --include='*.ts' --include='*.js' . | grep -v 'fp('               # decorate without fp wrapper
grep -rnE 'fastify\.(post|put|patch|delete)\([^,]+,\s*async' --include='*.ts' --include='*.js' .   # check missing schema
```

Reference: https://fastify.dev/docs/latest/Reference/Plugins/
