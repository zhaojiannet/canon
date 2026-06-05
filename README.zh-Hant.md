# canon

> 讓 Claude Code 守規範。兩個 plugin：`canon` 管程式碼寫法，強制跟框架官方最新做法走；`canon-chinese` 管中文表達，強制說平實大白話、嚴禁網路黑話和 AI 腔。

**Languages**: [简体中文](README.md) · **繁體中文** · [English](README.en.md) · [日本語](README.ja.md)

> 維護：[zhaoJian](https://www.zhaojian.net) · 倉庫：https://github.com/zhaojiannet/canon

## 這是什麼

一個 Claude Code 外掛市集，名字叫 `canon`（取 canonical，「按正典／規範做法來」）。下面兩個獨立 plugin，可以各裝各的：

| Plugin | 管什麼 | 怎麼生效 |
|---|---|---|
| **`canon`** | 程式碼寫法：10 個框架 skill，強制 Vue 3.5 / Nuxt UI v4 / Tailwind v4 / TypeScript / Go Echo v5 + sqlc / Node Fastify v5 / PostgreSQL / Astro 用各自官方最新穩定版的推薦做法，禁用已廢棄寫法；另含 `go`/`cm` 兩個手動工作流命令 | 框架 skill 編輯對應檔案時按 `paths` 自動啟動；`go`/`cm` 手動呼叫 `/canon:go`、`/canon:cm` |
| **`canon-chinese`** | 中文表達：強制平實中文，禁網路黑話、職場黑話和 AI 腔，保留真正的專業術語 | 一個 output-style，啟用後一直生效 |

兩者是同一種事——給 Claude 立規矩、不讓它將就。一個清理程式碼裡的廢棄寫法，一個清理中文裡的黑話。

---

## canon（程式碼規範）

### 解決什麼問題

寫程式反覆遇到這種情形：Tailwind、Nuxt UI、Vue、TypeScript、Echo、Fastify、PostgreSQL 官方明明給了最直接的寫法，AI 在長會話之後轉頭就忘，自製 CSS、繞過官方 API、寫上一代舊語法。每次發現都得手動修正一遍。CLAUDE.md、全域 memory 這種軟約束在長會話後壓不住。

`canon` 用 Claude Code 官方的 [skill 機制](https://code.claude.com/docs/en/skills) 解決：每個 skill 是一份 markdown 規則文件，按檔案類型 `paths` 自動啟動——編輯 `.vue` 時 Vue / Nuxt UI 規則進 context，編輯 `.css` 時 Tailwind 規則進 context，編輯 `.go` 時 Echo / sqlc 規則進 context，編輯 `migrations/*.sql` 時 PostgreSQL migration 安全規則進 context。規則現讀現用，不會像 CLAUDE.md 在長會話後被淡忘。

### 安裝

```bash
# 1. 註冊 marketplace
/plugin marketplace add zhaojiannet/canon

# 2. 裝程式碼規範 plugin（含全部 10 個框架 skill + go/cm 工作流命令）
/plugin install canon@canon

# 3. 重新載入使其生效
/reload-plugins
```

驗證：輸入 `/plugin` 進 **Installed** 分頁，能看到 `canon`。再用 `What skills are available?` 讓 Claude 列出 skill。

### 包含的 skill

| Skill | 觸發 paths | 功能 | 手動呼叫 |
|---|---|---|---|
| `vue` | `**/*.vue` | Vue 3.5+ SFC：強制 `<script setup>` + 類型化 `defineProps`/`defineEmits` + `defineModel` + `useTemplateRef`；禁 Options API / mixins | `/canon:vue` |
| `nuxt-ui` | `**/*.vue` | 強制用 Nuxt UI v4 元件（U 前綴）而非手寫 raw `<button>` / `<input>` / `<dialog>` | `/canon:nuxt-ui` |
| `tailwind` | `.vue/.html/.tsx/.css/.scss` | Tailwind v4 utility-first：禁 `<style scoped>` / 舊 utility / 任意值變數；強制 `oklch()` / `(--xxx)` 圓括號 / `@custom-variant dark` | `/canon:tailwind` |
| `typescript` | `**/*.ts, .tsx` | TypeScript strict：禁 `any` / `@ts-ignore` / namespace / 非 const enum；強制 strict tsconfig + `unknown` 替代 any | `/canon:typescript` |
| `echo` | `**/*.go` | Echo v5 錯誤處理：error 冒泡、`echo.NewHTTPError` 統一錯誤、集中 `HTTPErrorHandler`、`errors.Is`/`errors.As`、`%w` wrap | `/canon:echo` |
| `sqlc` | `**/queries/*.sql, sqlc.yaml` | sqlc v2 + pgx/v5 + 命名規範 + 禁手寫 SQL 呼叫 | `/canon:sqlc` |
| `fastify` | 含 fastify import 的 `.ts/.js` | Fastify v5 plugin async 寫法、`fastify-plugin` (fp) 何時用、JSON schema 驗證、encapsulation、graceful onClose | `/canon:fastify` |
| `pg-schema` | `**/migrations/*.sql, **/schema/*.sql` | snake_case + BIGSERIAL/UUID 主鍵 + timestamptz + 外鍵 ON DELETE 顯式 + jsonb 僅用於 schemaless | `/canon:pg-schema` |
| `pg-migrate` | `**/migrations/*.sql` | 交易包裹 + IF EXISTS 守衛 + 禁裸 DROP/TRUNCATE + NOT NULL 加欄位分三步 + `CREATE INDEX CONCURRENTLY` | `/canon:pg-migrate` |
| `astro` | `**/*.astro` | Astro 靜態優先：預設 zero JS、`client:visible`/`client:idle` 優於 `client:load`、`server:defer` 替代全頁 SSR、Content Collections 替代 `Astro.glob` | `/canon:astro` |

啟動方式：

- **自動**：編輯匹配 `paths` 的檔案時 Claude Code 自動載入對應 skill 進 context
- **手動**：`/canon:<skill-name>`，例如 `/canon:echo`

### 工作流命令（手動觸發）

除了上面 10 個按檔案類型自動啟動的框架 skill，canon 還含兩個手動觸發的工作流命令。它們設了 `disable-model-invocation: true`：只在你輸入 `/` 時手動呼叫，Claude 不會自動觸發，description 也不進 context、只當 `/` 選單裡的標籤。

| 命令 | 做什麼 | 呼叫 |
|---|---|---|
| `go` | 一套貫穿任務的工作紀律：開始幹活先查官方最新文件再動手、選最佳方案不選臨時做法；遇到問題不糊弄、不繞過；做錯時誠實承認；有遺留時如實交代、不謊報完成 | `/canon:go` |
| `cm` | 把當前改動分批分類提交：按主題分組、一個 commit 只做一件事，commit message 按規範寫，提交前列給你確認，未經同意不 push | `/canon:cm` |

> 這兩個是作者的個人工作流命令。`go` 是通用紀律，誰裝都能用；`cm` 的 commit message 格式引用作者全域 `~/.claude/CLAUDE.md` 裡的規範，你可以換成自己的提交規範。

---

## canon-chinese（樸素中文）

### 做什麼

強制 Claude Code 用平實中文，禁網路黑話與職場黑話（根因、二開、兜底、對齊、抓手、閉環、賦能、沉澱、鏈路、落地……），同時保留真正的專業術語（解耦、冪等、並行、複用……），不矯枉過正。還專門管 AI 自造腔和口頭禪（底層邏輯、本質上、「不是 X 而是 Y」拉踩句式、「綜上所述」總結腔……）。

核心是一個 [output-style](https://code.claude.com/docs/en/output-styles)。output-style 直接改 Claude Code 的系統提示詞，比 CLAUDE.md（使用者訊息層）約束力更強，是目前唯一能穩定影響 Claude 措辭的官方手段。

需要說清楚：**沒有任何方法能 100% 杜絕**。模型是機率生成文字，提示詞約束是「強烈傾向」而非硬過濾。這個 plugin 用的是約束力最強的軟辦法，能讓這類詞幾乎絕跡，但偶爾仍可能漏一兩個。

### 安裝

```bash
# marketplace 已註冊的話跳過這步
/plugin marketplace add zhaojiannet/canon

# 裝樸素中文 plugin
/plugin install canon-chinese@canon
/reload-plugins
```

### 啟用

這個 plugin 的 output-style 設了 `force-for-plugin: true`，**裝完啟用就自動生效，不用手動選**。它會覆蓋你目前的 output-style 設定。

> output-style 是系統提示的一部分，Claude Code 每次會話開始時讀一次。改完要 `/clear` 或開新會話才生效。

想暫時關掉：在 `/plugin` 裡停用 `canon-chinese`。想自己手動管理 output-style（而不是自動強制）：把 `plugins/chinese/output-styles/plain-chinese.md` 裡的 `force-for-plugin: true` 刪掉，改用 `/config` → Output style 手動選。

> 注意：舊的 `/output-style` 命令在 Claude Code v2.1.73 棄用、v2.1.91 移除，現在統一用 `/config`。

---

## 從舊版（lockstep-run）遷移

這個 repo 之前叫 `claude-skills`、marketplace 叫 `lockstep-run`。升級到 `canon`：

```bash
# 1. 卸載舊 plugin
/plugin uninstall lockstep-run@lockstep-run

# 2. 移除舊 marketplace
/plugin marketplace remove lockstep-run

# 3. 註冊新 marketplace 並安裝
/plugin marketplace add zhaojiannet/canon
/plugin install canon@canon
/plugin install canon-chinese@canon
/reload-plugins
```

skill 呼叫名也變短了：`/lockstep-run:tailwind-utility-first` → `/canon:tailwind`。

## How it works

`canon` 的每個 skill 是一份 markdown 規則文件，包含：核心原則、禁用項 + 簡短原因、舊 API → 新 API 對照表、表達不出來時報告而非繞過的 STOP 信號、場景化規則、寫完自查的 grep 清單。Claude Code 在編輯匹配 `paths` 的檔案時把對應 SKILL.md 載入 context，按需載入而非長期佔用。

`canon-chinese` 的 output-style 把禁用詞對照表、AI 口頭禪清單、專業術語白名單、自檢清單加進系統提示，每輪回覆都生效。

## Development

本地載入（開發／除錯用，不需安裝）：

```bash
cd <你的專案>
claude --plugin-dir ~/Cores/Projects/canon/plugins/code \
       --plugin-dir ~/Cores/Projects/canon/plugins/chinese
```

改完 SKILL.md 或 output-style 後，在已執行的 Claude Code 內執行 `/reload-plugins` 生效。校驗結構：`claude plugin validate .`。

官方參考：[Skills](https://code.claude.com/docs/en/skills) / [Plugins](https://code.claude.com/docs/en/plugins) / [Plugin Marketplaces](https://code.claude.com/docs/en/plugin-marketplaces) / [Output styles](https://code.claude.com/docs/en/output-styles)。

## 名字由來

`canon` 取自 canonical——「按正典、規範的做法來」。兩個 plugin 是一回事：程式碼按官方規範寫，中文按平實規範說。

這個專案前身代號 `lockstep`（「齊步走」）。改名時想起我的好兄弟薛貴文（跑哥）。大學軍訓，教官一臉嚴肅讓他帶全系喊口號，他連續 N 次把「齊步走」喊成「齊步跑」，從此成為傳說中的跑哥。

## License

MIT
