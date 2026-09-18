---
title: GitHub Actions 工作流程详解：从概念到自动部署
date: 2026-09-17 10:30:00
categories:
  - 开发工具
tags:
  - GitHub Actions
  - CI/CD
  - 自动化
  - DevOps
---

GitHub Actions 是 GitHub 内置的 **CI/CD 和自动化平台**：当仓库发生某个事件（push、提 PR、发 Release、定时……），GitHub 自动在一台云端机器上按你写的 YAML 文件执行一系列任务——装依赖、跑测试、构建、部署，全程不用你手动操作。本文从核心概念讲到完整实战（最后会给出本站 Hexo 博客的自动部署配置），读完就能给自己的项目接上自动化（版本以 2026 年主流稳定版为准：`actions/checkout@v5`、`actions/setup-node@v5`）。

## 一、先搞懂五个核心概念

Actions 的所有知识都围绕这五个词：

| 概念 | 含义 | 类比 |
|---|---|---|
| **Workflow（工作流）** | 一个 `.yml` 文件，描述"什么时候、干什么" | 一份完整的施工方案 |
| **Event（事件）** | 触发 workflow 的条件：push、PR、release、schedule…… | 开工哨声 |
| **Job（任务）** | workflow 里的一组步骤，默认多个 job **并行**跑在不同虚拟机上 | 施工队的一个班组 |
| **Step（步骤）** | job 里的一条命令或一个 action，**按顺序串行**执行 | 班组干的每一道工序 |
| **Runner（运行器）** | 实际执行任务的机器，GitHub 提供 Ubuntu/Windows/macOS 云端机，也可自建 | 工地和工具 |
| **Action（动作）** | 可复用的"步骤零件"，比如"拉代码""装 Node"，别人写好你直接 `uses` | 预制构件，不用自己砌砖 |

它们的层级关系：

```text
Workflow (.github/workflows/ci.yml)
├── Event: on: push / pull_request / schedule ...
└── Jobs
    ├── Job A (runs-on: ubuntu-latest)
    │   ├── Step 1: uses: actions/checkout@v5
    │   ├── Step 2: run: npm ci
    │   └── Step 3: run: npm test
    └── Job B (needs: Job A)   ← 依赖 A，A 成功后才跑
        └── Steps ...
```

## 二、一个最小 Workflow，逐行拆解

在仓库根目录创建 `.github/workflows/ci.yml`（文件名随意，目录固定）：

```yaml
# 工作流的显示名称（Actions 页面看到的名字）
name: CI

# 触发器：什么事件发生时运行
on:
  push:
    branches: [main]        # push 到 main 分支
  pull_request:
    branches: [main]        # 向 main 提 PR 或 PR 更新

# 给自动令牌的权限（最小权限原则，后面细讲）
permissions:
  contents: read

# 任务集合
jobs:
  test:                     # job 的 ID（自己起名）
    name: 运行测试           # 显示名称
    runs-on: ubuntu-latest  # 在最新版 Ubuntu 虚拟机上跑

    steps:                  # 步骤，从上往下串行
      - name: 拉取代码
        uses: actions/checkout@v5       # 官方 action：把仓库代码 checkout 到机器上

      - name: 安装 Node.js
        uses: actions/setup-node@v5    # 官方 action：装指定版本 Node
        with:
          node-version: 22
          cache: npm                    # 自动缓存 npm 依赖

      - name: 安装依赖
        run: npm ci                     # run：直接执行 shell 命令

      - name: 运行测试
        run: npm test
```

把这个文件 commit 并 push，GitHub 立刻就会运行它。在仓库页点 **Actions** 标签就能看到每次运行的状态、点开每一步的实时日志。

> **第一次使用只需要记住三件事**：文件放 `.github/workflows/`；`on` 决定何时跑；`steps` 里 `uses` 是用别人的零件、`run` 是自己敲命令。

## 三、触发器（on）详解

`on` 是 workflow 的灵魂，常用触发器：

```yaml
on:
  # 1. push 到任意分支 / 指定分支 / 指定路径
  push:
    branches: [main, 'release/**']
    paths: ['src/**', 'package.json']   # 只有这些路径变化才触发
    paths-ignore: ['docs/**', '*.md']   # 或忽略这些路径

  # 2. Pull Request
  pull_request:
    types: [opened, synchronize, reopened]
    branches: [main]

  # 3. 发布 Release
  release:
    types: [published]

  # 4. 定时任务（cron 是 UTC 时间！）
  schedule:
    - cron: '0 1 * * *'     # UTC 01:00 = 北京时间 09:00

  # 5. 手动触发（Actions 页面点 "Run workflow" 按钮）
  workflow_dispatch:
    inputs:                       # 还可以传参数
      environment:
        description: '部署环境'
        type: choice
        options: [staging, production]
        default: staging

  # 6. Issue / 评论（配合 Cline 自动排查就是用这个）
  issues:
    types: [opened]
  issue_comment:
    types: [created]
```

## 四、Job 的编排：并行、依赖、条件、矩阵

### 4.1 串行依赖：needs

默认所有 job 并行。用 `needs` 声明依赖，形成 DAG（有向无环图）：

```yaml
jobs:
  lint:
    runs-on: ubuntu-latest
    steps: [{ run: npm run lint }]
  test:
    runs-on: ubuntu-latest
    steps: [{ run: npm test }]
  deploy:
    needs: [lint, test]      # lint 和 test 都成功后才跑 deploy
    runs-on: ubuntu-latest
    steps: [{ run: ./deploy.sh }]
```

### 4.2 条件执行：if

```yaml
steps:
  - name: 部署到生产
    if: github.ref == 'refs/heads/main' && github.event_name == 'push'
    run: ./deploy.sh
```

常用的上下文表达式：

| 表达式 | 含义 |
|---|---|
| `github.ref == 'refs/heads/main'` | 当前是 main 分支 |
| `github.event_name == 'pull_request'` | 由 PR 触发 |
| `matrix.os == 'ubuntu-latest'` | 矩阵中的某一项（见下） |
| `success()` | 之前所有步骤都成功（默认行为） |
| `failure()` | 之前有步骤失败（常用于发通知） |
| `always()` | 无论成败都执行（常用于清理、上传日志） |
| `github.actor != 'github-actions[bot]'` | 防止机器人自触发死循环 |

### 4.3 矩阵：一套代码多环境跑

用 `strategy.matrix` 让同一 job 在多个操作系统 / 语言版本上并行展开：

```yaml
jobs:
  test:
    runs-on: ${{ matrix.os }}
    strategy:
      fail-fast: false          # 某个组合失败不要取消其他组合
      matrix:
        os: [ubuntu-latest, windows-latest, macos-latest]
        node: [18, 20, 22]
    steps:
      - uses: actions/checkout@v5
      - uses: actions/setup-node@v5
        with:
          node-version: ${{ matrix.node }}
      - run: npm ci && npm test
```

这会展开成 3 × 3 = **9 个并行 job**，一次性验证三个系统 × 三个 Node 版本的兼容性。

### 4.4 并发控制：concurrency

防止重复运行互相打架（比如连续 push 三次，只保留最新一次）：

```yaml
concurrency:
  group: ci-${{ github.ref }}        # 同一分支只允许一个
  cancel-in-progress: true           # 新的来了就取消旧的
```

> 部署类任务建议 `cancel-in-progress: false`，避免正在部署生产时被中途掐断。

## 五、Action：复用生态零件

### 5.1 怎么用

```yaml
steps:
  - uses: 所有者/仓库@版本          # 引用方式
  # 例：actions/checkout@v5
  # 例：docker/setup-buildx-action@v3
```

- 官方 action 在 `actions/*` 组织下，市场在 [github.com/marketplace?type=actions](https://github.com/marketplace?type=actions)；
- `with` 传输入参数，动作产生的结果可以通过 `id` 在后续步骤拿到。

### 5.2 常用官方 Action 速查（2026 主流版本）

| Action | 作用 |
|---|---|
| `actions/checkout@v5` | 拉取仓库代码（几乎是每个 workflow 的第一步） |
| `actions/setup-node@v5` | 安装 Node.js，支持缓存 |
| `actions/setup-python@v6` | 安装 Python |
| `actions/cache@v5` | 通用缓存（依赖、构建产物） |
| `actions/upload-artifact@v4` | 上传构建产物，供下载或其他 job 使用 |
| `actions/download-artifact@v4` | 下载产物 |
| `actions/github-script@v8` | 在 workflow 里直接写 JS 调 GitHub API |

### 5.3 Step 间传值：outputs

```yaml
steps:
  - id: vars
    run: echo "short_sha=$(git rev-parse --short HEAD)" >> $GITHUB_OUTPUT
  - name: 使用上一步的输出
    run: echo "本次构建版本: ${{ steps.vars.outputs.short_sha }}"
```

## 六、Secrets、Variables 与环境变量

自动化经常需要密钥（部署 token、API key），**绝不能明文写进 YAML**。

### 6.1 配置位置

仓库页 → **Settings** → **Secrets and variables** → **Actions**：

- **Secrets**：加密存储，日志里自动打码，用 `${{ secrets.名字 }}` 引用；
- **Variables**：明文的配置项（如环境地址），用 `${{ vars.名字 }}` 引用。

```yaml
steps:
  - name: 部署
    env:
      API_KEY: ${{ secrets.DEPLOY_API_KEY }}
      SITE_URL: ${{ vars.PRODUCTION_URL }}
    run: ./deploy.sh
```

### 6.2 自动令牌 GITHUB_TOKEN

每个 workflow 运行时 GitHub 会**自动注入**一个临时令牌 `${{ secrets.GITHUB_TOKEN }}`，用于操作本仓库（push、发 release、评论 PR），任务结束自动失效，不用手动创建：

```yaml
permissions:
  contents: write          # 推送代码 / 发 Release
  pull-requests: write     # PR 评论
  pages: write             # 部署 GitHub Pages
  id-token: write          # OIDC 联合登录（见下）
```

> 遵循**最小权限原则**：只给当前任务需要的权限。2023 年后新建仓库默认大部分权限是 read-only，需要写操作时显式声明。

### 6.3 OIDC：不存长期密钥访问云厂商

配置信任关系后，可以用联合身份直接登录 AWS/Azure/GCP，无需在 GitHub 存 Access Key：

```yaml
permissions:
  id-token: write
steps:
  - uses: aws-actions/configure-aws-credentials@v4
    with:
      role-to-assume: arn:aws:iam::123456789012:role/gh-actions-deploy
      aws-region: ap-northeast-1
```

## 七、缓存与产物

### 7.1 缓存依赖，加速构建

```yaml
- uses: actions/cache@v5
  with:
    path: ~/.npm
    key: ${{ runner.os }}-node-${{ hashFiles('**/package-lock.json') }}
    restore-keys: ${{ runner.os }}-node-
```

> `setup-node@v5` 加 `cache: npm` 已经自动做这件事，多数语言的 setup action 同理，不用手写 cache。

### 7.2 Artifacts：产物上传 / 跨 job 传递

```yaml
- name: 构建
  run: npm run build

- name: 上传构建产物
  uses: actions/upload-artifact@v4
  with:
    name: dist-package
    path: dist/
    retention-days: 7       # 保留 7 天
```

上传后可以在该次运行的页面底部手动下载；另一个 job 用 `actions/download-artifact@v4` 取回——典型模式是"build job 和 deploy job 分离，靠 artifact 传包"。

## 八、实战一：Node 项目完整 CI

一个典型的"PR 检查 + main 合并后发版"配置：

```yaml
name: Build & Release

on:
  pull_request:
    branches: [main]
  push:
    branches: [main]
    tags: ['v*']

permissions:
  contents: write

jobs:
  quality:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v5
      - uses: actions/setup-node@v5
        with: { node-version: 22, cache: 'npm' }
      - run: npm ci
      - run: npm run lint
      - run: npm test
      - run: npm run build

  release:
    needs: quality
    if: startsWith(github.ref, 'refs/tags/v')   # 只在打 v 开头的 tag 时执行
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v5
      - uses: actions/setup-node@v5
        with: { node-version: 22, cache: 'npm', registry-url: 'https://registry.npmjs.org' }
      - run: npm ci
      - run: npm publish
        env:
          NODE_AUTH_TOKEN: ${{ secrets.NPM_TOKEN }}
      - name: 生成 GitHub Release
        run: gh release create "$GITHUB_REF_NAME" --generate-notes
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

流程：平时提 PR 自动跑 lint/test/build；打 tag（`git tag v1.0.0 && git push --tags`）后自动发布 npm 包并创建 Release。

## 九、实战二：Hexo 博客自动部署（本站同款思路）

我们目前是在本地执行 `hexo deploy`。有了 Actions，可以做到**只 push Markdown 源码，云端自动构建发布**：

```yaml
# .github/workflows/deploy.yml
name: Deploy Hexo

on:
  push:
    branches: [main]
  workflow_dispatch:

permissions:
  contents: write
  pages: write

concurrency:
  group: pages
  cancel-in-progress: false

jobs:
  build-deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v5
        with:
          submodules: recursive      # 如果主题用 submodule 管理
      - uses: actions/setup-node@v5
        with:
          node-version: 22
          cache: 'npm'
      - run: npm ci
      - name: 构建静态文件
        run: npx hexo clean && npx hexo generate
      - name: 部署到 Pages 仓库
        run: |
          cd public
          git init
          git config user.name "github-actions[bot]"
          git config user.email "41898282+github-actions[bot]@users.noreply.github.com"
          git add -A
          git commit -m "auto deploy: ${{ github.sha }}"
          git branch -M main
          git remote add origin https://x-access-token:${{ secrets.GITHUB_TOKEN }}@github.com/KernelDriver-star/KernelDriver-star.github.io.git
          git push -f origin main
```

> 注意：自动令牌 `GITHUB_TOKEN` 默认**不能跨仓库**推送。上面这种"源码仓库 → Pages 仓库"是两个仓库，需要在源码仓库 Settings → Secrets 里存一个有 `repo` 权限的 **PAT（Personal Access Token）**，把 URL 里的 token 换成 `${{ secrets.PAT }}`。若把 Pages 直接配置在源码仓库的 Settings → Pages → Source: GitHub Actions，则可以用官方的 `actions/deploy-pages` 系列 action，连 PAT 都省了。

## 十、从零开始的使用流程（操作步骤）

以给任意项目添加 CI 为例：

1. **确认 Actions 已开启**：仓库 Settings → General → Features → 勾选 Actions（默认开启）。
2. **创建 workflow 文件**：两种方式任选——
   - 网页：仓库 **Actions** 标签 → 左侧选模板（如 "Node.js"）→ Configure → Commit；
   - 本地：手动建 `.github/workflows/xxx.yml`，写完 commit、push。
3. **观察第一次运行**：push 后 Actions 页面会出现带黄圈的运行记录，点进去看每个 job/step 的实时日志。
4. **配置密钥**（如需要）：Settings → Secrets and variables → Actions → New repository secret。
5. **失败时排查**：点红色失败的 step，看日志报错；修代码后重新 push 即自动重跑；也可以在运行页点 **Re-run all jobs** 手动重跑。
6. **接入分支保护**（团队协作推荐）：Settings → Branches → Add rule，勾选 *Require status checks to pass before merging*，选中你的测试 job——以后测试不过，PR 无法合并。
7. **查看账单**：公开仓库和 self-hosted runner **免费**；私有仓库每月有免费额度（2000 分钟 Linux，Windows/macOS 按倍率计费），在 Settings → Billing 查看用量。

## 十一、常见坑与最佳实践

| 坑 | 正确做法 |
|---|---|
| 直接 `uses: some/action@main` | 固定到大版本 tag（`@v5`）甚至 commit SHA，防止上游更新突然搞坏你的流水线 |
| 在日志里 `echo $SECRET` 调试 | 密钥会被打码但仍可能从上下文泄漏，调试后及时删除 |
| `npm install` 而非 `npm ci` | CI 用 `npm ci`：严格按 lockfile 安装，更快、更可复现 |
| 忘记 `actions/checkout` | 没有代码后续全挂，它几乎总是第一步 |
| schedule 不准时 | cron 用 UTC 时间（北京时间减 8 小时），且高负载时段可能延迟 |
| 矩阵 fail-fast 默认 true | 想看到所有组合的失败结果时设 `fail-fast: false` |
| workflow 自己 push 又触发自己 | 加 `if: github.actor != 'github-actions[bot]'` 防护 |
| 私有仓库 fork PR 拿不到 secret | 这是安全设计，防止外部 PR 窃取密钥；需配合 pull_request_target 谨慎处理 |
| 每个 job 重复装环境 | job 间不共享文件系统，用 cache 加速、用 artifact 传文件 |

## 十二、速查表

| 需求 | 写法 |
|---|---|
| 触发：push main | `on: push: branches: [main]` |
| 触发：PR | `on: pull_request` |
| 触发：手动按钮 | `on: workflow_dispatch` |
| 触发：定时（北京 9 点） | `on: schedule: - cron: '0 1 * * *'` |
| 运行环境 | `runs-on: ubuntu-latest`（或 windows / macos） |
| Job 依赖 | `needs: [jobA, jobB]` |
| 条件 | `if: github.ref == 'refs/heads/main'` |
| 多版本矩阵 | `strategy.matrix` |
| 拉代码 | `uses: actions/checkout@v5` |
| 装 Node | `uses: actions/setup-node@v5 with: node-version: 22` |
| 执行命令 | `run: npm test`（多行用 `|`） |
| 引用密钥 | `${{ secrets.NAME }}` |
| 引用普通变量 | `${{ vars.NAME }}` |
| 自动令牌 | `${{ secrets.GITHUB_TOKEN }}` |
| 上传产物 | `uses: actions/upload-artifact@v4` |
| 步骤间传值 | `echo "k=v" >> $GITHUB_OUTPUT` |
| 并发控制 | `concurrency: { group, cancel-in-progress }` |
| 权限声明 | 顶层 `permissions:` |

## 小结

GitHub Actions 的心智模型可以浓缩成一句话：**Event 触发 Workflow，Workflow 包含若干 Job，Job 在 Runner 上按 Step 执行，Step 要么 `run` 命令、要么 `uses` 复用 Action**。

上手路径建议：

1. 先给项目加一个只会 `lint + test` 的 CI，熟悉 push 触发、看日志、失败重跑；
2. 再学 Secrets、矩阵、产物，把测试矩阵和构建流水线搭起来；
3. 最后接入部署（Release、Pages、服务器、云厂商 OIDC），形成"push 即上线"的完整 CI/CD 闭环。

当这套流程跑顺之后，你的日常发布就只剩"写代码 → push"，剩下的检查、构建、分发全部由 Actions 自动完成——这正是 CI/CD 的核心价值：**把重复的劳动交给机器，把精力留给代码本身。**
