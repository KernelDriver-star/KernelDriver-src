---
title: Cline 使用指南：AI 编程代理的自动化流程与 GitHub 部署
date: 2026-09-17 15:00:00
categories:
  - 开发工具
tags:
  - Cline
  - AI编程
  - GitHub Actions
  - 自动化
---

`Cline`（前身叫 Claude Dev）是一个开源的 AI 编程代理（AI Coding Agent），以 VS Code / JetBrains 插件或命令行工具的形式运行。和代码补全工具不同，它不是"你写一半它补一行"，而是能**自主读代码、改文件、跑命令、开浏览器验证、根据报错自我修正**，并把每一步操作摊开给你审批。本文讲清楚三件事：Cline 的自动化流程是怎么转起来的、本地如何用它开发并推送到 GitHub、以及如何把它部署到 GitHub Actions 实现无人值守的 CI/CD（内容以 [Cline 官方文档](https://docs.cline.bot/) 为准）。

## 一、Cline 是什么

- **开源**：Apache 2.0 协议，仓库 [cline/cline](https://github.com/cline/cline)，68k+ stars，全平台 11M+ 安装。
- **形态**：VS Code 扩展（4.5M+ 安装）、JetBrains 插件（早期访问）、独立 Desktop 应用、Cline CLI（终端）、SDK（嵌入自己的产品）。
- **自带密钥（BYOK）**：Cline 本身免费，你只需要给它配一个模型——Claude、GPT、Gemini、DeepSeek、OpenRouter，或本地的 Ollama / LM Studio 都可以，费用直接付给模型厂商，Cline 不收平台费。

### 1.1 和 GitHub Copilot / Cursor 有什么区别

| | Copilot（补全） | Cursor（AI 编辑器） | Cline（Agent） |
|---|---|---|---|
| 交互方式 | 行内自动补全 | Chat + Composer | 对话驱动的**任务执行** |
| 能否跑终端命令 | 不能 | 有限 | 可以，且能读回输出继续推理 |
| 能否多文件协调修改 | 弱 | 可以 | 核心能力，linter 感知 |
| 能否开浏览器自测 | 不能 | 不能 | 可以（驱动 Chromium 截图、读 DOM） |
| 操作审批 | 无 | 部分 | **每次写文件/执行命令前都可批准/拒绝** |
| 开源 / 模型自由 | 闭源 / 绑定 | 闭源 / 绑定 | 开源 / 任意模型 |
| CI 无头运行 | 不支持 | 不支持 | 原生支持（Cline CLI） |

一句话：Copilot 是"自动补全"，Cline 是"你派活、它干活、每步给你看 diff 的初级工程师"。

## 二、安装与配置

### 2.1 VS Code 扩展（最常用）

1. 在 VS Code 扩展市场搜索 **Cline**（发布者 saoudrizwan，标识 `claude-dev`）并安装；
2. 打开侧边栏的 Cline 图标，选择模型提供商并填入 API Key；
3. 比如用 Anthropic：填 `ANTHROPIC_API_KEY`，模型选 Claude Sonnet（编码能力和成本平衡最好）；预算敏感可选 DeepSeek，本地隐私优先可选 Ollama。

### 2.2 Cline CLI（终端 / CI 必备）

Cline CLI 和插件共用同一个 agent 核心，Plan/Act、MCP、检查点、规则的行为完全一致：

```bash
# 需要 Node.js 22+
npm install -g cline

# 认证（交互式，支持 Cline 账号、ChatGPT 订阅或自有 API Key）
cline auth

# 非交互式认证（CI 环境用）
cline auth --provider anthropic --apikey sk-xxxx --modelid claude-sonnet-4-6
```

验证安装：

```bash
cline --version
cline doctor     # 诊断配置问题
```

## 三、Cline 的自动化流程（核心）

### 3.1 Agent Loop：它到底是怎么"自己干活"的

Cline 接到一个任务后，内部是一个不断循环的 **Agent Loop（代理循环）**：

```text
你下达任务
   │
   ▼
┌─────────────────────────────────────┐
│ 1. 思考：读哪些文件、用什么方案      │
├─────────────────────────────────────┤
│ 2. 调用工具：读文件 / 写文件 /       │
│    跑命令 / 开浏览器 / 调 MCP        │
├─────────────────────────────────────┤
│ 3. 观察：拿到命令输出、报错、截图     │
├─────────────────────────────────────┤
│ 4. 决策：成功就继续下一步，          │
│    失败就根据报错换方案再来一遍      │
└──────────────┬──────────────────────┘
               └──── 循环直到任务完成
```

关键在于第 3 步——它跑 `npm test` 失败了，会**读测试报错、改代码、再跑一遍**，不需要你转述错误。这是它和"聊天机器人给你贴一段代码让你自己试"的本质区别。

### 3.2 Plan / Act 两种模式

| 模式 | 行为 | 适用场景 |
|---|---|---|
| **Plan（计划）** | 只读代码、调研、提问、输出实施方案，**不动任何文件** | 复杂任务先对齐思路 |
| **Act（执行）** | 真正写文件、跑命令 | 方案确认后干活 |

典型用法：先在 Plan 模式让它分析"把项目从 Webpack 迁到 Vite 需要改哪些地方"，你审完计划，切到 Act 模式让它执行。CLI 里对应 `-p` / `-a` 参数：

```bash
cline -p "design the migration plan"    # 只规划
cline -a "apply the migration"          # 直接执行（默认）
```

### 3.3 它有哪些工具

- **读 / 写文件**：写文件前展示 diff，支持跨多文件协调修改，并感知 linter 报错自动修。
- **终端**：在你的真实终端里执行命令（`npm install`、`pytest`、`git push`……），长驻的 dev server 也能管理。
- **浏览器**：启动 Chromium 实例，打开它刚写的页面，截图、读 DOM、点击操作，发现 UI 不对会自我修正。
- **MCP 工具**：通过 [Model Context Protocol](https://modelcontextprotocol.io/) 接数据库、内部 API、Jira、云平台等外部系统，内置 MCP Marketplace 可直接浏览安装。

### 3.4 安全护栏：审批、检查点、规则

**（1）逐步审批 / 自动批准**

默认每次操作都要你点 Approve / Reject。信任流程后，可以按操作类型开启自动批准（比如自动放行"读文件"，写文件和删文件仍需手动确认）。CLI 中全自动模式叫 **YOLO**：

```bash
cline --yolo "run tests and fix failures"
# 等价于 --auto-approve true，无人值守时用
```

> YOLO 会无提示地改文件、执行任意命令。**务必在干净的 git 分支上跑**，方便一键回滚。

**（2）Checkpoints（检查点）**

每一次工具调用都会自动生成一个工作区快照（独立于 git）。改坏了不用 `git reset`，直接在对话里输入：

```text
/undo
```

即可回到上一个检查点。

**（3）@提及**

在对话框用 `@` 直接把文件、文件夹、当前 Problems 面板错误或 git diff 喂给它，不用复制粘贴：

```text
@src/mmu.c 这个文件里的页表映射逻辑有问题，帮我排查
```

**（4）.clinerules 项目规则**

在仓库根目录放 `.clinerules` 文件，把团队规范写进去，Cline 每次任务都会遵守：

```text
# 编码规范
- C 代码遵循 Linux kernel style（缩进用 tab）
- 所有新函数必须写 kernel-doc 注释
- 不要修改 include/uapi/ 下的用户态接口
- 提交信息格式：drivers/gpu: xxxxx
```

规则文件随仓库走，团队每个人（和每个 CI 里的 Cline）行为都一致。

**（5）Skills 与 Hooks**

- **Skills**：把可复用的经验打包（"怎么写我们的 PR""怎么跑测试套件"）。
- **Hooks**：用脚本对每次工具调用做拦截——允许读、写需审批、禁止碰生产环境：

```bash
export CLINE_COMMAND_PERMISSIONS='{"allow": ["npm *", "git *"], "deny": ["rm -rf *", "sudo *"]}'
```

## 四、本地开发并推送到 GitHub

下面走一遍完整链路：用 Cline 开发功能 → 提交 → 推送到 GitHub → 开 PR。

### 4.1 准备：Git 认证

Cline 底层就是调用你机器上的 `git`，所以先确保本机能正常推送（SSH key 或 HTTPS 凭据二选一）：

```bash
git --version
ssh -T git@github.com    # 看到 Hi KernelDriver-star! 即 SSH 通
```

### 4.2 让 Cline 完成开发

在 VS Code 的 Cline 对话框里直接下达任务，它会经历"读代码 → 改文件 → 跑测试 → 修报错"的完整循环：

```text
给驱动加一个 dma_bench 的 debugfs 节点：
1. 在 drivers/gpu/dma.c 里实现 show/store 回调
2. 注册到 /sys/kernel/debug/dma_bench
3. 跑 make 确认编译通过
```

执行中你会逐个看到 diff 和命令审批点，编译报错它会自己改完再编译，直到通过。

### 4.3 让 Cline 提交并推送

代码满意后，继续在同一个对话里说：

```text
把改动提交了，commit message 用 "drivers/gpu: add dma_bench debugfs node"，然后 push 到 origin 的当前分支
```

Cline 会自己执行：

```bash
git status
git add drivers/gpu/dma.c
git commit -m "drivers/gpu: add dma_bench debugfs node"
git push origin HEAD
```

每一步都在你的终端真实执行，输出会回到它的上下文。

### 4.4 配合 gh CLI 开 PR（推荐组合）

如果本机装了 [GitHub CLI](https://cli.github.com/)（见上一篇 [gh CLI 使用指南](/2026/09/17/gh-CLI-使用指南/)），Cline 可以直接调 `gh` 完成建仓、开 PR：

```text
这是个新项目，帮我在 GitHub 上创建私有仓库并推送上去
```

```bash
# Cline 实际执行
gh repo create dma-bench --private --source=. --push
```

或者功能分支开发完：

```bash
gh pr create --title "drivers/gpu: add dma_bench debugfs node" \
  --body "## 变更内容
- 新增 dma_bench debugfs 节点
- 提供吞吐量读写统计
## 测试
- 本地 make 编译通过"
```

> Cline + gh 是非常顺手的组合：Cline 负责"写代码 + 跑 git"，gh 负责"和 GitHub 平台打交道"，两者都不用开浏览器。

### 4.5 Worktree：多个任务并行不打架

CLI 支持给每个任务开独立的 git worktree，多个代理并行干活时不会在同一分支上互相覆盖：

```bash
cline --worktree "migrate to flat config"
```

## 五、把 Cline 部署到 GitHub Actions（无人值守）

这是"部署上线到 GitHub"的第二种含义：让 Cline 跑在 GitHub Actions 里，事件触发后自动分析代码、审查 PR、回复 Issue。Cline CLI 的 **headless 模式**就是为此设计的。

### 5.1 Headless 模式原理

满足以下任一条件，Cline 自动进入无界面的 headless 模式：

| 调用方式 | 触发原因 |
|---|---|
| `cline --yolo "task"` | 全自动批准 |
| `cline --json "task"` | JSON 输出，便于脚本解析 |
| `git diff \| cline "task"` | stdin 是管道 |
| `cline "task" > out.txt` | stdout 被重定向 |

CI 里的标准形态：

```bash
cline --json --auto-approve true "你的任务"
```

### 5.2 实战：PR 自动代码审查

在仓库里新建 `.github/workflows/cline-review.yml`：

```yaml
name: AI Code Review

on:
  pull_request:
    types: [opened, synchronize]

permissions:
  contents: read
  pull-requests: write

jobs:
  review:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0          # 拉全历史，才能拿到完整 diff

      - uses: actions/setup-node@v4
        with:
          node-version: '22'

      - name: Install Cline CLI
        run: npm install -g cline

      - name: Configure authentication
        env:
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
        run: cline auth --provider anthropic --apikey "$ANTHROPIC_API_KEY"

      - name: Review the pull request
        env:
          PR_NUMBER: ${{ github.event.pull_request.number }}
        run: |
          git diff origin/${{ github.base_ref }}...HEAD | \
          cline --yolo "请审查这些改动：指出潜在 bug、内存安全问题和风格问题，
                        用中文输出 Markdown 格式的审查意见" > review.md

      - name: Post review as comment
        run: gh pr comment "$PR_NUMBER" --body-file review.md
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          PR_NUMBER: ${{ github.event.pull_request.number }}
```

工作流说明：

1. **触发**：每次 PR 新建或更新；
2. **认证**：API Key 存在仓库的 Settings → Secrets and variables → Actions 里，名字如 `ANTHROPIC_API_KEY`（用 OpenRouter 则存 `OPENROUTER_API_KEY`，认证命令换成 `cline auth --provider openrouter --apikey ...`）；
3. **喂上下文**：把 PR 的 diff 通过管道喂给 Cline；
4. **全自动**：`--yolo` 让它在 CI 里无需人工点审批；
5. **回写评论**：审查结果用官方自带的 `GITHUB_TOKEN` 通过 `gh` 发回 PR。

提交这个文件并 push 后，后续每个 PR 都会自动收到一条 AI 审查评论。

### 5.3 实战：Issue 里 @cline 自动排查

Cline 官方提供了 [GitHub Integration 样例](https://docs.cline.bot/cli/samples/github-integration)：在 Issue 评论里 `@cline`，就触发一个 Action，Cline 自动读仓库代码、定位问题、把分析结果回复到 Issue。核心触发逻辑：

```yaml
on:
  issue_comment:
    types: [created, edited]

permissions:
  issues: write

jobs:
  respond:
    runs-on: ubuntu-latest
    steps:
      - name: Check for @cline mention
        id: detect
        uses: actions/github-script@v7
        with:
          script: |
            const body = context.payload.comment?.body || ""
            const hit = body.toLowerCase().includes("@cline")
            core.setOutput("hit", hit ? "true" : "false")
      # hit 为 true 时：checkout → setup-node 22 → npm i -g cline
      # → cline auth → cline --yolo "分析这个 issue..." → gh issue comment 回写
```

完整 workflow 可以从官方样例仓库直接拷贝：

```bash
mkdir -p .github/workflows
curl -o .github/workflows/cline-responder.yml \
  https://raw.githubusercontent.com/cline/cline/main/src/samples/cli/github-integration/cline-responder.yml
```

### 5.4 配置 Secrets 的步骤

1. GitHub 仓库页 → **Settings** → **Secrets and variables** → **Actions**；
2. 点 **New repository secret**；
3. Name 填 `ANTHROPIC_API_KEY`（或 `OPENROUTER_API_KEY`），Secret 填你的 Key；
4. 在 workflow 里用 `${{ secrets.ANTHROPIC_API_KEY }}` 引用——日志里会自动打码，不会泄漏。

### 5.5 本地脚本与定时任务

Cline CLI 也能直接塞进本地 shell 脚本和 cron：

```bash
# 提交前让 Cline 先审一遍本地改动
git diff | cline "review these changes for regressions"

# 链式：先解释改动，再生成本次 commit message
git diff | cline -y "explain these changes" | cline -y "write a commit message"

# 定时任务（CLI 内置 schedule）
cline schedule create "每天早上检查依赖漏洞并提 PR" --cron "0 9 * * 1-5"
```

JSON 输出方便用 `jq` 解析：

```bash
cline --json "list all TODO comments" | jq -r '.text'
```

## 六、安全与最佳实践

| 建议 | 原因 |
|---|---|
| 本地首次使用保持逐步审批 | 看清它对每个文件做了什么，别一上来 YOLO |
| CI / YOLO 必须在新分支或 worktree 上跑 | 出问题随时丢弃分支，保护主分支 |
| 用 `.clinerules` 写清禁区 | 如"禁止改 uapi 接口""禁止 force push main" |
| 用 `CLINE_COMMAND_PERMISSIONS` 限制命令白名单 | CI 里只放行 `npm *`、`git *`，拒绝 `rm -rf`、`sudo` |
| API Key 只存 GitHub Secrets，不要写进 workflow | 写进 yaml 等于公开泄露 |
| 给 CI 的 PAT 最小权限 | 只用官方 `GITHUB_TOKEN` 能满足就别用个人 PAT |
| 给任务设超时 | `cline --timeout 600 "..."`，避免死循环烧 token |
| 任务结束检查 git diff | 它的改动最终由你负责，合入前自己过一遍 |

## 七、速查表

| 命令 / 功能 | 作用 |
|---|---|
| VS Code 扩展市场搜 Cline | 安装 IDE 插件 |
| `npm install -g cline` | 安装 Cline CLI（需 Node 22+） |
| `cline auth` / `cline doctor` | 认证 / 诊断配置 |
| Plan 模式 / Act 模式 | 先只读规划，确认后再执行 |
| `@文件/文件夹/@problems` | 把上下文直接塞进对话 |
| `/undo` | 回到上一个检查点 |
| `.clinerules` | 随仓库走的项目规则 |
| `cline -p "..."` / `cline -a "..."` | CLI 下只规划 / 直接执行 |
| `cline --yolo "..."` | 全自动执行（CI / 无人值守） |
| `cline --json "..."` | NDJSON 机器可读输出 |
| `cline --worktree "..."` | 在独立 git worktree 中执行 |
| `git diff \| cline "..."` | 管道喂入上下文 |
| `cline schedule create "..." --cron "..."` | 定时自动化任务 |
| `CLINE_COMMAND_PERMISSIONS` | 命令执行白/黑名单 |
| `cline --timeout 600 "..."` | 任务超时控制 |
| GitHub Secrets + `cline auth --provider ...` | CI 中的认证方式 |

## 小结

Cline 的价值在于把"读代码 → 改代码 → 跑验证 → 看报错 → 再修"这个开发中最耗时的循环自动化，而 Plan/Act 分离、逐步审批、Checkpoints 和 `.clinerules` 又把"AI 乱改"的风险关进笼子。

与 GitHub 的协作有两条路径：

1. **本地开发流**：Cline 写代码并直接调 `git` / `gh` 提交、推送、开 PR——适合日常开发；
2. **CI 部署流**：把 Cline CLI 以 headless 模式放进 GitHub Actions，用 `--yolo --json` 无人值守地做 PR 审查、Issue 排查、定时维护——适合团队规模化使用。

建议先用本地插件 + 逐步审批跑顺一个小任务，建立信任后再逐步打开自动批准，最后才把它放进 Actions。让 AI 干活的同时，审批权和最终合并权始终握在自己手里，这才是正确的打开方式。
