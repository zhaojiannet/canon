# 计划模板（`.scratch/e2e/plan.md`）

格式沿用官方 `test-generation.md` §1.4（`**Seed:**` / `**File:**` / `**Steps:**` / `- expect:`），generator 直接认。额外加账号、前置数据、副作用等级三行，这是官方格式没有、而我们的护栏要读的。

```markdown
Status: draft
Profile: .scratch/e2e/profile.md (Verified 2026-09-12)
Scope: smoke + risk:1-3

# <项目名> 测试计划

## 冒烟层（自动生成，不探索）

| 页面 | 角色 | 检查 | 跳过原因 |
|---|---|---|---|
| /dashboard/orders | shop_admin | checks.md 全部 | |
| /dashboard/hq-today | shop_admin | 期望 403 / 重定向 | |
| /admin/refunds | platform_admin | — | admin 强制 TOTP，未配置 |

## 风险路径

### 1. 顾客现金结账

**Seed:** `tests/e2e/seed.spec.ts`
**Account:** 匿名（桌台码开台） + shop_admin（收银）
**Data:** 店铺 1 桌台 3 空闲；商品 A 上架
**Side-effect:** 2（写订单、账单；快照可回滚）

#### 1.1. cash-checkout-single-table

**File:** `tests/e2e/risk/cash-checkout-single-table.spec.ts`

**Steps:**
  1. 用桌台码 URL 开台，选人数 2
    - expect: 进入 /menu，购物车为空
  2. 加购商品 A ×1，去 /cart 提交订单
    - expect: 订单页出现该商品，金额含税与商品页一致
  3. shop_admin 在 /tables/overview 对该桌执行现金收银
    - expect: 账单状态 paid；顾客端 /orders 显示已支付
    - expect: 收银后桌台回到空闲

#### 1.2. cash-checkout-rejects-when-online-in-flight
...

### 2. <下一条路径>
```

写计划的规则：

- 一个场景一个文件，场景名 kebab-case 等于文件名。
- 场景之间独立，从 seed 的干净状态开始，不串联。
- 步骤写用户视角（「点『ログイン』」），不写 API 视角。
- 每个 `- expect:` 都会变成一条断言，写可观察的结果。
- 3 / 4 级的场景照样写出来，标 `Skip: <级别> <原因>`——让人看到没测什么，比看不到强。
- 负向场景（错角色、错店、过期 token）至少每条路径一个。
