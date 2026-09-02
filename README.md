# 沉舟的技术手记

> 记录后端开发日志、项目复盘与面试经验。

一个用 Hugo 搭建的中文技术博客，承载作者的实践沉淀：SignalDrift 游戏服务端与客户端的完整开发过程、Nexus Gateway / MiniSQL 等项目复盘、以及求职面试的经验整理。

在线访问：<https://chenzhou0071.github.io>

## 内容分类

| 分类 | 内容 | 代表文章 |
|---|---|---|
| `dev-log` | 开发日志：按计划推进的项目过程记录，含设计决策、踩坑与验证 | SignalDrift 开发日志 ①-⑤（网关 → 大厅 → Unity 客户端 → 房间对战） |
| `project-review` | 项目复盘：技术总结与提炼，聚焦架构决策与问题根因 | Nexus Gateway、MiniSQL、SignalDrift 项目复盘 |
| `interview` | 面试经验：求职过程中的题目与总结 | — |

## 技术栈

- **Hugo**（Extended 版，含 SCSS 支持）+ **PaperMod** 主题（git submodule 引入）
- **GitHub Actions**：push 到 `main` 自动构建并部署 GitHub Pages
- **giscus**：基于 GitHub Discussions 的评论区
- **mermaid**：流程图渲染（render hook 拦截 ` ```mermaid ` 代码块，按需加载脚本，跟随浅色/暗色主题重渲染）

## 功能特性

- 中文站点，支持浅色 / 深色 / 跟随系统三种主题
- mermaid 流程图、代码高亮（GitHub Light/Dark 配色）
- giscus 评论、字数统计、归档与标签页
- 自定义 404 页面、emoji favicon

## 目录结构

```
.
├── content/posts/        # 文章（按分类分目录）
│   ├── dev-log/          # 开发日志
│   ├── interview/        # 面试经验
│   └── project-review/   # 项目复盘
├── layouts/              # 模板覆盖与自定义（render hook、extend_head 等）
├── assets/css/           # 自定义样式
├── static/               # 静态资源（图片、简历等）
├── themes/PaperMod/      # 主题（submodule）
└── hugo.toml             # 站点配置
```

## 本地开发

需要 Hugo Extended 0.164+：

```bash
hugo server --port 1313
```

打开 http://localhost:1313 预览。

## 部署

不需要手动部署。push 到 `main` 分支后，GitHub Actions（`hugo --gc --minify` 构建 + `deploy-pages` 动作）自动部署到 GitHub Pages：

## 相关项目

博客内容对应的实际项目：

- [SignalDrift](https://github.com/chenzhou0071) —— Go + Unity 的双人对战游戏（自定义 TCP 协议、网关/大厅/房间三层服务端、30Hz 实时同步）
- Nexus Gateway —— C 语言自研 HTTP 网关
- MiniSQL —— C++ 轻量级关系型数据库

---

© 沉舟。原创内容，转载需注明出处。
