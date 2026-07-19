---
title: "读代码之前，先读 Git 历史：五条命令给项目做体检"
date: 2026-07-19T16:33:00+08:00
draft: false
tags: ["Git"]
categories: ["工程实践"]
description: "Ally Piechowski 的文章《The Git Commands I Run Before Reading Any Code》读后摘要：接手陌生代码库时，先用五条 git log 命令看清变更热点、Bus Factor、Bug 聚集、项目节奏与救火频率。"
---

接手一个陌生代码库时，大多数人的第一反应是打开源码目录开始翻文件。Ally Piechowski 在
[The Git Commands I Run Before Reading Any Code](https://piechowski.io/post/git-commands-before-reading-code/)
里给出了另一条路：**先别读代码，先读 git 历史**。提交记录里藏着代码质量、团队健康度和项目风险的大量信号，
五条命令就能把它们挖出来。本文是这篇文章的摘要笔记。

## 1. 变更热点：哪些文件被改得最多

```bash
git log --format=format: --name-only --since="1 year ago" \
  | sort | uniq -c | sort -nr | head -20
```

列出过去一年改动次数最多的 20 个文件。高频变更本身不一定是坏事，但**高变更 + 低所有权**的文件
是典型的风险区：改动零碎、难以预测，还会悄悄拖垮团队的排期估算。作者引用了微软研究院 2005 年的
一项研究：按组件规模归一化后的「相对 churn」，比绝对改动次数更能预测缺陷密度。

## 2. Bus Factor：项目压在几个人身上

```bash
git shortlog -sn --no-merges
```

按提交量给贡献者排序。如果一个人占了 60% 以上的提交，这就是一个关键依赖风险——尤其当这个人
已经半年没出现在提交记录里。作者建议顺手确认一下：历史上的头部贡献者，最近是否还活跃。

**注意**：squash merge 工作流会掩盖真实作者——统计反映的是「谁合并的」，不是「谁写的」。

## 3. Bug 聚集：哪些文件反复出问题

```bash
git log -i -E --grep="fix|bug|broken" --name-only --format='' \
  | sort | uniq -c | sort -nr | head -20
```

找出提交信息里带 fix / bug / broken 的文件。把这份榜单和第 1 条的 churn 榜单对照着看，
交集就是风险最高的代码：**反复出 bug、却从未被认真重构的文件**。

**注意**：这条命令的可信度取决于团队写提交信息的纪律。

## 4. 项目节奏：动量是在涨还是在跌

```bash
git log --format='%ad' --date=format:'%Y-%m' | sort | uniq -c
```

按月份统计整个仓库历史的提交量。平稳的节奏说明项目健康；断崖式下跌往往对应人员流失；
持续下滑说明动量在流失；周期性的尖峰则暗示是攒一批再发布，而非持续交付。
作者提到一个案例：某位 CTO 一直没把提交速度的下滑，和资深工程师离职的时间点联系起来——
这些图形反映的其实是组织层面的动态。

## 5. 救火频率：团队信不信自己的发布流程

```bash
git log --oneline --since="1 year ago" \
  | grep -iE 'revert|hotfix|emergency|rollback'
```

数一数 revert 和 hotfix。偶尔出现很正常，但频繁的紧急提交说明团队不信任自己的部署流程——
背后通常是测试不可靠、缺少 staging 环境，或者回滚很痛苦。

**注意**：结果为零有两种解释——要么真的稳定，要么只是提交信息没写清楚，需要结合上下文判断。

## 小结

这五条命令的共同点是：都不看一行源码，却能回答「这个项目的雷区在哪」「关键知识在谁脑子里」
「团队现在的状态怎么样」。花五分钟跑一遍，再决定从哪里开始读代码，往往比直接扎进
`src/` 目录高效得多。原文还有更多展开，推荐阅读
[原文](https://piechowski.io/post/git-commands-before-reading-code/)。
