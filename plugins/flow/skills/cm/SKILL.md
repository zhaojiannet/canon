---
name: cm
description: 按规范提交当前改动：先确认代码审过，每个 commit 前查密钥、调 vet 核对 message
disable-model-invocation: true
argument-hint: "[可选：限定文件或范围]"
---

把当前改动提交。带参数时只提交参数指定的部分，其余改动留在工作区。

## 先看改动

1. 跑 `git status --porcelain` 和 `git diff`。前者列出未跟踪的新文件，后者看不到它们。分组以这两条命令的输出为准，上下文里的任务列表只当线索。
2. 改动里有代码文件（`.md`、`.txt`、图片、锁文件以外的都算）时，确认本会话跑过 `/code-review`，并且那之后的代码改动只是在修它确认的问题。修这些问题不用再审：不带参数的 `/code-review` 每次都重审未推送的提交和全部未提交的改动，修完就要求再审会一轮接一轮。
   - 没跑过，或者审查之后又写了别的代码，就用 AskUserQuestion 问我，问题里列出这些代码文件，选项两个：「跑 `/code-review`」「跳过」。
   - 选跑：调 `/code-review`，等结果回来，修掉它确认的问题，再往下走。选跳过：直接往下走。

   审查放在提交前，审出的问题能修进同一个 commit，已提交的历史保持不动。

## 分几个 commit

- 默认全部放进一个 commit。改动里有两件互不相干的事时才分开，比如一个 bug 修复加一个无关的新功能。同一件事改了多少文件、跨了多少目录，都放在一起。
- 一两行的清理（`.gitignore` 条目、拼写、格式）跟着最相关的 commit 走，message 里不提。
- 分开后，每个 commit 单独看都要完整：函数签名和它的调用点、改动和它的测试，放在同一个 commit。
- 一个文件里混了两件事时，整个文件放进主要的那个 commit，在 message 末尾用一句话交代这个文件顺带了哪件事的改动。

## 每个 commit

1. 用 `git add <路径…>` 暂存这一组，全程保持工作区原样。`git add -A`、`git commit -a` 会把别的组带进来；用 `git stash`、`git reset`、`git checkout` 切割改动可能丢数据（claude-code issue #14293）。
2. 查密钥，用下面第一个能用的：
   - `gitleaks git --pre-commit --staged --redact --verbose --no-banner .`
   - `docker run --rm -v "$PWD":/repo ghcr.io/gitleaks/gitleaks:v8.30.1 git --pre-commit --staged --redact --verbose --no-banner /repo`
   - 两个都用不了，就自己把暂存的 diff 扫一遍。

   退出码是 1 时停下，把查到的文件和行号告诉我。我确认是误报后，在那一行加 `gitleaks:allow` 注释，再继续。
3. 按下面「Message」一节写草稿。
4. 调一次 `/flow:vet`，把草稿原文作为参数传给它。按它的建议改 message；它标出的注释改在文件里，再 `git add`。
5. `git commit`。

## Message

- 格式和禁忌照我全局 `~/.claude/CLAUDE.md` 里的 commit 规范。
- 对着这个 commit 的 diff 写，对话只用来补 why。每句话都要指向 diff 里的某处改动，并说出看 diff 得不到的 why；做不到的句子删掉。
- diff 之外能写的只有最终代码的 why：设计取舍、约束，包括为什么选这个做法、不选读者会想到的另一个。开发经过、试过又删掉的写法、会话待办的编号都不写。
- 没有值得写的 why，就只写 subject。

## 收尾

- 全部提交完，列出每个 commit 的 hash 和 subject，再跑一次 `git status`。剩下的改动只能是参数范围外、本来就不提交的那些。
- 我明确说 push 才 push。
