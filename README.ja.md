# canon

> Claude Code に規範を守らせる。一つの marketplace に 11 個の plugin：`lockstep-*` シリーズがコードの書き方を管理し（各フレームワーク公式の最新安定版のパターンに固定）、`flow` がワークフローコマンドを提供し、`plain-chinese` が中国語の表現を管理します（平易な言葉、ネット隠語や AI 口調を禁止）。

**Languages**: [简体中文](README.md) · [繁體中文](README.zh-Hant.md) · [English](README.en.md) · **日本語**

> メンテナンス：[zhaoJian](https://www.zhaojian.net) · リポジトリ：https://github.com/zhaojiannet/canon

## これは何か

`canon`（canonical「正典・規範のやり方に従う」から）という名前の Claude Code プラグインマーケットプレイスです。下記の 11 個の独立した plugin が、役割別に 3 グループに分かれており、それぞれ別々にインストールできます：

| Plugin | 何を管理するか | どう効くか |
|---|---|---|
| **`lockstep-*`**（9 個のフレームワーク plugin） | コードの書き方：1 フレームワークにつき 1 plugin。Astro / Vue 3.5 / Nuxt UI v4 / Tailwind v4 / TypeScript / Echo v5 / sqlc / Fastify v5 / PostgreSQL をそれぞれの公式最新安定版の推奨パターンに強制し、deprecated な書き方を禁止 | `paths` に一致するファイルを扱うときに自動的に有効化 |
| **`flow`** | 手動ワークフローコマンド `go`/`cm`/`e2e`/`pin` の 4 つ | `/flow:go`、`/flow:cm`、`/flow:e2e`、`/flow:pin` で手動呼び出し |
| **`plain-chinese`** | 中国語の表現：平易な中国語を強制し、ネット隠語・職場隠語・AI 口調を禁止しつつ、本物の専門用語は保持 | 一つの output-style で、有効にすると常に効く |

**なぜフレームワークを 1 つにまとめず 9 個の plugin に分けたか**：plugin はインストールの最小単位で、1 つ入れるとその skill が全部ついてきて、選べません。分けることで、プロジェクトの技術スタックに合わせて個別にインストールできます。Astro サイトなら `lockstep-astro` だけ入れればよく、使わないバックエンドのフレームワークは 1 token も消費しません（各 skill の description は Claude の skill 一覧予算を消費します）。

---

## lockstep-*（コード規範）

### 何を解決するか

コードを書いていると繰り返しこういう状況に出会います。Tailwind、Nuxt UI、Vue、TypeScript、Echo、Fastify、PostgreSQL は公式が一番直接的な書き方を提示しているのに、AI は長い会話のあと忘れて、独自 CSS を書き、公式 API を迂回し、一世代前の構文を書き始めます。気づくたびに手で直すしかありません。CLAUDE.md やグローバルメモリのようなソフトな制約は、長い会話のあとでは効きません。

`lockstep-*` は Claude Code 公式の [skill 機構](https://code.claude.com/docs/en/skills) でこれを解決します。各 skill は、有効化されるタイミングを `paths` でファイルタイプごとに限定した markdown のルール文書です。`.vue` ファイルを扱うときは Vue / Nuxt UI のルールが、`.css` を扱うときは Tailwind のルールが、`.go` を扱うときは Echo / sqlc のルールが、`migrations/*.sql` を扱うときは PostgreSQL の表設計と migration 安全のルールが context に入ります。ルールはマッチするファイルを扱うときだけ読み込まれ、context を常時占有しません。長い会話で自動 compaction が走ると、各 skill は先頭 5,000 token だけが残り、全 skill で 25,000 token の予算を共有するため、早い時点で呼ばれた skill は丸ごと落ちることがあります。

### インストール

```bash
# 1. marketplace を登録
/plugin marketplace add zhaojiannet/canon

# 2. プロジェクトの技術スタックに合わせて必要なフレームワーク plugin を入れる（例：Vue フロントエンド）
/plugin install lockstep-vue@canon
/plugin install lockstep-nuxt-ui@canon
/plugin install lockstep-tailwind@canon
/plugin install lockstep-typescript@canon

# 3. 再読み込みして有効化
/reload-plugins
```

**シナリオ別のよくある組み合わせ**：

| シナリオ | 入れるもの |
|---|---|
| Astro サイト | `lockstep-astro` + `lockstep-tailwind` + `lockstep-typescript` |
| Vue フロントエンド | `lockstep-vue` + `lockstep-nuxt-ui` + `lockstep-tailwind` + `lockstep-typescript` |
| Go バックエンド | `lockstep-echo` + `lockstep-sqlc` + `lockstep-postgres` |
| Node バックエンド | `lockstep-fastify` + `lockstep-typescript` + `lockstep-postgres` |
| 共通（全員に推奨） | `flow` + `plain-chinese` |

確認：`/plugin` を入力して **Installed** タブを開くと、入れた `lockstep-*` が見えます。さらに `What skills are available?` を Claude に聞けば skill 一覧が出ます。

### 含まれるフレームワーク plugin

| Plugin | トリガー paths | 機能 | 自動呼び出し名 |
|---|---|---|---|
| `lockstep-astro` | `**/*.astro` | Astro 7 静的優先：デフォルト zero JS、`client:visible`/`client:idle` を `client:load` より優先、`server:defer` で全ページ SSR を代替、Content Collections で `Astro.glob` を代替 | `/lockstep-astro:astro` |
| `lockstep-vue` | `**/*.vue` | Vue 3.5+ SFC：`<script setup>` + 型ベース `defineProps`/`defineEmits` + `defineModel` + `useTemplateRef` を強制。Options API / mixins を禁止 | `/lockstep-vue:vue` |
| `lockstep-nuxt-ui` | `**/*.vue` | Nuxt UI v4 コンポーネント（U 接頭辞）の使用を強制し、生の `<button>` / `<input>` / `<dialog>` の手書きを禁止 | `/lockstep-nuxt-ui:nuxt-ui` |
| `lockstep-tailwind` | `.vue/.astro/.html/.tsx/.jsx/.css` | Tailwind v4 utility-first：`<style>` ブロック / 旧 utility を禁止。`oklch()` / 括弧構文 / v4 構文を強制 | `/lockstep-tailwind:tailwind` |
| `lockstep-typescript` | `.ts/.tsx` | TypeScript 7 strict：strict は 7 のデフォルトで無効化しない。`unknown` + 絞り込みで `any` を代替、const オブジェクトで enum を代替、ES module で namespace を代替。7 で削除された `baseUrl` / `target es5` / `outFile` は書き換える | `/lockstep-typescript:typescript` |
| `lockstep-echo` | `**/*.go` | Echo v5 エラー処理：`HTTPError` + 集中型 `HTTPErrorHandler`、`errors.Is`/`errors.As`、`%w` wrap、graceful shutdown | `/lockstep-echo:echo` |
| `lockstep-sqlc` | `queries/*.sql, sqlc.yaml` | sqlc codegen：SQL を源とし、生成された `Querier` を通す。手書きの `database/sql` を禁止 | `/lockstep-sqlc:sqlc` |
| `lockstep-fastify` | `.ts/.tsx/.js/.mjs`（fastify を import しているかは問わない） | Fastify v5：カプセル化された plugin、`fastify-plugin` (fp) でスコープをまたぐ、JSON schema 検証で手書きを代替 | `/lockstep-fastify:fastify` |
| `lockstep-postgres` | `migrations/`、`schema/`、`migrate/`、`sqitch/` 下の `*.sql` | PostgreSQL 表設計 + migration 安全（統合）：snake_case + BIGSERIAL/UUID + timestamptz + 外部キー ON DELETE の明示；トランザクション包囲 + NOT NULL 追加は 3 ステップ + `CREATE INDEX CONCURRENTLY` + 破壊的操作は事前承認 | `/lockstep-postgres:postgres` |

有効化の方法：

- **自動**：`paths` にマッチするファイルを扱うとき、Claude が対応する skill を自動的に context に読み込みます（日常はほぼこれで、コマンドを手で打つことはまずありません）
- **手動**：`/lockstep-<フレームワーク>:<skill>`、例：`/lockstep-echo:echo`

### ワークフローコマンド：flow（手動トリガー）

`flow` plugin は手動トリガーのワークフローコマンドを 4 つ含みます。`disable-model-invocation: true` を設定しており、`/` を入力したときだけ呼び出され、Claude が自動トリガーすることはなく、description も context に入りません——`/` メニューのラベルでしかありません。

```bash
/plugin install flow@canon
/reload-plugins
```

| コマンド | 機能 | 呼び出し |
|---|---|---|
| `go` | タスク全体を貫く作業規律：着手前に公式の最新ドキュメントを確認し、その場しのぎより最善の方法を選ぶ。セキュリティに関わるときは能動的に防御し、既知の脆弱性を残さない。何かが壊れたときはごまかしたり迂回したりしない。自分のミスは正直に引き受ける。作業が未完了なら、成功を装わず正直に報告する | `/flow:go` |
| `cm` | 現在の変更をグループ分け・分類してコミット：実際の `git diff` を基準にまとめ（context 内のタスクリストは手がかりにとどめる）、既定ではまとめたまま無理に分けず、関心事 1 つにつき 1 コミット、各メッセージを規約どおりに書き、コミット前にプランを提示し、同意なしに push しない | `/flow:cm` |
| `e2e` | 既存のアカウントと環境で現在のプロジェクトを全量 E2E テスト：プロジェクトを読んでプロファイル化（入口、アカウント、副作用レベル、バックアップ／復元、業務ルール）→ 計画を書いてレビュー待ち → 公式 `playwright-cli` skill で Playwright spec を生成 → 全ページの横断スモーク → 独立コンテナで実行し、失敗は分類してから修正（アプリのバグは報告のみ、修正しない）→ レポート。計画の各シナリオに状態を持たせ、セッションをまたいで続行、終了時に「未着手」は 0 でなければならない。対象プロジェクトに増えるのは `e2e/` ディレクトリと `.gitignore` の 1 行だけ；ホストには Playwright 関連を一切インストールしない | `/flow:e2e` |
| `pin` | 目視では確認しにくい UI バグ（タイミング、断続的、複数ステップの操作）を 1 つ固定してから修正：まず Playwright spec で再現し、再現率から実行回数を決める → 手順を最小化し仮説を挙げる → `error-context.md` と trace で原因を特定 → アプリのコードを修正 → grep で見つけた関連 spec だけを実行し、全量は流さない。spec は `e2e/tests/repro/` に回帰テストとして残す。アサーションの変更、バックアップコマンドがない場合、3 回続けて直らない場合は止まって確認する。コンテナは `e2e` と同じ構成を使う | `/flow:pin` |

> この 4 つは作者個人のワークフローコマンドです。`go` は誰でも使える一般的な規律です。`cm` のコミットメッセージ書式は作者のグローバルな `~/.claude/CLAUDE.md` のルールを参照しているので、自分のコミット規約に差し替えてください。`e2e` は計画・生成・修復に公式 `playwright-cli` skill（コンテナがプロジェクトの `e2e/` 内に導入）を使います。実行環境の構築は `plugins/flow/skills/e2e/references/runner.md` を参照。`pin` はこのコンテナと `references/` の原因特定・タイミングの資料を `e2e` と共用します。

---

## plain-chinese（平易な中国語）

### 何をするか

Claude Code に平易な中国語を強制し、ネット隠語と職場隠語（根因、二开、兜底、对齐、抓手、闭环、赋能、沉淀、链路、落地……）を禁止しつつ、本物の専門用語（解耦、幂等、并发、复用……）は保持して、行き過ぎないようにします。さらに AI 特有の作り口調や口癖（底层逻辑、本质上、「不是 X 而是 Y」というマウント構文、「综上所述」というまとめ口調……）も専門に管理します。

中核は一つの [output-style](https://code.claude.com/docs/en/output-styles) です。output-style は Claude Code のシステムプロンプトを直接書き換えるため、CLAUDE.md（ユーザーメッセージ層）より制約力が強く、現時点で Claude の言い回しを安定して左右できる唯一の公式手段です。

はっきり言っておくべきこと：**100% 根絶できる方法は存在しません**。モデルは確率的にテキストを生成するもので、プロンプトによる制約は「強い傾向」であって、ハードなフィルターではありません。この plugin は制約力が最も強いソフトな手段を使っており、この種の語をほぼ絶滅させられますが、たまに 1〜2 個漏れることはあり得ます。

### インストール

```bash
# marketplace を登録済みならこのステップは飛ばす
/plugin marketplace add zhaojiannet/canon

# 平易な中国語 plugin をインストール
/plugin install plain-chinese@canon
/reload-plugins
```

### 有効化

この plugin の output-style は `force-for-plugin: true` を設定しているので、**インストールして有効にすれば自動的に効き、手動で選ぶ必要はありません**。現在の output-style 設定を上書きします。

> 会話の途中で output-style を切り替えると、次のメッセージから新しいスタイルで応答します（v2.1.251 より前は `/clear` するか新しい会話を始めるまで反映されませんでした）。plugin の output-style ファイルを編集したら、`/reload-plugins` を実行するか Claude Code を再起動します。

一時的に切りたいとき：`/plugin` の中で `plain-chinese` を無効化します。自分で output-style を手動管理したいとき（自動強制ではなく）：`plugins/plain-chinese/output-styles/plain-chinese.md` の中の `force-for-plugin: true` を削除し、`/config` → Output style で手動選択に切り替えます。

> 注意：`/output-style` コマンドは Claude Code v2.1.73 で deprecated、v2.1.91 で削除されましたが、v2.1.269 で `/output-style [name]` として再追加され、スタイルの一覧表示と切り替えができます。`/config` → Output style でも選べます。

---

## 旧版からの移行

### v0.5（`canon` / `canon-chinese` の 2 つの統合 plugin）から

v0.5 ではフレームワーク skill を全部 `canon` という 1 つの plugin にまとめていました。v0.6 では `lockstep-*` シリーズ（1 フレームワーク 1 plugin、シナリオ別に個別インストール）に分割し、ワークフローを `flow` として独立させ、中国語 plugin を `plain-chinese` に改名しました。

```bash
# 1. marketplace を更新
/plugin marketplace update canon

# 2. 古い統合 plugin をアンインストール
/plugin uninstall canon@canon
/plugin uninstall canon-chinese@canon

# 3. プロジェクトの技術スタックに合わせて新 plugin をインストール（上記「シナリオ別のよくある組み合わせ」参照）
/plugin install flow@canon
/plugin install plain-chinese@canon
/plugin install lockstep-vue@canon      # 必要に応じて、プロジェクトで使うフレームワークを
/reload-plugins
```

コマンド名の変化：`/canon:vue` → `/lockstep-vue:vue`；`/canon:go` → `/flow:go`；`/canon:cm` → `/flow:cm`。`pg-schema` と `pg-migrate` は `lockstep-postgres`（1 つの `postgres` skill）に統合されました。

### さらに古い `lockstep-run` から

この repo はさらに以前 `claude-skills`、marketplace は `lockstep-run` という名前でした。まず `/plugin uninstall lockstep-run@lockstep-run` + `/plugin marketplace remove lockstep-run` を実行し、その後上記のように `canon` を登録してインストールします。

## How it works

各 `lockstep-*` の skill は markdown のルール文書で、次を含みます：核心原則、禁止項目 + 簡潔な理由、旧 API → 新 API の対応表、表現できないときは迂回せず報告する STOP シグナル、シナリオ別ルール、書き終えたあとの自己チェック用 grep リスト。Claude は `paths` にマッチするファイルを扱うときだけ該当する SKILL.md を context に読み込みます。常時占有するのではなく、必要なときに読み込みます。

`plain-chinese` の output-style は、禁止語の対応表、AI 口癖リスト、専門用語のホワイトリスト、セルフチェックリストをシステムプロンプトに追加し、毎ターンの返信で効きます。

## Development

ローカル読み込み（開発・デバッグ用、インストール不要）。各 plugin は独立したディレクトリなので、必要な分だけ `--plugin-dir` を渡します：

```bash
cd <あなたのプロジェクト>
claude --plugin-dir ~/Cores/Projects/canon/plugins/lockstep-vue \
       --plugin-dir ~/Cores/Projects/canon/plugins/flow \
       --plugin-dir ~/Cores/Projects/canon/plugins/plain-chinese
```

SKILL.md の編集は現在のセッションにすぐ反映されます。output-style や hooks など他のコンポーネントを編集したら、起動中の Claude Code で `/reload-plugins` を実行します。構造を検証するには：`claude plugin validate .`（marketplace.json を検証）+ 各 plugin に対して `claude plugin validate ./plugins/<plugin>`（plugin.json と skill frontmatter を検証）。

> 注意：skill の `description` に `": "`（コロン + 空白）を入れないでください。YAML がネストしたマッピングとして解釈し、frontmatter 全体のパースが失敗します。

公式リファレンス：[Skills](https://code.claude.com/docs/en/skills) / [Plugins](https://code.claude.com/docs/en/plugins) / [Plugin Marketplaces](https://code.claude.com/docs/en/plugin-marketplaces) / [Output styles](https://code.claude.com/docs/en/output-styles)。

## 名前の由来

`canon` は canonical——「正典・規範のやり方に従う」から取りました。コードは公式の規範どおりに書き、中国語は平易な規範どおりに話す。

フレームワークシリーズの接頭辞 `lockstep`（「歩調を揃えて進め」）は「公式バージョンと歩調を合わせる」という意味です。この名前には一つの来歴があります。改名するとき、私の親友、薛貴文（通称：跑哥／パオゴー）を思い出しました。大学の軍事訓練で、教官が真顔で彼に系全員の号令をかけさせたとき、彼は「齐步走（歩調をそろえて進め）」と言うべきところを N 回連続で「齐步跑（歩調をそろえて走れ）」と叫び、その日から伝説の跑哥（走る兄貴）になりました。`lockstep` は一度引退しかけましたが、最後はフレームワーク plugin の接頭辞として残りました。

## License

MIT
