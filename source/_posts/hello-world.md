---
title: 如何部署和启动一个网站（以 Hexo 博客为例）
---

很多人以为"做一个网站"很难，其实用 [Hexo](https://hexo.io/) 这类静态博客框架，不需要买服务器、不需要懂后端，几十分钟就能让一个网站上线。这篇文章以本站为例，完整讲一遍从环境准备、本地启动到部署上线的全过程。

## 一、整体思路

Hexo 是一个静态网站生成器：你用 Markdown 写文章，它负责把文章套用主题模板，生成一堆 HTML/CSS/JS 静态文件；然后把这些静态文件推送到 GitHub Pages，就能通过域名访问了。

整个流程分三步：

1. **本地环境搭建** —— 安装 Node.js 和 Hexo
2. **本地启动预览** —— 在自己电脑上看效果、写文章
3. **部署上线** —— 生成静态文件并推送到 GitHub Pages

## 二、环境准备

需要两个基础软件：

- **Node.js**（建议 16 以上版本）：Hexo 运行的基础，安装后自带 npm 包管理器
- **Git**：用于把网站文件推送到 GitHub

安装完成后打开终端（Windows 推荐 PowerShell），验证一下：

```bash
node --version
npm --version
git --version
```

能显示版本号就说明安装成功。然后全局安装 Hexo 命令行工具：

```bash
npm install -g hexo-cli
```

> 小提示：在 Windows PowerShell 下如果提示命令找不到或脚本被禁用，可以改用 `npm.cmd`、`npx.cmd` 这样的 `.cmd` 入口来调用。

## 三、获取项目并安装依赖

如果是全新建站，用下面的命令初始化一个项目（会自动创建目录并写入配置）：

```bash
hexo init my-blog
cd my-blog
npm install
```

如果是从已有仓库拉取的项目（比如本站），进入项目目录后直接安装依赖即可：

```bash
cd KernelDriver-src
npm install
```

项目的核心文件说明：

| 文件 / 目录 | 作用 |
| --- | --- |
| `_config.yml` | 网站总配置：标题、主题、部署地址等 |
| `source/_posts/` | 你写的文章都放在这里（Markdown 格式） |
| `themes/` | 主题目录，决定网站长什么样 |
| `package.json` | 项目依赖清单 |

## 四、本地启动预览

在项目目录下执行：

```bash
hexo server
```

看到 `Hexo is running at http://localhost:4000/` 后，用浏览器打开 **http://localhost:4000/** 就能看到网站了。修改文章或配置后刷新页面即可看到效果，非常适合写稿和调试。

> 如果启动后页面空白并出现 `WARN No layout: index.html`，通常是主题没装好：检查 `_config.yml` 里的 `theme` 配置，并确认主题已安装（主题可以放在 `themes/` 目录，也可以直接 `npm install hexo-theme-主题名` 安装）。

## 五、写一篇新文章

```bash
hexo new "我的第一篇文章"
```

执行后会在 `source/_posts/` 下生成一个对应的 Markdown 文件，打开它用 Markdown 语法写作即可。文件顶部 `---` 之间的部分叫 Front-matter，可以设置标题、日期、标签等：

```yaml
---
title: 文章标题
date: 2026-09-10 10:00:00
tags:
  - 教程
categories:
  - 建站
---
```

## 六、生成静态文件

网站确认无误后，执行生成命令：

```bash
hexo clean      # 清除旧的缓存和生成文件（可选，但推荐）
hexo generate   # 生成静态文件，结果在 public/ 目录
```

`public/` 目录里就是整个网站的全部内容，把它放到任何静态服务器上都能跑。

## 七、部署到 GitHub Pages（上线）

以本站为例，先在 `_config.yml` 末尾配置部署信息：

```yaml
deploy:
  type: git
  repository: 你的仓库地址（如 代码托管平台 GitHub：username/username.github.io.git）
  branch: main
```

首次部署需要先安装部署插件：

```bash
npm install hexo-deployer-git --save
```

然后一条命令完成上线：

```bash
hexo deploy
```

Hexo 会自动把 `public/` 里的静态文件提交并推送到你配置的仓库。在仓库 Settings → Pages 里开启 GitHub Pages 服务后，等一两分钟，就能通过 `https://用户名.github.io/` 访问你的网站了——这就是一个真正上线的网站。

## 八、日常使用小结

以后每次更新网站，只需记住三条命令：

```bash
hexo new "文章名"   # 写新文章
hexo server         # 本地预览
hexo clean && hexo generate && hexo deploy   # 生成并部署上线
```

更多用法可以查阅官方文档：[Hexo 文档](https://hexo.io/docs/)。遇到问题也可以在 [故障排查页](https://hexo.io/docs/troubleshooting.html) 找答案。
