# 时序问题的复现手段

核对：Playwright 1.63.0 文档源文件，2026-09。复现异步加载、防抖、定时器、竞态类 bug 时按需取用；目标是让 spec 每次都走到出问题的那个时序，不靠碰运气。

| 场景 | 用什么 | 要点 | 出处 |
|---|---|---|---|
| 等元素出现、文字变化 | web-first 断言 `await expect(locator).toHaveText(...)` | 自动重试到超时。不用 `isVisible()` 这类立即返回的手动判断 | [best-practices](https://playwright.dev/docs/best-practices) |
| 等的不是 DOM（接口计数、store 值） | `expect.poll(() => ...)` | 把普通断言变成轮询 | [test-assertions](https://playwright.dev/docs/test-assertions#expectpoll) |
| 一段操作要整体重试直到成立 | `expect(async () => {...}).toPass({ timeout })` | 默认 timeout 是 0，且不读 expect timeout，必须显式传 | [test-assertions](https://playwright.dev/docs/test-assertions#expecttopass) |
| 点击触发请求，要在响应后断言 | `const p = page.waitForResponse(url)` → 点击 → `await p` | 先开始等再点，第一行不加 `await`，否则会错过响应 | [class-page](https://playwright.dev/docs/api/class-page#page-wait-for-response) |
| 页面依赖当前时间 | `page.clock.setFixedTime(date)` | 官方首选；只固定 `Date.now()` / `new Date()`，定时器照常走 | [clock](https://playwright.dev/docs/clock) |
| 防抖、轮询、倒计时 | `page.clock.install()` 后 `runFor(ms)` 或 `fastForward(ms)` | `install` 必须在任何其他 clock 调用之前；`runFor` 触发区间内所有回调，`fastForward` 到期定时器最多各触发一次 | [clock](https://playwright.dev/docs/clock) · [class-clock](https://playwright.dev/docs/api/class-clock) |
| 需要让某个请求变慢来撞出竞态 | `page.route(url, async route => { ...延迟...; await route.continue() })` | 官方没有专门的竞态示例，这是 `route` 的组合用法；延迟用 Promise 等待另一个请求完成，别写死毫秒 | [mock](https://playwright.dev/docs/mock) |
| 分段看是哪一步出问题 | `await test.step('名', async () => {...}, { box: true })` | 报告按步显示；`box` 让报错指向调用处 | [class-test](https://playwright.dev/docs/api/class-test#test-step) |
| 失败时带上日志 | `testInfo.attach('名', { body, contentType })`；`page.consoleMessages()` / `page.pageErrors()`（1.56+） | `path` 与 `body` 二选一 | [class-testinfo](https://playwright.dev/docs/api/class-testinfo#test-info-attach) |
| 确认复现稳定、修复稳定 | `--repeat-each=<N>` | 本配置 `retries: 0`，失败不会被重试掩盖成通过 | [test-cli](https://playwright.dev/docs/test-cli) |

不用：`page.waitForTimeout()`（官方原话 "Tests that wait for time are inherently flaky"）、`networkidle`、`sleep`。
