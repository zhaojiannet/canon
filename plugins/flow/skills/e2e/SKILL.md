---
name: e2e
description: 按项目画像做端到端测试。摸清项目 → 出计划 → 人审 → 生成 spec → 横切面检查 → 跑并修 → 出报告
disable-model-invocation: true
argument-hint: "[profile|plan|generate|run|report] [范围]"
---

给当前项目做端到端测试。流程每个项目一样，项目事实（入口、账号、副作用、数据恢复、业务口径）写在画像里，每个项目不同。

## 前提

- 规划 / 生成 / 修复的操作步骤按官方 skill 做：`~/.claude/skills/playwright-cli/references/test-generation.md` 的 §1 / §2 / §3。没有这个目录就停下告诉我。
- 官方步骤假定 `playwright-cli` 装在宿主机。这里不是：**宿主机什么都不装**，凡官方步骤里的 `playwright-cli X` 一律执行 `<exec> npx playwright cli X`，`npx playwright X` 一律执行 `<exec> npx playwright X`，`<exec>` 是画像里定义的前缀（如 `docker compose exec e2e`）。运行环境的搭法见 `references/runner.md`。
- 一条官方文档与实测不符的地方：seed 只有一步时 `resume` 会让测试跑完、浏览器关闭。attach 后用 `step-over` 走到最后一步，页面停在「Close context」时才探索。attach 之后每条命令带 `-s=tw-xxxx`。

## 状态与入口

状态全在项目的 `.scratch/e2e/`（必须在 `.gitignore` 里，不在就先加）：

```
profile.md          画像，首行 Verified: <日期>
plan.md             计划，首行 Status: draft | approved
run-<日期>.json     playwright --reporter=json 原始输出
report-<日期>.md    报告
```

不带参数时看磁盘决定从哪继续：

| 磁盘状态 | 做 | 停 |
|---|---|---|
| 无 profile.md | 第 0 步 | 停，等我确认画像 |
| 有 profile、无 plan | 第 1 步 | 停，等我审计划 |
| plan 是 draft | 提醒我审，不往下走 | 停 |
| plan 已 approved、spec 未生成 | 第 2 → 3 → 4 → 5 步一口气做 | 只在「护栏」列的情况停 |
| spec 已有 | 第 4 → 5 步 | 同上 |
| 今天已有 report | 说明已跑过，问要不要重跑 | 停 |

带参数强制跳到某步：`profile`（对比 `git log` 自 Verified 日期以来路由 / 页面目录 / compose / 账号文档的变动，只更新差异）、`plan [范围]`（范围如 `menu`、`smoke`、`risk:1-3`）、`generate`、`run`（平时改完代码用这个）、`report`（从最近一份 run json 重出）。

同一会话里我说「批准」，把 `Status: approved` 写进 plan.md 再往下走；换会话敲 `/e2e` 读磁盘接着来。

## 0 画像

读 CLAUDE.md、compose、路由注册、页面目录、Taskfile / Makefile、已有测试账号文档、已有 spec，按 `references/profile-template.md` 填 `profile.md`。能自己查的自己查（跑 `curl` 看入口通不通、`psql` 核账号能不能登），只把查不到的问我。

必须停下让我确认的两项：数据恢复手段（没有就整份标 `只读`，不生成任何写操作测试）、副作用清单的等级划分。

Nuxt / Vite dev 项目要核一条：`vite.server.allowedHosts` 是否放行 `.local`，没放行则容器经 OrbStack 域名访问会 403。这是应用配置改动，列出来等我批准。

## 1 计划

按 `references/plan-template.md` 写 `plan.md`，`Status: draft`，停下等我审。

- 冒烟层直接从路由 / 页面清单生成，不探索。每个页面 × 每个角色一行，只做 `references/checks.md` 里的检查。
- 风险路径才用官方 planner 探索（带着 profile 一起读）。范围默认取画像里「范围」一项；没写就取风险路径前 3 条。
- 每条场景标账号、前置数据、副作用等级、跳过原因。3 / 4 级默认列为跳过。
- 不要盲探全部页面。上百个页面的项目，探索式规划一次就是十几万 token。

## 2 生成

按官方 §2 逐条生成到画像里定义的 spec 目录。生成前先 grep 已有 spec 有没有同名或同流程的，有就改不新建。文件头写 `// spec: .scratch/e2e/plan.md <编号>` 和 `// seed:`。登录走 `auth.setup.ts` + `storageState`，每个角色一份，见 runner.md。

## 3 横切面

按 `references/checks.md` 生成 `smoke.spec.ts`。这一层不需要探索，覆盖计划里每个页面。

## 4 跑并修

1. 跑前按画像的命令做数据快照。
2. `<exec> npx playwright test --reporter=json`，输出存 `run-<日期>.json`。
3. 每条失败按官方 §3 单独处理：`--grep` 限定这一条、`--debug=cli` 暂停、attach 看现场。先定性再动手：
   - 测试写错（选择器、等待、断言）→ 改测试，重跑。
   - 环境（数据没种、容器没起、账号密码不对）→ 修环境，重跑。同一处两次修不好 → 停，问我。
   - 应用 bug → 不改应用代码。记下复现路径、期望、实际，测试标 `test.fixme('<原因>')`，继续下一条。
   - 分不清是应用改了还是回归 → 按官方 §3.4 停下问我。
4. 同一测试修 3 次没过 → 标 `fixme` 附注释，记进报告，继续。
5. 跑后按画像的命令恢复数据。快照 / 恢复失败要如实写进报告，不能当没发生。

## 5 报告

按 `references/report-template.md` 写 `report-<日期>.md`。覆盖了什么、跳过了什么及原因、应用 bug 列表、环境问题、下一步建议。报告是给没看过对话的人的，每条 bug 要有复现路径。

## 护栏

- **副作用四级**：1 只读 / 2 库内可逆 / 3 库内不可逆 / 4 出库外部（支付、邮件、推送、打印、第三方 API）。1 / 2 自动跑；3 / 4 每次运行前逐条问我批不批，没批就跳过并写明。没有数据恢复手段的项目只跑 1。
- **不改应用代码**。e2e 流程发现的 bug 只进报告。
- **不 mock 外部网关**。做不到就标跳过，不假装测过。
- 只用 Playwright 自带 chromium。不接宿主机浏览器。
- 生成 / 修复用到的密码从 `.env` 或画像指向的文件读，spec 里用 `process.env.E2E_*`，不写明文。
- 冒烟和只读测试可并行；写操作测试串行，或按官方 auth 文档「每个 worker 一个账号」。
- 不用 `networkidle`、不加 `sleep`、不跳过 hook 来让测试变绿。

## 收尾

只说三件事：跑了什么、结果如何（通过 / 失败 / 跳过各多少）、有什么要我拍板。报告路径给出来。
