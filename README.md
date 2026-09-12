# canon

> 让 Claude Code 守规范。一个 marketplace 下 11 个插件：`lockstep-*` 系列管代码写法（锁定各框架官方最新稳定版的做法），`flow` 管工作流命令，`plain-chinese` 管中文表达（平实大白话、严禁互联网黑话和 AI 腔）。

**Languages**: **简体中文** · [繁體中文](README.zh-Hant.md) · [English](README.en.md) · [日本語](README.ja.md)

> 维护：[zhaoJian](https://www.zhaojian.net) · 仓库：https://github.com/zhaojiannet/canon

## 这是什么

一个 Claude Code 插件市场，名字叫 `canon`（取 canonical，"按正典/规范做法来"）。下面 11 个独立插件，按职责分三类，各装各的：

| 插件 | 管什么 | 怎么生效 |
|---|---|---|
| **`lockstep-*`**（9 个框架插件） | 代码写法：一框架一插件，强制 Astro / Vue 3.5 / Nuxt UI v4 / Tailwind v4 / TypeScript / Echo v5 / sqlc / Fastify v5 / PostgreSQL 用各自官方最新稳定版的推荐做法，禁用已废弃写法 | 编辑对应文件时按 `paths` 自动激活 |
| **`flow`** | 三个手动工作流命令 `go`/`cm`/`e2e` | 手动调用 `/flow:go`、`/flow:cm`、`/flow:e2e` |
| **`plain-chinese`** | 中文表达：强制平实中文，禁互联网黑话、职场黑话和 AI 腔，保留真正的专业术语 | 一个 output-style，启用后一直生效 |

**为什么框架拆成 9 个插件、而不是打包成一个**：插件是最小安装单元，装一个就把它的 skill 全带来、装的人挑不了。拆开后才能按项目技术栈单独装——写 Astro 站的项目只装 `lockstep-astro`，用不到的后端框架一个 token 不占（每个 skill 的描述会占 Claude 的 skill 列表预算）。

---

## lockstep-*（代码规范）

### 解决什么问题

写代码反复遇到这种情形：Tailwind、Nuxt UI、Vue、TypeScript、Echo、Fastify、PostgreSQL 官方明明给了最直接的写法，AI 在长会话之后转头就忘，自己造 CSS、绕过官方 API、写上一代旧语法。每次发现都得手动纠正一遍。CLAUDE.md、全局 memory 这种软约束在长会话后压不住。

`lockstep-*` 用 Claude Code 官方的 [skill 机制](https://code.claude.com/docs/en/skills) 解决：每个 skill 是一份 markdown 规则文档，按文件类型 `paths` 自动激活——编辑 `.vue` 时 Vue / Nuxt UI 规则进 context，编辑 `.css` 时 Tailwind 规则进 context，编辑 `.go` 时 Echo / sqlc 规则进 context，编辑 `migrations/*.sql` 时 PostgreSQL 表设计与迁移安全规则进 context。规则现读现用，不会像 CLAUDE.md 在长会话后被淡忘。

### 安装

```bash
# 1. 注册 marketplace
/plugin marketplace add zhaojiannet/canon

# 2. 按项目技术栈装需要的框架插件（示例：Vue 前端）
/plugin install lockstep-vue@canon
/plugin install lockstep-nuxt-ui@canon
/plugin install lockstep-tailwind@canon
/plugin install lockstep-typescript@canon

# 3. 重新加载使其生效
/reload-plugins
```

**按场景的常见组合**：

| 场景 | 装哪些 |
|---|---|
| Astro 企业站 | `lockstep-astro` + `lockstep-tailwind` + `lockstep-typescript` |
| Vue 前端 | `lockstep-vue` + `lockstep-nuxt-ui` + `lockstep-tailwind` + `lockstep-typescript` |
| Go 后端 | `lockstep-echo` + `lockstep-sqlc` + `lockstep-postgres` |
| Node 后端 | `lockstep-fastify` + `lockstep-typescript` + `lockstep-postgres` |
| 通用（建议都装） | `flow` + `plain-chinese` |

验证：输入 `/plugin` 进 **Installed** 标签，能看到装的 `lockstep-*`。再用 `What skills are available?` 让 Claude 列出 skill。

### 包含的框架插件

| 插件 | 触发 paths | 功能 | 自动调用名 |
|---|---|---|---|
| `lockstep-astro` | `**/*.astro` | Astro 7 静态优先：默认 zero JS、`client:visible`/`client:idle` 优于 `client:load`、`server:defer` 替代全页 SSR、Content Collections 替代 `Astro.glob` | `/lockstep-astro:astro` |
| `lockstep-vue` | `**/*.vue` | Vue 3.5+ SFC：强制 `<script setup>` + 类型化 `defineProps`/`defineEmits` + `defineModel` + `useTemplateRef`；禁 Options API / mixins | `/lockstep-vue:vue` |
| `lockstep-nuxt-ui` | `**/*.vue` | 强制用 Nuxt UI v4 组件（U 前缀）而非手写 raw `<button>` / `<input>` / `<dialog>` | `/lockstep-nuxt-ui:nuxt-ui` |
| `lockstep-tailwind` | `.vue/.astro/.html/.tsx/.jsx/.css` | Tailwind v4 utility-first：禁 `<style>` 块 / 旧 utility；强制 `oklch()` / 圆括号语法 / v4 语法 | `/lockstep-tailwind:tailwind` |
| `lockstep-typescript` | `.ts/.tsx` | TypeScript 7 strict：strict 在 7 里是默认值、不许关；`unknown` + 收窄替代 `any`，const 对象替代 enum，ES module 替代 namespace；7 移除的 `baseUrl` / `target es5` / `outFile` 一律改写 | `/lockstep-typescript:typescript` |
| `lockstep-echo` | `**/*.go` | Echo v5 错误处理：`HTTPError` + 集中 `HTTPErrorHandler`、`errors.Is`/`errors.As`、`%w` wrap、graceful shutdown | `/lockstep-echo:echo` |
| `lockstep-sqlc` | `queries/*.sql, sqlc.yaml` | sqlc codegen：SQL 为源、走生成的 `Querier`、禁手写 `database/sql` | `/lockstep-sqlc:sqlc` |
| `lockstep-fastify` | 含 fastify import 的 `.ts/.js/.mjs` | Fastify v5：封装式 plugin、`fastify-plugin` (fp) 跨作用域、JSON schema 验证替代手写 | `/lockstep-fastify:fastify` |
| `lockstep-postgres` | `migrations/`、`schema/`、`migrate/`、`sqitch/` 下 `*.sql` | PostgreSQL 表设计 + 迁移安全（合并）：snake_case + BIGSERIAL/UUID + timestamptz + 外键 ON DELETE；事务包裹 + NOT NULL 加列分三步 + `CREATE INDEX CONCURRENTLY` + 破坏性操作先审批 | `/lockstep-postgres:postgres` |

激活方式：

- **自动**：编辑匹配 `paths` 的文件时 Claude Code 自动加载对应 skill 进 context（日常主要靠这个，几乎不用手敲命令）
- **手动**：`/lockstep-<框架>:<skill>`，例如 `/lockstep-echo:echo`

### 工作流命令：flow（手动触发）

`flow` 插件含三个手动触发的工作流命令，设了 `disable-model-invocation: true`：只在你输入 `/` 时手动调用，Claude 不会自动触发，description 也不进 context、只当 `/` 菜单里的标签。

```bash
/plugin install flow@canon
/reload-plugins
```

| 命令 | 做什么 | 调用 |
|---|---|---|
| `go` | 一套贯穿任务的工作纪律：开始干活先查官方最新文档再动手、选最佳方案不选临时做法；涉及安全时主动防护、不留已知漏洞；遇到问题不糊弄、不绕过；做错时诚实承认；有遗留时如实交代、不谎报完成 | `/flow:go` |
| `cm` | 把当前改动分批分类提交：优先按上下文里的任务列表分组、一个 commit 只做一件事，commit message 按规范写，提交前列给你确认，未经同意不 push | `/flow:cm` |
| `e2e` | 给当前项目做端到端测试：先读项目出画像（入口、账号、副作用分级、数据恢复、业务口径）→ 出计划等你审 → 按官方 `playwright-cli` skill 生成 Playwright spec → 每页横切面冒烟 → 容器内跑、失败先定性再修（应用 bug 只报不改）→ 出报告。状态存 `.scratch/e2e/`，跨会话可续；宿主机不装任何 Playwright 组件 | `/flow:e2e` |

> 这三个是作者的个人工作流命令。`go` 是通用纪律，谁装都能用；`cm` 的 commit message 格式引用作者全局 `~/.claude/CLAUDE.md` 里的规范，你可以换成自己的提交规范；`e2e` 依赖官方 `playwright-cli` skill（`~/.claude/skills/playwright-cli/`，用 `playwright-cli install --skills -g` 安装）和 OrbStack 容器域名，运行环境搭法见 `plugins/flow/skills/e2e/references/runner.md`。

---

## plain-chinese（朴素中文）

### 做什么

强制 Claude Code 用平实中文，禁互联网黑话与职场黑话（根因、二开、兜底、对齐、抓手、闭环、赋能、沉淀、链路、落地……），同时保留真正的专业术语（解耦、幂等、并发、复用……），不矫枉过正。还专门管 AI 自造腔和口癖（底层逻辑、本质上、"不是 X 而是 Y"拉踩句式、"综上所述"总结腔……）。

核心是一个 [output-style](https://code.claude.com/docs/en/output-styles)。output-style 直接改 Claude Code 的系统提示词，比 CLAUDE.md（用户消息层）约束力更强，是目前唯一能稳定影响 Claude 措辞的官方手段。

需要说清楚：**没有任何方法能 100% 杜绝**。模型是概率生成文本，提示词约束是"强烈倾向"而非硬过滤。这个插件用的是约束力最强的软办法，能让这类词几乎绝迹，但偶尔仍可能漏一两个。

### 安装

```bash
# marketplace 已注册的话跳过这步
/plugin marketplace add zhaojiannet/canon

# 装朴素中文插件
/plugin install plain-chinese@canon
/reload-plugins
```

### 启用

这个插件的 output-style 设了 `force-for-plugin: true`，**装完启用就自动生效，不用手动选**。它会覆盖你当前的 output-style 设置。

> output-style 是系统提示的一部分，Claude Code 每次会话开始时读一次。改完要 `/clear` 或开新会话才生效。

想临时关掉：在 `/plugin` 里禁用 `plain-chinese`。想自己手动管理 output-style（而不是自动强制）：把 `plugins/plain-chinese/output-styles/plain-chinese.md` 里的 `force-for-plugin: true` 删掉，改用 `/config` → Output style 手动选。

> 注意：旧的 `/output-style` 命令在 Claude Code v2.1.73 弃用、v2.1.91 移除，现在统一用 `/config`。

---

## 从旧版迁移

### 从 v0.5（`canon` / `canon-chinese` 两个合并插件）

v0.5 把框架 skill 全打包在 `canon` 一个插件里。v0.6 拆成 `lockstep-*` 系列（一框架一插件，按场景单独装）、工作流独立为 `flow`、中文改名 `plain-chinese`。

```bash
# 1. 刷新 marketplace
/plugin marketplace update canon

# 2. 卸掉旧的合并插件
/plugin uninstall canon@canon
/plugin uninstall canon-chinese@canon

# 3. 按项目技术栈装新插件（见上面"按场景的常见组合"）
/plugin install flow@canon
/plugin install plain-chinese@canon
/plugin install lockstep-vue@canon      # 按需,装你项目用得上的框架
/reload-plugins
```

命令名变化：`/canon:vue` → `/lockstep-vue:vue`；`/canon:go` → `/flow:go`；`/canon:cm` → `/flow:cm`。`pg-schema` 和 `pg-migrate` 合并成了 `lockstep-postgres`（一个 `postgres` skill）。

### 从更旧的 `lockstep-run`

这个 repo 更早叫 `claude-skills`、marketplace 叫 `lockstep-run`。先 `/plugin uninstall lockstep-run@lockstep-run` + `/plugin marketplace remove lockstep-run`，再按上面注册 `canon` 安装。

## How it works

每个 `lockstep-*` 的 skill 是一份 markdown 规则文档，包含：核心原则、禁用项 + 简短原因、旧 API → 新 API 对照表、表达不出来时报告而非绕过的 STOP 信号、场景化规则、写完自查的 grep 清单。Claude Code 在编辑匹配 `paths` 的文件时把对应 SKILL.md 载入 context，按需加载而非长期占用。

`plain-chinese` 的 output-style 把禁用词对照表、AI 口癖清单、专业术语白名单、自检清单加进系统提示，每轮回复都生效。

## Development

本地加载（开发/调试用，不需安装）。每个插件是独立目录，按需 `--plugin-dir` 多个：

```bash
cd <你的项目>
claude --plugin-dir ~/Cores/Projects/canon/plugins/lockstep-vue \
       --plugin-dir ~/Cores/Projects/canon/plugins/flow \
       --plugin-dir ~/Cores/Projects/canon/plugins/plain-chinese
```

改完 SKILL.md 或 output-style 后，在已运行的 Claude Code 内执行 `/reload-plugins` 生效。校验结构：`claude plugin validate .`（校验 marketplace.json）+ 对每个插件 `claude plugin validate ./plugins/<插件>`（校验 plugin.json 和 skill frontmatter）。

> 注意：skill 的 `description` 里不要出现 `": "`（冒号+空格），YAML 会把它当成嵌套映射、导致 frontmatter 整段解析失败。

官方参考：[Skills](https://code.claude.com/docs/en/skills) / [Plugins](https://code.claude.com/docs/en/plugins) / [Plugin Marketplaces](https://code.claude.com/docs/en/plugin-marketplaces) / [Output styles](https://code.claude.com/docs/en/output-styles)。

## 名字由来

`canon` 取自 canonical——"按正典、规范的做法来"。代码按官方规范写，中文按平实规范说。

框架系列的前缀 `lockstep`（"齐步走"）取意"跟官方版本步调一致"。这个名字还有段来历：改名时想起我的好兄弟薛贵文（跑哥）。大学军训，教官一脸严肃让他带全系喊口号，他连续 N 次把"齐步走"喊成"齐步跑"，从此成为传说中的跑哥。`lockstep` 一度要退休，最后留下来做了框架插件的前缀。

## License

MIT
