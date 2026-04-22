---
title: "Hello World: the blog is live"
date: 2026-04-22T20:00:00+08:00
draft: false
tags: ["Hugo"]
categories: ["Notes"]
description: "First post: why I'm writing again, and the stack behind this site."
---

## Why this blog

Ideas end up scattered across Slack threads, notes, and chats. A blog is a small forcing function:
if I cannot explain it clearly, I probably do not understand it yet.

## What I plan to write

- **Engineering practice**: debugging stories, tuning, and lessons learned
- **System design**: distributed systems and platform thinking
- **Tools & productivity**: small automations that improve day-to-day work
- **Reading & life notes**: not everything is code

## Stack

This site is built with [Hugo](https://gohugo.io/) and the
[PaperMod](https://github.com/adityatelange/hugo-PaperMod) theme.
Source and posts live in a GitHub repository; pushing to `main` triggers GitHub Actions, which builds and deploys to GitHub Pages.

```bash
git add .
git commit -m "post: new article"
git push
```

After a short wait, changes show up at https://ethan6188.github.io/.

## Next

No fixed plan yet. The goal is to keep the pipeline running and fill in content over time.
