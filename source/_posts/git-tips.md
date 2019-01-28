---
title: git tips
date: 2019-01-28 13:00:47
tags: git
categories: tools
---

> 日常 git tips，持续更新...

## 撤销单个文件修改

1. 还没有`add`到`stage`

- `git checkout filename`

2. `add`到`stage`但还没有`commit`

- `git reset HEAD filename` // 移除暂存区
- `git checkout filename`

3. 已经`commit`

- `git reset HEAD^` // 回退
- `git checkout filename`

## `push`时遇到 missing tree / unpackaed error

- `git push --no-thin xxxx` // 禁止 thin pack transfer 优化

## `commit`之后需要修改`author`

- `git commit --amend --author="username <username@meow.com>"`

## `push`之后修改特定的`commit`

- `git rebase -i HEAD~N` // 修改前`N`个commits
- 将对应要修改的commit前面的`pick`改成`edit`（`p` -> `e`），保存退出
- `git log` 检查当前 `HEAD` 已经位于要修改的commit
- 修改需要修改的文件，`git add -u`，`git commit --amend`，`git push xxx` // 再次提交本commit
- `git rebase --continue` // 恢复
- **PS：`squash`可以将多个commit合并为1个**