# 仓库约定（供 Agent / 自动化 / 协作者）

本文档描述**本仓库内应遵守的规范**；面向读者的介绍与上手说明见 `README.md`。

## 技术栈与部署

- **Hugo**（extended）+ **PaperMod**（`themes/PaperMod`，git submodule）
- **GitHub Actions**：`.github/workflows/deploy.yml`，推 `main` 构建并部署到 **GitHub Pages**
- 本地构建：`hugo --gc --minify`；预览：`hugo server -D`

## 多语言（整站双语）

- **语言代码**：`zh`（默认）、`en`
- **URL**：中文在站点根路径 `/`；英文在 **`/en/`** 前缀下
- **内容文件命名**：`*.zh.md` / `*.en.md`（同一 basename 表示同一篇的两种语言，由 Hugo 关联）
- **文章路径**：`content/posts/<slug>.zh.md`、`content/posts/<slug>.en.md`
- **归档 / 搜索**：仅维护 `content/archives.zh.md`、`archives.en.md`、`search.zh.md`、`search.en.md`，勿改回单文件 `archives.md` / `search.md`
- **站点配置**：`hugo.toml` 中 `[languages.zh]`、`[languages.en]` 分别维护描述、首页文案、菜单文案、RSS（如 `index.xml` 与 `/en/index.xml`）

新增同一主题时，**中英文成对添加**；避免只加一种语言导致另一种语言缺页。

## 标签与分类

- **`tags`**：每篇文章 **只使用一个** tag，且必须 **具体、可检索**（如 `Hugo`、`Spark`、`GitHub Actions`）
- **禁止**：过于宽泛的 tag，例如「文章」「博客」「随笔」等（除非正文主题就是该抽象概念本身）
- **`categories`**：可用于稍宽的分组（如「杂谈」「Notes」）；与 tags 分工——**细粒度用 tag，栏目感用 category**

## 文档分工

- **`README.md`**：给人读的简介、安装、常用命令、链接；不写冗长的机器规则
- **`AGENTS.md`**（本文件）：约定、边界、目录结构细节、Agent 易错点
- 若规范变更：**先改 `AGENTS.md`，再视情况在 `README.md` 做一句话摘要或链接**

## 目录与文件（维护时）

```
content/posts/          # 文章：*.zh.md / *.en.md
content/archives.*.md
content/search.*.md
hugo.toml
themes/PaperMod/        # submodule，勿手写覆盖主题目录
.github/workflows/deploy.yml
archetypes/default.md   # 新文章模板
```

## Git / 提交（建议）

- 文章：`post: <简短英文或中文描述>`
- 配置/主题：`chore:`、`feat:`、`fix:` 等常规前缀即可
- 推送含 `.github/workflows/` 变更时，需具备能更新 workflow 的 GitHub token 权限（如 `workflow` scope）

## 边界

- 不擅自删除双语中已有的一种语言版本，除非用户明确要求
- 不扩大改动范围：无关主题、布局的「顺手重构」默认不做
