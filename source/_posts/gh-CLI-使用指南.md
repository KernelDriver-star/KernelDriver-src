---
title: gh CLI 使用指南：命令行玩转 GitHub
date: 2026-09-17 10:00:00
categories:
  - 开发工具
tags:
  - GitHub
  - gh
  - CLI
  - 开发工具
---

很多人每天都在用 GitHub——提 PR、看 Issue、查 CI 结果、下 Release，但每次都要切到浏览器、等页面加载、点来点去。`gh`（GitHub CLI）是 GitHub 官方推出的命令行工具，让你在终端里完成绝大部分 GitHub 操作。这篇整理一下 `gh` 和 GitHub 的关系、安装配置、常用命令和实战流程（命令语法以 [gh 官方手册](https://cli.github.com/manual) 为准，版本截至 2.100.0）。

## 一、gh 和 GitHub 是什么关系

这是本文最核心的问题，先讲清楚。

### 1.1 GitHub 是平台

**GitHub** 是一个**Web 平台**（网站），提供代码托管、协作、CI/CD（GitHub Actions）、Issue 追踪、项目管理等功能。你平时通过浏览器访问 `github.com` 来使用它。

### 1.2 gh 是工具

**gh**（GitHub CLI）是一个**本地命令行程序**，安装在你的电脑上，通过 GitHub 的 REST/GraphQL API 与 GitHub 平台交互。简单说：

| | GitHub | gh |
|---|---|---|
| 本质 | Web 平台（SaaS 服务） | 本地命令行工具（Go 编写的二进制） |
| 使用方式 | 浏览器打开网页 | 终端输入命令 |
| 底层协议 | HTTP/HTML | HTTP/REST API + GraphQL API |
| 操作对象 | 仓库、Issue、PR、Actions、Release…… | **同样的东西**，只是换了入口 |
| 认证 | 浏览器登录（用户名+密码/OAuth） | 本地 token（`gh auth login` 后自动管理） |
| 能力范围 | GitHub 全部功能 | 覆盖 ~90% 高频操作，少数设置仍需网页 |

### 1.3 一个类比

- **GitHub** 就像银行网点（柜台），你带着身份证去排队办理业务。
- **gh** 就像手机银行 App，你在家里点几下就办完了，底层调的是同一套银行系统。

### 1.4 什么时候用 gh、什么时候用网页

| 场景 | 推荐方式 | 原因 |
|---|---|---|
| 创建 PR / 查看 PR diff | gh | `gh pr create` 一条命令搞定，不用切窗口 |
| 批量关闭/标签 Issue | gh | 脚本批量操作，网页要点很多次 |
| 查看 CI 运行日志 | gh | `gh run view`，还能 `--log-failed` 只看失败 |
| 创建 Release 并上传产物 | gh | `gh release create` 一条命令上传多文件 |
| 克隆仓库 | 两者都可 | `gh repo clone` 自动用你的认证，不用配 SSH key |
| 修改仓库 Settings（如改默认分支、配 Secrets） | 网页 | gh 不支持这些操作 |
| 管理 GitHub Actions 密钥 | 网页 | 需要在仓库 Settings → Secrets 里加 |

一句话：**日常高频操作用 gh，管理类配置用网页。**

## 二、安装与认证

### 2.1 安装

```bash
# macOS
brew install gh

# Ubuntu / Debian
curl -fsSL https://cli.github.com/packages/githubcli-archive-keyring.gpg | sudo dd of=/usr/share/keyrings/githubcli-archive-keyring.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/githubcli-archive-keyring.gpg] https://cli.github.com/packages stable main" | sudo tee /etc/apt/sources.list.d/github-cli.list > /dev/null
sudo apt update && sudo apt install gh

# Windows (PowerShell)
winget install --id GitHub.cli
# 或
scoop install gh
# 或
choco install gh
```

验证：

```bash
gh --version
# gh version 2.100.0 (2026-09-03)
```

### 2.2 认证

```bash
gh auth login
```

交互式引导会让你选：

1. **GitHub.com** 还是 GitHub Enterprise → 选 GitHub.com
2. **协议** → HTTPS（推荐，不需要配 SSH key）或 SSH
3. **认证方式** → 浏览器登录（推荐）或粘贴 token

浏览器登录方式：终端会给你一个 8 位验证码，自动打开浏览器，粘贴验证码后点 Authorize 即可。认证信息保存在本地，以后所有 `gh` 命令自动携带。

验证认证状态：

```bash
gh auth status
# ✓ Logged in to github.com account KernelDriver-star
#   - Active account: true
#   - Git operations protocol: https
#   - Token: gho_***********************************
#   - Token scopes: repo, read:org, workflow, etc.
```

> **Token 权限**：`gh auth login` 给的 token 默认包含 `repo`、`read:org`、`workflow` 等权限。如果你后续用 `gh workflow run` 触发 Actions 发现没权限，检查一下 token scopes 是否包含 `workflow`。

## 三、仓库操作（gh repo）

### 3.1 克隆仓库

```bash
# 克隆自己的仓库（不用写完整 URL，不用配 SSH key）
gh repo clone KernelDriver-src

# 克隆别人的
gh repo clone cli/cli
```

### 3.2 创建仓库

```bash
# 把当前本地目录推到 GitHub 新仓库
gh repo create my-project --public --source=. --push

# 在 GitHub 上创建空仓库（不克隆到本地）
gh repo create my-project --public --clone

# 私有仓库 + 添加描述
gh repo create my-project --private --description "我的实验项目"
```

### 3.3 查看仓库信息

```bash
# 仓库基本信息（描述、star、语言、默认分支等）
gh repo view KernelDriver-star/KernelDriver-src

# 直接在浏览器打开仓库首页
gh repo view --web
```

### 3.4 Fork

```bash
gh repo fork cli/cli --clone
# fork 的同时克隆到本地
```

## 四、Issue 管理（gh issue）

### 4.1 查看 Issue

```bash
# 列出分配给你的 open issue
gh issue list --assignee @me

# 查看某个 issue 详情（标题、正文、评论）
gh issue view 42

# 在浏览器打开
gh issue view 42 --web
```

### 4.2 创建 Issue

```bash
# 交互式（自动打开编辑器写正文）
gh issue create --title "修复登录页空白" --body "复现步骤：..."

# 从文件读正文
gh issue create --title "Bug: crash on startup" --body-file ./bug-report.md

# 指定标签和负责人
gh issue create --title "文档更新" --label "documentation" --assignee "@me"

# v2.99.0 新功能：附带截图
gh issue create --title "UI 错位" --body "见截图" --attach ./screenshot.png
```

> `--attach` 是 v2.99.0 引入的功能，支持 PNG/JPEG/GIF/WebP/SVG/MP4/MOV/WebM，图片 ≤10MB，视频 ≤10MB（免费版）或 ≤100MB（付费版）。重复使用 `--attach` 可一次传多个文件。

### 4.3 关闭 / 评论 Issue

```bash
gh issue close 42 --comment "已修复，请验证"
gh issue comment 42 --body "我在本地复现了，正在排查"
gh issue reopen 42
```

### 4.4 批量操作（配合脚本）

```bash
# 批量给带有 "bug" 标签的 issue 加上 "priority:high"
gh issue list --label "bug" --json number --jq '.[].number' | ForEach-Object { gh issue edit $_ --add-label "priority:high" }
```

## 五、Pull Request（gh pr）

这是 `gh` 用得最多的场景。

### 5.1 创建 PR

```bash
# 最简形式（当前分支 → 默认分支）
gh pr create --title "重构内存管理模块" --body "变更说明..."

# 指定目标分支
gh pr create --base develop --head feature/mmu --title "..." --body "..."

# 从模板创建（自动填充 PR 模板）
gh pr create --fill
# --fill 会从 .github/PULL_REQUEST_TEMPLATE.md 读取模板

# 附带前后对比截图
gh pr create --title "UI 优化" --body "前后对比见附件" --attach ./before.png --attach ./after.png
```

### 5.2 查看 PR

```bash
# 列出我的 PR
gh pr list --assignee @me

# PR 详情（含 diff、review 状态、CI 状态）
gh pr view 15

# 只看 diff
gh pr diff 15

# CI 检查状态
gh pr checks 15
# ✓  build / compile (pull_request) Successful
# ✓  test / unit (pull_request) Successful
# ✗  lint (pull_request) Failed
```

### 5.3 Review 和合并

```bash
# 审查（approve / request changes / comment）
gh pr review 15 --approve --body "LGTM"
gh pr review 15 --request-changes --body "请修复 XX 问题"

# 合并（三种策略）
gh pr merge 15 --squash --delete-branch    # squash 合并后删分支（最常用）
gh pr merge 15 --merge                     # 创建 merge commit
gh pr merge 15 --rebase                    # rebase 合并
```

### 5.4 Checkout 别人的 PR

```bash
# 把 PR #15 的分支拉到本地
gh pr checkout 15
# 改完之后
gh pr checkout main   # 切回主分支
```

## 六、GitHub Actions（gh run / gh workflow）

### 6.1 查看运行状态

```bash
# 最近的运行
gh run list --limit 10

# 某个 workflow 的运行
gh run list --workflow=ci.yml

# 失败的运行
gh run list --status failure
```

### 6.2 查看运行详情

```bash
# 概览（含每个 job 的状态）
gh run view 12345

# 只看失败的日志（超实用！）
gh run view 12345 --log-failed

# 看完整日志
gh run view 12345 --log
```

> `--log-failed` 是日常排查 CI 失败的利器——不用打开浏览器翻日志，直接在终端看报错。

### 6.3 手动触发和重跑

```bash
# 触发 workflow_dispatch
gh workflow run deploy.yml --ref main

# 重跑失败的 job
gh run rerun 12345 --failed

# 取消正在跑的
gh run cancel 12345
```

## 七、Release（gh release）

### 7.1 创建 Release

```bash
# 最简形式（给最新 tag 创建 release）
gh release create v1.0.0 --title "v1.0.0" --notes "首个正式版本"

# 上传产物（二进制 / 压缩包）
gh release create v1.0.0 \
  --title "v1.0.0" \
  --notes "首个正式版本" \
  ./dist/app-linux-amd64 \
  ./dist/app-windows-amd64.exe \
  ./dist/app-darwin-arm64

# 从 CHANGELOG 生成 release notes
gh release create v1.0.0 --generate-notes
```

> `--generate-notes` 会自动根据 PR 标题生成变更日志，非常省事。

### 7.2 下载 Release

```bash
# 下载最新 release 的所有产物
gh release download --pattern "*linux*"

# 下载指定版本的某个文件
gh release download v1.2.0 --pattern "app-linux-amd64"

# 下载源码包
gh release download v1.0.0 --archive zip
```

## 八、API 直调（gh api）

当 `gh` 内置命令不够用时，可以直接调 GitHub API：

```bash
# REST API
gh api repos/KernelDriver-star/KernelDriver-src/commits --jq '.[0].commit.message'

# GraphQL API
gh api graphql -f query='
{
  viewer {
    login
    repositories(first: 5) {
      nodes { nameWithOwner stargazerCount }
    }
  }
}'

# 用 jq 提取字段
gh api repos/cli/cli/pulls --jq '.[].title'
```

> `gh api` 自动携带你的认证 token，不用手动加 Header。支持 `--paginate`（自动翻页）、`-X`（指定 HTTP 方法）、`-f`（表单字段）等。

## 九、扩展（gh extension）

`gh` 支持社区扩展，相当于插件系统：

```bash
# 列出已安装扩展
gh extension list

# 安装扩展（从 GitHub 仓库）
gh extension install dlvhdr/gh-dash      # 仪表盘，终端看 PR/Issue 概览
gh extension install cli/gh-extension-precompile  # 预编译辅助

# v2.100.0 新增：webhook 官方扩展
gh extension install github/gh-webhook
```

> v2.100.0 还引入了实验性的 `api_host` 配置，可以把 API 流量路由到自定义网关（企业级用途）：
> ```bash
> gh config set api_host gh-gateway.example.com --host github.com
> ```

## 十、常用技巧

### 10.1 --json + --jq：数据提取

几乎所有 `gh` 命令都支持 JSON 输出，配合 `jq` 提取你想要的字段：

```bash
# 列出所有仓库的名称和 star 数
gh repo list --json nameWithOwner,stargazerCount --jq '.[] | "\(.nameWithOwner): \(.stargazerCount) stars"'

# 列出 PR 的所有 reviewer
gh pr view 15 --json reviews --jq '.reviews[].author.login'
```

### 10.2 --jq 省去管道

`--jq` 是内置的，不需要装 `jq` 命令：

```bash
# 不用 --jq（需要管道）
gh pr list --json number,title | jq -r '.[] | "#\(.number) \(.title)"'

# 用 --jq（更简洁）
gh pr list --json number,title --jq '.[] | "#\(.number) \(.title)"'
```

### 10.3 在脚本中使用 gh

```bash
# CI 脚本：如果构建产物存在，自动创建 release
if [ -f ./dist/app ]; then
  gh release create "$VERSION" --generate-notes ./dist/app
fi

# 自动给 PR 加标签
gh pr create --title "feat: add GPU profiler" --body "..." --label "enhancement"
```

### 10.4 设置默认仓库

如果你经常在某个仓库下操作，不用每次写 `-R owner/repo`：

```bash
# 在仓库目录下执行（会读取 .git/config 的 remote）
cd ~/code/KernelDriver-src
gh pr list    # 自动识别为 KernelDriver-star/KernelDriver-src
```

> 如果你的 git remote 指向代理而非 github.com，每次都要加 `-R owner/repo`。

### 10.5 gh config：全局配置

```bash
# 查看所有配置
gh config list

# 设置默认编辑器
gh config set editor "code --wait"

# 设置 git 协议
gh config set git_protocol https
```

## 十一、速查表

| 命令 | 作用 |
|---|---|
| `gh auth login` / `gh auth status` | 登录 / 查看认证状态 |
| `gh repo clone <name>` | 克隆仓库 |
| `gh repo create <name> --public --source=. --push` | 创建仓库并推送本地代码 |
| `gh repo view [--web]` | 查看仓库信息 / 浏览器打开 |
| `gh issue list --assignee @me` | 列出分配给我的 Issue |
| `gh issue create --title "..." --body "..."` | 创建 Issue |
| `gh issue view <N> [--web]` | 查看 Issue |
| `gh issue close <N>` / `gh issue comment <N> --body "..."` | 关闭 / 评论 Issue |
| `gh pr create --title "..." --body "..."` | 创建 PR |
| `gh pr list --assignee @me` | 列出我的 PR |
| `gh pr view <N>` / `gh pr diff <N>` | 查看 PR 详情 / diff |
| `gh pr checks <N>` | 查看 PR 的 CI 检查状态 |
| `gh pr review <N> --approve` | 审查通过 |
| `gh pr merge <N> --squash --delete-branch` | Squash 合并并删分支 |
| `gh pr checkout <N>` | 拉取 PR 分支到本地 |
| `gh run list` / `gh run view <N>` | 列出 / 查看 CI 运行 |
| `gh run view <N> --log-failed` | 只看失败的 CI 日志 |
| `gh run rerun <N> --failed` | 重跑失败的 job |
| `gh workflow run <file>` | 手动触发 workflow |
| `gh release create <tag>` | 创建 Release |
| `gh release create <tag> --generate-notes` | 自动生成变更日志 |
| `gh release download <tag>` | 下载 Release 产物 |
| `gh api <endpoint> --jq '.field'` | 直接调 GitHub API |
| `gh extension install <repo>` | 安装社区扩展 |
| `gh config set <key> <value>` | 设置全局配置 |

## 小结

`gh` 不是 GitHub 的替代品，而是**加速器**。GitHub 平台提供全部功能，`gh` 让你在终端里高频操作时不用切窗口、不用等网页加载、还能写脚本批量自动化。

核心思路：**人机交互用 `gh`（创建 PR、查 CI、关 Issue），管理配置用网页（仓库 Settings、Secrets、Branch protection）。** 把 `gh pr create`、`gh run view --log-failed`、`gh release create --generate-notes` 这三板斧用熟，日常工作效率就能提升一大截。
