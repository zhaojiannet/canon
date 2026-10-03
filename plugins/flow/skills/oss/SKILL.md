---
name: oss
description: 调研、选型、对比开源项目时的筛选标准和安全核对。用户要调研、选型、对比、推荐库、框架、工具、GitHub 项目，或提到 star 数、替代方案时，先读这里再找候选。
---

# 调研开源项目

## 候选门槛
- star 少于 1k 的不进候选。
- 列多个候选对比时，第 2、第 3 名与第 1 名的 star 差距不超 2k，或不低于第 1 名的一半，两条满足一条即可。知名项目、大厂出品、官方推荐的不受此限，但要在报告里写明按哪一条例外放进来。
- star 数当场查，不凭记忆，报告里附查到的数字：

```sh
gh api repos/<owner>/<repo> --jq '{stars: .stargazers_count, forks: .forks_count, issues: .open_issues_count, created: .created_at, pushed: .pushed_at}'
gh api repos/<owner>/<repo>/contributors --jq length
```

## 危险、钓鱼项目核对
候选进报告前逐个核对，命中任意一条就剔除并写明原因：
- 名字、README、仓库布局模仿知名项目，但 owner 不是原项目的组织或作者（typosquatting）。
- star 多但 fork、issue、contributor 极少，或仓库很新却 star 暴涨（刷 star）。
- 安装方式要求 `curl ... | sh` 走非官方域名、下载预编译二进制而源码里没有对应的构建脚本或 CI、要求粘贴钱包私钥 / API key / 登录凭证。
- 仓库里有混淆代码、来路不明的二进制或压缩包、Release 附件和源码对不上。
- 一年以上没有提交且 issue 无人回应，按已废弃处理。

不确定是否安全就不进候选，不替它找理由。
