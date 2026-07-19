---
title: "Before Reading the Code, Read the Git History: A Five-Command Checkup"
date: 2026-07-19T16:33:00+08:00
draft: false
tags: []
categories: ["Engineering Practice"]
description: "Notes on Ally Piechowski's article \"The Git Commands I Run Before Reading Any Code\": five git log commands that reveal churn hotspots, bus factor, bug clustering, project momentum, and firefighting frequency before you open a single source file."
---

When handed an unfamiliar codebase, most people's first instinct is to open the source tree and
start browsing files. In
[The Git Commands I Run Before Reading Any Code](https://piechowski.io/post/git-commands-before-reading-code/),
Ally Piechowski suggests a different route: **don't read the code yet — read the git history
first**. The commit log carries strong signals about code quality, team health, and project
risk, and five commands are enough to surface them. This post is my summary of the article.

## 1. Churn hotspots: which files change the most

```bash
git log --format=format: --name-only --since="1 year ago" \
  | sort | uniq -c | sort -nr | head -20
```

Lists the 20 most-modified files over the past year. High churn alone isn't necessarily bad,
but **high churn combined with low ownership** marks a classic risk zone: changes there are
patchy and unpredictable, and they quietly inflate the team's estimates. The author cites a
2005 Microsoft Research study showing that *relative* churn — normalized by component size —
predicts defect density better than absolute change counts.

## 2. Bus factor: how many shoulders the project rests on

```bash
git shortlog -sn --no-merges
```

Ranks contributors by commit volume. A single person accounting for 60%+ of commits is a
critical dependency risk — especially if that person hasn't appeared in the log for six
months. The author also recommends checking whether the top historical contributors are still
active recently.

**Caveat**: squash-merge workflows obscure true authorship — the stats reflect who *merged*
the code, not who *wrote* it.

## 3. Bug clustering: which files keep breaking

```bash
git log -i -E --grep="fix|bug|broken" --name-only --format='' \
  | sort | uniq -c | sort -nr | head -20
```

Surfaces files that appear in commits whose messages mention fix / bug / broken. Cross-check
this list against the churn list from step 1: the intersection is your highest-risk code —
**files that break repeatedly yet never got properly refactored**.

**Caveat**: this one is only as trustworthy as the team's commit-message discipline.

## 4. Project momentum: is it rising or fading

```bash
git log --format='%ad' --date=format:'%Y-%m' | sort | uniq -c
```

Counts commits per month across the repo's entire history. A steady rhythm signals health;
a sharp drop often maps to people leaving; a long decline means momentum is draining away;
periodic spikes suggest batched releases rather than continuous delivery. The author recalls
a CTO who had never connected their declining commit velocity to the moment senior engineers
departed — these charts are really about organizational dynamics.

## 5. Firefighting frequency: does the team trust its own releases

```bash
git log --oneline --since="1 year ago" \
  | grep -iE 'revert|hotfix|emergency|rollback'
```

Counts reverts and hotfixes. The occasional one is normal, but frequent emergency commits
mean the team doesn't trust its own deploy process — usually rooted in unreliable tests, a
missing staging environment, or painful rollbacks.

**Caveat**: a result of zero has two readings — genuine stability, or merely vague commit
messages. Interpret in context.

## Takeaway

What these five commands share: without reading a single line of source, they answer "where
are the minefields", "whose head holds the critical knowledge", and "what shape is the team
in right now". Spending five minutes on them before deciding where to start reading usually
beats diving straight into `src/`. The original article goes deeper — worth a
[full read](https://piechowski.io/post/git-commands-before-reading-code/).
