# canon

> Keep Claude Code on the rails. One marketplace, 11 plugins: the `lockstep-*` series enforces how code is written (locked onto each framework's latest stable release), `flow` provides workflow commands, and `plain-chinese` governs Chinese writing (plain language, no internet or AI jargon).

**Languages**: [简体中文](README.md) · [繁體中文](README.zh-Hant.md) · **English** · [日本語](README.ja.md)

> Maintained by [zhaoJian](https://www.zhaojian.net) · Repo: https://github.com/zhaojiannet/canon

## What this is

A Claude Code plugin marketplace named `canon` (from *canonical* — "do it the canonical way"). It holds 11 independent plugins in three groups; install whichever you need:

| Plugin | Governs | How it applies |
|---|---|---|
| **`lockstep-*`** (9 framework plugins) | Code conventions: one plugin per framework, enforcing the latest stable official practices for Astro / Vue 3.5 / Nuxt UI v4 / Tailwind v4 / TypeScript / Echo v5 / sqlc / Fastify v5 / PostgreSQL, and forbidding deprecated patterns | Auto-activates when Claude works with files matching `paths` |
| **`flow`** | Workflow commands: `go`, `cm`, `e2e`, `pin`, `vet` | The first four are invoked manually: `/flow:go`, `/flow:cm`, `/flow:e2e`, `/flow:pin`; `vet` is called by `cm` before each commit |
| **`plain-chinese`** | Chinese writing: forces plain Simplified Chinese, bans internet/workplace buzzwords and AI-tic phrasing, keeps real technical terms | An output style, always on once enabled |

**Why the frameworks are split into 9 plugins instead of one bundle**: a plugin is the smallest install unit — install one and all its skills come along, with no way to pick. Splitting lets you install per project stack. An Astro site installs only `lockstep-astro`; the backend frameworks you don't use cost zero tokens (each skill's description consumes Claude's skill-listing budget).

---

## lockstep-* (code conventions)

### The problem it solves

You hit this constantly: Tailwind, Nuxt UI, Vue, TypeScript, Echo, Fastify, PostgreSQL all document the most direct way to do something, yet deep into a long session the AI forgets, hand-rolls CSS, bypasses the official API, or writes last-generation syntax. You correct it by hand every time. Soft constraints like CLAUDE.md or global memory don't hold up over a long session.

`lockstep-*` solves this with Claude Code's official [skill mechanism](https://code.claude.com/docs/en/skills): each skill is a markdown rule document whose `paths` limit by file type when it activates — work with `.vue` files and the Vue / Nuxt UI rules enter context, work with `.css` and Tailwind enters, work with `.go` and Echo / sqlc enter, work with `migrations/*.sql` and the PostgreSQL schema + migration-safety rules enter. Rules load only while Claude works with matching files, not permanently resident in context; after auto-compaction in a long session, each skill keeps only its first 5,000 tokens within a shared 25,000-token budget, and skills invoked earlier can be dropped entirely.

### Install

```bash
# 1. Register the marketplace
/plugin marketplace add zhaojiannet/canon

# 2. Install the framework plugins your project needs (example: a Vue frontend)
/plugin install lockstep-vue@canon
/plugin install lockstep-nuxt-ui@canon
/plugin install lockstep-tailwind@canon
/plugin install lockstep-typescript@canon

# 3. Reload to take effect
/reload-plugins
```

**Common combinations by scenario**:

| Scenario | Install |
|---|---|
| Astro site | `lockstep-astro` + `lockstep-tailwind` + `lockstep-typescript` |
| Vue frontend | `lockstep-vue` + `lockstep-nuxt-ui` + `lockstep-tailwind` + `lockstep-typescript` |
| Go backend | `lockstep-echo` + `lockstep-sqlc` + `lockstep-postgres` |
| Node backend | `lockstep-fastify` + `lockstep-typescript` + `lockstep-postgres` |
| Universal (recommended for all) | `flow` + `plain-chinese` |

Verify: type `/plugin`, open the **Installed** tab, and you'll see the `lockstep-*` plugins you installed. Then ask `What skills are available?` to have Claude list the skills.

### The framework plugins

| Plugin | Trigger paths | What it enforces | Auto call name |
|---|---|---|---|
| `lockstep-astro` | `**/*.astro` | Astro 7 static-first: zero JS by default, `client:visible`/`client:idle` over `client:load`, `server:defer` over full-page SSR, Content Collections over `Astro.glob` | `/lockstep-astro:astro` |
| `lockstep-vue` | `**/*.vue` | Vue 3.5+ SFC: `<script setup>` + typed `defineProps`/`defineEmits` + `defineModel` + `useTemplateRef`; no Options API / mixins | `/lockstep-vue:vue` |
| `lockstep-nuxt-ui` | `**/*.vue` | Use Nuxt UI v4 components (U-prefix) instead of raw `<button>` / `<input>` / `<dialog>` | `/lockstep-nuxt-ui:nuxt-ui` |
| `lockstep-tailwind` | `.vue/.astro/.html/.tsx/.jsx/.css` | Tailwind v4 utility-first: no `<style>` blocks / deprecated utilities; force `oklch()` / paren syntax / v4 syntax | `/lockstep-tailwind:tailwind` |
| `lockstep-typescript` | `.ts/.tsx` | TypeScript 7 strict: strict is the default in 7 and stays on; `unknown` plus narrowing over `any`, const objects over enum, ES modules over namespace; rewrite the flags 7 removed (`baseUrl` / `target es5` / `outFile`) | `/lockstep-typescript:typescript` |
| `lockstep-echo` | `**/*.go` | Echo v5 error handling: `HTTPError` + central `HTTPErrorHandler`, `errors.Is`/`errors.As`, `%w` wrap, graceful shutdown | `/lockstep-echo:echo` |
| `lockstep-sqlc` | `queries/*.sql, sqlc.yaml` | sqlc codegen: SQL is the source of truth, calls go through the generated `Querier`, no hand-written `database/sql` | `/lockstep-sqlc:sqlc` |
| `lockstep-fastify` | `.ts/.tsx/.js/.mjs` (whether or not they import fastify) | Fastify v5: encapsulated plugins, `fastify-plugin` (fp) for cross-scope, JSON schema validation over manual | `/lockstep-fastify:fastify` |
| `lockstep-postgres` | `*.sql` under `migrations/`, `schema/`, `migrate/`, `sqitch/` | PostgreSQL schema design + migration safety (merged): snake_case + BIGSERIAL/UUID + timestamptz + explicit FK ON DELETE; transaction-wrapped + 3-step NOT NULL + `CREATE INDEX CONCURRENTLY` + destructive ops gated behind approval | `/lockstep-postgres:postgres` |

Activation:

- **Automatic**: when Claude works with a file matching `paths`, it loads the corresponding skill into context (this is the main path day to day — you rarely type the command)
- **Manual**: `/lockstep-<framework>:<skill>`, e.g. `/lockstep-echo:echo`

### Workflow commands: flow (manual)

The `flow` plugin holds four manual workflow commands with `disable-model-invocation: true`: they fire only when you type `/`, Claude never triggers them automatically, and their description stays out of context — it's just a label in the `/` menu. The fifth, `vet`, is a pre-commit checker that Claude may invoke itself; `cm` calls it before every commit.

```bash
/plugin install flow@canon
/reload-plugins
```

| Command | What it does | Invoke |
|---|---|---|
| `go` | A discipline that runs through the whole task: check the latest official docs before acting, pick the best approach over a quick hack; actively guard security and leave no known vulnerabilities; don't fudge or work around problems; own mistakes honestly; report leftovers truthfully, never fake "done" | `/flow:go` |
| `cm` | Commit the current changes in classified batches: group by the actual `git diff` (a task list in context is only a hint), keep changes together by default rather than splitting, one commit does one thing, commit messages follow the spec, shown to you before committing, never push without consent | `/flow:cm` |
| `e2e` | Full end-to-end test of the current project using its existing accounts and environment: read the project into a profile (entry points, accounts, side-effect levels, backup/restore, business rules) → write a plan and wait for your review → generate Playwright specs via the official `playwright-cli` skill → cross-cutting smoke on every page → run in a standalone container, classify each failure before fixing (app bugs are reported, never patched) → write a report. Every scenario carries a status in the plan, work resumes across sessions, and "todo" must be 0 at the end. The project under test gains only an `e2e/` directory and one `.gitignore` line; nothing Playwright-related is installed on the host | `/flow:e2e` |
| `pin` | Pin down one UI bug that is hard to confirm by eye (timing, intermittent, multi-step) before fixing it: write a Playwright spec that reproduces it, choose how many runs from the reproduction rate → minimise the steps and list hypotheses → locate the cause from `error-context.md` and the trace → fix the app code → run only the related specs found by grep, never the full suite. The spec stays in `e2e/tests/repro/` as a regression test. Stops to ask before changing assertions, when no backup command exists, or after 3 failed fixes. Uses the same container setup as `e2e` | `/flow:pin` |
| `vet` | Pre-commit check: runs with `context: fork` in a subagent that cannot see the conversation, given only the diff and the draft message. Flags sentences with no counterpart in the diff, references no one can open in the repo, comments that narrate process or run too long, and non-Chinese text. Reports only, never edits. `cm` calls it before each commit; you can also run it mid-work to check comments alone | `/flow:vet` |

> These are the author's personal workflow commands. `go` is general discipline anyone can use; `cm`'s commit-message format references the author's global `~/.claude/CLAUDE.md` spec — swap in your own; `e2e` drives planning, generation and healing through the official `playwright-cli` skill (installed by the container into the project's `e2e/`) — see `plugins/flow/skills/e2e/references/runner.md` for the runtime setup; `pin` shares that container and the diagnosis and timing references under `references/`.

---

## plain-chinese

### What it does

Forces Claude Code to write plain Simplified Chinese, banning internet and workplace buzzwords while keeping genuine technical terms — no overcorrecting. It also targets AI-invented phrasing and verbal tics (the "not X but Y" framing, "essentially", summary-tic closers, etc.).

At its core is an [output style](https://code.claude.com/docs/en/output-styles). An output style edits Claude Code's system prompt directly — stronger than CLAUDE.md (user-message layer), and currently the only official lever that reliably shapes Claude's wording.

To be clear: **nothing can eliminate it 100%**. The model generates text probabilistically; a prompt constraint is a "strong tendency", not a hard filter. This plugin uses the strongest soft approach available — it makes such words nearly vanish, but one or two may still slip through.

### Install

```bash
# Skip if the marketplace is already registered
/plugin marketplace add zhaojiannet/canon

# Install the plain-chinese plugin
/plugin install plain-chinese@canon
/reload-plugins
```

### Enable

This plugin's output style sets `force-for-plugin: true`, so **it applies automatically once installed and enabled — no manual selection**. It overrides your current output style.

> When you switch output styles mid-session, Claude uses the new style starting with your next message (before v2.1.251 it applied only after `/clear` or a new session). After editing a plugin's output style file, run `/reload-plugins` or restart Claude Code.

To turn it off temporarily: disable `plain-chinese` in `/plugin`. To manage the output style yourself (instead of forcing it): remove `force-for-plugin: true` from `plugins/plain-chinese/output-styles/plain-chinese.md` and select it via `/config` → Output style.

> Note: the `/output-style` command was deprecated in Claude Code v2.1.73 and removed in v2.1.91; v2.1.269 added `/output-style [name]` back to list and switch styles. `/config` → Output style also works.

---

## Migrating from older versions

### From v0.5 (the merged `canon` / `canon-chinese` plugins)

v0.5 bundled all framework skills into a single `canon` plugin. v0.6 splits them into the `lockstep-*` series (one plugin per framework, installable per scenario), breaks the workflow out as `flow`, and renames the Chinese plugin to `plain-chinese`.

```bash
# 1. Refresh the marketplace
/plugin marketplace update canon

# 2. Uninstall the old merged plugins
/plugin uninstall canon@canon
/plugin uninstall canon-chinese@canon

# 3. Install the new plugins per project stack (see "Common combinations" above)
/plugin install flow@canon
/plugin install plain-chinese@canon
/plugin install lockstep-vue@canon      # as needed, for the frameworks your project uses
/reload-plugins
```

Command name changes: `/canon:vue` → `/lockstep-vue:vue`; `/canon:go` → `/flow:go`; `/canon:cm` → `/flow:cm`. `pg-schema` and `pg-migrate` are merged into `lockstep-postgres` (a single `postgres` skill).

### From the older `lockstep-run`

This repo was once called `claude-skills` with a marketplace named `lockstep-run`. First `/plugin uninstall lockstep-run@lockstep-run` + `/plugin marketplace remove lockstep-run`, then register `canon` and install as above.

## How it works

Each `lockstep-*` skill is a markdown rule document with: core principles, forbidden items + brief reasons, old-API → new-API tables, STOP signals to report rather than work around when something can't be expressed, scenario rules, and a grep checklist to self-verify after writing. Claude loads the matching SKILL.md into context only when working with a file matching `paths` — on demand, not permanently resident.

`plain-chinese`'s output style adds the banned-word tables, AI-tic list, technical-term allowlist, and self-check list into the system prompt, applying on every turn.

## Development

Local loading (for development/debugging, no install needed). Each plugin is its own directory — pass `--plugin-dir` for each as needed:

```bash
cd <your project>
claude --plugin-dir ~/Cores/Projects/canon/plugins/lockstep-vue \
       --plugin-dir ~/Cores/Projects/canon/plugins/flow \
       --plugin-dir ~/Cores/Projects/canon/plugins/plain-chinese
```

Edits to a SKILL.md take effect immediately in the current session; after editing an output style, hooks or other components, run `/reload-plugins` in the running Claude Code. Validate structure: `claude plugin validate .` (checks marketplace.json) plus `claude plugin validate ./plugins/<plugin>` for each plugin (checks plugin.json and skill frontmatter).

> Note: don't put `": "` (colon + space) in a skill's `description` — YAML reads it as a nested mapping and the whole frontmatter fails to parse.

Official references: [Skills](https://code.claude.com/docs/en/skills) / [Plugins](https://code.claude.com/docs/en/plugins) / [Plugin Marketplaces](https://code.claude.com/docs/en/plugin-marketplaces) / [Output styles](https://code.claude.com/docs/en/output-styles).

## About the name

`canon` is from *canonical* — "do it the canonical, by-the-book way." Code written to official spec, Chinese written to a plain-language spec.

The framework series' prefix `lockstep` means "keep in step with the official version." It also has a backstory: when renaming, I thought of my good friend Xue Guiwen ("Brother Run"). During college military training, the stone-faced drill instructor had him lead the whole department in chants, and he kept yelling "quick march!" as "quick *run*!" — N times in a row, becoming the legendary Brother Run. `lockstep` was nearly retired, but stayed on as the framework plugins' prefix.

## License

MIT
