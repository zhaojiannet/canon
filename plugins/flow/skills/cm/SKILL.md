---
name: cm
description: 按规范提交当前改动：先查代码审过没有，每个 commit 前查密钥、调 vet 核对 message
disable-model-invocation: true
argument-hint: "[可选：限定文件或范围]"
---

把当前改动提交。带参数时只提交指定的那部分，其余不动。

## 先看改动

- `git status --porcelain`（含未跟踪文件，`git diff` 看不到它们）+ `git diff`。以实际改动为准，上下文里的任务列表只是线索。
- 改动里有代码文件（`.md`、`.txt`、图片、锁文件之外的都算），而本会话在最后一次改代码之后没跑过 `/code-review`：停下告诉我「这批代码还没审」，我回「跳过」才继续。审查放在提交前，审出的问题修进同一个 commit，不用事后改写已提交的历史。

## 分几个 commit

- 默认全部放进一个 commit。只有改动里明显是互不相干的两件事，比如一个 bug 修复加一个无关的新功能，才分开。一件事改了多少文件、跨了多少目录都放一起。
- 一两行的清理（`.gitignore` 条目、拼写、格式）跟着最相关的 commit 走，message 里不提。
- 分开时每个 commit 单独看都要完整：改函数签名和它的调用点、改动和它的测试放在同一个。
- 一个文件里混了两件事，整个文件放进主要那边，不做文件内拆分，message 末尾用一句话说明这个文件顺带了另一件事的改动。

## 每个 commit

1. `git add <路径…>`。不用 `git add -A`、`git commit -a`；不用 `git stash` / `git reset` / `git checkout` 切割改动，这条路会走到 `git reset --hard` 丢数据（claude-code issue #14293）。
2. 查密钥，按顺序用第一个能用的：
   - `gitleaks git --pre-commit --staged --redact --verbose --no-banner .`
   - `docker run --rm -v "$PWD":/repo ghcr.io/gitleaks/gitleaks:v8.30.1 git --pre-commit --staged --redact --verbose --no-banner /repo`
   - 都没有就自己把暂存的 diff 扫一遍。

   退出码 1 就停下，把查到的文件和行号告诉我，不提交。我确认是误报的，在那一行加 `gitleaks:allow` 注释再继续。
3. 按下面的规则写 message 草稿。
4. 调一次 `/flow:vet`，把草稿原文当参数传给它。按它的建议改 message；它标出的注释改在文件里，再 `git add`。改完就提交，不再调第二次。
5. `git commit`。

## Message

- 格式和禁忌照我全局 `~/.claude/CLAUDE.md` 里的 commit 规范执行，这里不重复。
- 先只看这个 commit 的 diff 写，不要从对话出发。
- 每句话要指向 diff 里某处改动，并给出看 diff 得不到的 why；做不到的删。
- diff 外能写的只有最终代码的 why：设计取舍、约束，包括为什么选这个做法、而不选读者会想到的另一个。开发经过、试过又删掉的写法、会话待办的编号都不写。
- 没有值得写的 why 就只写 subject。

## 收尾

- 全部提交完，列出每个 commit 的 hash 和 subject，再跑一次 `git status`，剩余改动必须是参数范围外、本来就不提交的那些。
- 未经我明确同意不要 push。
