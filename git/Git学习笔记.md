# Git 学习笔记

> 整理日期：2026-08-01
> 内容：Git 核心区域、工作流程命令、实战案例、rebase、取消追踪等完整总结

---

## 目录

1. [Git 的四个核心区域](#一git-的四个核心区域)
2. [完整工作流程（命令含义）](#二完整工作流程命令含义)
3. [完整流程实战案例](#三完整流程实战案例)
4. [分支创建：从本地还是远程](#四分支创建从本地还是远程)
5. [分支命名约定](#五分支命名约定)
6. [git diff：比较已提交版本](#六git-diff比较已提交版本)
7. [git rebase：变基](#七git-rebase变基)
8. [取消追踪文件（git rm --cached）](#八取消追踪文件git-rm---cached)
9. [.gitignore 与 git rm 的关系](#九gitignore-与-git-rm-的关系)
10. [实战场景：误提交 node_modules 的处理](#十实战场景误提交-node_modules-的处理)
11. [速查表与关键概念](#十一速查表与关键概念)

---

## 一、Git 的四个核心区域

理解 Git 工作流，先要明白数据在这四个区域之间流动：

```
┌─────────────┐  git add   ┌──────────────┐  git commit  ┌───────────┐  git push  ┌──────────┐
│  工作区     │ ─────────► │   暂存区     │ ──────────►  │  本地仓库 │ ────────►  │ 远程仓库 │
│ Working Dir │            │ Staging Area │              │ Local Repo│            │ Remote   │
└─────────────┘            └──────────────┘              └───────────┘            └──────────┘
       ▲                         │                            │                        │
       │  git checkout            │  git reset                 │  git reset --hard      │
       └──────────────────────────┴────────────────────────────┘                       │
                                                                                         │
                                                        git pull  ◄────────────────────┘
```

| 区域 | 说明 |
|------|------|
| **工作区** | 你能在文件系统里直接看到、编辑的文件 |
| **暂存区** | `git add` 后进入，是"准备提交"的快照暂存地 |
| **本地仓库** | `git commit` 后，变更被永久记录到本地历史 |
| **远程仓库** | GitHub/GitLab 上的仓库，团队协作的共享中心 |

---

## 二、完整工作流程（命令含义）

### 1. 初始化 / 克隆

```bash
git init                    # 在当前目录初始化一个新仓库（创建 .git/）
git clone <url>             # 从远程仓库克隆完整历史到本地
git clone <url> <dir>       # 克隆到指定目录名
```

### 2. 查看状态（最常用，随时了解"我在哪一步"）

```bash
git status                  # 查看工作区/暂存区状态：哪些文件改了、哪些已暂存
git status -s               # 精简版输出
git diff                    # 查看工作区相对暂存区的改动（还没 add 的）
git diff --staged           # 查看暂存区相对最新提交的改动（已 add、待 commit 的）
git log --oneline --graph   # 查看提交历史（一行一提交，带分支图）
```

### 3. 暂存 -> 提交（最核心的两步）

```bash
git add <file>              # 把指定文件的改动放入暂存区
git add .                   # 暂存当前目录所有改动（含新增）
git add -u                  # 只暂存已跟踪文件的修改/删除（不含新文件）
git add -p                  # 交互式逐块选择暂存（同一文件部分改动）

git commit -m "说明"        # 把暂存区的内容提交为一个新快照，-m 写提交说明
git commit -am "说明"       # 对已跟踪文件：add + commit 一步完成（跳过暂存区）
git commit --amend          # 修改最近一次提交（合并新改动 / 改说明），不要对已推送的提交用
```

> **add 和 commit 为什么分开？** 让你能把一次大修改拆成几个逻辑清晰的小提交。

### 4. 推送到远程

```bash
git push                    # 把本地分支推到远程同名分支
git push origin <branch>    # 显式指定推送到 origin 的某分支
git push -u origin <branch> # 首次推送并建立追踪关系（之后 git push 即可）
```

### 5. 拉取远程更新

```bash
git fetch                   # 下载远程更新，但不合并到工作区（只动远程跟踪分支）
git pull                    # = git fetch + git merge，拉取并合并到当前分支
git pull --rebase           # 拉取后用 rebase 代替 merge，历史更线性
```

### 6. 分支操作（并行开发的关键）

```bash
git branch                  # 列出本地分支，* 标记当前分支
git branch <name>           # 创建分支但不切换
git checkout <branch>       # 切换到某分支
git checkout -b <name>      # 创建并切换（= 上面两步合一）
git switch <branch>         # 新版切换分支命令（推荐，比 checkout 语义更清晰）
git switch -c <name>        # 创建并切换

git merge <branch>          # 把指定分支合并进当前分支
git branch -d <name>        # 删除已合并的分支
git branch -D <name>        # 强制删除（未合并也删）
```

### 7. 撤销操作（容易混淆，重点理解）

```bash
git restore <file>          # 丢弃工作区的修改（还没 add）-- 新版命令
git checkout -- <file>      # 同上（旧写法）

git restore --staged <file> # 把文件从暂存区移回工作区（已 add 但想撤回 add）
git reset HEAD <file>       # 同上（旧写法）

git reset --soft HEAD~1     # 撤销最近一次提交，改动保留在暂存区
git reset --mixed HEAD~1    # 撤销提交 + 撤销暂存，改动保留在工作区（默认）
git reset --hard HEAD~1     # 彻底丢弃最近一次提交及其改动（⚠️不可恢复）
```

| 场景 | 命令 |
|------|------|
| 改了文件还没 add，想丢弃 | `git restore <file>` |
| 已 add，想撤回暂存 | `git restore --staged <file>` |
| 已 commit，想撤销提交但保留改动 | `git reset --soft HEAD~1` |
| 已 commit，想彻底丢弃 | `git reset --hard HEAD~1` ⚠️ |

---

## 三、完整流程实战案例

### 📌 故事背景

写一篇《DaemonSet 学习笔记》，从新建文件到最终合并到主干，完整经历一次 Git 协作流程。

### 第 1 步：配置身份（首次使用必做）

```bash
git config --global user.name "akon"
git config --global user.email "akon@example.com"
```

> **含义**：设置全局的用户名和邮箱，每次 commit 会记录这个身份。`--global` 表示对当前用户所有仓库生效；去掉则只对当前仓库生效。

查看是否已配置：

```bash
git config --list
```

### 第 2 步：查看当前仓库状态

```bash
git status
```

> **含义**：查看工作区状态。假设你已经在仓库里，输出类似：

```
On branch master
nothing to commit, working tree clean
```

说明当前在 `master` 分支，工作区干净（没有未提交的改动）。

### 第 3 步：先拉取最新代码（协作前必做）

```bash
git pull
```

> **含义**：从远程仓库拉取并合并最新代码到当前分支。养成"开始工作前先 pull"的习惯，能避免后面冲突。

如果提示 `Already up to date.`，说明本地已是最新的。

### 第 4 步：创建功能分支（不在主干直接改）

```bash
git switch -c feature/daemonset-note
```

> **含义**：创建并切换到新分支 `feature/daemonset-note`。`-c` = create。`switch` 是新版命令，等价于旧版的 `git checkout -b`。

验证当前分支：

```bash
git branch
# * feature/daemonset-note    <-- * 表示当前所在
#   master
```

### 第 5 步：创建文件并写入内容

```bash
echo "# DaemonSet 学习笔记" > daemonset-note.md
```

> 这是普通 shell 命令，创建一个笔记文件。此刻文件在工作区，但 Git 还没跟踪它。

查看状态：

```bash
git status
```

输出：

```
Untracked files:
  (use "git add <file>..." to include in what will be committed)
        daemonset-note.md
```

> **含义**：Git 发现一个"未跟踪"的新文件，提示你用 `git add` 把它纳入跟踪。

### 第 6 步：暂存文件

```bash
git add daemonset-note.md
```

> **含义**：把文件改动放入**暂存区**。此时文件进入"准备提交"状态，但还没真正提交。

再看状态：

```bash
git status
```

输出变化：

```
Changes to be committed:
  (use "git restore --staged <file>..." to unstage)
        new file:   daemonset-note.md
```

> **含义**：文件已进入暂存区（`Changes to be committed`），等待 commit。

### 第 7 步：提交到本地仓库

```bash
git commit -m "docs: 新增 DaemonSet 学习笔记"
```

> **含义**：把暂存区的内容拍成一个快照，永久写入本地仓库历史。`-m` 后面是提交说明。

查看提交历史：

```bash
git log --oneline --graph -5
```

输出类似：

```
* 9a3b1c2 (HEAD -> feature/daemonset-note) docs: 新增 DaemonSet 学习笔记
* c055faa (master) docs: 添加 StatefulSet 与有状态工作负载学习笔记
* 4c4a8d0 deploy
* ...
```

> **含义**：`--oneline` 每个提交一行；`--graph` 显示分支拓扑；`-5` 只看最近 5 条。`HEAD` 指向当前分支最新提交。

### 第 8 步：继续修改，演示"暂存区的作用"

继续往笔记里补充内容：

```bash
echo "## DaemonSet 的特点" >> daemonset-note.md
echo "## 典型使用场景" >> daemonset-note.md
```

此时查看差异（**还没 add**）：

```bash
git diff
```

输出：

```
+## DaemonSet 的特点
+## 典型使用场景
```

> **含义**：`git diff` 显示工作区相对暂存区的改动--也就是"改了但还没暂存"的部分。

现在**只暂存一部分**，体验分块暂存：

```bash
git add -p daemonset-note.md
```

> **含义**：`-p` = patch 模式，交互式逐块选择。Git 会逐块问你 `y`(暂存)/`n`(不暂存)/`s`(拆更细)。这样你能把一次修改拆成多个提交。

简化演示，直接全部暂存并提交：

```bash
git add daemonset-note.md
git commit -m "docs: 补充 DaemonSet 特点与使用场景"
```

### 第 9 步：推送到远程仓库

首次推送当前分支：

```bash
git push -u origin feature/daemonset-note
```

> **含义**：把本地 `feature/daemonset-note` 分支推送到远程 `origin` 的同名分支。`-u`（`--set-upstream`）建立追踪关系，**之后**只需 `git push` 即可。

之后再次推送就简化为：

```bash
git push
```

### 第 10 步：模拟队友更新了主干（拉取与合并）

假设在你写笔记期间，队友往 `master` 推送了新提交。你切回主干更新：

```bash
git switch master
git pull
```

> **含义**：切回 `master` 并拉取最新代码。现在你的本地 `master` 比分支创建时多了队友的提交。

切回功能分支，把主干的新内容合并进来（防止冲突积压）：

```bash
git switch feature/daemonset-note
git merge master
```

> **含义**：把 `master` 合并进当前分支。如果两边改了不同文件/不同位置，会自动合并；如果改了同一处，会**冲突**。

#### 如果出现冲突

冲突时 Git 会暂停，状态显示 `Unmerged paths`，文件里出现标记：

```
<<<<<<< HEAD
你写的内容
=======
队友写的内容
>>>>>>> master
```

解决步骤：

```bash
# 1. 手动编辑文件，删掉 <<<<<<< ======= >>>>>>> 标记，保留正确内容
# 2. 标记冲突已解决
git add daemonset-note.md
# 3. 完成合并提交
git commit -m "merge: 合并 master 解决冲突"
```

### 第 11 步：演示撤销操作（三种常见场景）

#### 场景 A：改了文件还没 add，想丢弃

```bash
echo "这行写错了" >> daemonset-note.md
git restore daemonset-note.md      # 丢弃工作区改动，恢复成最近提交的样子
```

> **含义**：`git restore <file>` 丢弃工作区的未暂存改动（旧写法 `git checkout -- <file>`）。

#### 场景 B：已 add，想撤回暂存

```bash
echo "测试内容" >> daemonset-note.md
git add daemonset-note.md
git restore --staged daemonset-note.md   # 从暂存区移出，改动仍保留在工作区
```

> **含义**：`--staged` 表示"反向暂存"，即把文件从暂存区退回工作区（旧写法 `git reset HEAD <file>`）。

#### 场景 C：已 commit，想撤销最近一次提交

```bash
# 撤销提交但保留改动在工作区（最常用、最安全）
git reset --soft HEAD~1

# 想彻底丢弃（危险！）
git reset --hard HEAD~1
```

> **含义**：`HEAD~1` = 上一个提交。`--soft` 只动 HEAD 指针，改动保留在暂存区；`--hard` 连工作区一起清空，**不可恢复**。

### 第 12 步：合并分支到主干（两种方式）

#### 方式一：本地合并（小项目/个人项目常用）

```bash
git switch master              # 切回主干
git merge feature/daemonset-note   # 把功能分支合并进来
git push                       # 推送到远程主干
```

#### 方式二：Pull Request（团队协作推荐）

在 GitHub 网页上对 `feature/daemonset-note` 分支发起 PR，团队评审通过后点击 Merge。本地再拉取：

```bash
git switch master
git pull
```

### 第 13 步：清理已合并的分支

```bash
git branch -d feature/daemonset-note          # 删除本地分支（-d 要求已合并）
git push origin --delete feature/daemonset-note  # 删除远程分支
```

> **含义**：合并完成后清理分支，保持仓库整洁。`-d`（小写）会检查是否已合并，安全；`-D`（大写）强制删除。

### 第 14 步：打标签（标记里程碑版本）

```bash
git tag v1.0 -m "完成 Kubernetes 工作负载系列笔记"
git push origin v1.0            # 推送标签到远程（标签不会随 push 自动推送）
```

> **含义**：`tag` 给某个提交打上版本标记，常用于发布。`-m` 带说明。

---

## 四、分支创建：从本地还是远程

**`git switch -c <新分支>` 默认从当前 HEAD 创建**，也就是从你当前所在的本地分支的最新提交创建，**不是**从远程分支创建。

### 四种创建方式对比

#### 1. 从当前本地分支创建（默认行为）

```bash
git switch master                 # 先切到本地 master
git switch -c feature/new         # 从本地 master 的当前提交创建新分支
```

> 新分支的起点 = 本地 master 此刻的提交。**如果本地 master 不是最新的，新分支也不是最新的。**

这正是案例里的写法--之所以能保证新分支是新的，是因为**前一步先执行了 `git pull`** 把本地 master 更新到最新。

#### 2. 显式从远程分支创建（最稳妥）

```bash
git switch -c feature/new origin/master
```

> 直接以远程 `origin/master` 为起点创建，不依赖本地分支是否最新。即使本地 master 落后，新分支也基于远程最新代码。

#### 3. 自动追踪创建（DWIM 行为，不带 `-c`）

```bash
git switch feature/new
```

> 如果本地没有 `feature/new`，但远程存在 `origin/feature/new`，Git 会**自动**创建本地分支并建立追踪关系。这叫 "Do What I Mean" 行为。前提是远程已有该分支（比如队友先推了）。

#### 4. 从指定提交/标签创建

```bash
git switch -c hotfix abc1234      # 从某个具体 commit 创建
git switch -c release v1.0        # 从某个 tag 创建
```

### 关键区别图示

```
情况 A：本地 master 落后于远程
远程:  origin/master ──●──●──●──●(队友的新提交)  最新
本地:  master ────────●──●──●                     落后1个

git switch -c feature/x              -> 从本地 ●(落后) 创建 ❌ 不是最新
git switch -c feature/x origin/master -> 从远程 ●(最新) 创建 ✅
```

### 实际开发建议

| 场景 | 推荐命令 |
|------|---------|
| 个人项目、本地 master 刚 pull 过 | `git pull && git switch -c feature/x` |
| 想确保基于远程最新代码 | `git switch -c feature/x origin/master` |
| 远程已有该分支，想拉到本地 | `git switch feature/x`（自动追踪） |

---

## 五、分支命名约定

`feature/daemonset-note` 就是一个**分支名**，按**命名约定**写，分两部分：

```
feature  /  daemonset-note
  │            │
  │            └── 具体做什么（这个分支的内容描述）
  └── 分支类型（前缀分类）
```

### 1. Git 本身不关心这个名字

对 Git 来说，`feature/daemonset-note` 和 `abc123`、`my-branch` 没有任何区别，都只是一个普通分支名字。Git 不会因为叫 `feature/` 就给它特殊功能。

### 2. `/` 斜杠的作用

斜杠在 Git 里**会被当成路径分隔符**，所以：

- 在 GitHub、GitLab、各类 Git GUI 工具里，`feature/daemonset-note` 会**显示成 `feature` 文件夹下折叠的样子**，方便分组管理
- 在底层，分支其实是存在 `.git/refs/heads/feature/daemonset-note` 这个路径下的文件

### 3. 常见的命名约定（团队规范）

| 前缀 | 含义 | 示例 |
|------|------|------|
| `feature/` | 新功能开发 | `feature/user-login` |
| `bugfix/` | 修复 bug | `bugfix/null-pointer` |
| `hotfix/` | 紧急修复（通常直接合主干） | `hotfix/payment-crash` |
| `release/` | 发布准备 | `release/v2.0` |
| `docs/` | 文档改动 | `docs/api-reference` |
| `refactor/` | 重构 | `refactor/auth-module` |
| `chore/` | 杂项（依赖升级、配置等） | `chore/upgrade-deps` |

> 这套约定源自 **Git Flow** 工作流，现在被广泛采用，但**不是强制**的，团队可以自定义。

---

## 六、git diff：比较已提交版本

把 `git diff` 理解成**"比较任意两个版本之间的差异"**，而版本可以是工作区、暂存区、或任意一个提交。

### 1. 三个"比较对象"

```
提交历史        暂存区        工作区
HEAD~1 ── HEAD ── 暂存区 ── 工作区
  │        │        │         │
  └────────┴────────┴─────────┘
      这些都可以作为 diff 的两端
```

### 2. `git diff` 参数规则

```bash
git diff              # 暂存区  vs  工作区     （省略两端，取默认值）
git diff <X>          # X       vs  工作区     （给一个端，另一端默认工作区）
git diff <X> <Y>      # X       vs  Y          （明确指定两端）
git diff --staged     # HEAD    vs  暂存区     （--staged 固定比较暂存区）
```

### 3. 比较"已提交的版本"

```bash
git diff HEAD             # 工作区 vs 最新提交（含未 add 的改动）
git diff HEAD~1           # 工作区 vs 上一个提交
git diff HEAD~1 HEAD              # 上一个提交 vs 最新提交
git diff <commit1> <commit2>      # 任意两个提交之间
git diff master feature/x         # 两个分支之间
git diff HEAD~1 HEAD -- <文件>    # 某文件在两次提交间的差异（-- 过滤文件）
```

### 4. 速查表

| 我想比较... | 命令 |
|-----------|------|
| 工作区 vs 暂存区（还没 add 的） | `git diff` |
| 暂存区 vs 最新提交（已 add 待 commit） | `git diff --staged` |
| 工作区 vs 最新提交（所有未提交改动） | `git diff HEAD` |
| 工作区 vs 上一个提交 | `git diff HEAD~1` |
| 两个提交之间 | `git diff HEAD~1 HEAD` |
| 两个分支之间 | `git diff master feature/x` |
| **某文件在两次提交间的差异** | `git diff HEAD~1 HEAD -- 文件` |

### 5. 补充：查看"某次提交改了什么"

如果目的不是比较两个版本，而是想看**某一次提交具体做了哪些改动**，用 `git show` 更直接：

```bash
git show                       # 显示最新提交的改动内容
git show HEAD~1                # 显示上一个提交改了什么
git show <commit-hash>         # 显示指定提交的改动
git show <commit-hash> -- <文件>  # 只看那次提交对某文件的改动
```

> `git show <commit>` = 显示该提交的元信息 + 它相对父提交的 diff，等价于 `git diff <commit>^ <commit>`。

---

## 七、git rebase：变基

`git rebase` 的核心是**"变基"--把分支的起点挪到另一个提交上，让历史变成一条直线**。

### 1. rebase 和 merge 的本质区别

场景：feature 分支从 master 分出来后，master 又有了新提交

```
      A---B---C  feature       （你在 feature 上做了3个提交）
     /
*---*---D---E  master          （master 上别人又推了2个提交）
```

#### 方式一：merge（合并）

```bash
git switch feature
git merge master
```

```
      A---B---C-------M  feature       （M 是合并提交）
     /             /
*---*---D---E-----┘  master
```

> 会产生一个**合并提交 M**，历史是**分叉再汇合**的，保留了"两条线"的真实记录。

#### 方式二：rebase（变基）

```bash
git switch feature
git rebase master
```

```
              A'--B'--C'  feature      （A'B'C' 是"重新应用"后的新提交）
             /
*---*---D---E  master
```

> 把 A、B、C 的改动**"摘下来"**，逐个**重新应用**到 master 最新提交 E 之上，变成 A'、B'、C'（**新的 commit hash**）。历史变成**一条直线**，就好像 feature 一直基于最新的 master 开发一样。

### 2. 两者的取舍

| | merge | rebase |
|---|------|--------|
| 历史 | 分叉+合并提交，真实记录 | 线性、干净 |
| commit hash | 不变 | **会变**（重写历史） |
| 冲突 | 一次性解决 | 可能逐个提交都要解决 |
| 适用 | 公共分支、团队协作 | 个人功能分支整理历史 |

### 3. 基本用法

```bash
git rebase master              # 把当前分支变基到 master 最新提交
git rebase origin/master       # 变基到远程 master（更常用，确保基于最新）
git rebase --continue          # 解决冲突后继续 rebase
git rebase --abort             # 放弃 rebase，回到 rebase 前的状态
git rebase --skip              # 跳过当前提交（该提交的改动已被上游包含）
```

#### rebase 遇到冲突的处理流程

```bash
git rebase master
# 冲突！Git 暂停，提示哪些文件冲突

# 1. 手动解决冲突（编辑文件，删掉 <<<<<<< ======= >>>>>>> 标记）
# 2. 标记已解决
git add <冲突文件>
# 3. 继续 rebase（不是 commit！）
git rebase --continue

# 如果中途想反悔：
git rebase --abort              # 完全撤销，回到 rebase 之前
```

> 注意：rebase 冲突解决后用 `git rebase --continue`，**不是** `git commit`。因为 rebase 是逐个提交重放，每解决一个提交的冲突就 continue 进入下一个。

### 4. 交互式 rebase（最强大的用法）⭐

```bash
git rebase -i HEAD~3           # 交互式整理最近 3 个提交
```

执行后打开编辑器，列出最近 3 个提交：

```
pick 9a3b1c2 docs: 新增 DaemonSet 学习笔记
pick 8f2a4d1 docs: 补充特点
pick 3c1b9e0 docs: 补充使用场景
```

把每行开头的 `pick` 改成动作词即可：

| 动作 | 含义 |
|------|------|
| `pick`（p） | 保留该提交（默认） |
| `reword`（r） | 保留提交，但修改提交说明 |
| `squash`（s） | 把该提交**合并进上一个提交**（说明也合并） |
| `fixup`（f） | 同 squash，但**丢弃**该提交的说明 |
| `edit`（e） | 暂停，让你修改该提交的内容 |
| `drop`（d） | **删除**该提交 |
| 调整顺序 | 直接交换行顺序即可 |

#### 典型场景

**场景 1：把 3 个零碎提交合并成 1 个干净的提交**

```
pick   9a3b1c2 docs: 新增 DaemonSet 学习笔记
squash 8f2a4d1 docs: 补充特点
squash 3c1b9e0 docs: 补充使用场景
```

**场景 2：删掉某个提交**

```
pick   9a3b1c2 docs: 新增 DaemonSet 学习笔记
drop   8f2a4d1 docs: 补充特点        # 删掉这行对应的提交
pick   3c1b9e0 docs: 补充使用场景
```

**场景 3：只改某个提交的说明**

```
reword 9a3b1c2 docs: 新增 DaemonSet 学习笔记   # 改这行
pick   8f2a4d1 docs: 补充特点
pick   3c1b9e0 docs: 补充使用场景
```

### 5. rebase 的黄金法则 ⚠️

> **永远不要对已经推送到远程、且别人可能基于它工作的分支做 rebase。**

原因：rebase 会**重写历史**（生成新的 commit hash）。如果别人已经基于旧的提交在开发，你 rebase 后强行推送，别人的本地就会和远程对不上，造成混乱。

```
✅ 安全：rebase 自己的功能分支（还没推送，或只有你一个人用）
❌ 危险：rebase master / 共享分支后 git push --force
```

如果确实需要 rebase 已推送的**个人分支**，要用：

```bash
git push --force-with-lease     # 比 --force 安全，会检查远程是否有别人的新提交
```

### 6. 一句话总结

- **merge** 保留真实历史，安全，适合公共分支
- **rebase** 重塑历史，干净，适合整理个人分支
- **交互式 rebase**（`-i`）是整理提交历史的利器：合并、改说明、删提交、调顺序
- **黄金法则**：别 rebase 公共分支

---

## 八、取消追踪文件（git rm --cached）

核心命令是 `git rm --cached <文件>`，关键是 `--cached` 这个参数。

### 1. 核心命令

```bash
git rm --cached <file>          # 停止追踪，但保留本地文件 ⭐（最常用）
git rm <file>                   # 停止追踪，同时删除本地文件
git rm -r --cached <目录>        # 停止追踪整个目录，保留本地文件
```

> **`--cached` 的含义**：只从 Git 索引（暂存区）里移除该文件，**不碰工作区的实际文件**。这样文件还在你电脑上，但 Git 不再追踪它了。

### 2. `--cached` 的区别（重要）

| 命令 | Git 追踪 | 本地文件 |
|------|---------|---------|
| `git rm --cached .env` | 停止追踪 | **保留** ✅ |
| `git rm .env` | 停止追踪 | **删除** ❌ |

所以取消追踪文件，**一定要带 `--cached`**，否则本地文件就没了。

### 3. 查看哪些文件被追踪

```bash
git ls-files                    # 列出所有被追踪的文件
git ls-files *.env              # 筛选特定文件
```

### 4. 速查表

| 需求 | 命令 |
|------|------|
| 停止追踪文件，本地保留 | `git rm --cached <file>` |
| 停止追踪目录，本地保留 | `git rm -r --cached <dir>` |
| 停止追踪并删除本地文件 | `git rm <file>` |
| 查看被追踪的文件 | `git ls-files` |

---

## 九、.gitignore 与 git rm 的关系

### 1. 关键事实：`.gitignore` 只对"未追踪"的文件生效

一旦文件**已经被 Git 追踪**，`.gitignore` 就**完全失效**了--Git 会继续追踪它的每一次改动。

做个实验：

```bash
git add config.ini            # 1. 开始追踪
git commit -m "add config"    # 2. 已被追踪

echo "config.ini" >> .gitignore   # 3. 加进 ignore（试图让它"消失"）

# 4. 修改 config.ini
echo "new setting" >> config.ini

git status                    # 5. 仍然显示 config.ini 被修改！
```

> 原因：Git 已经"认识"这个文件了（索引里有记录），`.gitignore` 的作用是"决定要不要认识新文件"，对"已经认识的文件"没有发言权。

所以必须**先用 `git rm --cached` 让 Git"忘记"这个文件**，`.gitignore` 才能开始生效。

### 2. 两者的本质区别

| | `.gitignore` | `git rm --cached` |
|---|---|---|
| 作用对象 | **未追踪**的文件 | **已追踪**的文件 |
| 性质 | **预防**（别让新文件被追踪） | **治疗**（让已追踪的文件停止追踪） |
| 生效时机 | 文件还没进 Git 之前 | 文件已经进了 Git 之后 |

它们是**互补**关系：

```
还没追踪 ──(.gitignore 拦截)──► 永远不被追踪      ✅ 预防成功
                                    │
                              若已误追踪 ↓
已追踪 ──(git rm --cached)──► 取消追踪 ──(.gitignore 接管)──► 不再被追踪  ✅ 治愈
```

### 3. 为什么 `git rm --cached` 不自动写 `.gitignore`？

四个原因：

1. **职责单一，不做隐式副作用**：`git rm` 的职责就是"从索引移除文件"，偷偷改 `.gitignore` 会让使用者困惑
2. **`.gitignore` 用的是模式匹配，不是文件路径**：取消追踪 `app.log`，ignore 该写 `app.log`、`*.log` 还是 `logs/`？忽略范围不同，Git **无法替你决定**
3. **可能已经有规则覆盖了**：文件可能已被某条 `.gitignore` 规则匹配，自动写入反而产生冗余
4. **你可能压根不想忽略它**：取消追踪 ≠ 想忽略，可能改用别的机制管理

### 4. 一句话总结

- **`.gitignore` 是"门卫"**：只拦还没进门的新文件，对已经在里面的文件无能为力
- **`git rm --cached` 是"请出去"**：把已经进门的文件请出去，门卫才管得到它
- 两者必须**配合使用**：先请出去（`git rm --cached`），再设门卫（`.gitignore`）

---

## 十、实战场景：误提交 node_modules 的处理

### 场景

不小心把 `node_modules` 文件夹提交并 push 了，现在想：删除远程仓库的 node_modules，保留本地的，同时取消追踪。

### 完整处理步骤（4 步）

```bash
# 第1步：停止追踪 node_modules，本地文件保留
git rm -r --cached node_modules

# 第2步：加入 .gitignore，防止以后又被追踪
echo "node_modules/" >> .gitignore

# 第3步：提交这个"取消追踪"的变更
git add .gitignore
git commit -m "chore: 停止追踪 node_modules"

# 第4步：推送到远程（普通 push，不需要 force）
git push
```

### 每一步发生了什么

| 步骤 | 命令 | 效果 |
|------|------|------|
| 1 | `git rm -r --cached node_modules` | 从 Git 索引移除整个目录，**本地文件保留** ✅。`-r` 递归，`--cached` 是关键 |
| 2 | `echo ... >> .gitignore` | 设"门卫"，防止以后 `npm install` 后又被追踪 |
| 3 | `git commit` | 把"删除追踪"记录成一个提交 |
| 4 | `git push` | 推送后，**远程仓库的 node_modules 被删除**，本地保留 ✅ |

> **为什么第 4 步不用 `--force`？** 因为没有重写历史，只是新增了一个"删除 node_modules"的普通提交，是正常的前进式更新，普通 `push` 即可。`--force` 只在 rebase 等重写历史的场景才需要。

### ⚠️ 两个需要注意的问题

#### 问题 1：历史提交里仍然有 node_modules

上面的做法只在**最新版本**删除了 node_modules，但**历史提交里仍然保留着**。带来的影响：

- 仓库 `.git` 体积**不会变小**
- 别人 clone 时仍会下载完整历史（包含 node_modules）

**如果 node_modules 很大，想让历史也彻底清除**，需要重写历史：

```bash
# 方法一：git filter-repo（推荐，需先 pip install git-filter-repo）
git filter-repo --invert-paths --path node_modules

# 方法二：git filter-branch（Git 自带，较慢）
git filter-branch --force --index-filter \
  'git rm -r --cached --ignore-unmatch node_modules' \
  --prune-empty --tag-name-filter cat -- --all

# 清除后强推（历史被重写，必须 force）
git push origin --force --all
git push origin --force --tags

# 清理本地无用对象，释放空间
git reflog expire --expire=now --all
git gc --prune=now --aggressive
```

> 这属于破坏性操作，参考 rebase 黄金法则：重写历史后必须强推，且要通知所有协作者。

#### 问题 2：其他机器/协作者 pull 时会发生什么

push 这个"删除 node_modules"的提交后：

- 远程最新版本已没有 node_modules
- 协作者 `git pull` 时，Git 会尝试**删除他们本地的 node_modules**

**给协作者的建议**：pull 前先备份，或 pull 后重新 `npm install` 生成 node_modules（它本是依赖产物，随时能重建，本来就不该进 Git）。

### 推荐的 .gitignore 模板

```gitignore
# 依赖
node_modules/

# 构建产物
dist/
build/

# 环境变量
.env
.env.local

# 系统文件
.DS_Store
Thumbs.db

# 编辑器
.vscode/
.idea/
```

---

## 十一、速查表与关键概念

### 命令速查表（按使用频率排序）

| 顺序 | 命令 | 用途 |
|------|------|------|
| 日常 | `git status` | 随时查看状态 |
| 日常 | `git pull` | 开始工作前拉取 |
| 核心 | `git add` + `git commit` | 暂存 + 提交 |
| 核心 | `git push` | 推送到远程 |
| 分支 | `git switch -c` / `git merge` | 建分支 / 合并 |
| 撤销 | `git restore` / `git reset` | 丢弃改动 / 撤销提交 |
| 查看 | `git log` / `git diff` | 看历史 / 看改动 |
| 比较 | `git diff <X> <Y> -- <文件>` | 比较两个版本某文件 |
| 查看提交 | `git show <commit>` | 看某次提交改了什么 |
| 变基 | `git rebase` / `git rebase -i` | 变基 / 交互式整理历史 |
| 取消追踪 | `git rm --cached <file>` | 停止追踪但保留本地 |

### 关键概念备忘

- **HEAD**：指向当前所在分支的最新提交，可以理解为"你现在站在哪"
- **`HEAD~1`**：HEAD 的上一个提交
- **origin**：远程仓库的默认别名（`git remote -v` 可查看）
- **追踪分支**：本地分支与远程分支的绑定关系，建立后 `git push`/`git pull` 不用带参数
- **工作流黄金法则**：永远不要在共享分支（main/master）上 `git push --force` 或 `reset --hard` 后推送
- **rebase 黄金法则**：别 rebase 已推送的公共分支（会重写历史）
- **`.gitignore` 门卫法则**：只对未追踪文件生效，已追踪文件必须先 `git rm --cached` 取消追踪
