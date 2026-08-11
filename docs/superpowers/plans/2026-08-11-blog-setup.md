# 舟上码途博客搭建 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 在 `e:\pro\blog` 从零搭建「舟上码途」Hugo 技术博客，部署到 GitHub Pages（chenzhou0071.github.io），实现写 Markdown → push → 自动上线的完整流程。

**Architecture:** Hugo 静态站点（PaperMod 主题，git submodule 引入）托管在 GitHub 仓库 `chenzhou0071.github.io`；GitHub Actions 在每次 push 到 `main` 时自动构建并部署到 GitHub Pages。文章内容为 Markdown，分类固定三个（project-review / interview / dev-log），个性化定制（二次元头像、404 页、favicon）通过配置与模板覆盖实现，不修改主题源码。

**Tech Stack:** Hugo (extended)、PaperMod 主题、GitHub Actions、GitHub Pages、giscus 评论

**Design Doc:** `docs/superpowers/specs/2026-08-11-github-pages-blog-design.md`

## Global Constraints

- Hugo 必须使用 **extended** 版本（PaperMod 需要 SCSS 支持）
- PaperMod 主题以 git submodule 引入（`themes/PaperMod`），**禁止修改主题源码**；一切定制走 `hugo.toml` 配置、`layouts/` 模板覆盖、`assets/css/extended/` 自定义样式
- `baseURL` = `https://chenzhou0071.github.io/`
- 站点语言 `zh-cn`；文章中文写作，分类/标签 slug 用英文
- 文章 URL 结构：`/posts/<分类>/<文章slug>/`（即内容放在 `content/posts/<分类>/` 下即可实现，无需 permalinks 配置）
- 分类固定三个：`project-review`（项目复盘）/ `interview`（面试经验）/ `dev-log`（开发日志）
- 默认分支 `main`；push 到 `main` 自动触发部署
- 未完成文章标记 `draft: true`（本地可见、线上不发布）
- 明确不做：自定义域名、fork 魔改主题、后端/数据库、访问统计
- GitHub 仓库已建好：`https://github.com/chenzhou0071/chenzhou0071.github.io`（**不要**再创建同名仓库）

---

### Task 1: 初始化 Hugo 站点与 PaperMod 主题

**Files:**
- Create: `hugo.toml`（基础骨架，Task 2 会扩充为完整版）
- Create: `themes/PaperMod`（git submodule）
- Create: `.gitignore`

**Interfaces:**
- Consumes: 已初始化的 git 仓库（`e:\pro\blog`，已有 docs/ 与首个提交）
- Produces: 可成功构建的 Hugo 站点骨架；Task 2 在此基础上扩展配置

- [ ] **Step 1: 安装本地 Hugo（extended）**

```powershell
hugo version
```

若无输出（未安装），执行：

```powershell
winget install -e --id Hugo.Hugo.Extended
```

安装完成后**打开新的终端**（刷新 PATH）再执行 `hugo version`，确认输出包含 `extended`。
> 若 winget 失败：从 https://github.com/gohugoio/hugo/releases 下载 `hugo_extended_*_windows-amd64.zip`，解压到 `C:\tools\hugo\` 并把该目录加入 PATH。若实在装不上，可跳过本地 Hugo，验证一律以 GitHub Actions 构建为准（Task 6 验收）。

- [ ] **Step 2: 初始化站点骨架**

```powershell
hugo new site . --force
```

> `--force` 是必须的：当前目录已含 `docs/` 与 `.git`。执行后应生成 `hugo.toml`、`content/`、`themes/`、`assets/` 等目录。

- [ ] **Step 3: 引入 PaperMod 主题（submodule）**

```powershell
git submodule add https://github.com/adityatelange/hugo-PaperMod.git themes/PaperMod
```

验证：`themes/PaperMod/theme.toml` 存在（Hugo 识别主题的标志）。

- [ ] **Step 4: 写入基础配置**

用 Write 工具将 `hugo.toml` 替换为以下内容：

```toml
baseURL = "https://chenzhou0071.github.io/"
languageCode = "zh-cn"
title = "舟上码途"
theme = "PaperMod"
enableEmoji = true
enableRobotsTXT = true
paginate = 10
timeZone = "Asia/Shanghai"
```

- [ ] **Step 5: 创建 .gitignore**

```text
/public/
/resources/
/.hugo_build.lock
```

- [ ] **Step 6: 构建验证**

```powershell
hugo --gc --minify
```

预期：命令成功退出；`public/index.html` 存在且包含「舟上码途」。

- [ ] **Step 7: 提交**

```bash
git add hugo.toml .gitignore .gitmodules themes/PaperMod
git commit -m "feat: 初始化 Hugo 站点并引入 PaperMod 主题"
```

---

### Task 2: 站点完整配置与功能页（搜索/归档）

**Files:**
- Modify: `hugo.toml`（替换为完整配置）
- Create: `content/search.md`
- Create: `content/archives.md`

**Interfaces:**
- Consumes: Task 1 的可构建站点骨架
- Produces: 功能齐全的站点配置；Task 3 的内容页依赖本任务的菜单与分类体系

- [ ] **Step 1: 写入完整站点配置**

用 Write 工具将 `hugo.toml` 整体替换为：

```toml
baseURL = "https://chenzhou0071.github.io/"
languageCode = "zh-cn"
title = "舟上码途"
theme = "PaperMod"
enableEmoji = true
enableRobotsTXT = true
paginate = 10
timeZone = "Asia/Shanghai"

[taxonomies]
  category = "categories"
  tag = "tags"

[markup]
  [markup.goldmark]
    [markup.goldmark.renderer]
      unsafe = true

  [markup.highlight]
    codeFences = true
    guessSyntax = true
    lineNos = true
    style = "dracula"

[params]
  env = "production"
  defaultTheme = "auto"
  disableThemeToggle = false
  ShowReadingTime = true
  ShowWordCount = true
  ShowToc = true
  TocOpen = false
  ShowCodeCopyButtons = true
  ShowPostNavLinks = true
  ShowBreadCrumbs = true
  ShowRssButtonInSectionTermList = true
  description = "沉舟的技术手记：后端开发 / C 语言 / 网关 / 面试经验"

  [params.homeInfoParams]
    Title = "舟上码途"
    Content = "沉舟的技术手记。记录后端开发日志、项目复盘（Nexus Gateway）与面试经验。"

  [params.fuseOpts]
    isCaseSensitive = false
    shouldSort = true
    location = 0
    distance = 1000
    threshold = 0.4
    minMatchCharLength = 0
    limit = 10
    keys = ["title", "permalink", "summary"]

  [[params.socialIcons]]
    name = "github"
    url = "https://github.com/chenzhou0071"

  [[params.socialIcons]]
    name = "rss"
    url = "/index.xml"

[[menu.main]]
  identifier = "archives"
  name = "归档"
  url = "/archives/"
  weight = 10

[[menu.main]]
  identifier = "search"
  name = "搜索"
  url = "/search/"
  weight = 20

[[menu.main]]
  identifier = "categories"
  name = "分类"
  url = "/categories/"
  weight = 30

[[menu.main]]
  identifier = "tags"
  name = "标签"
  url = "/tags/"
  weight = 40

[[menu.main]]
  identifier = "about"
  name = "关于"
  url = "/about/"
  weight = 50

[[menu.main]]
  identifier = "resume"
  name = "简历"
  url = "/resume/"
  weight = 60
```

> 注意：`[params.homeInfoParams]` 与 `[[params.socialIcons]]` 分别是 table 与 array-of-tables，二者不能合并写；`unsafe = true` 是 Task 4 关于页内嵌 HTML 头像的前置要求。

- [ ] **Step 2: 创建搜索页**

创建 `content/search.md`：

```markdown
---
title: "搜索"
layout: "search"
url: "/search/"
summary: "search"
placeholder: "输入关键词搜索文章"
---
```

- [ ] **Step 3: 创建归档页**

创建 `content/archives.md`：

```markdown
---
title: "归档"
layout: "archives"
url: "/archives/"
summary: "archives"
---
```

- [ ] **Step 4: 构建验证**

```powershell
hugo --gc --minify
```

预期：构建成功；`public/search/index.html`、`public/archives/index.html` 存在。

- [ ] **Step 5: 本地预览抽查**

```powershell
hugo server
```

浏览器打开 `http://localhost:1313/`：确认标题「舟上码途」、顶部菜单 6 项、GitHub 图标、首页简介文字正常；打开 `/search/` 无报错。

- [ ] **Step 6: 提交**

```bash
git add hugo.toml content/search.md content/archives.md
git commit -m "feat: 完成站点完整配置（菜单/搜索/归档/暗色模式）"
```

---

### Task 3: 内容骨架（关于/简历/三篇示例文章）

**Files:**
- Create: `content/about.md`
- Create: `content/resume.md`
- Create: `content/posts/project-review/nexus-gateway-recap.md`（draft）
- Create: `content/posts/interview/interview-experience-template.md`（draft）
- Create: `content/posts/dev-log/blog-launch.md`（发布）
- Create: `static/resume/`（目录，放简历 PDF）

**Interfaces:**
- Consumes: Task 2 的菜单（/about/、/resume/）与分类体系
- Produces: 站点首批内容；Task 4 的头像图被 about.md 引用；Task 5 的评论依赖文章页（blog-launch.md 等）

- [ ] **Step 1: 创建关于页**

创建 `content/about.md`：

```markdown
---
title: "关于"
url: "/about/"
---

## 你好，我是沉舟

后端开发方向，正在寻找实习机会。

## 技术栈

- 语言：C / Go
- 方向：网络编程、网关、系统编程
- 项目：Nexus Gateway（自研 C 语言网关）

## 联系我

- GitHub: [chenzhou0071](https://github.com/chenzhou0071)
- 邮箱：（待补充）

## 关于这个博客

「舟上码途」记录我的开发日志、项目复盘与面试经验，欢迎交流。
```

- [ ] **Step 2: 创建简历页**

创建 `content/resume.md`：

```markdown
---
title: "简历"
url: "/resume/"
---

## 在线简历

### 教育背景

（待补充）

### 项目经历

#### Nexus Gateway —— 自研 C 语言网关

（待补充：职责、技术选型、量化成果）

### 技能清单

（待补充）

---

📄 [下载 PDF 版简历](/resume/chenzhou-resume.pdf)
```

- [ ] **Step 3: 创建 resume 目录并放置 PDF**

```powershell
New-Item -ItemType Directory -Force -Path static\resume
```

将真实简历 PDF 命名为 `chenzhou-resume.pdf` 放入 `static/resume/`。**若暂时没有 PDF，跳过本步的文件放置**——链接会暂时 404，验收时排除此项，简历就位后自动生效（见 Task 6 验收说明）。

- [ ] **Step 4: 创建 Nexus Gateway 复盘骨架（draft）**

创建 `content/posts/project-review/nexus-gateway-recap.md`：

```markdown
---
title: "Nexus Gateway：自研 C 语言网关复盘"
date: 2026-08-11
draft: true
categories: ["project-review"]
tags: ["c语言", "网关", "网络编程"]
summary: "从零实现一个 C 语言网关的心路历程：架构设计、技术难点与踩坑记录。"
---

> 本文为骨架，请补充你的真实项目经历。写作指引见文末。

## 项目背景

（为什么做这个项目、目标是什么）

## 总体架构

（模块划分、数据流）

## 关键技术决策

（事件驱动模型？多线程？内存管理？）

## 踩坑记录

（最有价值的故障排查经历）

## 性能与效果

（压测数据、对比结果）

## 复盘与收获

（面试官最想听的部分）

---

*写作指引：每节 200-500 字；踩坑记录按「现象→排查→根因→修复」组织；贴关键代码片段；写完删除本指引并把 `draft` 改为 `false`。*
```

- [ ] **Step 5: 创建面经模板（draft）**

创建 `content/posts/interview/interview-experience-template.md`：

```markdown
---
title: "面经：XXX 公司后端实习一面"
date: 2026-08-11
draft: true
categories: ["interview"]
tags: ["面经"]
summary: "XXX 公司后端实习一面记录：流程、问题、复盘。"
---

> 模板：复制后替换为实际面试记录。

## 基本信息

公司 / 岗位 / 时间 / 轮次：

## 面试流程

## 考察的问题

### 算法与手写

### 八股（操作系统 / 网络 / C）

### 项目深挖

## 回答得好的 / 翻车的

## 复盘总结
```

- [ ] **Step 6: 创建上线首篇日志（发布）**

创建 `content/posts/dev-log/blog-launch.md`：

```markdown
---
title: "舟上码途上线了"
date: 2026-08-11
categories: ["dev-log"]
tags: ["博客"]
summary: "为什么做这个博客，以及它会写些什么。"
---

这里是「舟上码途」的第一篇文章。

## 为什么开这个博客

沉淀技术成长，记录后端开发之路，也把项目复盘和面试经验整理成文，与同路人交流。

## 你会在这里看到什么

- **项目复盘**：Nexus Gateway 的完整回顾（架构、难点、踩坑）
- **面试经验**：面经与八股整理
- **开发日志**：学习与编码日常

## 关于站点的技术

本博客使用 Hugo + PaperMod 构建，GitHub Actions 自动部署到 GitHub Pages，源码在 [chenzhou0071.github.io 仓库](https://github.com/chenzhou0071/chenzhou0071.github.io)。
```

- [ ] **Step 7: 构建验证**

```powershell
hugo --gc --minify
```

预期：构建成功；`public/about/index.html`、`public/resume/index.html` 存在；`public/posts/dev-log/blog-launch/index.html` 存在；`public/posts/project-review/` 下**没有** nexus-gateway-recap 页面（draft 未发布）。

- [ ] **Step 8: 提交**

```bash
git add content/ static/resume/
git commit -m "feat: 添加关于页、简历页与首批文章骨架"
```

---

### Task 4: 个性化定制（二次元头像 / 404 / favicon）

**Files:**
- Create: `static/img/avatar.png`（ImageGen 生成）
- Create: `static/img/404-anime.png`（ImageGen 生成）
- Create: `layouts/404.html`
- Create: `layouts/partials/extend_head.html`
- Create: `assets/css/extended/custom.css`
- Modify: `content/about.md`（加入头像 HTML）

**Interfaces:**
- Consumes: Task 3 的 about.md（加入头像）、Task 2 的 `unsafe = true`（HTML 渲染）
- Produces: 个性化静态资源；Task 5/6 无依赖

- [ ] **Step 1: 生成二次元头像**

用 ImageGen 生成（prompt 参考）：*「二次元动漫风格头像插画，一位年轻程序员男孩，半身像，简洁干净的背景，柔和色彩，高质量动漫插画，方形构图」*，size 选 `1024x1024`，保存为 `static/img/avatar.png`。
> 若 ImageGen 不可用：用任意现有二次元图片（用户提供）放入该路径；再不行则跳过，about.md 不显示头像，不影响其余功能。

- [ ] **Step 2: 生成 404 插画**

用 ImageGen 生成（prompt 参考）：*「二次元动漫风格插画，一艘小木船在浩瀚星空中航行，暖色灯塔光，治愈系，温柔色调，宽幅横图」*，size 选 `1536x1024`，保存为 `static/img/404-anime.png`。
> 兜底方案同上。

- [ ] **Step 3: 覆盖 404 页面**

创建 `layouts/404.html`：

```html
{{- define "main" }}
<main id="main" class="not-found">
  <img src="/img/404-anime.png" alt="404" loading="lazy">
  <h1>404</h1>
  <p>你来到了沉舟号航线的未知海域，这片海图上没有标记。</p>
  <p><a href="{{ "" | relLangURL }}">返回首页 →</a></p>
</main>
{{- end }}
```

- [ ] **Step 4: 添加 favicon（emoji 方案，零文件依赖）**

创建 `layouts/partials/extend_head.html`：

```html
<link rel="icon" href="data:image/svg+xml,<svg xmlns='http://www.w3.org/2000/svg' viewBox='0 0 100 100'><text y='.9em' font-size='90'>🚢</text></svg>">
<meta name="description" content="{{ .Site.Params.description }}">
```

- [ ] **Step 5: 自定义样式**

创建 `assets/css/extended/custom.css`：

```css
/* 关于页头像 */
.about-avatar img {
  border-radius: 50%;
  width: 120px;
  height: 120px;
  object-fit: cover;
  margin-bottom: 0.5rem;
}

/* 404 页面 */
.not-found {
  text-align: center;
  padding: 3rem 1rem;
}
.not-found img {
  max-width: 480px;
  width: 100%;
  border-radius: 8px;
  margin: 0 auto 1.5rem;
}
.not-found h1 {
  font-size: 3rem;
  margin-bottom: 0.5rem;
}
.not-found p {
  color: var(--secondary);
}
```

- [ ] **Step 6: 关于页加入头像**

在 `content/about.md` 的 `## 你好，我是沉舟` 之前插入：

```html
<div class="about-avatar">
  <img src="/img/avatar.png" alt="沉舟的头像">
</div>
```

- [ ] **Step 7: 构建与页面验证**

```powershell
hugo --gc --minify
hugo server
```

验证：`http://localhost:1313/about/` 显示圆形头像；`http://localhost:1313/404.html` 显示插画 + 「404」+ 返回首页链接（404.html 是静态页，直接访问路径即可）；浏览器标签页出现 🚢 favicon；首页暗色/亮色切换正常。

- [ ] **Step 8: 提交**

```bash
git add static/img/ layouts/ assets/css/extended/ content/about.md
git commit -m "feat: 个性化定制（二次元头像、404 页、favicon）"
```

---

### Task 5: giscus 评论接入

**Files:**
- Modify: `hugo.toml`（加入 comments 与 giscus 配置块）

**Interfaces:**
- Consumes: Task 3 的文章页（评论框渲染目标）
- Produces: 文章页底部评论框；线上生效依赖 Task 6 部署

- [ ] **Step 1: 写入 giscus 配置**

在 `hugo.toml` 的 `[params]` 内追加（放在 `[[params.socialIcons]]` 之前）：

```toml
  comments = true

  [params.giscus]
    repo = "chenzhou0071/chenzhou0071.github.io"
    repoId = ""
    category = "Announcements"
    categoryId = ""
    mapping = "pathname"
    strict = "0"
    reactionsEnabled = "1"
    emitMetadata = "0"
    inputPosition = "bottom"
    theme = "preferred_color_scheme"
    lang = "zh-CN"
    crossorigin = "anonymous"
```

- [ ] **Step 2: 用户获取 repoId 与 categoryId（需要用户操作，浏览器）**

1. 打开 https://giscus.app/zh-CN
2. 在「仓库」输入 `chenzhou0071/chenzhou0071.github.io`，点击页面提示的 **Install giscus app**（授权该仓库，跳转 GitHub 完成安装）
3. 回到 giscus.app，页面自动生成配置；「Discussion 分类」选择 **Announcements**
4. 从页面底部的 `<script>` 片段中复制 `data-repo-id="..."` 与 `data-category-id="..."` 两个值
5. 把两个值分别填入 `hugo.toml` 的 `repoId` 与 `categoryId`
6. 若用户暂时无法完成：保持空字符串，评论区不渲染，**不影响上线**，Task 6 后可随时补

- [ ] **Step 3: 用户开启仓库 Discussions（需要用户操作）**

仓库 `chenzhou0071/chenzhou0071.github.io` → Settings → General → 拉到最底部 **Features** → 勾选 **Discussions** → Save changes。

- [ ] **Step 4: 本地验证构建与页面**

```powershell
hugo --gc --minify
hugo server
```

验证：构建成功；打开 `http://localhost:1313/posts/dev-log/blog-launch/`，页面底部：
- 若 repoId 已填：出现 giscus 评论框（GitHub 登录后可见完整效果，本地无登录也可看框架）
- 若未填：无评论框、页面无 JS 报错（F12 Console 检查）

- [ ] **Step 5: 提交**

```bash
git add hugo.toml
git commit -m "feat: 接入 giscus 评论系统"
```

---

### Task 6: CI/CD 工作流与首次上线

**Files:**
- Create: `.github/workflows/hugo.yml`
- Modify: git remote、默认分支（`master` → `main`）

**Interfaces:**
- Consumes: Task 1-5 的全部产物；GitHub 仓库 `chenzhou0071/chenzhou0071.github.io`（用户已建好）
- Produces: 线上站点 `https://chenzhou0071.github.io/`

- [ ] **Step 1: 创建工作流**

创建 `.github/workflows/hugo.yml`：

```yaml
name: Deploy Hugo site to Pages

on:
  push:
    branches: ["main"]
  workflow_dispatch:

permissions:
  contents: read
  pages: write
  id-token: write

concurrency:
  group: "pages"
  cancel-in-progress: true

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4
        with:
          submodules: recursive
          fetch-depth: 0
      - name: Setup Hugo
        uses: peaceiris/actions-hugo@v3
        with:
          hugo-version: "0.145.0"
          extended: true
      - name: Build
        run: hugo --gc --minify
      - name: Upload artifact
        uses: actions/upload-pages-artifact@v3
        with:
          path: ./public

  deploy:
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    runs-on: ubuntu-latest
    needs: build
    steps:
      - name: Deploy to GitHub Pages
        id: deployment
        uses: actions/deploy-pages@v4
```

> 若 `hugo-version: "0.145.0"` 在 Actions 中报「版本不存在」，改为任一已发布的 0.14x 版本号即可。

- [ ] **Step 2: 用户开启 Pages 的 Actions 部署源（需要用户操作，浏览器）**

仓库 `chenzhou0071/chenzhou0071.github.io` → Settings → Pages → **Build and deployment** → Source 选 **GitHub Actions**。此步必须在首次部署前完成，否则 deploy 任务会失败。

- [ ] **Step 3: 关联远程仓库并统一分支**

```powershell
git remote add origin https://github.com/chenzhou0071/chenzhou0071.github.io.git
git branch -M main
git push -u origin main
```

预期：推送成功；push 会自动触发 Actions（仓库 Actions 页面出现运行记录）。

- [ ] **Step 4: 确认 Actions 构建部署成功**

浏览器打开仓库 Actions 页面（或装了 gh CLI 则 `gh run watch`），观察两个任务：`build` → `deploy` 均绿色通过。若失败：点进失败任务看日志，常见原因与修复：
- `hugo: not found` / 版本不存在 → 调整 `hugo-version`
- submodule 拉取失败 → 检查 checkout 步骤的 `submodules: recursive`
- 部署失败且提示 Pages 未启用 → 回到 Step 2 检查部署源设置

- [ ] **Step 5: 线上验收**

```powershell
$paths = @("/", "/about/", "/resume/", "/archives/", "/search/", "/categories/", "/tags/", "/index.xml", "/404.html", "/posts/dev-log/blog-launch/")
foreach ($p in $paths) {
  $code = curl.exe -s -o NUL -w "%{http_code}" "https://chenzhou0071.github.io$p"
  Write-Host "$code  $p"
}
```

预期：所有路径返回 `200`（`/index.xml` 为 XML，`/404.html` 返回 200 属正常——静态页）。
> 例外：`/resume/chenzhou-resume.pdf` 在用户放入 PDF 前返回 404，属已知待办，不视为失败。

浏览器逐项确认（含移动端，用户手机可访问 https://chenzhou0071.github.io/ 检查排版）：
- 首页标题/简介/文章列表/暗色切换
- 文章页代码高亮、TOC、面包屑
- 归档/搜索/分类/标签页
- 404 页插画与返回链接
- 关于页头像、简历页内容
- RSS 订阅 `/index.xml`
- 若 Task 5 已填 repoId：文章页底部 giscus 评论框可加载

- [ ] **Step 6: 提交收尾**

```bash
git add .github/workflows/hugo.yml
git commit -m "ci: 添加 GitHub Pages 自动部署工作流"
git push
```

> 说明：workflow 文件随首次 push 一起上线（Step 3 提交内容已含本文件时，本步可合并为一次提交再 push；若 Step 3 已 push 全部内容则本步跳过，确认无未提交更改即可）。

---

## 验收清单（最终）

- [ ] `https://chenzhou0071.github.io/` 200 且标题为「舟上码途」
- [ ] 三大分类 slug（project-review / interview / dev-log）在 /categories/ 可见
- [ ] 菜单 6 项全部可达（归档/搜索/分类/标签/关于/简历）
- [ ] 关于页显示二次元圆形头像；404 页显示插画与返回链接；favicon 为 🚢
- [ ] 暗色/亮色切换正常；文章页代码高亮与 TOC 正常
- [ ] 搜索页可搜到「blog-launch」文章
- [ ] RSS `/index.xml` 可访问
- [ ] 无死链（`/resume/chenzhou-resume.pdf` 在 PDF 就位前除外）
- [ ] 手机端打开首页与一篇文章排版正常
- [ ] giscus 配置完成后文章页出现评论框（未配置则页面无报错）
- [ ] 再次 push 一篇文章能自动触发部署（验证流程闭环）
