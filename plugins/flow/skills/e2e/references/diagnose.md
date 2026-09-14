# 定位失败

`/flow:e2e` 第 4 步和 `/flow:pin` 共用。按顺序查，前一步能说清原因就不往下走。`<exec>` 是画像里的执行前缀。

1. 终端输出里 `Error Context:` 指向的 `test-results/<用例>/error-context.md`：错误信息、失败时的页面结构快照、出错处源码。
2. 同目录的 `trace.zip`，用命令行读，不开浏览器：`<exec> npx playwright trace open <trace.zip>` → `trace actions --errors-only` → `trace action <id>` / `trace snapshot <id>` → `trace requests --failed` → `trace console --errors-only`，查完 `trace close`。
3. 还不清楚：spec 里用 `test.step` 分段、用 `testInfo.attach` 挂上关键响应和 `page.consoleMessages()`，重跑同一条。时序类手段见 `timing.md`。
4. 最后才 `--debug=cli` 暂停、attach 看现场（官方 playwright-cli skill 的 `test-generation.md` §3，实测行为见 `runner.md`）。attach 后每看一次页面都要把快照读进上下文，所以放最后。
