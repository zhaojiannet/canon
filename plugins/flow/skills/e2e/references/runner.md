# 运行环境（容器内跑 Playwright）

核对：Playwright 1.63.0，2026-09。实测环境 OrbStack + 官方镜像。

规则一句话：**Playwright 相关的一切只存在于容器里，宿主机零安装。** 不装 `@playwright/test`、不装浏览器、不装 `@playwright/cli`。`playwright-cli` 本身就在 `playwright-core` 里，`npx playwright cli <cmd>` 就有它。

## compose 服务

```yaml
services:
  e2e:
    image: mcr.microsoft.com/playwright:v1.63.0-noble   # 版本必须等于 package.json 里 @playwright/test 的版本，差一位找不到浏览器
    container_name: ${PROJECT}-e2e
    profiles: [e2e]              # docker compose up 不会顺带起它
    init: true                   # 官方要求，否则常驻进程变僵尸
    ipc: host                    # 官方要求，否则 Chromium 撑爆 64MB /dev/shm 中途崩
    working_dir: /app
    volumes:
      - ./:/app
      - e2e_node_modules:/app/node_modules   # 不 bind 宿主机 node_modules，平台二进制会崩
    environment:
      BASE_URL: http://${PROJECT}-nginx.orb.local   # 入口容器的 OrbStack 域名；没有 nginx 就写前端容器
      PLAYWRIGHT_HTML_OPEN: never
    command: sleep infinity      # 必须长驻，CLI 的常驻进程和 socket 都在容器里
volumes:
  e2e_node_modules:
```

起：`docker compose --profile e2e up -d e2e && docker compose exec e2e npm ci`
跑：`docker compose exec e2e npx playwright test`
画像里 `<exec>` 就写 `docker compose exec e2e`。

镜像里没有 Google Chrome，CLI 默认找它会报 `Chromium distribution 'chrome' is not found`。项目根放 `.playwright/cli.config.json`（入库，它是基建）：

```json
{ "browser": { "browserName": "chromium", "launchOptions": { "channel": "chromium" } } }
```

`.gitignore` 加 `.playwright-cli/`（CLI 在项目根写快照和 console 日志，可能含凭证）和 `playwright-report/`、`test-results/`、`playwright/.auth/`。

## 网络

容器内能解析 OrbStack 域名（`<容器名>.orb.local`），不需要接任何 docker 网络。Nuxt / Vite dev 服务校验 Host 头，不放行会返回 403 `Blocked request. This host ... is not allowed`；`nuxt.config.ts` 里：

```ts
vite: { server: { allowedHosts: ['.local'] } }
```

Astro 在 `astro.config.mjs` 的 `vite.server.allowedHosts` 同理。生产构建不经 Vite dev server，没有这个问题。

## playwright.config.ts

```ts
import { defineConfig, devices } from '@playwright/test'

export default defineConfig({
  testDir: './tests/e2e',
  fullyParallel: true,
  forbidOnly: !!process.env.CI,
  retries: process.env.CI ? 2 : 0,
  reporter: process.env.CI ? [['github'], ['list']] : 'list',
  use: {
    baseURL: process.env.BASE_URL,
    trace: 'on-first-retry',
    screenshot: 'only-on-failure',
    video: 'retain-on-failure',
  },
  projects: [
    { name: 'setup', testMatch: /.*\.setup\.ts/ },
    {
      name: 'chromium',
      use: { ...devices['Desktop Chrome'] },
      dependencies: ['setup'],
    },
  ],
})
```

只跑 chromium。三浏览器是需求出现时再加。

## 登录态

官方 auth 文档的做法：`setup` project 登录一次存 `storageState`，每个角色一份，测试文件用 `test.use({ storageState })` 挑角色。

```ts
// tests/e2e/auth.setup.ts
import { test as setup, expect } from '@playwright/test'

setup('shop admin', async ({ page }) => {
  await page.goto('/dashboard/login')
  await page.getByRole('textbox', { name: 'Email' }).fill(process.env.E2E_SHOP_EMAIL!)
  await page.getByRole('textbox', { name: 'Password' }).fill(process.env.E2E_SHOP_PASSWORD!)
  await page.getByRole('button', { name: 'Sign In' }).click()
  await expect(page).toHaveURL(/\/dashboard\//)
  await page.context().storageState({ path: 'playwright/.auth/shop-admin.json' })
})
```

账号密码走 `.env` 的 `E2E_*`，compose 的 `env_file` 带进容器。强制 TOTP 的端（如平台超管）要在画像里写清处理办法：dev 库种一个 secret 已知的账号，用 `otpauth` 之类在 setup 里算码；做不到就该端只跑匿名冒烟。

## seed 与 attach 的实测行为

- `<exec> npx playwright test tests/e2e/seed.spec.ts --debug=cli` 要后台跑并把输出写到文件（`docker compose exec -d e2e sh -c '... > /tmp/debug.log 2>&1'`），从文件里取 `tw-xxxx`。
- attach 后测试暂停在第一步。**用 `step-over` 逐步走，不用 `resume`**——seed 只有 `goto` 一步时 `resume` 会直接跑完、浏览器关闭。走到「Paused - Close context」就是可探索状态。
- attach 后每条命令带 `-s=tw-xxxx`。
- 探索完 `resume` 让测试结束，或 `pkill -f "playwright test"`。一次只开一个 debug 会话。

## 常用命令

```bash
<exec> npx playwright test --grep "<测试名>"                       # 单跑一条
<exec> sh -c 'PLAYWRIGHT_JSON_OUTPUT_NAME=.scratch/e2e/run-$(date +%F).json npx playwright test --reporter=json'
<exec> npx playwright cli open <url> / snapshot / fill e5 "x" / click e6 / --raw generate-locator e6
<exec> npx playwright cli state-save /app/playwright/.auth/x.json
<exec> npx playwright cli list / close / kill-all
<exec> npx playwright show-trace test-results/<...>/trace.zip    # 无头环境看不了，trace 拷到宿主机用浏览器版 trace.playwright.dev 看
```
