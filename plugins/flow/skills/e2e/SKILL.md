---
name: e2e
description: 用现有账号和环境给当前项目做全量端到端测试。摸清项目 → 出计划 → 人审 → 生成 spec → 横切面检查 → 跑并修 → 出报告，不大改被测项目
disable-model-invocation: true
argument-hint: "[profile|plan|generate|run|report] [范围]"
---

给当前项目做端到端测试。目标是把现有系统认真跑一遍、没有遗留，不是搭一套测试基建。要复现并修某个界面 bug，用 `/flow:pin`。

## 三条底线

1. **不大改被测项目，也不往项目外放东西。** 允许的改动只有两处：项目根新建 `e2e/` 工作区（spec、计划、报告、依赖、官方 skill 全在里面），`.gitignore` 加一行 `/e2e/`。不动 compose、应用配置、`.env`、`package.json`，不重建任何容器和数据库。Playwright 装在一个独立容器里，挂载 `e2e/`，搭法见 `references/runner.md`。项目非改不可才能测的地方，写进报告的「阻塞」项，不自己改。
2. **只用现有账号、现有环境、现有数据。** 账号先从项目已有的账号文档 / seed 脚本 / 数据库里找，登一遍核实；找不到的角色在会话里问我要测试账号，不自己建、不改密码。跑前用现有工具备份数据，恢复命令写进画像；出了问题随时能还原，但还原前要问我。测试自己创建的数据自己收尾删掉。
3. **全量，不漏。** 默认范围是全部：每个页面 × 每个角色的冒烟，加清单里每一个用户操作一条场景。计划里每条带状态，收尾时「待做」必须为 0，或逐条写明为什么没做。

## 前提

- 规划 / 生成 / 修复的操作步骤按官方 skill 做：`e2e/.claude/skills/playwright-cli/references/test-generation.md` 的 §1 / §2 / §3。它由容器里 `npx playwright cli install --skills` 生成（见 runner.md），没有就先装。修复时先按 `references/diagnose.md` 读现成的失败产物，读完仍不清楚才用 §3 的 attach。
- 官方步骤假定 `playwright-cli` 装在宿主机。这里不是：**宿主机什么都不装**，官方步骤里的 `playwright-cli X` 一律执行 `<exec> npx playwright cli X`，`npx playwright X` 一律执行 `<exec> npx playwright X`，`<exec>` 是画像里定义的前缀（如 `docker exec -w /work <项目>-e2e`）。
- 一条官方文档与实测不符：seed 只有一步时 `resume` 会让测试跑完、浏览器关闭。attach 后用 `step-over` 走到最后一步，页面停在「Close context」时才探索。attach 之后每条命令带 `-s=tw-xxxx`。

## 状态与入口

状态全在项目根 `e2e/`：

```
e2e/
├── profile.md          画像，首行 Verified: <日期>
├── plan.md             计划，首行 Status: draft | approved；每条场景一行状态
├── .env                账号密码（gitignore 已整目录忽略）
├── .claude/skills/playwright-cli/   官方 skill（容器生成）
├── node_modules/       @playwright/test（容器安装）
├── tests/              生成的 spec
├── run-<日期>.json     playwright --reporter=json 原始输出
└── report-<日期>.md    报告
```

不带参数时看磁盘决定从哪继续：

| 磁盘状态 | 做 | 停 |
|---|---|---|
| 无 profile.md | 第 0 步 | 停，等我确认画像 |
| 有 profile、无 plan | 第 1 步 | 停，等我审计划 |
| plan 是 draft | 提醒我审，不往下走 | 停 |
| plan 已 approved、有场景「待做」 | 第 2 → 3 → 4 步按模块分批做，每批更新状态 | 只在「护栏」列的情况停 |
| 所有场景无「待做」 | 第 5 步 | — |
| 今天已有 report | 说明已跑过，问要不要重跑 | 停 |

带参数强制跳到某步：`profile`（对比 `git log` 自 Verified 日期以来路由 / 页面目录 / 账号文档的变动，只更新差异）、`plan [范围]`（范围如 `menu`、`smoke`、`dashboard/finance`，不写就是全量）、`generate [模块]`、`run [模块]`（平时改完代码用这个，只跑已有 spec）、`report`。

同一会话里我说「批准」，把 `Status: approved` 写进 plan.md 再往下走；换会话敲 `/e2e` 读磁盘接着来。全量是大活，一个会话做不完是正常的，状态在磁盘上不会丢。

## 0 画像

读 CLAUDE.md、compose、路由注册、页面目录、已有测试清单 / 账号文档 / seed 脚本 / 已有 spec，按 `references/profile-template.md` 填 `profile.md`。能自己查的自己查（`curl` 看入口通不通、用 CLI 把每个账号登一遍），查不到的问我。

必须停下让我确认的三项：账号表（哪些角色能登、哪些缺账号——缺的直接向我要）、备份与恢复命令、副作用清单的等级划分。

## 1 计划

按 `references/plan-template.md` 写 `plan.md`，`Status: draft`，停下等我审。

- 冒烟层直接从路由 / 页面清单生成，不探索。每个页面 × 每个角色一行。
- 操作层：清单里每个写操作一条场景，按端 / 模块分组。有已有测试清单（如 `docs/test-inventory.md`）就以它为准，没有就自己从路由和页面抽。
- 每条场景标账号、前置数据、副作用等级、清理方式。3 / 4 级默认列为跳过，原因写清。
- 只对步骤拿不准的场景用官方 planner 探索，其余从清单直接写。上百页面的项目盲探一次就是十几万 token。

## 2 生成

按官方 §2 逐条生成到 `e2e/tests/<端>/<模块>/`。生成前先 grep 已有 spec 有没有同名或同流程的，有就改不新建。文件头写 `// spec: plan.md <编号>`。登录走 `auth.setup.ts` + `storageState`，每个角色一份。按模块分批，每批生成完立刻跑这批，状态写回 plan.md。

## 3 横切面

按 `references/checks.md` 生成 `e2e/tests/smoke.spec.ts`，覆盖计划里每个页面 × 角色。不探索。

## 4 跑并修

1. 按画像里的命令备份一次。备份失败就停，不往下跑。
2. `<exec> npx playwright test [模块] --reporter=json`，输出存 `run-<日期>.json`。
3. 每条失败 `--grep` 限定这一条，按 `references/diagnose.md` 的顺序查清原因。先定性再动手：
   - 测试写错（选择器、等待、断言）→ 改测试，重跑。
   - 环境（服务没起、账号登不上、前置数据不在）→ 能在底线内解决就解决；不能就标「阻塞」写进报告，继续下一条。
   - 应用 bug → 不改应用代码。记下复现路径、期望、实际，测试标 `test.fixme('<原因>')`，继续下一条。
   - 分不清是应用改了还是回归 → 按官方 §3.4 停下问我。
4. 同一测试修 3 次没过 → 标 `fixme` 附注释，记进报告，继续。
5. 每批跑完把每条场景的结果写回 plan.md 的状态列。
6. 测试创建的数据在 `afterEach` / `afterAll` 里删掉；删不掉的写进报告「遗留数据」。数据被弄乱需要还原时，先问我再跑恢复命令。

## 5 报告

按 `references/report-template.md` 写 `report-<日期>.md`。计划总数、通过 / 失败 / 跳过 / 阻塞各多少、「待做」为 0 的核对、应用 bug 列表、阻塞项、遗留数据、下一步建议。报告是给没看过对话的人的，每条 bug 要有复现路径。

## 护栏

- **副作用四级**：1 只读 / 2 库内可逆（自己建自己删）/ 3 库内不可逆 / 4 出库外部（支付、邮件、推送、打印、第三方 API）。1 / 2 自动跑；3 / 4 每次运行前逐条问我批不批，没批就跳过并写明。
- **不改应用代码**。e2e 流程发现的 bug 只进报告；要修用 `/flow:pin`。
- **不 mock 外部网关**。做不到就标跳过，不假装测过。
- 只用 Playwright 自带 chromium。
- 密码放 `e2e/.env`，spec 里用 `process.env.E2E_*`，不写明文进 spec。
- 全部测试串行（`workers: 1`），避免写操作互相踩数据。
- 不用 `networkidle`、不加 `sleep`、不跳过 hook 来让测试变绿。

## 收尾

只说三件事：跑了什么、结果如何（计划 N 条：通过 / 失败 / 跳过 / 阻塞 / 待做）、有什么要我拍板。报告路径给出来。
