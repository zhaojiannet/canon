# 报告模板（`e2e/report-<日期>.md`）

读者是没看过对话的人。数字从 `run-<日期>.json` 的 `stats` 和 plan.md 的状态列取，不手数。「待做」必须是 0，不是 0 就逐条列出来说明。

```markdown
# e2e 报告 <日期>

Profile: Verified <日期>  ·  Plan: <范围>  ·  Run: run-<日期>.json  ·  备份: backups/<文件>

## 结果

| | 数 |
|---|---|
| 计划场景 | 212 |
| 通过 | 178 |
| 失败（应用 bug） | 6 |
| 跳过（3/4 级、无账号） | 24 |
| 阻塞（环境 / 需改项目） | 4 |
| 待做 | 0 |
| 修过的测试 | 11（列于下） |

## 覆盖

- 冒烟：<N> 页 × <M> 角色，全部通过 / 失败 <K>
- 操作层按模块：dashboard/orders 12/12、dashboard/products 9/10、…

## 应用 bug（不改代码，交人处理）

### 1. 合并结账后账单不标 paid
- 场景：plan orders-2 `merge-checkout-marks-paid`
- 复现：shop_admin 在 /orders 勾选桌 3、桌 5 → 合并结账 → 现金收银 → 查看账单
- 期望：账单 `status = paid`
- 实际：`status = open`，前端提示「支払完了」
- 证据：`test-results/.../trace.zip`；`POST /api/dashboard/orders/merge-checkout` 返回 200 但 `bills` 未更新
- 测试当前状态：`test.fixme`

## 阻塞（不改项目测不了）

| 场景 | 卡在哪 | 要改什么才能测 |
|---|---|---|
| admin/* 全部 | 强制 TOTP，无 secret 已知账号 | 给一个测试用 TOTP secret |

## 跳过及原因

| 场景 | 级别 | 原因 |
|---|---|---|
| pay-3 线上支付回调 | 4 | 走真实支付网关，未批准 |

## 修过的测试（测试或环境问题，已解决）

| 场景 | 类型 | 改了什么 |
|---|---|---|
| smoke /dashboard/products | 测试 | 选择器改为 getByRole('table') |
| orders-1 | 环境 | 前置订单不存在，场景内改为先代客下单 |

## 遗留数据

| 数据 | 位置 | 为什么没删 |
|---|---|---|
| 无 | | |

## 下一步建议

- <按价值排序，最多 5 条：该补的路径、该给的账号、该接进 CI 的层>
```
