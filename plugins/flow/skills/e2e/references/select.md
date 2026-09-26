# 按改动挑测试

`/flow:e2e run` 不带参数、`/flow:pin` 第 7 步都按这里挑。核对：Playwright 1.63.0，2026-09，下面的命令组合都在官方镜像里实测过。

## 查对照表

对照表在画像的「改动 → 测试」一节。画像里还没有这一节，就先按 `profile-template.md` 补上，停下让我确认后再挑。

改动文件取未提交的全部改动：`git diff HEAD --name-only`，加上 `git status --porcelain` 里 `??` 开头的未跟踪文件（`git diff` 看不到它们）。两者都为空时取最近一个 commit 的文件（`git show --name-only --format= HEAD`），并告诉我取的是哪个 commit。

每个文件逐个查表，第一条匹配的行生效：

- spec 路径或模块目录：加入本次范围。
- `smoke`：跑 `smoke.spec.ts`（每个页面 × 角色打开一遍）。布局、导航外壳、共享组件改了，要确认的是页面都还能打开。
- `全量`：`/flow:e2e run` 停下告诉我是哪个文件命中，我同意才跑全量；`/flow:pin` 不停，照跑其余选中的，收尾提醒我。
- `无`：不选。
- 表里查不到的文件：列给我，我定跑什么，定了把这一行补进画像。

## 一条命令跑完

按路径筛选和 `--grep` 叠用取的是交集：`npx playwright test tests/dashboard --grep @critical` 只跑 dashboard 下带 `@critical` 的，别处的关键测试一条都不跑。`--grep` 匹配的标题里带 spec 的相对路径，所以把选中的路径和标签写进同一个 `--grep`，取的是并集：

```bash
<exec> npx playwright test --grep 'dashboard/orders/[^ ]*\.spec\.ts|pay/refund\.spec\.ts|@critical' --list   # 先看选中哪些
<exec> npx playwright test --grep 'dashboard/orders/[^ ]*\.spec\.ts|pay/refund\.spec\.ts|@critical' --no-deps
```

目录一律写成 `<目录>/[^ ]*\.spec\.ts`，收在 spec 文件名上。标题里也有测试名，冒烟用例的测试名就是页面 URL，只写 `dashboard/` 会把 `smoke.spec.ts` 里所有 `/dashboard/...` 页面一起选中。

## 复用登录态

本次要用的角色在 `playwright/.auth/` 下都有文件，就加 `--no-deps`，跳过 `setup` 项目的重新登录。文件不在会直接报 `ENOENT`，去掉 `--no-deps` 重跑。

登录态在服务端过期后，失败看起来和普通断言失败一样：期望的页面内容，实际是登录页。认不出来，所以带 `--no-deps` 的运行只要有失败，先跑 `<exec> npx playwright test --last-failed`，不带 `--no-deps`，它会先重新登录再只跑失败的那几条。仍然失败的才按 `diagnose.md` 查。两次运行之间不插别的运行，`--last-failed` 读的是上一次运行的记录。

## 关键测试

计划里标 `@critical` 的场景，生成时写成 `test('<场景名>', { tag: '@critical' }, async ({ page }) => { … })`。每次按改动挑测试都带上它们，改动映射漏掉的核心流程靠它们补。

## 什么时候跑全量

- `/flow:e2e run all`，或完整流程的第 4 步。
- 对照表命中 `全量`，我同意之后。
