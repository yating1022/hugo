---
title: "Git 常用命令速查"
date: 2026-11-18T09:20:00+08:00
tags: ["Git", "版本控制"]
---

日常开发真正高频的 Git 命令，按场景整理。不追求全，只收真正会用的。

<!--more-->

## 一、先配好环境

```bash
git config --global user.name "你的名字"
git config --global user.email "you@example.com"
git config --global core.editor "vim"
git config --global init.defaultBranch main

# 有用的别名
git config --global alias.st status
git config --global alias.co checkout
git config --global alias.br branch
git config --global alias.lg "log --oneline --graph --decorate --all"
```

## 二、日常提交

```bash
git status -s              # 简短状态
git add -p                 # 逐块选择要提交的改动，比 git add . 精细得多
git commit -m "feat: 说明"
git commit --amend         # 修补最后一次提交（还没推送时用）
```

`git add -p` 是减少无意义 diff 的关键：一次提交只做一件事，review 的人会感谢你。

## 三、分支操作

```bash
git switch -c feature/login     # 新建并切换（比 checkout -b 语义清楚）
git switch main                 # 切回
git branch -d 分支名            # 删除已合并的分支
git branch -D 分支名            # 强制删除
git branch -vv                  # 看本地分支对应哪个远程
```

## 四、合并与变基

```bash
git merge feature            # 快进合并为主
git merge --no-ff feature    # 强制生成合并提交，保留分支痕迹
git rebase main              # 把当前分支变基到 main 上，历史更线性

git rebase -i HEAD~3         # 整理最近 3 个提交（squash / reword / drop）
```

经验：**个人分支用 rebase 保持干净，公共分支永远不要 rebase**，否则会把别人的历史改掉。

## 五、撤销（最容易搞混的一组）

| 目标 | 命令 |
|---|---|
| 丢弃工作区某个文件的修改 | `git restore 文件名` |
| 把文件从暂存区拿回来（不影响改动） | `git restore --staged 文件名` |
| 撤销最后一次提交，改动留在工作区 | `git reset --soft HEAD~1` |
| 撤销最后一次提交，改动留在暂存区 | `git reset --mixed HEAD~1` |
| **彻底丢弃**最后一次提交和改动 | `git reset --hard HEAD~1` |
| 撤销一个**已推送**的提交 | `git revert 提交号` |

已经推送的提交不要用 `reset` 硬改历史，用 `revert` 反向提交一次，安全且不影响别人。

## 六、找回误删的东西

```bash
git reflog
```

`reflog` 记录了 HEAD 的每一次移动，误 `reset --hard`、误删分支之后都能靠它捞回来：

```bash
git reset --hard HEAD@{3}      # 回到 reflog 里的某个位置
git switch -c 救回来的分支 提交号
```

这条命令救过很多人，值得记住。

## 七、查历史

```bash
git log --oneline --graph --all --decorate
git log -p 文件名              # 某个文件的完整变更历史
git log -S "关键词"            # 哪次提交引入/删除了这段代码（俗称搜代码的皮卡丘）
git blame 文件名               # 每行是谁改的
git diff --stat HEAD~5 HEAD    # 最近 5 次提交动了哪些文件
```

`git log -S` 在排查「这段逻辑什么时候没的」时非常好用。

## 八、暂存现场

```bash
git stash              # 收起当前改动
git stash -u           # 连未跟踪文件一起收
git stash list
git stash pop          # 取回并删除记录
git stash apply        # 取回但保留记录
```

临时切分支修 bug 时必备。

## 九、摘取单个提交

```bash
git cherry-pick 提交号
```

把另一个分支的某个提交单独摘过来，比整分支合并灵活。冲突时解决完 `git cherry-pick --continue`。

## 十、远程协作

```bash
git fetch --prune              # 拉取并清理已删除的远程分支引用
git pull --rebase              # 拉取时用变基，避免无意义的 merge 提交
git push -u origin 分支名      # 首次推送并建立跟踪
git push --force-with-lease    # 比 --force 安全：别人有新提交时会拒绝
```

用 `--force-with-lease` 替代 `--force`，这是条硬规矩。

## 十一、标签

```bash
git tag -a v1.0.0 -m "第一个正式版本"
git push origin v1.0.0
git tag                        # 列出所有标签
```

## 十二、冲突处理流程

1. `git status` 看哪些文件冲突
2. 打开文件，找 `<<<<<<<` `=======` `>>>>>>>` 标记，手工决定保留哪边
3. 删掉冲突标记
4. `git add 文件` 标记已解决
5. `git rebase --continue` 或 `git commit`

冲突不可怕，可怕的是不看内容直接选「接受全部」。

## 十三、几个实用技巧

```bash
# 查看某行代码是谁在哪次提交改的
git log -L 10,20:文件路径

# 临时保存某个文件的当前版本再恢复
git stash push 文件名

# 统计代码量贡献
git shortlog -sn

# 忽略本地修改（不提交，但保留改动）
git update-index --skip-worktree 配置文件
```

---

Git 的学习曲线主要在「撤销」这一块。记住一句话：**只要提交过，就基本丢不了**，`reflog` 是最后的保险。
