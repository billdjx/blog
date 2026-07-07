---
title: Git 常用命令笔记
date: 2026-07-06 18:01:30
tags:
  - Git
  - 版本控制
  - 效率
categories:
  - 技术笔记
---

## 前言

日常开发中 Git 用的频率很高，但有些命令总记不住。这篇笔记整理了最常用的 Git 命令，方便查阅。

---

## 基础操作

```bash
# 克隆仓库
git clone <repository-url>

# 查看状态
git status

# 添加文件到暂存区
git add <file>          # 添加指定文件
git add .               # 添加所有变更

# 提交
git commit -m "提交说明"

# 推送
git push origin <branch>

# 拉取最新代码
git pull origin <branch>
```

## 分支操作

```bash
# 查看分支
git branch              # 本地分支
git branch -r           # 远程分支
git branch -a           # 所有分支

# 创建并切换分支
git checkout -b <branch-name>
# 或（新版 Git）
git switch -c <branch-name>

# 切换分支
git checkout <branch-name>
# 或
git switch <branch-name>

# 合并分支
git merge <branch-name>

# 删除分支
git branch -d <branch-name>     # 本地
git push origin --delete <branch-name>  # 远程
```

## 撤销与回退

```bash
# 撤销工作区修改
git checkout -- <file>

# 撤销暂存区
git reset HEAD <file>

# 回退到某个 commit
git reset --soft HEAD~1     # 保留工作区和暂存区
git reset --mixed HEAD~1    # 保留工作区，清空暂存区（默认）
git reset --hard HEAD~1     # 全部丢弃（慎用！）

# 回退到指定 commit
git reset --hard <commit-hash>

# 撤销某个 commit（生成新的撤销 commit）
git revert <commit-hash>
```

## 查看历史

```bash
# 查看提交历史
git log
git log --oneline           # 简洁模式
git log --graph --oneline   # 图形化查看

# 查看某文件的修改历史
git log -p <file>

# 查看某次提交的详情
git show <commit-hash>
```

## 暂存与恢复

```bash
# 暂存当前工作区
git stash

# 查看暂存列表
git stash list

# 恢复暂存
git stash pop              # 恢复并删除
git stash apply            # 恢复但不删除

# 删除暂存
git stash drop
```

## 实用技巧

```bash
# 修改最近一次 commit 信息
git commit --amend -m "新的说明"

# 将多个 commit 合并为一个
git rebase -i HEAD~3

# 查看某行代码是谁改的
git blame <file>

# 清理未跟踪的文件
git clean -fd
```

## 结语

以上是我日常最常用的 Git 命令，不需要都记住，收藏起来需要的时候查一下就行。

持续更新中... 🚀
