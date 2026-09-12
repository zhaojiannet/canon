# 报告模板（`.scratch/e2e/report-<日期>.md`）

读者是没看过对话的人。数字从 `run-<日期>.json` 的 `stats` 取，不手数。

```markdown
# e2e 报告 <日期>

Profile: Verified <日期>  ·  Plan: <范围>  ·  Run: run-<日期>.json

## 结果

| | 数 |
|---|---|
| 通过 | 41 |
| 失败（应用 bug） | 2 |
| 失败（未解决） | 0 |
| 跳过 | 6 |
| 修过的测试 | 5（列于下） |

## 覆盖

- 冒烟：<N> 页 × <M> 角色
- 风险路径：1 顾客现金结账（3 场景）、2 合并结账（2 场景）、3 ……

## 应用 bug（不改代码，交人处理）

### 1. 合并结账后账单不标 paid
- 场景：plan 2.1 `merge-checkout-marks-paid`
- 复现：shop_admin 在 /orders 勾选桌 3、桌 5 → 合并结账 → 现金收银 → 查看账单
- 期望：账单 `status = paid`
- 实际：`status = open`，前端提示「支払完了」
- 证据：`test-results/.../trace.zip`，接口 `POST /api/dashboard/orders/merge-checkout` 返回 200 但 `bills` 未更新
- 测试当前状态：`test.fixme`

## 跳过及原因

| 场景 | 级别 | 原因 |
|---|---|---|
| 1.3 线上支付回调 | 4 | 走真实支付网关，未批准 |
| admin/* 全部 | — | 强制 TOTP，画像未配置 |

## 修过的测试（测试或环境问题，已解决）

| 场景 | 类型 | 改了什么 |
|---|---|---|
| smoke /dashboard/products | 测试 | 选择器改为 getByRole('table') |
| 1.1 | 环境 | 桌台 3 被上次运行占用；快照恢复后通过 |

## 环境 / 数据

- 快照：<命令 / 文件>；恢复：<成功 / 失败及原因>
- 运行时长：<秒>

## 下一步建议

- <按价值排序，最多 5 条：该补的路径、该接进 CI 的层、该配置的账号>
```
