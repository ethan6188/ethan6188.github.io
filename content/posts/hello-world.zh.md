---
title: "Hello World：博客上线啦"
date: 2026-04-22T20:00:00+08:00
draft: false
tags: ["Hugo"]
categories: ["杂谈"]
description: "博客的第一篇文章：为什么重启写作，以及这里的技术栈。"
---

## 为什么开这个博客

很多想法散落在各种 Slack、笔记、聊天窗口里，时间一长什么都没留下来。开个博客，
强迫自己把问题想清楚再写出来。

## 这里会写什么

- **工程实践**：工作里踩过的坑、调优过程、排查故事
- **系统设计**：大数据 / 分布式 / 平台侧的架构思考
- **工具与效率**：让自己开发体验更好的小工具与自动化
- **读书与思考**：不完全是技术的内容

## 技术栈

这个博客用 [Hugo](https://gohugo.io/) 构建，主题是
[PaperMod](https://github.com/adityatelange/hugo-PaperMod)，
源码和文章都在 GitHub 仓库里，推到 `main` 分支后 GitHub Actions 会自动构建并部署到 GitHub Pages。

```bash
git add .
git commit -m "post: new article"
git push
```

就这样，几十秒后就能在 https://ethan6188.github.io 上看到更新。

## 下一篇

还没想好。先让这里运转起来，再慢慢填内容。
