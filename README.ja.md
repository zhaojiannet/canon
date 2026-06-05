# canon

> Claude Code に規範を守らせる。2 つの plugin：`canon` はコードの書き方を管理し、フレームワーク公式の最新パターンに強制的に従わせます。`canon-chinese` は中国語の表現を管理し、平易な口語を強制して、インターネット隠語や AI 口調を厳禁します。

**Languages**: [简体中文](README.md) · [繁體中文](README.zh-Hant.md) · [English](README.en.md) · **日本語**

> メンテナンス：[zhaoJian](https://www.zhaojian.net) · リポジトリ：https://github.com/zhaojiannet/canon

## これは何か

`canon`（canonical「正典・規範のやり方に従う」から）という名前の Claude Code プラグインマーケットプレイスです。下記の 2 つの独立した plugin があり、それぞれ別々にインストールできます：

| Plugin | 何を管理するか | どう効くか |
|---|---|---|
| **`canon`** | コードの書き方：10 個の framework skill で、Vue 3.5 / Nuxt UI v4 / Tailwind v4 / TypeScript / Go Echo v5 + sqlc / Node Fastify v5 / PostgreSQL / Astro をそれぞれの公式最新安定版の推奨パターンに強制し、deprecated な書き方を禁止。さらに手動ワークフローコマンド `go`/`cm` を 2 つ同梱 | framework skill は対応するファイルを編集すると `paths` に従って自動的に有効化。`go`/`cm` は `/canon:go`、`/canon:cm` で手動呼び出し |
| **`canon-chinese`** | 中国語の表現：平易な中国語を強制し、インターネット隠語・職場隠語・AI 口調を禁止しつつ、本物の専門用語は保持 | 一つの output-style で、有効にすると常に効く |

両者は同じことです——Claude にルールを課し、妥協させない。一方はコードの中の deprecated な書き方を、もう一方は中国語の中の隠語を片づけます。

---

## canon（コード規範）

### 何を解決するか

コードを書いていると繰り返しこういう状況に出会います。Tailwind、Nuxt UI、Vue、TypeScript、Echo、Fastify、PostgreSQL は公式が一番直接的な書き方を提示しているのに、AI は長い会話のあと忘れて、独自 CSS を書き、公式 API を迂回し、一世代前の構文を書き始めます。気づくたびに手で直すしかありません。CLAUDE.md やグローバルメモリのようなソフトな制約は、長い会話のあとでは効きません。

`canon` は Claude Code 公式の [skill 機構](https://code.claude.com/docs/en/skills) でこれを解決します。各 skill はファイルタイプ `paths` で自動的に有効化される markdown のルール文書です。`.vue` を編集すれば Vue / Nuxt UI のルールが、`.css` を編集すれば Tailwind のルールが、`.go` を編集すれば Echo / sqlc のルールが、`migrations/*.sql` を編集すれば PostgreSQL migration 安全のルールが context に入ります。ルールはその場で読んで使うので、CLAUDE.md のように長い会話の途中で薄れることがありません。

### インストール

```bash
# 1. marketplace を登録
/plugin marketplace add zhaojiannet/canon

# 2. コード規範 plugin をインストール（10 個の framework skill + go/cm ワークフローコマンドを全部含む）
/plugin install canon@canon

# 3. 再読み込みして有効化
/reload-plugins
```

確認：`/plugin` を入力して **Installed** タブを開くと `canon` が見えます。さらに `What skills are available?` を Claude に聞けば skill 一覧が出ます。

### 含まれる skill

| Skill | トリガー paths | 機能 | 手動呼び出し |
|---|---|---|---|
| `vue` | `**/*.vue` | Vue 3.5+ SFC：`<script setup>` + 型ベース `defineProps`/`defineEmits` + `defineModel` + `useTemplateRef` を強制。Options API / mixins を禁止 | `/canon:vue` |
| `nuxt-ui` | `**/*.vue` | Nuxt UI v4 コンポーネント（U 接頭辞）の使用を強制し、生の `<button>` / `<input>` / `<dialog>` の手書きを禁止 | `/canon:nuxt-ui` |
| `tailwind` | `.vue/.html/.tsx/.css/.scss` | Tailwind v4 utility-first：`<style scoped>` / 旧 utility / 任意値変数を禁止。`oklch()` / `(--xxx)` 括弧構文 / `@custom-variant dark` を強制 | `/canon:tailwind` |
| `typescript` | `**/*.ts, .tsx` | TypeScript strict：`any` / `@ts-ignore` / namespace / 非 const enum を禁止。strict tsconfig + `unknown` で any を代替することを強制 | `/canon:typescript` |
| `echo` | `**/*.go` | Echo v5 エラー処理：error の冒泡、`echo.NewHTTPError` でエラーを統一、集中型 `HTTPErrorHandler`、`errors.Is`/`errors.As`、`%w` wrap | `/canon:echo` |
| `sqlc` | `**/queries/*.sql, sqlc.yaml` | sqlc v2 + pgx/v5 + 命名規約 + 手書き SQL 呼び出しの禁止 | `/canon:sqlc` |
| `fastify` | fastify を import する `.ts/.js` | Fastify v5 plugin の async 書法、`fastify-plugin` (fp) を使うべきタイミング、JSON schema 検証、encapsulation、graceful onClose | `/canon:fastify` |
| `pg-schema` | `**/migrations/*.sql, **/schema/*.sql` | snake_case + BIGSERIAL/UUID PK + timestamptz + 外部キー ON DELETE の明示 + jsonb は schemaless にのみ使用 | `/canon:pg-schema` |
| `pg-migrate` | `**/migrations/*.sql` | トランザクション包囲 + IF EXISTS ガード + 裸の DROP/TRUNCATE 禁止 + NOT NULL 追加は 3 ステップ + `CREATE INDEX CONCURRENTLY` | `/canon:pg-migrate` |
| `astro` | `**/*.astro` | Astro 静的優先：デフォルト zero JS、`client:visible`/`client:idle` を `client:load` より優先、`server:defer` で全ページ SSR を代替、Content Collections で `Astro.glob` を代替 | `/canon:astro` |

有効化の方法：

- **自動**：`paths` にマッチするファイルを編集すると Claude Code が対応する skill を自動的に context に読み込みます
- **手動**：`/canon:<skill-name>`、例：`/canon:echo`

### ワークフローコマンド（手動トリガー）

ファイルタイプで自動有効化される 10 個の framework skill とは別に、canon は手動ワークフローコマンドを 2 つ同梱しています。これらは `disable-model-invocation: true` を設定しており、`/` を入力したときだけ呼び出され、Claude が自動トリガーすることはなく、description も context に入りません——`/` メニューのラベルでしかありません。

| コマンド | 機能 | 呼び出し |
|---|---|---|
| `go` | タスク全体を貫く作業規律：着手前に公式の最新ドキュメントを確認し、その場しのぎより最善の方法を選ぶ。何かが壊れたときはごまかしたり迂回したりしない。自分のミスは自分で引き受ける。作業が未完了なら、成功を装わず正直に報告する | `/canon:go` |
| `cm` | 現在の変更をグループ分け・分類してコミット：トピックごとにまとめ、関心事 1 つにつき 1 コミット、各メッセージを規約どおりに書き、コミット前にプランを提示し、同意なしに push しない | `/canon:cm` |

> この 2 つは作者個人のワークフローコマンドです。`go` は誰でも使える一般的な規律です。`cm` のコミットメッセージ書式は作者のグローバルな `~/.claude/CLAUDE.md` のルールを参照しているので、自分のコミット規約に差し替えてください。

---

## canon-chinese（平易な中国語）

### 何をするか

Claude Code に平易な中国語を強制し、インターネット隠語と職場隠語（根因、二开、兜底、对齐、抓手、闭环、赋能、沉淀、链路、落地……）を禁止しつつ、本物の専門用語（解耦、幂等、并发、复用……）は保持して、行き過ぎないようにします。さらに AI 特有の作り口調や口癖（底层逻辑、本质上、「不是 X 而是 Y」というマウント構文、「综上所述」というまとめ口調……）も専門に管理します。

中核は一つの [output-style](https://code.claude.com/docs/en/output-styles) です。output-style は Claude Code のシステムプロンプトを直接書き換えるため、CLAUDE.md（ユーザーメッセージ層）より制約力が強く、現時点で Claude の言い回しを安定して左右できる唯一の公式手段です。

はっきり言っておくべきこと：**100% 根絶できる方法は存在しません**。モデルは確率的にテキストを生成するもので、プロンプトによる制約は「強い傾向」であって、ハードなフィルターではありません。この plugin は制約力が最も強いソフトな手段を使っており、この種の語をほぼ絶滅させられますが、たまに 1〜2 個漏れることはあり得ます。

### インストール

```bash
# marketplace を登録済みならこのステップは飛ばす
/plugin marketplace add zhaojiannet/canon

# 平易な中国語 plugin をインストール
/plugin install canon-chinese@canon
/reload-plugins
```

### 有効化

この plugin の output-style は `force-for-plugin: true` を設定しているので、**インストールして有効にすれば自動的に効き、手動で選ぶ必要はありません**。現在の output-style 設定を上書きします。

> output-style はシステムプロンプトの一部で、Claude Code は会話を開始するたびに一度読み込みます。変更後は `/clear` するか新しい会話を始めないと反映されません。

一時的に切りたいとき：`/plugin` の中で `canon-chinese` を無効化します。自分で output-style を手動管理したいとき（自動強制ではなく）：`plugins/chinese/output-styles/plain-chinese.md` の中の `force-for-plugin: true` を削除し、`/config` → Output style で手動選択に切り替えます。

> 注意：旧来の `/output-style` コマンドは Claude Code v2.1.73 で deprecated、v2.1.91 で削除され、現在は `/config` に統一されています。

---

## 旧版（lockstep-run）からの移行

この repo は以前 `claude-skills`、marketplace は `lockstep-run` という名前でした。`canon` にアップグレードするには：

```bash
# 1. 旧 plugin をアンインストール
/plugin uninstall lockstep-run@lockstep-run

# 2. 旧 marketplace を削除
/plugin marketplace remove lockstep-run

# 3. 新 marketplace を登録してインストール
/plugin marketplace add zhaojiannet/canon
/plugin install canon@canon
/plugin install canon-chinese@canon
/reload-plugins
```

skill の呼び出し名も短くなりました：`/lockstep-run:tailwind-utility-first` → `/canon:tailwind`。

## How it works

`canon` の各 skill は markdown のルール文書で、次を含みます：核心原則、禁止項目 + 簡潔な理由、旧 API → 新 API の対応表、表現できないときは迂回せず報告する STOP シグナル、シナリオ別ルール、書き終えたあとの自己チェック用 grep リスト。Claude Code は `paths` にマッチするファイルを編集するときに該当する SKILL.md を context に読み込みます。常時占有するのではなく、必要なときに読み込みます。

`canon-chinese` の output-style は、禁止語の対応表、AI 口癖リスト、専門用語のホワイトリスト、セルフチェックリストをシステムプロンプトに追加し、毎ターンの返信で効きます。

## Development

ローカル読み込み（開発・デバッグ用、インストール不要）：

```bash
cd <あなたのプロジェクト>
claude --plugin-dir ~/Cores/Projects/canon/plugins/code \
       --plugin-dir ~/Cores/Projects/canon/plugins/chinese
```

SKILL.md や output-style を編集したら、起動中の Claude Code で `/reload-plugins` を実行すれば反映されます。構造を検証するには：`claude plugin validate .`。

公式リファレンス：[Skills](https://code.claude.com/docs/en/skills) / [Plugins](https://code.claude.com/docs/en/plugins) / [Plugin Marketplaces](https://code.claude.com/docs/en/plugin-marketplaces) / [Output styles](https://code.claude.com/docs/en/output-styles)。

## 名前の由来

`canon` は canonical——「正典・規範のやり方に従う」から取りました。2 つの plugin は同じことです：コードは公式の規範どおりに書き、中国語は平易な規範どおりに話す。

このプロジェクトの前身のコードネームは `lockstep`（「歩調を揃えて進め」）でした。改名するとき、私の親友、薛貴文（通称：跑哥／パオゴー）を思い出しました。大学の軍事訓練で、教官が真顔で彼に系全員の号令をかけさせたとき、彼は「齐步走（歩調をそろえて進め）」と言うべきところを N 回連続で「齐步跑（歩調をそろえて走れ）」と叫び、その日から伝説の跑哥（走る兄貴）になりました。

## License

MIT
</content>
</invoke>
