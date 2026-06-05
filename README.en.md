# canon

> Keep Claude Code on the rules. Two plugins: `canon` governs how code is written, forcing the latest official framework conventions; `canon-chinese` governs how Chinese reads, forcing plain everyday wording and strictly banning internet jargon and AI-speak.

**Languages**: [简体中文](README.md) · [繁體中文](README.zh-Hant.md) · **English** · [日本語](README.ja.md)

> Maintained by [zhaoJian](https://www.zhaojian.net) · Repo: https://github.com/zhaojiannet/canon

## What this is

A Claude Code plugin marketplace named `canon` (from "canonical" — "do it the canonical, by-the-book way"). It contains two independent plugins you can install separately:

| Plugin | What it governs | How it takes effect |
|---|---|---|
| **`canon`** | How code is written: 10 framework skills that force Vue 3.5 / Nuxt UI v4 / Tailwind v4 / TypeScript / Go Echo v5 + sqlc / Node Fastify v5 / PostgreSQL / Astro to follow each library's latest stable official conventions, and ban deprecated patterns; plus two manual workflow commands `go`/`cm` | Framework skills activate automatically by `paths` when you edit a matching file; `go`/`cm` are called manually via `/canon:go`, `/canon:cm` |
| **`canon-chinese`** | How Chinese reads: forces plain Chinese, bans internet and workplace jargon and AI-speak, while keeping genuine technical terms | A single output-style; once enabled, it stays in effect |

Both do the same thing — set rules for Claude and never let it settle for less. One cleans deprecated patterns out of your code, the other cleans jargon out of your Chinese.

---

## canon (code conventions)

### What problem it solves

Writing code, you run into this again and again: Tailwind, Nuxt UI, Vue, TypeScript, Echo, Fastify, PostgreSQL all ship the most direct way to do something, but after a long session the AI forgets and goes off — rolling its own CSS, bypassing the official API, writing last-generation syntax. Every time you catch it, you correct it by hand. Soft constraints like CLAUDE.md and global memory cannot hold up in a long session.

`canon` solves this with Claude Code's official [skill system](https://code.claude.com/docs/en/skills): each skill is a markdown rule document that activates automatically by file type via `paths` — editing `.vue` pulls the Vue / Nuxt UI rules into context, editing `.css` pulls the Tailwind rules in, editing `.go` pulls the Echo / sqlc rules in, editing `migrations/*.sql` pulls the PostgreSQL migration safety rules in. The rules are read fresh and used on the spot, so they do not fade like CLAUDE.md does after a long session.

### Installation

```bash
# 1. Register the marketplace
/plugin marketplace add zhaojiannet/canon

# 2. Install the code-conventions plugin (ships all 10 framework skills + go/cm workflow commands)
/plugin install canon@canon

# 3. Reload to activate
/reload-plugins
```

Verify: type `/plugin`, go to the **Installed** tab, and you should see `canon`. Then ask Claude `What skills are available?` to have it list the skills.

### Included skills

| Skill | Trigger paths | What it does | Manual call |
|---|---|---|---|
| `vue` | `**/*.vue` | Vue 3.5+ SFC: forces `<script setup>` + type-based `defineProps`/`defineEmits` + `defineModel` + `useTemplateRef`; bans Options API / mixins | `/canon:vue` |
| `nuxt-ui` | `**/*.vue` | Forces Nuxt UI v4 components (U-prefix) over hand-written raw `<button>` / `<input>` / `<dialog>` | `/canon:nuxt-ui` |
| `tailwind` | `.vue/.html/.tsx/.css/.scss` | Tailwind v4 utility-first: bans `<style scoped>` / deprecated utilities / arbitrary-value variables; forces `oklch()` / `(--xxx)` paren syntax / `@custom-variant dark` | `/canon:tailwind` |
| `typescript` | `**/*.ts, .tsx` | TypeScript strict: bans `any` / `@ts-ignore` / namespace / non-const enum; forces strict tsconfig + `unknown` over any | `/canon:typescript` |
| `echo` | `**/*.go` | Echo v5 error handling: error bubbling, unified errors via `echo.NewHTTPError`, centralized `HTTPErrorHandler`, `errors.Is`/`errors.As`, `%w` wrap | `/canon:echo` |
| `sqlc` | `**/queries/*.sql, sqlc.yaml` | sqlc v2 + pgx/v5 + naming convention + ban on hand-written SQL calls | `/canon:sqlc` |
| `fastify` | `.ts/.js` files importing fastify | Fastify v5 async plugin style, when to use `fastify-plugin` (fp), JSON schema validation, encapsulation, graceful onClose | `/canon:fastify` |
| `pg-schema` | `**/migrations/*.sql, **/schema/*.sql` | snake_case + BIGSERIAL/UUID PK + timestamptz + explicit FK ON DELETE + jsonb only for schemaless data | `/canon:pg-schema` |
| `pg-migrate` | `**/migrations/*.sql` | Transaction wrapping + IF EXISTS guards + no bare DROP/TRUNCATE + three-step NOT NULL adds + `CREATE INDEX CONCURRENTLY` | `/canon:pg-migrate` |
| `astro` | `**/*.astro` | Astro static-first: zero JS by default, `client:visible`/`client:idle` over `client:load`, `server:defer` over full-page SSR, Content Collections over `Astro.glob` | `/canon:astro` |

How they activate:

- **Automatic**: Claude Code loads the matching skill into context when you edit a file matching its `paths`.
- **Manual**: `/canon:<skill-name>`, e.g. `/canon:echo`.

### Workflow commands (manual trigger)

Beyond the 10 framework skills that auto-activate by file type, canon ships two manual workflow commands. They set `disable-model-invocation: true`: invoked only when you type `/`, never auto-triggered by Claude, and their description stays out of context — it is just a label in the `/` menu.

| Command | What it does | Call |
|---|---|---|
| `go` | A work discipline that runs through a whole task: check the latest official docs before starting and pick the best approach over a quick hack; when something breaks, don't fudge or work around it; own your mistakes; when work is unfinished, report it honestly instead of claiming success | `/canon:go` |
| `cm` | Commit current changes in grouped, classified batches: group by topic, one commit per concern, write each message to spec, show you the plan before committing, never push without consent | `/canon:cm` |

> These two are the author's personal workflow commands. `go` is general discipline anyone can use; `cm`'s commit-message format references the author's global `~/.claude/CLAUDE.md` rules — swap in your own commit conventions.

---

## canon-chinese (plain Chinese)

### What it does

Forces Claude Code to use plain Chinese, banning internet and workplace jargon (根因, 二开, 兜底, 对齐, 抓手, 闭环, 赋能, 沉淀, 链路, 落地, …) while keeping genuine technical terms (解耦, 幂等, 并发, 复用, …), without overcorrecting. It also specifically handles AI's self-invented tone and verbal tics (底层逻辑, 本质上, the "不是 X 而是 Y" put-down construction, the "综上所述" summary tone, …).

At its core is one [output-style](https://code.claude.com/docs/en/output-styles). An output-style edits Claude Code's system prompt directly, which is a stronger constraint than CLAUDE.md (which lives at the user-message layer). It is currently the only official means of reliably influencing Claude's wording.

To be clear: **no method can eliminate this 100%**. The model generates text probabilistically, and prompt constraints are a "strong tendency," not a hard filter. This plugin uses the strongest soft approach available — it can make these words nearly disappear, but one or two may still slip through occasionally.

### Installation

```bash
# Skip this step if the marketplace is already registered
/plugin marketplace add zhaojiannet/canon

# Install the plain-Chinese plugin
/plugin install canon-chinese@canon
/reload-plugins
```

### Enabling

This plugin's output-style sets `force-for-plugin: true`, so **it takes effect automatically once installed and enabled — no manual selection needed**. It overrides your current output-style setting.

> An output-style is part of the system prompt, which Claude Code reads once at the start of each session. After changing it, run `/clear` or start a new session for it to take effect.

To turn it off temporarily: disable `canon-chinese` in `/plugin`. To manage the output-style manually yourself (instead of auto-forcing): remove `force-for-plugin: true` from `plugins/chinese/output-styles/plain-chinese.md`, then select it manually via `/config` → Output style.

> Note: the old `/output-style` command was deprecated in Claude Code v2.1.73 and removed in v2.1.91; use `/config` now.

---

## Migrating from the old version (lockstep-run)

This repo was previously called `claude-skills` and the marketplace was called `lockstep-run`. To upgrade to `canon`:

```bash
# 1. Uninstall the old plugin
/plugin uninstall lockstep-run@lockstep-run

# 2. Remove the old marketplace
/plugin marketplace remove lockstep-run

# 3. Register the new marketplace and install
/plugin marketplace add zhaojiannet/canon
/plugin install canon@canon
/plugin install canon-chinese@canon
/reload-plugins
```

The skill call names are shorter too: `/lockstep-run:tailwind-utility-first` → `/canon:tailwind`.

## How it works

Each skill in `canon` is a markdown rule document containing: core principles, forbidden items with brief reasons, an old-API → new-API mapping table, STOP signals to report rather than work around when something can't be expressed, scenario-specific rules, and a grep checklist for self-checking after editing. Claude Code loads the matching SKILL.md into context when you edit a file matching its `paths` — loaded on demand rather than occupying context permanently.

The `canon-chinese` output-style adds the banned-word mapping table, the AI verbal-tic list, the technical-term allowlist, and the self-check list into the system prompt, so it takes effect on every reply.

## Development

Local load (for development/debugging, no install needed):

```bash
cd <your-project>
claude --plugin-dir ~/Cores/Projects/canon/plugins/code \
       --plugin-dir ~/Cores/Projects/canon/plugins/chinese
```

After editing a SKILL.md or an output-style, run `/reload-plugins` inside the running Claude Code to apply changes. Validate the structure with `claude plugin validate .`.

Official references: [Skills](https://code.claude.com/docs/en/skills) / [Plugins](https://code.claude.com/docs/en/plugins) / [Plugin Marketplaces](https://code.claude.com/docs/en/plugin-marketplaces) / [Output styles](https://code.claude.com/docs/en/output-styles).

## The name

`canon` comes from "canonical" — "do it the canonical, by-the-book way." The two plugins are one and the same idea: write code by the official conventions, write Chinese by the plain conventions.

This project was originally codenamed `lockstep` ("marching in step"). When renaming it, I thought of my good friend Xue Guiwen ("Paoge"). During college military training, the drill instructor, dead serious, handed him the whole department's roll call. He kept shouting "running in step" (齐步跑) instead of "marching in step" (齐步走), N times in a row — and from then on he became the legendary "Paoge" ("Brother Run").

## License

MIT
