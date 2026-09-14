# 横切面检查（`smoke.spec.ts`）

对计划里每个「页面 × 角色」都做，不探索、不录动作，从清单直接生成。这层最便宜，抓的是整页回归（路由吞掉、水合崩溃、权限放行）。

## 检查项

| # | 检查 | 断言 | 说明 |
|---|---|---|---|
| 1 | 页面可达 | `resp.status() < 400` | 匿名页 / 已登录页各按其角色；第 5 条的受限反例不走这条 |
| 2 | 无脚本崩溃 | `page.on('pageerror')` 收到的列表为空 | SSR 已绘但未水合时点击会静默失效，这条抓得到 |
| 3 | 应用壳渲染 | Nuxt `#__nuxt` 非空（`not.toBeEmpty()`）/ Astro `main` 非空 | 防止 200 但白屏。不用 `toBeVisible()`：布局用固定定位时壳子高度为 0，会误判 hidden |
| 4 | 无 i18n 缺 key | 正文不含 `t('`、`[missing`、形如 `xxx.yyy.zzz` 的裸 key | 按项目 i18n 库的缺 key 表现写正则 |
| 5 | 权限边界 | 错角色访问受限页 → 返回 403 / 跳离受限路径（登录或首页）/ 原 URL 显示 403 页 | 每个受限前缀至少一个反例 |
| 6 | 控制台无 error 级 | `console` 消息中 `type() === 'error'` 为空 | 第三方脚本被 CSP 拦的噪音要在画像里列白名单 |

不做：视觉回归（截图比对）、性能、无障碍。需要时另起一层。

## 写法

```ts
// e2e/tests/smoke.spec.ts
import { test, expect } from '@playwright/test'

const PAGES: Array<{ path: string; role: 'anon' | 'shop-admin'; expectBlocked?: boolean }> = [
  { path: '/dashboard/login', role: 'anon' },
  { path: '/dashboard/orders', role: 'shop-admin' },
  { path: '/dashboard/hq-today', role: 'shop-admin', expectBlocked: true },
]

for (const p of PAGES) {
  test.describe(`${p.role}`, () => {
    if (p.role !== 'anon')
      test.use({ storageState: `playwright/.auth/${p.role}.json` })

    test(`${p.path}`, async ({ page }) => {
      const pageErrors: string[] = []
      const consoleErrors: string[] = []
      page.on('pageerror', e => pageErrors.push(e.message))
      page.on('console', m => { if (m.type() === 'error') consoleErrors.push(m.text()) })

      const resp = await page.goto(p.path)

      if (p.expectBlocked) {
        const status = resp?.status()
        // 三种拦法任一即可：服务端 403、跳离受限路径（登录页带 ?redirect= 也算）、原 URL 显示 403 页
        await expect.poll(async () =>
          status === 403
          || new URL(page.url()).pathname.replace(/\/+$/, '') !== p.path
          || await page.locator(BLOCKED_MARK).count() > 0,
        { message: `${p.path} 应被拦截（status=${status}）` }).toBe(true)
        return
      }

      expect(resp?.status(), 'HTTP status').toBeLessThan(400)

      await expect(page.locator('#__nuxt')).not.toBeEmpty()
      await expect(page.locator('body')).not.toContainText(/\bt\('|\[missing|\b[a-z]+\.[a-z]+\.[a-z]+\b/)
      expect(pageErrors, 'uncaught page errors').toEqual([])
      expect(consoleErrors.filter(t => !IGNORE.some(re => re.test(t))), 'console errors').toEqual([])
    })
  })
}

// 画像里列出的已知噪音（如 CSP 拦第三方统计脚本）
const IGNORE: RegExp[] = []
// 画像里定义的 403 页标识（选择器）
const BLOCKED_MARK = '[data-testid="forbidden"]'
```

按项目改四处：`PAGES` 从画像的页面清单生成；应用壳选择器；缺 key 正则和 `IGNORE`；`BLOCKED_MARK` 按画像里定义的 403 页标识写。
