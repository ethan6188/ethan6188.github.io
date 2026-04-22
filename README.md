# ethan6188.github.io

个人博客，基于 **Hugo + PaperMod**，通过 **GitHub Actions** 自动部署到 **GitHub Pages**。

访问地址：

- 中文（默认）：https://ethan6188.github.io/
- English：https://ethan6188.github.io/en/

站点为 **整站双语**（Hugo `zh` + `en`）。主题头部可切换语言。

---

## 本地开发

### 依赖

- [Hugo extended](https://gohugo.io/installation/) (`brew install hugo`)
- Git

### 拉取仓库（含主题 submodule）

```bash
git clone --recurse-submodules https://github.com/ethan6188/ethan6188.github.io.git
cd ethan6188.github.io
```

如果已经 clone 忘了带 submodule：

```bash
git submodule update --init --recursive
```

### 本地预览

```bash
hugo server -D
# 打开 http://localhost:1313
```

`-D` 表示渲染 draft 草稿。

---

## 写一篇新文章（双语）

同一主题建议中英文各一篇，文件名用 **语言后缀** 配对（Hugo 会互相关联）：

```bash
hugo new content posts/my-topic.zh.md
hugo new content posts/my-topic.en.md
```

编辑 `content/posts/my-topic.zh.md` 与 `my-topic.en.md`，把 `draft` 改为 `false` 后推送。

**标签约定**：每篇文章 **`tags` 只放一个**、尽量具体（例如 `Hugo`、`Spark`），避免「文章」「博客」等泛词。

### 归档与搜索页

已拆成 `archives.zh.md` / `archives.en.md`、`search.zh.md` / `search.en.md`，一般无需改动。

### 发布

```bash
git add .
git commit -m "post: my new post"
git push
```

推到 `main` 后，GitHub Actions 会自动：

1. 拉取代码 + submodule 主题
2. 安装 Hugo extended
3. `hugo --gc --minify` 构建 `public/`
4. 部署到 Pages

几十秒后就能在 https://ethan6188.github.io 看到更新。

---

## 目录结构

```
.
├── .github/workflows/deploy.yml   # 自动部署 workflow
├── archetypes/                    # 新文章模板
├── assets/                        # Hugo 处理的资源（SCSS、JS）
├── content/                       # 内容（多语言：*.zh.md / *.en.md）
│   ├── posts/
│   ├── archives.zh.md / archives.en.md
│   └── search.zh.md / search.en.md
├── data/                          # 数据文件
├── i18n/                          # 多语言
├── layouts/                       # 自定义布局（覆盖主题）
├── static/                        # 静态资源（原样拷贝到 public/）
├── themes/PaperMod/               # 主题（git submodule）
└── hugo.toml                      # 站点配置
```

---

## 主题

[PaperMod](https://github.com/adityatelange/hugo-PaperMod)，作为 git submodule 引入。

升级主题：

```bash
git submodule update --remote themes/PaperMod
```
