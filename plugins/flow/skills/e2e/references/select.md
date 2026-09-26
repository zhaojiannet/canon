# 按改动挑测试

`/flow:e2e run`（后面不跟模块）和 `/flow:pin` 第 7 步都按这里挑。核对：Playwright 1.63.0，2026-09，下面的命令组合都在官方镜像里实测过。

## 1 取改动文件

取未提交的全部改动：`git diff HEAD --name-only`，加上 `git status --porcelain` 里 `??` 开头的未跟踪文件（`git diff` 看不到它们）。两者都为空时，取最近一个 commit 的文件（`git show --name-only --format= HEAD`），并告诉我取的是哪个 commit。

## 2 查对照表

对照表在画像的「改动 → 测试」一节。每个改动文件逐个查表，第一条匹配的行生效：

| 这一行写的是 | 做 |
|---|---|
| spec 路径或模块目录 | 加入本次范围 |
| `smoke` | 把 `smoke.spec.ts` 加入本次范围（每个页面 × 角色打开一遍） |
| `无` | 不选 |
| `全量` | `/flow:e2e run`：停下，告诉我是哪个文件命中，我同意才跑全量。`/flow:pin`：照跑其余选中的，收尾提醒我 |
| 表里查不到这个文件 | `/flow:e2e run`：停下列给我，我定跑什么，定了把这一行补进画像。`/flow:pin`：跑这个文件所在模块的目录，收尾列给我 |

画像里还没有这一节时：`/flow:e2e run` 先按 `profile-template.md` 补上，停下让我确认后再挑；`/flow:pin` 跑改动所在模块的目录，收尾提醒我补表。

## 3 一条命令跑完

选中的 spec 和全部 `@critical` 写进同一个 `--grep`，取的是并集：

```bash
<exec> npx playwright test --grep 'dashboard/orders/[^ ]*\.spec\.ts|pay/refund\.spec\.ts|@critical' --list   # 先看选中哪些
<exec> npx playwright test --grep 'dashboard/orders/[^ ]*\.spec\.ts|pay/refund\.spec\.ts|@critical' --no-deps
```

- `--grep` 匹配的标题里带 spec 的相对路径，所以路径能写进 `--grep`。按路径筛选再加 `--grep` 取的是交集：`npx playwright test tests/dashboard --grep @critical` 只跑 dashboard 下带 `@critical` 的，别处的关键测试一条都不跑。
- 目录一律写成 `<目录>/[^ ]*\.spec\.ts`，收在 spec 文件名上。标题里还有测试名，冒烟用例的测试名就是页面 URL，只写 `dashboard/` 会把 `smoke.spec.ts` 里所有 `/dashboard/...` 页面一起选中。

## 4 复用登录态

1. 本次要用的角色在 `playwright/.auth/` 下都有登录态文件时，加 `--no-deps`，跳过 `setup` 项目的重新登录。文件不在会报 `ENOENT`，去掉 `--no-deps` 重跑。
2. 带 `--no-deps` 的运行只要有失败，紧接着跑 `<exec> npx playwright test --last-failed`，不带 `--no-deps`：它先重新登录，再只跑失败的那几条。登录态在服务端过期时，失败看起来和普通断言失败一样（期望的页面内容，实际是登录页），靠这一步把它们筛掉。
3. 这次仍然失败的，才按 `diagnose.md` 查。

两次运行之间不插别的运行：`--last-failed` 读的是上一次运行的记录。

## 关键测试

计划里标 `@critical` 的场景，生成时写成 `test('<场景名>', { tag: '@critical' }, async ({ page }) => { … })`。每次按改动挑测试都带上它们，对照表漏掉的核心流程由它们补上。

## 什么时候跑全量

- `/flow:e2e run all`，或完整流程的第 4 步。
- 对照表命中 `全量`，我同意之后。
