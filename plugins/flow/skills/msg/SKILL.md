---
name: msg
description: 写或审 commit message 时的格式与禁忌。写 commit、起草 message、改 message、检查 message 时都先读这里。
---

# Commit message 规则

基于 cbea.ms 7 rules + Conventional Commits v1.0.0 + Linux kernel SubmittingPatches。

## 格式
- Subject: `<type>(<scope>): <desc>`，imperative（"修" 而非 "修了"），≤ 72 字符，无尾标点。type ∈ `feat / fix / refactor / docs / chore / style / perf / test / i18n`
- Body: 空行隔开 subject，行宽 72 列（中文约 36 字），断行落在标点处；按方面分段，每段 2 到 4 行，bullet 仅多并列项。版本号、顺带的一两行清理这类看 diff 就知道的不写
- Backtick 包所有 identifier / SQL / 路径 / 函数名 / 字段名
- Footer trailer: `Fixes #N` / `Closes #N` / `Refs: <ref>` / `BREAKING CHANGE: <desc>`
- 讲 what + why，不讲 how（diff 自己讲 how）
- 单一 problem per commit（kernel: split when description gets long）。body 各段讲同一件事的不同方面；哪一段讲的是另一件事，就是两个 commit
- 简体中文，不加 emoji，不加 AI 相关信息

## 严禁
- **商业品牌/产品名**（trademark 法律风险 + 暴露调研路径）。技术依赖名保留（编程语言 / 框架 / 库 / 标准 / 协议 / 工具链 / 开源生态）
- **受众化说教**（"用户看不懂 / 给程序员的话" — commit 是技术日志，不是用户手册）
- **文学修辞**（"UI 撒谎 / 契约 betray / 狼来了" — 改具体描述）
- **失败迭代叙事**（"v1 失败 v2 失败 v3 紧急修补" — 只讲最终方案 + 为什么这样选）
- **对话残留**。判断：每句话、每个引用都要能在最终 diff 或仓库里找到对应，找不到的删。被删掉的、试过没合入的、「为什么没有 X」、会话待办的编号、「本轮 / 这次 / 上一轮」，对读者不存在，否定句也不许提
- **PR 元语言**（"本 PR / 同 PR / 一并修复" — commit ≠ PR）
- **多层 markdown heading 堆叠**（最多 `## What / ## Why`）
- **业内对照虚词**（"业内主流 / 调研依据 / 对标业内" — 整段删）

## 提交前
- 必跑 `/flow:vet`：没看过对话的子代理拿 diff 和 message 草稿逐句核对，标出对不上的。清单为空才 commit。走 `/flow:cm` 时它自动调。

## 对话残留的样子
<examples>
<example>
反例：feat(auth): 登录页（不带「记住我」）
问题：diff 里从头到尾没有「记住我」，「不带」是在讲对话里被砍掉的东西
改成：feat(auth): 登录页
</example>
<example>
反例 body：「记住我」先不做，等会话模块定型再加。
问题：解释「为什么没有 X」，读者看 diff 不知道曾经打算有 X
改法：整句删
</example>
<example>
反例：fix(api): 按 #7 修复订单校验，#8 全部完成
问题：#7、#8 是会话待办的编号，仓库和 issue 里都没有
改成：fix(api): 订单等金额和币种都到齐再建
</example>
<example>
反例 body：先试了用 stash 切分改动，会丢数据，改成逐组 add。
问题：叙述被否掉的中间做法，diff 里只有逐组 add
改法：删；逐组 add 若有非直觉的原因，只写那个原因
</example>
<example>
反例 body：这次先只改左栏，右栏下一轮再动。
问题：「这次」「下一轮」是对话里的进度，读者没有轮次
改法：整句删
</example>
<example>
合格 body：原生 confirm 的按钮文案由浏览器给，带不上「删除」「离开」这类动作名，也套不上主题色。改用组件库的弹层后，点窗外、Esc 和右上角的 × 都按取消处理。
为什么合格：每句都对应 diff 里的改动，并说了 diff 看不出的取舍
</example>
</examples>
