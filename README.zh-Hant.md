# canon

> 讓 Claude Code 守規範。一個 marketplace 下 11 個插件：`lockstep-*` 系列管程式碼寫法（鎖定各框架官方最新穩定版的做法），`flow` 管工作流命令，`plain-chinese` 管中文表達（平實大白話、嚴禁網路黑話和 AI 腔）。

**Languages**: [简体中文](README.md) · **繁體中文** · [English](README.en.md) · [日本語](README.ja.md)

> 維護：[zhaoJian](https://www.zhaojian.net) · 倉庫：https://github.com/zhaojiannet/canon

## 這是什麼

一個 Claude Code 插件市場，名字叫 `canon`（取 canonical，「按正典/規範做法來」）。下面 11 個獨立插件，按職責分三類，各裝各的：

| 插件 | 管什麼 | 怎麼生效 |
|---|---|---|
| **`lockstep-*`**（9 個框架插件） | 程式碼寫法：一框架一插件，強制 Astro / Vue 3.5 / Nuxt UI v4 / Tailwind v4 / TypeScript / Echo v5 / sqlc / Fastify v5 / PostgreSQL 用各自官方最新穩定版的推薦做法，禁用已廢棄寫法 | 處理符合 `paths` 的檔案時自動啟用 |
| **`flow`** | 工作流命令 `go`/`cm`/`e2e`/`pin`/`vet` | 前四個手動呼叫 `/flow:go`、`/flow:cm`、`/flow:e2e`、`/flow:pin`；`vet` 由 `cm` 在提交前自動呼叫 |
| **`plain-chinese`** | 中文表達：強制平實中文，禁網路黑話、職場黑話和 AI 腔，保留真正的專業術語 | 一個 output-style，啟用後一直生效 |

**為什麼框架拆成 9 個插件、而不是打包成一個**：插件是最小安裝單元，裝一個就把它的 skill 全帶來、裝的人挑不了。拆開後才能按專案技術棧單獨裝——寫 Astro 站的專案只裝 `lockstep-astro`，用不到的後端框架一個 token 不佔（每個 skill 的描述會佔 Claude 的 skill 清單預算）。

---

## lockstep-*（程式碼規範）

### 解決什麼問題

寫程式碼反覆遇到這種情形：Tailwind、Nuxt UI、Vue、TypeScript、Echo、Fastify、PostgreSQL 官方明明給了最直接的寫法，AI 在長會話之後轉頭就忘，自己造 CSS、繞過官方 API、寫上一代舊語法。每次發現都得手動糾正一遍。CLAUDE.md、全域 memory 這種軟約束在長會話後壓不住。

`lockstep-*` 用 Claude Code 官方的 [skill 機制](https://code.claude.com/docs/en/skills) 解決：每個 skill 是一份 markdown 規則文件，用 `paths` 按檔案類型限定何時啟用——處理 `.vue` 檔案時 Vue / Nuxt UI 規則進 context，處理 `.css` 時 Tailwind 規則進 context，處理 `.go` 時 Echo / sqlc 規則進 context，處理 `migrations/*.sql` 時 PostgreSQL 表設計與遷移安全規則進 context。規則只在處理匹配的檔案時才載入，不長期佔用 context；長會話自動壓縮後，每個 skill 只保留前 5,000 token，所有 skill 共用 25,000 token 預算，呼叫較早的可能被整個丟掉。

### 安裝

```bash
# 1. 註冊 marketplace
/plugin marketplace add zhaojiannet/canon

# 2. 按專案技術棧裝需要的框架插件（範例：Vue 前端）
/plugin install lockstep-vue@canon
/plugin install lockstep-nuxt-ui@canon
/plugin install lockstep-tailwind@canon
/plugin install lockstep-typescript@canon

# 3. 重新載入使其生效
/reload-plugins
```

**按場景的常見組合**：

| 場景 | 裝哪些 |
|---|---|
| Astro 企業站 | `lockstep-astro` + `lockstep-tailwind` + `lockstep-typescript` |
| Vue 前端 | `lockstep-vue` + `lockstep-nuxt-ui` + `lockstep-tailwind` + `lockstep-typescript` |
| Go 後端 | `lockstep-echo` + `lockstep-sqlc` + `lockstep-postgres` |
| Node 後端 | `lockstep-fastify` + `lockstep-typescript` + `lockstep-postgres` |
| 通用（建議都裝） | `flow` + `plain-chinese` |

驗證：輸入 `/plugin` 進 **Installed** 分頁，能看到裝的 `lockstep-*`。再用 `What skills are available?` 讓 Claude 列出 skill。

### 包含的框架插件

| 插件 | 觸發 paths | 功能 | 自動呼叫名 |
|---|---|---|---|
| `lockstep-astro` | `**/*.astro` | Astro 7 靜態優先：預設 zero JS、`client:visible`/`client:idle` 優於 `client:load`、`server:defer` 替代全頁 SSR、Content Collections 替代 `Astro.glob` | `/lockstep-astro:astro` |
| `lockstep-vue` | `**/*.vue` | Vue 3.5+ SFC：強制 `<script setup>` + 類型化 `defineProps`/`defineEmits` + `defineModel` + `useTemplateRef`；禁 Options API / mixins | `/lockstep-vue:vue` |
| `lockstep-nuxt-ui` | `**/*.vue` | 強制用 Nuxt UI v4 元件（U 前綴）而非手寫 raw `<button>` / `<input>` / `<dialog>` | `/lockstep-nuxt-ui:nuxt-ui` |
| `lockstep-tailwind` | `.vue/.astro/.html/.tsx/.jsx/.css` | Tailwind v4 utility-first：禁 `<style>` 區塊 / 舊 utility；強制 `oklch()` / 圓括號語法 / v4 語法 | `/lockstep-tailwind:tailwind` |
| `lockstep-typescript` | `.ts/.tsx` | TypeScript 7 strict：strict 在 7 裡是預設值、不許關；`unknown` + 收窄替代 `any`，const 物件替代 enum，ES module 替代 namespace；7 移除的 `baseUrl` / `target es5` / `outFile` 一律改寫 | `/lockstep-typescript:typescript` |
| `lockstep-echo` | `**/*.go` | Echo v5 錯誤處理：`HTTPError` + 集中 `HTTPErrorHandler`、`errors.Is`/`errors.As`、`%w` wrap、graceful shutdown | `/lockstep-echo:echo` |
| `lockstep-sqlc` | `queries/*.sql, sqlc.yaml` | sqlc codegen：SQL 為源、走產生的 `Querier`、禁手寫 `database/sql` | `/lockstep-sqlc:sqlc` |
| `lockstep-fastify` | `.ts/.tsx/.js/.mjs`（不區分是否 import fastify） | Fastify v5：封裝式 plugin、`fastify-plugin` (fp) 跨作用域、JSON schema 驗證替代手寫 | `/lockstep-fastify:fastify` |
| `lockstep-postgres` | `migrations/`、`schema/`、`migrate/`、`sqitch/` 下 `*.sql` | PostgreSQL 表設計 + 遷移安全（合併）：snake_case + BIGSERIAL/UUID + timestamptz + 外鍵 ON DELETE；交易包裹 + NOT NULL 加欄分三步 + `CREATE INDEX CONCURRENTLY` + 破壞性操作先審批 | `/lockstep-postgres:postgres` |

啟用方式：

- **自動**：處理匹配 `paths` 的檔案時 Claude 自動載入對應 skill 進 context（日常主要靠這個，幾乎不用手敲命令）
- **手動**：`/lockstep-<框架>:<skill>`，例如 `/lockstep-echo:echo`

### 工作流命令：flow（手動觸發）

`flow` 插件含四個手動觸發的工作流命令，設了 `disable-model-invocation: true`：只在你輸入 `/` 時手動呼叫，Claude 不會自動觸發，description 也不進 context、只當 `/` 選單裡的標籤。第五個 `vet` 是提交前的檢查器，Claude 可以自己呼叫，`cm` 在每個 commit 前都會呼叫它。

```bash
/plugin install flow@canon
/reload-plugins
```

| 命令 | 做什麼 | 呼叫 |
|---|---|---|
| `go` | 一套貫穿任務的工作紀律：開始幹活先查官方最新文件再動手、選最佳方案不選臨時做法；涉及安全時主動防護、不留已知漏洞；遇到問題不糊弄、不繞過；做錯時誠實承認；有遺留時如實交代、不謊報完成 | `/flow:go` |
| `cm` | 把當前改動分批分類提交：以實際 `git diff` 為準分組（上下文裡的任務清單只當線索）、預設合在一起不硬拆、一個 commit 只做一件事，commit message 按規範寫，提交前列給你確認，未經同意不 push | `/flow:cm` |
| `e2e` | 用現有帳號和環境給當前專案做全量端到端測試：先讀專案出畫像（入口、帳號、副作用分級、備份恢復、業務口徑）→ 出計畫等你審 → 按官方 `playwright-cli` skill 生成 Playwright spec → 每頁橫切面冒煙 → 獨立容器裡跑、失敗先定性再修（應用 bug 只報不改）→ 出報告。計畫裡每條場景帶狀態，跨會話續做，收尾「待做」必須為 0。被測專案只多一個 `e2e/` 目錄和一行 `.gitignore`，宿主機不裝任何 Playwright 元件 | `/flow:e2e` |
| `pin` | 把一個肉眼難確認的介面 bug（時序、偶發、多步互動）釘住再修：先寫 Playwright spec 重現，按重現率決定跑幾次 → 縮到最少步驟、列假設 → 按 `error-context.md` 和 trace 定位 → 改應用程式碼 → 只跑 grep 出的相關 spec，不跑全量。spec 留在 `e2e/tests/repro/` 作回歸。改斷言、缺備份命令、連續 3 次修不好時停下問你。容器沿用 `e2e` 的搭法 | `/flow:pin` |
| `vet` | 提交前把關：用 `context: fork` 在一個看不到對話的子代理裡跑，只拿 diff 和 message 草稿逐句核對，標出指不到 diff 的句子、倉庫裡找不到的編號、敘述過程或超長的註解、非中文。只報告不改檔案。`cm` 每個 commit 前自動呼叫；寫程式中途也可以直接敲它只查註解 | `/flow:vet` |

> 這些是作者的個人工作流命令。`go` 是通用紀律，誰裝都能用；`cm` 的 commit message 格式引用作者全域 `~/.claude/CLAUDE.md` 裡的規範，你可以換成自己的提交規範；`e2e` 用官方 `playwright-cli` skill 做規劃 / 生成 / 修復（由容器裝進專案的 `e2e/` 內），執行環境搭法見 `plugins/flow/skills/e2e/references/runner.md`；`pin` 與 `e2e` 共用這個容器和 `references/` 下的定位、時序參考。

---

## plain-chinese（樸素中文）

### 做什麼

強制 Claude Code 用平實中文，禁網路黑話與職場黑話（根因、二開、兜底、對齊、抓手、閉環、賦能、沉澱、鏈路、落地……），同時保留真正的專業術語（解耦、冪等、並行、複用……），不矯枉過正。還專門管 AI 自造腔和口頭禪（底層邏輯、本質上、「不是 X 而是 Y」拉踩句式、「綜上所述」總結腔……）。

核心是一個 [output-style](https://code.claude.com/docs/en/output-styles)。output-style 直接改 Claude Code 的系統提示詞，比 CLAUDE.md（使用者訊息層）約束力更強，是目前唯一能穩定影響 Claude 措辭的官方手段。

需要說清楚：**沒有任何方法能 100% 杜絕**。模型是機率生成文字，提示詞約束是「強烈傾向」而非硬過濾。這個插件用的是約束力最強的軟辦法，能讓這類詞幾乎絕跡，但偶爾仍可能漏一兩個。

### 安裝

```bash
# marketplace 已註冊的話跳過這步
/plugin marketplace add zhaojiannet/canon

# 裝樸素中文插件
/plugin install plain-chinese@canon
/reload-plugins
```

### 啟用

這個插件的 output-style 設了 `force-for-plugin: true`，**裝完啟用就自動生效，不用手動選**。它會覆蓋你當前的 output-style 設定。

> 會話中途切換 output-style，從下一則訊息起就按新風格回覆（v2.1.251 之前要 `/clear` 或開新會話才生效）。改了插件裡的 output-style 檔案，要執行 `/reload-plugins` 或重新啟動 Claude Code。

想暫時關掉：在 `/plugin` 裡停用 `plain-chinese`。想自己手動管理 output-style（而不是自動強制）：把 `plugins/plain-chinese/output-styles/plain-chinese.md` 裡的 `force-for-plugin: true` 刪掉，改用 `/config` → Output style 手動選。

> 注意：`/output-style` 命令曾在 Claude Code v2.1.73 棄用、v2.1.91 移除，v2.1.269 又加回了 `/output-style [name]`，可以列出和切換風格；也可以用 `/config` → Output style。

---

## 從舊版遷移

### 從 v0.5（`canon` / `canon-chinese` 兩個合併插件）

v0.5 把框架 skill 全打包在 `canon` 一個插件裡。v0.6 拆成 `lockstep-*` 系列（一框架一插件，按場景單獨裝）、工作流獨立為 `flow`、中文改名 `plain-chinese`。

```bash
# 1. 重新整理 marketplace
/plugin marketplace update canon

# 2. 卸掉舊的合併插件
/plugin uninstall canon@canon
/plugin uninstall canon-chinese@canon

# 3. 按專案技術棧裝新插件（見上面「按場景的常見組合」）
/plugin install flow@canon
/plugin install plain-chinese@canon
/plugin install lockstep-vue@canon      # 按需，裝你專案用得上的框架
/reload-plugins
```

命令名變化：`/canon:vue` → `/lockstep-vue:vue`；`/canon:go` → `/flow:go`；`/canon:cm` → `/flow:cm`。`pg-schema` 和 `pg-migrate` 合併成了 `lockstep-postgres`（一個 `postgres` skill）。

### 從更舊的 `lockstep-run`

這個 repo 更早叫 `claude-skills`、marketplace 叫 `lockstep-run`。先 `/plugin uninstall lockstep-run@lockstep-run` + `/plugin marketplace remove lockstep-run`，再按上面註冊 `canon` 安裝。

## How it works

每個 `lockstep-*` 的 skill 是一份 markdown 規則文件，包含：核心原則、禁用項 + 簡短原因、舊 API → 新 API 對照表、表達不出來時報告而非繞過的 STOP 訊號、場景化規則、寫完自查的 grep 清單。Claude 在處理匹配 `paths` 的檔案時才把對應 SKILL.md 載入 context，按需載入而非長期佔用。

`plain-chinese` 的 output-style 把禁用詞對照表、AI 口頭禪清單、專業術語白名單、自檢清單加進系統提示，每輪回覆都生效。

## Development

本地載入（開發/除錯用，不需安裝）。每個插件是獨立目錄，按需 `--plugin-dir` 多個：

```bash
cd <你的專案>
claude --plugin-dir ~/Cores/Projects/canon/plugins/lockstep-vue \
       --plugin-dir ~/Cores/Projects/canon/plugins/flow \
       --plugin-dir ~/Cores/Projects/canon/plugins/plain-chinese
```

改完 SKILL.md 在當前會話裡立即生效；改了 output-style、hooks 等其他元件，要在已執行的 Claude Code 內執行 `/reload-plugins`。校驗結構：`claude plugin validate .`（校驗 marketplace.json）+ 對每個插件 `claude plugin validate ./plugins/<插件>`（校驗 plugin.json 和 skill frontmatter）。

> 注意：skill 的 `description` 裡不要出現 `": "`（冒號+空格），YAML 會把它當成巢狀對映、導致 frontmatter 整段解析失敗。

官方參考：[Skills](https://code.claude.com/docs/en/skills) / [Plugins](https://code.claude.com/docs/en/plugins) / [Plugin Marketplaces](https://code.claude.com/docs/en/plugin-marketplaces) / [Output styles](https://code.claude.com/docs/en/output-styles)。

## 名字由來

`canon` 取自 canonical——「按正典、規範的做法來」。程式碼按官方規範寫，中文按平實規範說。

框架系列的前綴 `lockstep`（「齊步走」）取意「跟官方版本步調一致」。這個名字還有段來歷：改名時想起我的好兄弟薛貴文（跑哥）。大學軍訓，教官一臉嚴肅讓他帶全系喊口號，他連續 N 次把「齊步走」喊成「齊步跑」，從此成為傳說中的跑哥。`lockstep` 一度要退休，最後留下來做了框架插件的前綴。

## License

MIT
