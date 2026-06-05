# canon

> 让 Claude Code 守规范。两个 plugin：`canon` 管代码写法，强制跟框架官方最新做法走；`canon-chinese` 管中文表达，强制说平实大白话、严禁互联网黑话和 AI 腔。

**Languages**: **简体中文** · [繁體中文](README.zh-Hant.md) · [English](README.en.md) · [日本語](README.ja.md)

> 维护：[zhaoJian](https://www.zhaojian.net) · 仓库：https://github.com/zhaojiannet/canon

## 这是什么

一个 Claude Code 插件市场，名字叫 `canon`（取 canonical，"按正典/规范做法来"）。下面两个独立 plugin，可以各装各的：

| Plugin | 管什么 | 怎么生效 |
|---|---|---|
| **`canon`** | 代码写法：10 个框架 skill，强制 Vue 3.5 / Nuxt UI v4 / Tailwind v4 / TypeScript / Go Echo v5 + sqlc / Node Fastify v5 / PostgreSQL / Astro 用各自官方最新稳定版的推荐做法，禁用已废弃写法；另含 `go`/`cm` 两个手动工作流命令 | 框架 skill 编辑对应文件时按 `paths` 自动激活；`go`/`cm` 手动调用 `/canon:go`、`/canon:cm` |
| **`canon-chinese`** | 中文表达：强制平实中文，禁互联网黑话、职场黑话和 AI 腔，保留真正的专业术语 | 一个 output-style，启用后一直生效 |

两者是同一种事——给 Claude 立规矩、不让它将就。一个清理代码里的废弃写法，一个清理中文里的黑话。

---

## canon（代码规范）

### 解决什么问题

写代码反复遇到这种情形：Tailwind、Nuxt UI、Vue、TypeScript、Echo、Fastify、PostgreSQL 官方明明给了最直接的写法，AI 在长会话之后转头就忘，自己造 CSS、绕过官方 API、写上一代旧语法。每次发现都得手动纠正一遍。CLAUDE.md、全局 memory 这种软约束在长会话后压不住。

`canon` 用 Claude Code 官方的 [skill 机制](https://code.claude.com/docs/en/skills) 解决：每个 skill 是一份 markdown 规则文档，按文件类型 `paths` 自动激活——编辑 `.vue` 时 Vue / Nuxt UI 规则进 context，编辑 `.css` 时 Tailwind 规则进 context，编辑 `.go` 时 Echo / sqlc 规则进 context，编辑 `migrations/*.sql` 时 PostgreSQL 迁移安全规则进 context。规则现读现用，不会像 CLAUDE.md 在长会话后被淡忘。

### 安装

```bash
# 1. 注册 marketplace
/plugin marketplace add zhaojiannet/canon

# 2. 装代码规范 plugin（含全部 10 个框架 skill + go/cm 工作流命令）
/plugin install canon@canon

# 3. 重新加载使其生效
/reload-plugins
```

验证：输入 `/plugin` 进 **Installed** 标签，能看到 `canon`。再用 `What skills are available?` 让 Claude 列出 skill。

### 包含的 skill

| Skill | 触发 paths | 功能 | 手动调用 |
|---|---|---|---|
| `vue` | `**/*.vue` | Vue 3.5+ SFC：强制 `<script setup>` + 类型化 `defineProps`/`defineEmits` + `defineModel` + `useTemplateRef`；禁 Options API / mixins | `/canon:vue` |
| `nuxt-ui` | `**/*.vue` | 强制用 Nuxt UI v4 组件（U 前缀）而非手写 raw `<button>` / `<input>` / `<dialog>` | `/canon:nuxt-ui` |
| `tailwind` | `.vue/.html/.tsx/.css/.scss` | Tailwind v4 utility-first：禁 `<style scoped>` / 旧 utility / 任意值变量；强制 `oklch()` / `(--xxx)` 圆括号 / `@custom-variant dark` | `/canon:tailwind` |
| `typescript` | `**/*.ts, .tsx` | TypeScript strict：禁 `any` / `@ts-ignore` / namespace / 非 const enum；强制 strict tsconfig + `unknown` 替代 any | `/canon:typescript` |
| `echo` | `**/*.go` | Echo v5 错误处理：error 冒泡、`echo.NewHTTPError` 统一错误、集中 `HTTPErrorHandler`、`errors.Is`/`errors.As`、`%w` wrap | `/canon:echo` |
| `sqlc` | `**/queries/*.sql, sqlc.yaml` | sqlc v2 + pgx/v5 + 命名规范 + 禁手写 SQL 调用 | `/canon:sqlc` |
| `fastify` | 含 fastify import 的 `.ts/.js` | Fastify v5 plugin async 写法、`fastify-plugin` (fp) 何时用、JSON schema 验证、encapsulation、graceful onClose | `/canon:fastify` |
| `pg-schema` | `**/migrations/*.sql, **/schema/*.sql` | snake_case + BIGSERIAL/UUID 主键 + timestamptz + 外键 ON DELETE 显式 + jsonb 仅用于 schemaless | `/canon:pg-schema` |
| `pg-migrate` | `**/migrations/*.sql` | 事务包裹 + IF EXISTS 守卫 + 禁裸 DROP/TRUNCATE + NOT NULL 加列分三步 + `CREATE INDEX CONCURRENTLY` | `/canon:pg-migrate` |
| `astro` | `**/*.astro` | Astro 静态优先：默认 zero JS、`client:visible`/`client:idle` 优于 `client:load`、`server:defer` 替代全页 SSR、Content Collections 替代 `Astro.glob` | `/canon:astro` |

激活方式：

- **自动**：编辑匹配 `paths` 的文件时 Claude Code 自动加载对应 skill 进 context
- **手动**：`/canon:<skill-name>`，例如 `/canon:echo`

### 工作流命令（手动触发）

除了上面 10 个按文件类型自动激活的框架 skill，canon 还含两个手动触发的工作流命令。它们设了 `disable-model-invocation: true`：只在你输入 `/` 时手动调用，Claude 不会自动触发，description 也不进 context、只当 `/` 菜单里的标签。

| 命令 | 做什么 | 调用 |
|---|---|---|
| `go` | 一套贯穿任务的工作纪律：开始干活先查官方最新文档再动手、选最佳方案不选临时做法；遇到问题不糊弄、不绕过；做错时诚实承认；有遗留时如实交代、不谎报完成 | `/canon:go` |
| `cm` | 把当前改动分批分类提交：按主题分组、一个 commit 只做一件事，commit message 按规范写，提交前列给你确认，未经同意不 push | `/canon:cm` |

> 这两个是作者的个人工作流命令。`go` 是通用纪律，谁装都能用；`cm` 的 commit message 格式引用作者全局 `~/.claude/CLAUDE.md` 里的规范，你可以换成自己的提交规范。

---

## canon-chinese（朴素中文）

### 做什么

强制 Claude Code 用平实中文，禁互联网黑话与职场黑话（根因、二开、兜底、对齐、抓手、闭环、赋能、沉淀、链路、落地……），同时保留真正的专业术语（解耦、幂等、并发、复用……），不矫枉过正。还专门管 AI 自造腔和口癖（底层逻辑、本质上、"不是 X 而是 Y"拉踩句式、"综上所述"总结腔……）。

核心是一个 [output-style](https://code.claude.com/docs/en/output-styles)。output-style 直接改 Claude Code 的系统提示词，比 CLAUDE.md（用户消息层）约束力更强，是目前唯一能稳定影响 Claude 措辞的官方手段。

需要说清楚：**没有任何方法能 100% 杜绝**。模型是概率生成文本，提示词约束是"强烈倾向"而非硬过滤。这个 plugin 用的是约束力最强的软办法，能让这类词几乎绝迹，但偶尔仍可能漏一两个。

### 安装

```bash
# marketplace 已注册的话跳过这步
/plugin marketplace add zhaojiannet/canon

# 装朴素中文 plugin
/plugin install canon-chinese@canon
/reload-plugins
```

### 启用

这个 plugin 的 output-style 设了 `force-for-plugin: true`，**装完启用就自动生效，不用手动选**。它会覆盖你当前的 output-style 设置。

> output-style 是系统提示的一部分，Claude Code 每次会话开始时读一次。改完要 `/clear` 或开新会话才生效。

想临时关掉：在 `/plugin` 里禁用 `canon-chinese`。想自己手动管理 output-style（而不是自动强制）：把 `plugins/chinese/output-styles/plain-chinese.md` 里的 `force-for-plugin: true` 删掉，改用 `/config` → Output style 手动选。

> 注意：旧的 `/output-style` 命令在 Claude Code v2.1.73 弃用、v2.1.91 移除，现在统一用 `/config`。

---

## 从旧版（lockstep-run）迁移

这个 repo 之前叫 `claude-skills`、marketplace 叫 `lockstep-run`。升级到 `canon`：

```bash
# 1. 卸载旧 plugin
/plugin uninstall lockstep-run@lockstep-run

# 2. 移除旧 marketplace
/plugin marketplace remove lockstep-run

# 3. 注册新 marketplace 并安装
/plugin marketplace add zhaojiannet/canon
/plugin install canon@canon
/plugin install canon-chinese@canon
/reload-plugins
```

skill 调用名也变短了：`/lockstep-run:tailwind-utility-first` → `/canon:tailwind`。

## How it works

`canon` 的每个 skill 是一份 markdown 规则文档，包含：核心原则、禁用项 + 简短原因、旧 API → 新 API 对照表、表达不出来时报告而非绕过的 STOP 信号、场景化规则、写完自查的 grep 清单。Claude Code 在编辑匹配 `paths` 的文件时把对应 SKILL.md 载入 context，按需加载而非长期占用。

`canon-chinese` 的 output-style 把禁用词对照表、AI 口癖清单、专业术语白名单、自检清单加进系统提示，每轮回复都生效。

## Development

本地加载（开发/调试用，不需安装）：

```bash
cd <你的项目>
claude --plugin-dir ~/Cores/Projects/canon/plugins/code \
       --plugin-dir ~/Cores/Projects/canon/plugins/chinese
```

改完 SKILL.md 或 output-style 后，在已运行的 Claude Code 内执行 `/reload-plugins` 生效。校验结构：`claude plugin validate .`。

官方参考：[Skills](https://code.claude.com/docs/en/skills) / [Plugins](https://code.claude.com/docs/en/plugins) / [Plugin Marketplaces](https://code.claude.com/docs/en/plugin-marketplaces) / [Output styles](https://code.claude.com/docs/en/output-styles)。

## 名字由来

`canon` 取自 canonical——"按正典、规范的做法来"。两个 plugin 是一回事：代码按官方规范写，中文按平实规范说。

这个项目前身代号 `lockstep`（"齐步走"）。改名时想起我的好兄弟薛贵文（跑哥）。大学军训，教官一脸严肃让他带全系喊口号，他连续 N 次把"齐步走"喊成"齐步跑"，从此成为传说中的跑哥。

## License

MIT
