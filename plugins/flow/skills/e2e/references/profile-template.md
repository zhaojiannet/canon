# 画像模板（`e2e/profile.md`）

每项都要填。查不到的写「未知」并列进「待确认」，不猜。

```markdown
Verified: 2026-09-12

# <项目名> e2e 画像

## 执行
- exec 前缀：`docker exec -w /work <项目>-e2e`
- 入口容器 / 端口：`<项目>-nginx` / 80（e2e 容器共用它的网络，BASE_URL=http://localhost）
- 依赖前置：<跑测试前必须起的容器 / 必须存在的 feature flag>

## 入口
| 端 | URL（容器视角） | 登录方式 | 备注 |
|---|---|---|---|
| dashboard | http://localhost/dashboard/ | 邮箱密码 | |

## 账号（每个角色一个；密码在 e2e/.env，这里只写变量名）
| 角色 | 邮箱 | 密码变量 | 能登（核过日期） | 来源 / 说明 |
|---|---|---|---|---|
| shop_admin | shop-a@example.test | E2E_SHOP_PASSWORD | 2026-09-12 | docs/ops/test-accounts.md，店铺 id=1 |
| platform_admin | — | — | — | 强制 TOTP，无 secret 已知的账号 → 该端只跑匿名冒烟 |

缺账号的角色：<列出来，向我要>

## 备份 / 恢复（用项目现有工具；没有就用 DB 容器自带的 pg_dump / mysqldump）
- 备份：`task db:dump -- <库>` → `backups/<库>-<日期>.sql.gz`
- 恢复：`task db:restore -- <文件>`（恢复前必须问我）
- 库所在：<容器名 / 库名；共享容器要写清只动哪个库>

## 副作用分级（有写操作的接口 / 页面动作）
| 级别 | 含义 | 本项目属于此级的操作 |
|---|---|---|
| 1 | 只读 | 所有 GET、列表、报表查看 |
| 2 | 库内可逆 | 建 / 改 / 删测试数据（自己建自己删） |
| 3 | 库内不可逆 | 日次締め、账期出金、重置密码、组织转移 |
| 4 | 出库外部 | 支付网关、退款、邮件、LINE 推送、打印、对象存储删除、机器翻译 |

3 / 4 级默认跳过；来源写清楚（文件 / 路由）。

## 业务口径（断言要用的规则）
- <例：税额必须经 effectiveTaxCategory(product, isTakeout) 算，前端不直接读 tax_category>
- <例：多租户——A 店账号访问 B 店资源必须 403>

## 已有测试
- 单元 / 集成：<数量、位置、CI 里跑不跑>
- 现有 e2e：<位置；「无」>
- 测试清单文档：<如 docs/test-inventory.md，有就以它为计划来源>

## 范围（默认全量）
- 层：smoke + 全部操作
- 端：全部
- 排除：<明确不测的，写原因>

## 待确认
- <只列真的查不到的>
```

画像来源优先级：项目 CLAUDE.md > compose / Taskfile > 路由注册代码 > 已有文档。文档和代码冲突以代码为准，冲突本身写进「待确认」。
