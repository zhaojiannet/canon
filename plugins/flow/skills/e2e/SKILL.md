---
name: e2e
description: 用现有账号和环境给当前项目做全量端到端测试。摸清项目 → 出计划 → 人审 → 生成 spec → 横切面检查 → 跑并修 → 出报告，被测项目只加 e2e/ 目录和一行 .gitignore
disable-model-invocation: true
argument-hint: "[profile|plan|generate|run|report] [范围]"
---

给当前项目做端到端测试：把现有系统完整跑一遍、不留遗漏。要复现并修某一个界面 bug，用 `/flow:pin`。

## 三条底线

1. **被测项目只动两处**：项目根新建 `e2e/` 工作区（spec、计划、报告、依赖、官方 skill 全在里面），`.gitignore` 加一行 `/e2e/`。compose、应用配置、`.env`、`package.json` 保持原样，容器和数据库不重建，项目目录外不放文件。唯一例外是备份：用项目现有的备份工具，写在它默认的位置。Playwright 装在一个独立容器里，挂载 `e2e/`，搭法见 `references/runner.md`。必须改项目才能测的地方，写进报告的「阻塞」项。
2. **只用现有账号、现有环境、现有数据。** 账号从项目已有的账号文档、seed 脚本、数据库里找，每个都登一遍核实；找不到的角色在会话里问我要测试账号，账号和密码保持原样。跑测试前用现有工具备份，恢复命令写进画像；要还原数据时先问我。测试自己建的数据，测试自己删。
3. **全量覆盖。** 默认范围是全部：每个页面 × 每个角色的冒烟，加清单里每一个用户操作一条场景。计划里每条场景带状态，收尾时状态为「待做」或「已生成」的场景为 0，否则逐条写明为什么没做。

## 前提

- 规划、生成、修复的操作步骤按官方 skill 做：`e2e/.claude/skills/playwright-cli/references/test-generation.md` 的 §1 / §2 / §3。这个文件不在时，按 `references/runner.md` 在容器里跑 `npx playwright cli install --skills` 生成。修复时定位失败的顺序见 `references/diagnose.md`。
- 所有 Playwright 命令都在 e2e 容器里跑。官方步骤默认 `playwright-cli` 装在宿主机，照下面改写：`playwright-cli X` 执行 `<exec> npx playwright cli X`，`npx playwright X` 执行 `<exec> npx playwright X`。`<exec>` 是画像里定义的前缀（如 `docker exec -w /work <项目>-e2e`）。
- seed 与 attach 按 `references/runner.md`「seed 与 attach 的实测行为」做，那里记了和官方文档不符的地方。

## 状态与入口

状态全在项目根 `e2e/`：

```
e2e/
├── profile.md                画像，首行 Verified: <日期>
├── plan.md                   计划，首行 Status: draft | approved；每条场景一行状态
├── .env                      账号密码（gitignore 已整目录忽略）
├── .claude/skills/playwright-cli/   官方 skill（容器生成）
├── node_modules/             @playwright/test（容器安装）
├── tests/                    生成的 spec
├── run-<日期>-<模块>.json     每批的 playwright --reporter=json 原始输出
└── report-<日期>.md          报告
```

`/flow:e2e` 不带参数时读磁盘决定从哪继续。从上往下，第一条符合的行生效：

| 磁盘状态 | 做 | 停 |
|---|---|---|
| 无 profile.md | 第 0 步 | 停，等我确认画像 |
| 有 profile、无 plan | 第 1 步 | 停，等我审计划 |
| plan 是 draft | 提醒我审计划 | 停 |
| 今天已有 report | 说明今天已跑过，问要不要重跑 | 停 |
| 有场景「待做」或「已生成」 | 第 2 → 3 → 4 步，按模块分批做，每批更新状态 | 见「护栏」里的停点 |
| 其余 | 第 5 步 | — |

带参数时直接跳到对应步骤：

- `profile`：对比 `git log` 自 Verified 日期以来路由、页面目录、账号文档的变动，只更新差异。
- `plan [范围]`：范围如 `menu`、`smoke`、`dashboard/finance`，省略就是全量。
- `generate [模块]`
- `run`（后面不跟模块）：改完代码平时用这个。按 `references/select.md` 查画像里的对照表，跑改动相关的已有 spec 加全部 `@critical`。
- `run <模块>`：跑该模块的已有 spec。
- `run all`：跑全部已有 spec。
- `report`

同一会话里我说「批准」，就把 `Status: approved` 写进 plan.md 再往下走。一个会话做不完全量时，把已完成批次的状态写回 plan.md 再停下；换会话后我敲 `/flow:e2e`，你读磁盘接着做。

## 0 画像

读 CLAUDE.md、compose、路由注册、页面目录、已有测试清单、账号文档、seed 脚本、已有 spec，按 `references/profile-template.md` 填 `profile.md`。能自己查的自己查：`curl` 看入口通不通，用 CLI 把每个账号登一遍；查不到的问我。

停下让我确认四项：账号表（哪些角色能登、哪些缺账号，缺的直接向我要）、备份与恢复命令、副作用等级划分、「改动 → 测试」对照表。

## 1 计划

按 `references/plan-template.md` 写 `plan.md`，`Status: draft`，停下等我审。

- 冒烟层从路由和页面清单生成，每个页面 × 每个角色一行。
- 操作层：清单里每个写操作一条场景，按端和模块分组。项目有测试清单（如 `docs/test-inventory.md`）就以它为准，没有就从路由和页面里抽。
- 每条场景标账号、前置数据、副作用等级、清理方式。3 / 4 级按「护栏」处理。
- 按 `references/plan-template.md` 的规则标 `@critical`。
- 操作层默认从清单直接写步骤。清单和路由代码里读不出要点哪个元素、会跳到哪、出现什么结果的场景，才用官方 planner 探索。上百页面的项目盲探一次就是十几万 token。

## 2 生成

按官方 §2 逐条生成到 `e2e/tests/<端>/<模块>/`：

1. 生成前 grep `e2e/tests/`：已有同名 spec，或同一页面同一操作的 spec，就在它上面改。
2. 文件头写 `// spec: plan.md <编号>`；计划里标了 `@critical` 的场景写 `{ tag: '@critical' }`。
3. 登录走 `auth.setup.ts` + `storageState`，每个角色一份。
4. 会建数据的场景，在 `afterEach` / `afterAll` 里删掉自己建的数据。
5. 按模块分批：一批生成完，就按第 4 步跑这一批。

## 3 横切面

按 `references/checks.md` 生成 `e2e/tests/smoke.spec.ts`，覆盖计划里每个页面 × 角色。

## 4 跑并修

1. 本次调用第一次跑测试前，按画像里的命令备份一次；同一次调用里后面的批次不再备份。备份失败就停下报告，不跑测试。
2. `<exec> sh -c 'PLAYWRIGHT_JSON_OUTPUT_NAME=run-<日期>-<模块>.json npx playwright test <模块> --reporter=json'`。本次要用的角色在 `playwright/.auth/` 下都有登录态文件时加 `--no-deps`。带 `--no-deps` 的运行有失败，先按 `references/select.md`「复用登录态」重跑失败项，仍失败的才进下一步。
3. 每条失败用 `--grep` 单跑这一条，按 `references/diagnose.md` 的顺序查清原因，先定性再动手：
   - 测试写错（选择器、等待、断言）→ 改测试，重跑。
   - 环境问题（服务没起、账号登不上、前置数据不在）→ 在三条底线内能解决就解决；解决不了就标「阻塞」写进报告，继续下一条。
   - 应用 bug → 记下复现路径、期望、实际，测试标 `test.fixme('<原因>')`，继续下一条。
   - 分不清是应用有意改了行为（该改测试）还是回归（算应用 bug）→ 按官方 §3.4 停下问我。
4. 同一测试按「测试写错」改过 3 次、每次重跑仍失败 → 标 `test.fixme('<原因>')`，plan.md 状态记「失败」、备注写「测试没修好」，报告里和应用 bug 分开单列，继续下一条。
5. 每批跑完，把每条场景的结果写回 plan.md 的状态列。
6. 删不掉的测试数据写进报告「遗留数据」。

## 5 报告

按 `references/report-template.md` 写 `report-<日期>.md`。

## 护栏

- **停点**：只在这几处停下问我：3 / 4 级场景跑之前；备份失败；分不清应用有意改了行为还是回归（官方 §3.4）；数据需要还原。
- **副作用四级**：1 只读 / 2 库内可逆（自己建自己删）/ 3 库内不可逆 / 4 出库外部（支付、邮件、推送、打印、第三方 API）。1 / 2 自动跑。3 / 4 在计划里先标跳过；跑这一批前逐条问我，批了的改成「待做」生成并跑，没批的保持跳过并写明原因。
- **应用代码保持原样**。e2e 发现的 bug 只进报告，要修用 `/flow:pin`。
- 外部网关走真实调用；只有 mock 才能测的场景标跳过，写明原因。
- 只用 Playwright 自带的 chromium。
- 密码放 `e2e/.env`，spec 里用 `process.env.E2E_*` 读。
- 全部测试串行（`workers: 1`），写操作之间不互相踩数据。
- 等待用 `references/timing.md` 里的手段。`networkidle`、`sleep`、跳过 `beforeEach` / `afterEach` / `beforeAll` / `afterAll` 都不拿来让测试变绿。

## 收尾

只说三件事：跑了什么；结果如何（计划 N 条：通过 / 失败 / 跳过 / 阻塞 / 待做 / 已生成）；有什么要我拍板。给出报告路径。
