---
title: AGENTS.md 使用指南：给 AI Agent 看的 README，一份文件跨工具通用
date: 2026-09-18 15:00:00
categories:
  - 开发工具
tags:
  - AGENTS.md
  - AI编程
  - Cline
  - Codex
  - MCP
---

用 AI 编程代理（Coding Agent）干活的人，大概率都被规则文件折磨过：Claude Code 读 `CLAUDE.md`，Cursor 读 `.cursorrules`，GitHub Copilot 读 `.github/copilot-instructions.md`，Cline 读 `.clinerules`，Gemini 读 `GEMINI.md`……同一个项目想让多个 AI 工具都听话，就得在仓库里维护好几份内容几乎一样的文件，还经常改漏。

**AGENTS.md** 就是为终结这种碎片化而来的：在仓库根目录放一个同名 Markdown 文件，把构建命令、测试方式、代码约定、禁区写清楚，任何支持该约定的代理动手改代码前都会先读它。官方对它的定位就一句话——**README.md 是给人看的，AGENTS.md 是给 Agent 看的**（见 [agents.md 官网](https://agents.md/)）。

本文讲清楚三件事：AGENTS.md 是什么、它的出处与归属、以及大家最关心的问题——**一份 AGENTS.md 能不能被多个 AI Agent 同时使用**（答案是：能，这正是它存在的意义）。

## 一、AGENTS.md 是什么

根据[官方网站](https://agents.md/)的定义：

> AGENTS.md 是一个简单、开放的格式，用于指导编程代理（coding agents）。把它想象成**给代理看的 README**：一个专门的、可预测的位置，用来提供上下文和指令，帮助 AI 编程代理在你的项目里工作。

它有几个关键特性：

- **就是一个普通 Markdown 文件**：通常放在仓库根目录，没有 schema、没有必填字段、没有版本号，代理只是把它当文本解析；
- **是"约定"（convention），不是"协议"（protocol）**：没有传输层、不需要运行时协商、不存在"合规认证"，价值完全来自各工具同意"都去这个路径找文件"；
- **随仓库走**：提交进版本库，团队所有人和所有 CI 里的代理共享同一份规则；
- **写的是"光看代码很难推断"的操作细节**：精确的测试命令、会卡住 CI 的 lint 规则、哪些目录是生成物绝不能手改。

一份最小可用的 AGENTS.md 长这样（[官网示例](https://agents.md/)）：

```markdown
# AGENTS.md

## Setup commands

- Install deps: `pnpm install`
- Start dev server: `pnpm dev`
- Run tests: `pnpm test`

## Code style

- TypeScript strict mode
- Single quotes, no semicolons
- Use functional patterns where possible
```

注意，它对读者的假设是"一个刚入职、需要 onboarding 的新同事"：你平时会口头交代新人的那些事，都适合写进去。

## 二、出处与发展简史

标明出处很重要，因为网上叫 "agents.md" 的第三方教程站不少，真正的官方源头是 **[agents.md](https://agents.md/)**，源码仓库在 GitHub 的 [agentsmd/agents.md](https://github.com/agentsmd/agents.md)。时间线如下：

| 时间 | 事件 |
|---|---|
| 2025 年 8 月 | AGENTS.md 随 **OpenAI Codex** 诞生。OpenAI 的原话是："Codex 需要一种可预测的方式来获取项目特定的说明（编码规范、构建步骤与测试要求），以确保智能体能在代码库中安全、高效地工作" |
| 2025 年下半年 | 演变为行业协作成果，共建方包括 [OpenAI Codex、Amp、Google Jules、Cursor、Factory](https://agents.md/) 等 |
| 2025 年 12 月 9 日 | **OpenAI 与 Anthropic、Block 共同发起**，在 Linux 基金会旗下成立 **Agentic AI 基金会（AAIF）**，OpenAI 将 AGENTS.md 捐赠给该基金会 |
| 至今 | 已被 **60,000+ 开源项目**采用（[官网统计](https://github.com/search?q=path%3AAGENTS.md+NOT+is%3Afork+NOT+is%3Aarchived&type=code)），包括 [openai/codex](https://github.com/openai/codex/blob/-/AGENTS.md)、[apache/airflow](https://github.com/apache/airflow/blob/-/AGENTS.md) 等知名仓库 |

捐赠这件事的分量在于：格式的演进不再受任何单一公司控制。同批进入 AAIF 的还有 **Anthropic 捐赠的 MCP（Model Context Protocol）** 和 **Block 捐赠的 goose**，Google、Microsoft、Amazon AWS、Bloomberg、Cloudflare 同为创始成员。以上均出自 OpenAI 官方公告 [《OpenAI 在 Linux 基金会旗下联合创立 Agentic AI Foundation》](https://openai.com/index/agentic-ai-foundation/)（2025-12-09）。

## 三、核心问题：一份文件能被多个 AI Agent 用吗

**能，而且这就是 AGENTS.md 的全部设计目标。** 官网直接用了一个小标题来强调：[One AGENTS.md works across many agents](https://agents.md/)（一份 AGENTS.md 可跨众多代理工作）。

截至目前，官网生态页列出的兼容工具已超过 20 款，按形态分类：

| 形态 | 支持的代理 / 工具 |
|---|---|
| 命令行 Agent | [OpenAI Codex CLI](https://openai.com/codex/)、[Google Gemini CLI](https://github.com/google-gemini/gemini-cli)、[Aider](https://aider.chat/docs/usage/conventions.html#always-load-conventions)、[opencode](https://opencode.ai/docs/rules/)、[Amp](https://ampcode.com/)、[goose (Block)](https://github.com/block/goose)、[Augment Code CLI](https://docs.augmentcode.com/cli/overview) |
| IDE / 编辑器 | [Cursor](https://cursor.com/)、[VS Code](https://code.visualstudio.com/docs/editor/artificial-intelligence)、[Zed](https://zed.dev/docs/ai/rules)、[JetBrains Junie](https://www.jetbrains.com/junie/)、[Windsurf (Cognition)](https://windsurf.com/)、[RooCode](https://roocode.com/)、[Kilo Code](https://kilocode.ai/) |
| 云端 / 平台 Agent | [GitHub Copilot coding agent](https://gh.io/coding-agent-docs)、[Google Jules](https://jules.google/)、[Devin (Cognition)](https://devin.ai/)、[Factory](https://factory.ai/)、[UiPath Autopilot](https://uipath.github.io/uipath-python)、[Warp](https://docs.warp.dev/knowledge-and-collaboration/rules#project-scoped-rules-1)、[Phoenix](https://phoenix.new/)、[Ona](https://ona.com/)、[Semgrep](https://semgrep.dev/) |

### 3.1 Cline 也支持（零配置）

上一篇刚讲过 [Cline](/2026/09/17/Cline-使用指南/)，它同样在官方支持之列。Cline 官方文档 [Rules 页](https://docs.cline.bot/customization/cline-rules)明确列出了四种可识别的规则文件：

| 规则类型 | 位置 | 说明 |
|---|---|---|
| Cline Rules | `.clinerules/` | Cline 原生主格式，支持条件规则 |
| Cursor Rules | `.cursorrules` | 自动检测 |
| Windsurf Rules | `.windsurfrules` | 自动检测 |
| **AGENTS.md** | `AGENTS.md`、`~/.agents/AGENTS.md` | **跨工具兼容的标准格式** |

也就是说：仓库里放了 AGENTS.md，用 Cline 打开项目时它会自动读到，无需任何配置；所有识别到的规则文件还会出现在 Cline 的 Rules 面板里，可以单独开关。当 `.clinerules` 与 AGENTS.md 内容冲突时，**`.clinerules` 优先**。

这给出了一个清晰的使用策略：**跨工具通用的团队标准写进 AGENTS.md；只有 Cline 才需要的特殊逻辑（比如按路径条件触发的规则）再放进 `.clinerules/`**，两者可以共存，Cline 会合并加载。

### 3.2 一个需要说清楚的前提：支持程度并不均等

虽然工具很多，但要诚实地提醒一点（[AgentProtocol 站点的分析](https://agentprotocol.ai/agents-md/)说得很到位）：因为 AGENTS.md **没有正式规范**，所以不存在"完全兼容"这回事——两个代理读同一份文件，对内容的取舍和执行严格程度可能不同。多数工具是"自动把文件内容注入上下文"，而具体怎么做、嵌套文件读几层，各家实现有差异。

好消息是：采用它零风险——**多加一个文件而已，任何工具忽略它也不会出任何问题**。这也是它能在几个月内铺开的原因。

## 四、和 CLAUDE.md / .cursorrules 们是什么关系

AGENTS.md 出现之前，各家代理"各自发明了同一个东西，只是名字不同"：

| 文件名 | 所属工具 |
|---|---|
| `CLAUDE.md` | Claude Code（Anthropic） |
| `.cursorrules` | Cursor |
| `.github/copilot-instructions.md` | GitHub Copilot |
| `GEMINI.md` | Google Gemini |
| `.clinerules` | Cline |
| `.windsurfrules` | Windsurf |

AGENTS.md 不是要发明又一个专有文件，而是推动大家**收敛到同一个文件名**，让任何代理都知道该去哪里找指令。现实中的过渡策略有两种：

1. **新工具主动兼容多种格式**：以 Cline 为代表，`.clinerules`、`.cursorrules`、`.windsurfrules`、`AGENTS.md` 全都读，老仓库无需迁移；
2. **老文件改名 + 符号链接**：保留对旧工具的向后兼容。官网 FAQ 给出的做法是：

```bash
mv AGENT.md AGENTS.md && ln -s AGENTS.md AGENT.md
```

Windows（需开发者模式或管理员权限的命令提示符）对应写法：

```powershell
New-Item -ItemType SymbolicLink -Path "AGENT.md" -Target "AGENTS.md"
```

## 五、动手写一份 AGENTS.md

### 5.1 该写什么

官网建议的"热门板块"是：项目概览、构建与测试命令、代码风格、测试说明、安全注意事项。结合[社区总结的最佳实践](https://agentprotocol.ai/agents-md/)，真正有用的内容具备三个特征——**具体、可验证、光读代码推断不出来**：

- **构建 / 测试 / Lint 命令**：精确到参数，比如"必须用 `pnpm test`，不要用 `npm test`"；
- **项目布局**：每个目录放什么，哪些路径是生成物或第三方 vendored 代码；
- **约定**：命名、错误处理、格式化规则——让代理去"匹配"周边代码而不是重写它；
- **禁区**：没被明确要求时不许碰的文件、目录或数据库迁移；
- **完成前的验证步骤**：怎样才算改完了（跑哪些命令、全绿才能汇报完成）。

### 5.2 实战示例：给本 Hexo 博客写一份

本博客（KernelDriver）是 Hexo + butterfly 主题，下面这份 AGENTS.md 可以直接放到仓库根目录，Codex、Cline、Cursor 等任何代理读完都能立刻按规矩干活：

```markdown
# AGENTS.md

本仓库是 KernelDriver 个人技术博客，基于 Hexo + hexo-theme-butterfly 构建。

## 命令

- 安装依赖：`npm install`
- 本地预览：`hexo s`（默认 http://localhost:4000）
- 生成静态文件：`hexo clean && hexo g`
- 部署到 GitHub Pages：`hexo d`

## 目录结构

- `source/_posts/`：Markdown 文章，文件名即 URL 的一部分
- `source/resume/index.md`：简历页内容（唯一存放位置）
- `source/css/resume.css`：简历页专属样式，不要与其他页面样式混用
- `source/img/`：Banner 等图片，一律本地存储并用相对路径引用
- `public/`：构建产物，禁止手改

## 约定

- 站点名必须是 `KernelDriver`，修改以 `_config.yml` 的 title / author 为准
- 主题定制只写进 `_config.butterfly.yml`，禁止改动 `node_modules` 里的主题文件
- 文章分类通过 Front-matter 的 `categories` 字段管理
- 简历页的 HTML 标签必须顶格写（行首不能有空格），否则会被 Markdown 渲染成代码块
- 简历页图标用纯 CSS 形状，不要用 Font Awesome 图标字体（会出现字符渲染问题）

## Git 与部署

- 文本文件统一 LF 行尾（由 `.gitattributes` 约束），Windows 脚本除外
- 部署目标仓库：`KernelDriver-star/KernelDriver-star.github.io`
- 禁止 `git push --force` 到主分支

## 完成前自检

1. `hexo clean && hexo g` 必须无报错；
2. `hexo s` 本地打开确认页面渲染正常；
3. 改动涉及文章时，确认 Front-matter 的分类与标签已填写。
```

这个例子也顺带演示了一个技巧：**把那些"吃过亏才知道"的坑写进文件**（比如 HTML 顶格、图标字体渲染问题），新接手的代理第一次就能绕开，不用反复试错。

另外，官网还提到一个省事的办法——**直接让代理帮你生成**："大多数编程代理只要你开口要求，就能帮你 scaffold 一份 AGENTS.md"（[原文](https://agents.md/)）。之后再根据它实际犯过的错迭代补充即可。

## 六、Monorepo：嵌套文件与冲突规则

大型仓库不必把所有子项目的说明塞进一个巨型文件。AGENTS.md 支持**目录嵌套**：

- 仓库根目录放一份，写组织级 / 全仓通用规则；
- 每个 package / 子项目目录里再放各自的 AGENTS.md，写该子项目专属命令；
- 代理编辑某个文件时，采用**目录树上离该文件最近的 AGENTS.md**。

官网给的数字很有说服力：写作本文时，**OpenAI 主代码仓库里有 88 个 AGENTS.md 文件**。

当指令之间发生冲突时，官网 [FAQ](https://agents.md/) 给了明确的优先级：

1. **离被编辑文件最近的 AGENTS.md 胜出**；
2. **用户在对话里给出的显式提示，覆盖一切文件规则**。

此外还有"全局"一层：部分工具（如 Cline）会读取用户主目录下的 `~/.agents/AGENTS.md`，用于放跨所有项目的个人偏好；项目规则与全局规则冲突时，项目规则优先。

## 七、各工具的接入方式

绝大多数工具**开箱自动读取**，无需配置（Codex、Cursor、Copilot、Cline、Zed、Junie 等都属此类）。少数需要显式开启，官网 FAQ 记录了两个典型：

**Aider**——在项目根目录的 `.aider.conf.yml` 中声明：

```yaml
read: AGENTS.md
```

**Google Gemini CLI**——在 `.gemini/settings.json` 中指定：

```json
{ "context": { "fileName": "AGENTS.md" } }
```

**Cline**——零配置，文件会自动出现在 Rules 面板；全局文件放 `~/.agents/AGENTS.md` 即可（见 [Cline 官方文档](https://docs.cline.bot/customization/cline-rules)）。

## 八、AGENTS.md 不是协议：和 MCP 的区别

这是最容易被混淆的一点。2025 年"智能体协议"很多，AGENTS.md 经常和 MCP、A2A 放在一起讨论，但它和这些根本不在同一层。[AgentProtocol 的对比](https://agentprotocol.ai/agents-md/)值得引用：

| 维度 | AGENTS.md（约定） | MCP（模型上下文协议） | A2A（Agent2Agent） |
|---|---|---|---|
| 本质 | 仓库里的一个文件 | 通信线协议（wire protocol） | 通信线协议 |
| 所在层 | Agent ↔ 代码库 | Agent ↔ 工具 / 数据 | Agent ↔ Agent |
| 有无传输层 | 无 | 有 | 有 |
| 运行时协商 | 无 | 有 | 有 |
| 机器可读 schema | 无 | 有 | 有 |
| 擅长场景 | 让代理快速上手一个仓库 | 接入工具与上下文 | 多代理协作 |

两者实际是**互补**关系：代理先读 AGENTS.md 知道"本项目测试要用 `pnpm test`"，再通过 MCP 拿到终端 / 文件系统工具去真正执行这条命令。一个告诉它"在这个代码库里该怎么做"，另一个给它"动手的能力"。

## 九、最佳实践与常见误区

| 建议 | 原因 |
|---|---|
| 保持简短、写具体命令 | 空泛的"代码要优雅"会被忽略或误用；精确的 `pnpm lint && pnpm tsc --noEmit` 不会 |
| 像维护其他文档一样持续更新 | **过时的 AGENTS.md 比没有更糟**——代理会老老实实执行已经失效的指令 |
| 明确标注生成物与禁区 | `src/generated/`、迁移目录、CI 配置等，写清楚"未经要求禁止修改" |
| 写清"完成定义" | 例如"提交前必须 `pnpm test` 与 `pnpm lint` 全绿，否则不得汇报完成" |
| 不要放任何密钥 | 它是要提交进仓库的普通文件，所有协作者和 CI 都可见 |
| 别和 README 重复 | README 讲"项目是什么、怎么入门"，AGENTS.md 讲"代理干活所需的操作细节" |
| 先写最小版本再迭代 | 观察代理实际犯过的错，把纠正措施逐条补进文件 |

## 十、速查表

| 项目 | 内容 |
|---|---|
| 官方网站 | [https://agents.md](https://agents.md/) |
| GitHub 仓库 | [agentsmd/agents.md](https://github.com/agentsmd/agents.md) |
| 治理机构 | Linux 基金会旗下 [Agentic AI Foundation (AAIF)](https://aaif.io/) |
| 发布时间 | 2025 年 8 月（随 OpenAI Codex） |
| 捐赠时间 | 2025 年 12 月 9 日（OpenAI 官方[公告](https://openai.com/index/agentic-ai-foundation/)） |
| 文件位置 | 仓库根目录 `AGENTS.md`；monorepo 可嵌套；全局 `~/.agents/AGENTS.md` |
| 文件格式 | 普通 Markdown，无必填字段、无 schema |
| 冲突优先级 | 最近的文件生效 > 根目录；用户对话指令 > 一切文件 |
| 多 Agent 通用 | 支持，Codex / Cursor / Copilot / Gemini CLI / Cline / Jules / Devin 等 20+ 工具 |
| 典型配置 | Aider：`.aider.conf.yml` 写 `read: AGENTS.md`；Gemini CLI：`.gemini/settings.json` 指定 `context.fileName` |
| 与 MCP 的关系 | 互补：AGENTS.md 提供静态上下文约定，MCP 提供运行时工具调用协议 |

## 小结

回到开头的三个问题：

1. **AGENTS.md 是什么？** 一个提交在仓库里、给 AI 编程代理阅读的 Markdown 文件，承载构建、测试、约定与禁区等"光看代码推断不出"的操作信息；它是约定而非协议，没有学习成本，加一个文件即可生效。
2. **出处是哪里？** 2025 年 8 月由 OpenAI Codex 团队提出，Codex、Amp、Jules、Cursor、Factory 等共建；2025 年 12 月 9 日 OpenAI 将其捐赠给 Linux 基金会旗下新成立的 Agentic AI Foundation，与 Anthropic 的 MCP、Block 的 goose 同属中立治理的开放标准，目前已有 6 万多个开源项目采用。
3. **能被多个 AI Agent 使用吗？** **能，这正是它的核心价值**——官网明言 "One AGENTS.md works across many agents"，Codex、Cursor、GitHub Copilot、Gemini CLI、Cline、Jules、Devin、Aider、Zed、Junie 等 20 余款工具均已支持；只是各家实现深浅不一，且它不替代各工具的专有规则文件（如 Cline 的 `.clinerules` 可与之共存并在冲突时优先）。

如果你已经在用 Cline、Cursor 或 Codex 之类的代理，建议现在就给自己的项目加一份 AGENTS.md：把"新人入职第一天你会交代的话"写进去，提交进仓库。一份文件，所有代理通用——这大概是 2025 年以来 AI 工程化领域里投入产出比最高的一个小习惯。

## 参考出处

- [AGENTS.md 官方网站](https://agents.md/)
- [GitHub：agentsmd/agents.md](https://github.com/agentsmd/agents.md)
- [OpenAI 官方公告：联合创立 Agentic AI Foundation（2025-12-09）](https://openai.com/index/agentic-ai-foundation/)
- [Cline 官方文档：Rules（AGENTS.md 支持说明）](https://docs.cline.bot/customization/cline-rules)
- [AgentProtocol：AGENTS.md 与 MCP/A2A 的对比分析](https://agentprotocol.ai/agents-md/)
- [Aider 文档：Conventions 文件加载方式](https://aider.chat/docs/usage/conventions.html#always-load-conventions)
