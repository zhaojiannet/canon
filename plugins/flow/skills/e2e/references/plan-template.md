# 计划模板（`e2e/plan.md`）

格式沿用官方 `test-generation.md` §1.4（`**Seed:**` / `**File:**` / `**Steps:**` / `- expect:`），generator 直接认。额外加账号、前置数据、副作用等级、清理方式和状态，这是官方格式没有、而我们的护栏和「不漏」要读的。

状态取值：`待做` / `已生成` / `通过` / `失败` / `跳过` / `阻塞`。每批跑完就更新，收尾时不能有「待做」。

```markdown
Status: draft
Profile: e2e/profile.md (Verified 2026-09-12)
Scope: 全量

# <项目名> 测试计划

## 总览

| 层 / 模块 | 场景数 | 待做 | 通过 | 失败 | 跳过 | 阻塞 |
|---|---|---|---|---|---|---|
| smoke | 135 × 3 | … | | | | |
| dashboard/orders | 12 | … | | | | |

## 冒烟层（自动生成，不探索）

| 页面 | 角色 | 期望 | 状态 | 备注 |
|---|---|---|---|---|
| /dashboard/orders | shop_admin | checks.md 全部 | 待做 | |
| /dashboard/hq-today | shop_admin | 403 / 重定向 | 待做 | |
| /admin/refunds | platform_admin | — | 跳过 | 强制 TOTP，无账号 |

## 操作层

### dashboard/orders

**Seed:** `tests/seed.spec.ts`
**Account:** shop_admin
**Data:** 店铺 1 有至少一张未付订单（没有则场景内先代客下单）
**Side-effect:** 2
**Cleanup:** 场景结束取消自己建的订单

#### orders-1. cancel-unpaid-order  `状态: 待做`

**File:** `tests/dashboard/orders/cancel-unpaid-order.spec.ts`

**Steps:**
  1. 打开 /dashboard/orders，点第一张未付订单
    - expect: 进入 /dashboard/orders/<id>
  2. 点「キャンセル」，填理由，确认
    - expect: 状态变为 cancelled
    - expect: 列表页该单不再出现在未付筛选

#### orders-2. merge-checkout-marks-paid  `状态: 待做`
...

### dashboard/finance/daily-close  `Skip: 3 級，日次締め不可逆`

| 编号 | 场景 | 状态 | 原因 |
|---|---|---|---|
| daily-close-1 | settle-today | 跳过 | 冻结营业日不可重算 |
```

写计划的规则：

- 一个场景一个文件，场景名 kebab-case 等于文件名。
- 场景之间独立，从 seed 的干净状态开始，不串联。
- 步骤写用户视角（「点『ログイン』」），不写 API 视角。
- 每个 `- expect:` 都会变成一条断言，写可观察的结果。
- 3 / 4 级的场景照样列出来标跳过——让人看到没测什么，比看不到强。
- 有测试清单文档的项目，清单里每个「用户操作」都要在计划里找得到对应场景或跳过行；对不上的写进「待确认」。
- 负向场景（错角色、错店、过期 token）每个模块至少一个。
