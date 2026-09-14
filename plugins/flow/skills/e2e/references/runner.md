# 运行环境

核对：Playwright 1.63.0，2026-09。实测环境 OrbStack + 官方镜像。

原则：Playwright 相关的一切只在一个独立容器里，宿主机零安装；文件全部落在项目根 `e2e/` 内（依赖、官方 skill、spec、账号、报告），项目目录外不放任何东西；被测项目除了 `e2e/` 和一行 `.gitignore` 什么都不改。`playwright-cli` 本身就在 `playwright-core` 里，`npx playwright cli` 就有它，不另装 `@playwright/cli`。

## 一次性搭建

```bash
P=<仓库目录名>              # 如 myshop
ENTRY=<入口容器名>           # 如 myshop-nginx；没有 nginx 就写前端 dev 容器
PORT=<入口容器内端口>        # nginx 80；Nuxt dev 3000；Astro dev 4321

cd <项目根>
mkdir -p e2e/tests
grep -qx '/e2e/' .gitignore || printf '\n# e2e 工作区：依赖、账号、报告，不入库\n/e2e/\n' >> .gitignore

docker run -d --name $P-e2e --init --ipc=host \
  --network container:$ENTRY \
  -v "$PWD/e2e:/work" -w /work \
  --env-file e2e/.env \
  -e BASE_URL=http://localhost:$PORT -e PLAYWRIGHT_HTML_OPEN=never \
  mcr.microsoft.com/playwright:v1.63.0-noble sleep infinity

docker exec -w /work $P-e2e sh -c 'npm init -y >/dev/null && npm i -D @playwright/test@1.63.0 --no-audit --no-fund'
# 装官方 skill 到 e2e/.claude/skills/playwright-cli/，并生成 e2e/.playwright/cli.config.json 指定用自带 chromium
# （CLI 默认找 Google Chrome，镜像里没有）。Playwright 升级后重跑一次。
docker exec -w /work $P-e2e npx playwright cli install --skills
```

画像里 `<exec>` 写 `docker exec -w /work $P-e2e`。`e2e/.env` 先建好再起容器（`--env-file` 要求文件存在），内容形如 `E2E_SHOP_EMAIL=...` / `E2E_SHOP_PASSWORD=...`；改了 `.env` 要 `docker restart $P-e2e`。

每个参数的理由：

- 镜像 tag 必须等于 `@playwright/test` 版本，差一位找不到浏览器。
- `--init`：官方要求，否则常驻进程变僵尸。
- `--ipc=host`：官方要求，否则 Chromium 撑爆 64MB `/dev/shm` 中途崩。
- `--network container:$ENTRY`：与入口容器共用网络命名空间，`http://localhost:$PORT` 就是被测站。这样绕开 Vite dev server 的 Host 头校验（用 OrbStack 域名或容器名访问 Nuxt dev 会 403 `Blocked request`），不用改项目的 `allowedHosts`。代价：入口容器被重建（不是重启）后，e2e 容器要 `docker rm -f` 再起一次。
- `node_modules` 直接落在 `e2e/node_modules/`：由容器安装，宿主机从不执行它，整个 `e2e/` 已被 gitignore。
- `sleep infinity`：CLI 的常驻进程和 socket 都在容器里，容器必须长驻；一次性 `docker run --rm` 每次调用进程都没了。

## 拆掉

```bash
docker rm -f $P-e2e
```

`e2e/` 目录留着（计划、报告、spec 还在），要彻底清就 `rm -rf e2e` 并删掉 `.gitignore` 那行。

## playwright.config.ts（放 `e2e/`）

```ts
import { defineConfig, devices } from '@playwright/test'

export default defineConfig({
  testDir: './tests',
  forbidOnly: !!process.env.CI,
  retries: 0,
  workers: 1,                       // 写操作共用一套现有数据，串行
  reporter: 'list',
  use: {
    baseURL: process.env.BASE_URL,
    trace: 'retain-on-failure',
    screenshot: 'only-on-failure',
    video: 'retain-on-failure',
  },
  projects: [
    { name: 'setup', testMatch: /.*\.setup\.ts/ },
    { name: 'chromium', use: { ...devices['Desktop Chrome'] }, dependencies: ['setup'] },
  ],
})
```

只跑 chromium。冒烟层只读，可以在它自己的 describe 里 `test.describe.configure({ mode: 'parallel' })`。

## 登录态

官方 auth 文档的做法：`setup` project 登录一次存 `storageState`，每个角色一份，测试文件用 `test.use({ storageState })` 挑角色。

```ts
// e2e/tests/auth.setup.ts
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

强制 TOTP 的端：问我要一个 secret 已知的测试账号，setup 里用 `otpauth` 算码；给不了就该端只跑匿名冒烟，写进报告。

## seed 与 attach 的实测行为

- `<exec> npx playwright test tests/seed.spec.ts --project=chromium --no-deps --debug=cli` 要后台跑并把输出写到文件（`docker exec -d -w /work $P-e2e sh -c '... > /tmp/debug.log 2>&1'`），从文件里取 `tw-xxxx`。**必须带 `--no-deps`**：否则 `setup` 依赖项目先跑，暂停停在登录用例上而不是 seed。seed 用 `test.use({ storageState })` 直接带登录态。
- attach 后测试暂停在第一步。**用 `step-over` 逐步走，不用 `resume`**——seed 只有 `goto` 一步时 `resume` 会直接跑完、浏览器关闭。走到「Paused - Close context」就是可探索状态。
- attach 后每条命令带 `-s=tw-xxxx`。
- 探索完 `resume` 让测试结束，或 `pkill -f "playwright test"`。一次只开一个 debug 会话。

## 常用命令

```bash
<exec> npx playwright test tests/dashboard/orders            # 跑一个模块
<exec> npx playwright test --grep "<测试名>"                  # 单跑一条
<exec> sh -c 'PLAYWRIGHT_JSON_OUTPUT_NAME=run-$(date +%F).json npx playwright test --reporter=json'
<exec> npx playwright cli open <url> / snapshot / fill e5 "x" / click e6 / --raw generate-locator e6
<exec> npx playwright cli list / close / kill-all
```

失败现场在 `e2e/test-results/<用例>/`：`error-context.md` 和 `trace.zip`。agent 用 `<exec> npx playwright trace ...` 在命令行读（顺序见 `diagnose.md`）；人要看就在宿主机浏览器打开 https://trace.playwright.dev 把 `trace.zip` 拖进去。
