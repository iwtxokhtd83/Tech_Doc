# Git 与 IDE 源代码管理讲义

> 本讲义覆盖 Git 核心原理、常用命令、协作工作流、排错技巧，以及在主流 IDE（VS Code、JetBrains、Visual Studio）中进行 Source Control 操作的实操指南。每章配"知识点 + 笔试题"。
>
> 约定：示例以 Git 2.40+ 为主；远端以 GitHub 为例（GitLab/Bitbucket 语法基本一致）。

## 目录

1. [Git 基础概念](#1-git-基础概念)
2. [首次配置与仓库初始化](#2-首次配置与仓库初始化)
3. [工作区、暂存区与提交](#3-工作区暂存区与提交)
4. [分支与合并](#4-分支与合并)
5. [Rebase 与变基策略](#5-rebase-与变基策略)
6. [远程仓库与协作](#6-远程仓库与协作)
7. [撤销与回滚](#7-撤销与回滚)
8. [Stash、Cherry-pick 与 Reflog](#8-stashcherry-pick-与-reflog)
9. [标签与发布](#9-标签与发布)
10. [Git 工作流模型](#10-git-工作流模型)
11. [代码审查与 Pull Request](#11-代码审查与-pull-request)
12. [钩子、子模块与高级特性](#12-钩子子模块与高级特性)
13. [常见冲突与排错](#13-常见冲突与排错)
14. [VS Code 中的源代码管理](#14-vs-code-中的源代码管理)
15. [JetBrains (IntelliJ/PyCharm 等) 中的 VCS](#15-jetbrains-intellijpycharm-等-中的-vcs)
16. [Visual Studio 的 Git 集成](#16-visual-studio-的-git-集成)
17. [GUI 工具：SourceTree / GitKraken / GitHub Desktop](#17-gui-工具sourcetree--gitkraken--github-desktop)
18. [最佳实践与 .gitignore](#18-最佳实践与-gitignore)
19. [综合笔试练习](#19-综合笔试练习)

---

## 1. Git 基础概念

### 1.1 什么是 Git

Git 是一个**分布式版本控制系统**（DVCS）。与 SVN/CVS 等集中式不同，每个工作副本都是一个**完整仓库**，包含完整历史。

### 1.2 核心对象模型

Git 底层只有四类对象，全部由 SHA-1（2.29+ 可用 SHA-256）做内容寻址：

| 对象 | 含义 |
|------|------|
| **blob** | 文件内容快照（不含文件名） |
| **tree** | 目录结构，指向 blob 或子 tree |
| **commit** | 一次提交：指向一个 tree、父 commit(s)、作者、信息 |
| **tag** | 带签名/说明的命名指针 |

一次 commit 的本质：指向某个 tree 的指针 + 指向父 commit 的指针 + 元数据。

### 1.3 三个区域

```
┌──────────────┐      add       ┌──────────────┐      commit     ┌──────────────┐
│   Working    │ ───────────▶  │    Index     │ ─────────────▶  │  Repository  │
│   Directory  │ ◀───────────  │  (Staging)   │ ◀─────────────  │  (.git)      │
└──────────────┘    checkout    └──────────────┘    reset         └──────────────┘
```

- **工作区（Working Directory）**：实际文件
- **暂存区（Index / Staging Area）**：`git add` 后的内容
- **本地仓库（Repository）**：`git commit` 后的对象

另加**远程仓库（Remote）**，是第四个抽象层。

### 1.4 引用（Refs）

- `HEAD`：当前位置指针
- `refs/heads/main`：本地分支
- `refs/remotes/origin/main`：远程分支跟踪
- `refs/tags/v1.0`：标签

分支本质是**指向某 commit 的可移动指针**，非常轻量。

### 📝 笔试题 1-1：Git 与 SVN 的本质区别？

- **分布式 vs 集中式**：Git 每个克隆都是完整仓库，SVN 依赖中心服务器
- **快照 vs 差异**：Git 每次提交是文件快照，SVN 存差异
- **离线能力**：Git 可离线 commit/diff/log，SVN 多数操作需联网
- **分支成本**：Git 分支极轻量，SVN 分支是目录拷贝

### 📝 笔试题 1-2：一次 `git commit` 到底保存了什么？

不是保存差异，而是保存：

1. 所有被追踪文件的**完整快照**（相同内容的 blob 复用，不重复存储）
2. 目录结构（tree 对象）
3. 父 commit 引用 + 作者 + 时间戳 + message

相同内容哈希一致，所以 Git 存储其实是"**按内容寻址的压缩对象库**"。

---

## 2. 首次配置与仓库初始化

### 2.1 全局配置

```bash
git config --global user.name "Alice"
git config --global user.email "alice@example.com"
git config --global core.editor "code --wait"      # 或 vim / nano
git config --global init.defaultBranch main         # 默认分支名
git config --global pull.rebase false               # 或 true / 'merges'
git config --global core.autocrlf input             # Linux/macOS
git config --global core.autocrlf true              # Windows
git config --global push.autoSetupRemote true       # 首次推送自动关联上游
```

查看配置：

```bash
git config --list --show-origin
git config user.email                    # 当前仓库的 email
```

配置文件优先级（高到低）：**仓库 `.git/config` → 用户 `~/.gitconfig` → 系统 `/etc/gitconfig`**。

### 2.2 初始化仓库

```bash
# 新建
git init                        # 当前目录变成仓库
git init --initial-branch=main

# 克隆
git clone https://github.com/user/repo.git
git clone git@github.com:user/repo.git        # SSH
git clone --depth 1 url                       # 浅克隆，只取最新一次
git clone --branch dev --single-branch url
```

### 2.3 别名（Alias）

常用配置，显著提速：

```bash
git config --global alias.co checkout
git config --global alias.br branch
git config --global alias.ci commit
git config --global alias.st status
git config --global alias.last 'log -1 HEAD'
git config --global alias.lg "log --oneline --graph --decorate --all"
git config --global alias.unstage 'reset HEAD --'
```

### 📝 笔试题 2-1：`~/.gitconfig` 里设置了 email A，某仓库里设置了 email B，`git commit` 用哪个？

用 **B**。仓库级优先级高于全局级。可通过 `git config user.email` 验证当前生效值。

---

## 3. 工作区、暂存区与提交

### 3.1 查看状态

```bash
git status
git status -s          # 简短格式 (?? 未追踪 / M 修改 / A 新增 / D 删除)
git diff               # 工作区 vs 暂存区
git diff --staged      # 暂存区 vs 上次提交（等同 --cached）
git diff HEAD          # 工作区 vs 上次提交
git diff branch1 branch2 -- path/file
```

### 3.2 添加与提交

```bash
git add file.txt
git add dir/
git add .                          # 当前目录下所有变化
git add -u                         # 仅更新已追踪文件（不含新文件）
git add -A                         # 所有（新+改+删），等价 git add --all
git add -p                         # 交互式选择 hunk（强推）

git commit -m "feat: add login"
git commit                         # 打开编辑器写详细信息
git commit -a -m "..."             # 跳过 add，自动 stage 已追踪文件的修改
git commit --amend                 # 修正上一次提交（改 message 或追加内容）
git commit --amend --no-edit       # 只追加内容，不改 message
git commit -m "..." --no-verify    # 跳过 pre-commit hook
```

### 3.3 提交信息规范（Conventional Commits）

```
<type>(<scope>): <subject>

<body>

<footer>
```

常见 type：`feat` / `fix` / `docs` / `style` / `refactor` / `perf` / `test` / `chore` / `build` / `ci` / `revert`。

示例：

```
feat(auth): support SSO login

- Add OIDC client
- Redirect to IdP on /login

BREAKING CHANGE: drops /basic-login endpoint
Closes #123
```

### 3.4 查看历史

```bash
git log
git log --oneline
git log --oneline --graph --decorate --all
git log -p                          # 带 diff
git log --stat                      # 文件变更统计
git log --author="Alice"
git log --since="2 weeks ago" --until="yesterday"
git log --grep="fix"
git log -- path/file                # 某文件历史
git log -S "functionName"           # 搜代码内容变动（pickaxe）
git log --follow -- file            # 跨越重命名
git show <commit>                   # 某次提交详情
git blame file                      # 每行最后修改者
```

### 3.5 移动 / 删除 / 忽略

```bash
git mv old new                      # 等价 mv + git add
git rm file                         # 从工作区和暂存区同时删
git rm --cached file                # 只从暂存区删（保留文件，标记为不追踪）
```

### 📝 笔试题 3-1：`git add .` 和 `git add -A` 的区别？

- `git add .`：添加**当前目录及子目录**的所有变化（含新增、修改、删除）
- `git add -A` / `--all`：添加**整个仓库**的所有变化

在仓库根目录下两者等价；在子目录下 `.` 只影响当前子树，`-A` 仍影响全仓库。

### 📝 笔试题 3-2：误把秘钥文件 commit 了，只是还没 push，怎么补救？

```bash
# 方法 1：只是最近一次 commit
git rm --cached secrets.env          # 从暂存区删
echo 'secrets.env' >> .gitignore
git add .gitignore
git commit --amend --no-edit

# 方法 2：commit 已多层深，用 filter-repo（重写历史）
git filter-repo --path secrets.env --invert-paths
```

**重要**：即使未 push 也应立即**轮换密钥**；若已 push 到远端，务必全员重 clone + 轮换凭证。

---

## 4. 分支与合并

### 4.1 分支操作

```bash
git branch                          # 列本地分支（* 当前）
git branch -a                       # 含远程
git branch -r                       # 仅远程
git branch -vv                      # 带跟踪关系
git branch feature                  # 创建但不切
git switch feature                  # 切换（推荐，语义清晰）
git switch -c feature               # 创建并切换
git checkout feature                # 传统写法
git checkout -b feature origin/dev  # 基于远程分支创建

git branch -d feature               # 删除已合并分支
git branch -D feature               # 强删（未合并也删）
git branch -m old new               # 重命名
```

`git switch` / `git restore` 是 2.23 拆分自 `git checkout` 的更清晰命令——推荐新项目使用。

### 4.2 合并（Merge）

```bash
git switch main
git merge feature                   # 产生合并 commit（若非 fast-forward）
git merge --no-ff feature           # 强制产生合并 commit，保留分支信息
git merge --ff-only feature         # 只允许 fast-forward，否则报错
git merge --squash feature          # 把 feature 所有变更压成一次提交
git merge --abort                   # 中止当前合并（有冲突时）
```

### 4.3 Fast-forward vs Three-way merge

- **Fast-forward**：主分支是 feature 的直系祖先，指针直接前移，无新 commit
- **Three-way merge**：两条分支分叉，Git 基于共同祖先生成合并 commit

示意：

```
Fast-forward:
A --- B --- C (main, feature)           A --- B --- C --- D --- E (main, feature)
              \
               D --- E

Three-way:
A --- B --- C --- F (main)              合并后生成 M：
              \                          A --- B --- C --- F --- M (main)
               D --- E                              \         /
                                                     D ------E
```

### 4.4 冲突解决

冲突标记：

```
<<<<<<< HEAD
current content
=======
incoming content
>>>>>>> feature
```

解决步骤：

```bash
# 编辑文件，解决所有冲突
git add <resolved-files>
git commit                # 或 git merge --continue
# 若放弃：
git merge --abort
```

### 📝 笔试题 4-1：`git switch` 和 `git checkout` 的关系？

`git switch`（分支切换）和 `git restore`（文件恢复）是 **Git 2.23** 从 `git checkout` 拆出来的，语义更清晰。`checkout` 仍可用，兼容老脚本；新项目推荐用 `switch`/`restore`。

### 📝 笔试题 4-2：`--no-ff` 合并有什么好处？

即使可 fast-forward 也生成合并 commit，**完整保留功能分支的存在痕迹**。回溯时能清楚看到"哪几个 commit 属于同一 feature"，`git log --graph` 更可读。代价是历史非线性。

---

## 5. Rebase 与变基策略

### 5.1 rebase 基本用法

```bash
git switch feature
git rebase main                     # 把 feature 的 commit "搬"到 main 最新之上

# 示意
# Before:
#   A --- B --- C (main)
#         \
#          D --- E (feature)
# After  git rebase main:
#   A --- B --- C --- D' --- E' (feature)
```

优点：**线性历史**，回溯清晰。
代价：**commit hash 被改写**（D→D', E→E'），已公开的分支切忌 rebase。

### 5.2 交互式 rebase（整理历史）

```bash
git rebase -i HEAD~5                # 修改最近 5 个 commit
```

编辑器中操作选项：

| 关键字 | 作用 |
|--------|------|
| `pick` | 保留 |
| `reword` | 保留但改 message |
| `edit` | 暂停以便修改内容 |
| `squash` | 合并到上一个 commit，合并 message |
| `fixup` | 合并到上一个，丢弃本次 message |
| `drop` | 删除 |
| `exec` | 执行 shell 命令 |

### 5.3 解决 rebase 冲突

```bash
# 出现冲突 → 手动解决 → git add → 继续
git add <files>
git rebase --continue
git rebase --skip                   # 跳过本次 commit（慎用）
git rebase --abort                  # 回到 rebase 前
```

### 5.4 merge vs rebase 的选择

| 场景 | 推荐 |
|------|------|
| 公共/已推送分支 | **merge**（不能改写历史） |
| 个人分支整理再合入 | **rebase**（线性整洁） |
| 团队/开源项目，强调历史保真 | merge + `--no-ff` |
| 主干保持线性 | rebase + fast-forward merge |

**Golden Rule of Rebase**：**永远不要对已推送给别人的分支做 rebase**。

### 5.5 `pull` 的两种模式

```bash
git pull                            # 等价于 fetch + 合并
git pull --rebase                   # 等价于 fetch + rebase（线性）
git config --global pull.rebase true
```

推荐 `pull --rebase`（尤其个人日常），避免在主线上产生无意义的合并 commit。

### 📝 笔试题 5-1：`git rebase main` 成功后，远端已有同名分支，`git push` 被拒怎么办？

因为 rebase 改写了历史。需要**强制推送**：

```bash
git push --force-with-lease          # 推荐，比 --force 安全
git push --force                     # 不推荐
```

`--force-with-lease` 会先检查远端是否已被他人更新（避免覆盖别人的提交）。

---

## 6. 远程仓库与协作

### 6.1 远端操作

```bash
git remote -v
git remote add origin git@github.com:u/r.git
git remote rename origin upstream
git remote remove old
git remote set-url origin <new-url>

git fetch                            # 拉取但不合并
git fetch --all --prune              # 清理已删除的远端分支
git pull                             # fetch + merge/rebase

git push                             # 推到跟踪的上游
git push -u origin feature           # 首次关联上游
git push origin :old-branch          # 删除远端分支（等价 --delete）
git push --tags                      # 推送所有本地标签
git push --dry-run                   # 演练，不真推
```

### 6.2 跟踪关系

```bash
git branch -vv                       # 看每个本地分支的上游
git branch --set-upstream-to=origin/main main
git push -u origin feature           # 一次搞定
```

### 6.3 协作工作流骨架

```bash
# 每天开工
git switch main
git pull --rebase

# 开分支干活
git switch -c feature/login
# 多次 commit ...
git push -u origin feature/login

# 其间同步主干
git fetch origin
git rebase origin/main               # 或 git merge origin/main

# 提 PR / MR，由审核方合入

# 合入后清理
git switch main
git pull --rebase
git branch -d feature/login
git push origin --delete feature/login
```

### 📝 笔试题 6-1：`git fetch` 与 `git pull` 的区别？

- `fetch`：只从远端**拉取**到本地 `origin/xxx`，**不修改**当前分支
- `pull`：`fetch` + **自动合并/变基**到当前分支

推荐习惯：先 `fetch` 看 `git log HEAD..origin/main` 有哪些新提交，再决定 merge/rebase。

---

## 7. 撤销与回滚

### 7.1 `reset` 三种模式

```bash
git reset --soft  HEAD~1           # 撤销 commit，保留暂存和工作区
git reset --mixed HEAD~1           # 默认，撤销 commit 和暂存，保留工作区
git reset --hard  HEAD~1           # 撤销 commit、暂存、工作区（⚠️不可逆）
```

对比：

| 模式 | HEAD | Index | Working |
|------|------|-------|---------|
| `--soft` | 移动 | 保持 | 保持 |
| `--mixed` | 移动 | 重置 | 保持 |
| `--hard` | 移动 | 重置 | 重置 |

### 7.2 revert：反向 commit

```bash
git revert <commit>                # 生成一个"反操作" commit，不改写历史
git revert <commit1> <commit2>     # 连续 revert
git revert -n <commit>             # 不自动 commit，继续累加
```

**已推送的公共分支用 `revert`，不要用 `reset`**。

### 7.3 restore：文件级恢复

```bash
git restore file                    # 从暂存区覆盖工作区
git restore --staged file           # 从 HEAD 覆盖暂存区（取消 add）
git restore --source=HEAD~2 file    # 从指定 commit 恢复文件
```

### 7.4 clean：清理未追踪

```bash
git clean -n                        # 预览
git clean -f                        # 删除未追踪文件
git clean -fd                       # 含目录
git clean -fdx                      # 连 .gitignore 中的也删（危险）
```

### 📝 笔试题 7-1：`git reset --hard HEAD~1` 后想找回怎么办？

```bash
git reflog                          # 找到被重置前的 commit hash
git reset --hard <hash>             # 回到那里
```

只要工作在 reflog 过期（默认 90 天）之内，仍可恢复；未提交的工作区内容丢了就是丢了。

### 📝 笔试题 7-2：`reset` 与 `revert` 的使用场景？

- **reset**：**本地、私人分支**，丢弃最近若干 commit
- **revert**：**公共、已推送分支**，以新 commit 抵消旧 commit；不改写历史

---

## 8. Stash、Cherry-pick 与 Reflog

### 8.1 stash：临时储藏

```bash
git stash                           # 存当前工作区和暂存
git stash -u                        # 含未追踪文件
git stash push -m "wip login"       # 带说明
git stash list                      # 查看
git stash show -p stash@{0}         # 看内容
git stash pop                       # 弹出并应用（删除）
git stash apply                     # 仅应用（不删）
git stash drop stash@{1}
git stash clear
```

典型场景：改到一半需要切换分支修 bug，先 `stash`，处理完回来 `pop`。

### 8.2 cherry-pick：挑取 commit

```bash
git cherry-pick <commit>
git cherry-pick <c1> <c2> <c3>
git cherry-pick A..B                # 区间（不含 A，含 B）
git cherry-pick -x <commit>         # message 中记录来源
git cherry-pick --continue          # 冲突解决后继续
git cherry-pick --abort
```

典型场景：把某修复从主干拣到发布分支。

### 8.3 reflog：引用历史

`reflog` 记录**本地**所有 HEAD 移动（commit / reset / rebase / checkout 等）：

```bash
git reflog
git reflog show feature
git log -g                          # 类似，带 diff 信息
```

**这是 Git 的后悔药**：即便 reset/rebase 似乎丢了 commit，只要还在 reflog 中，用 hash 即可找回。

### 📝 笔试题 8-1：`cherry-pick` 与 `rebase` 的区别？

- **cherry-pick**：把**指定的若干 commit** 挑到当前分支顶部
- **rebase**：把当前分支**全部 commit** 搬到另一个基础之上

cherry-pick 是"精准点菜"，rebase 是"整锅搬家"。两者都会生成新的 commit hash。

---

## 9. 标签与发布

### 9.1 轻量标签 vs 附注标签

```bash
git tag v1.0.0                      # 轻量：仅指针
git tag -a v1.0.0 -m "Release 1.0"  # 附注：含元数据，推荐
git tag -s v1.0.0 -m "..."          # GPG 签名标签

git tag                             # 列出
git tag -l "v1.*"
git show v1.0.0
```

### 9.2 推送和删除标签

```bash
git push origin v1.0.0
git push origin --tags              # 推送全部标签

# 删除
git tag -d v1.0.0                   # 本地
git push origin :refs/tags/v1.0.0   # 远端
git push origin --delete v1.0.0     # 同上，新写法
```

### 9.3 语义化版本

```
MAJOR.MINOR.PATCH[-PRERELEASE][+BUILD]
```

- `MAJOR`：破坏性变更
- `MINOR`：向后兼容的新功能
- `PATCH`：向后兼容的 bug 修复
- 示例：`1.2.3`、`2.0.0-rc.1`、`1.4.0+20250101`

### 📝 笔试题 9-1：轻量标签与附注标签的区别？

- **轻量**：只是一个指向 commit 的引用，不可携带信息
- **附注**：Git 对象之一，含 tagger、日期、message，可签名；对**发布版本**强烈推荐

---

## 10. Git 工作流模型

### 10.1 Git Flow（传统）

分支：

- `main`：生产可发布
- `develop`：集成分支
- `feature/*`：功能分支（从 develop 出，合回 develop）
- `release/*`：预发布（从 develop 出，合到 main + develop）
- `hotfix/*`：紧急修复（从 main 出，合到 main + develop）

**适合**：有明确版本发布节奏的产品，如 APP、桌面软件。
**缺点**：流程重，微服务/Web 持续交付不必要。

### 10.2 GitHub Flow（轻量）

- `main` 永远可部署
- 功能分支 → PR → 评审 → 合并 → 部署

**适合**：SaaS、Web 服务、持续部署团队。

### 10.3 GitLab Flow

介于两者之间，引入环境分支（`pre-production`、`production`），适合多环境发布。

### 10.4 Trunk-Based Development

- 所有人在 `main` 上小步提交，用 feature flag 隐藏未完成功能
- 分支极短（一天以内）

**适合**：CI/CD 成熟的高频发布团队（如 Google、Facebook）。

### 📝 笔试题 10-1：中小团队、Web 服务，推荐什么工作流？

**GitHub Flow**：主分支永远可部署，功能分支 + PR + 评审 + 合并。配合 CI/CD 和必需评审分支保护，足以满足大多数持续交付场景。Git Flow 反而过重。

---

## 11. 代码审查与 Pull Request

### 11.1 PR 的生命周期（GitHub 为例）

1. **开分支** `git switch -c feature/xxx`
2. **提交 + 推送**
3. **创建 PR**：写清 Summary / Motivation / Testing
4. **自动检查**：CI、lint、测试、覆盖率
5. **代码审查**：评论、建议、讨论
6. **修改并再提交**：可 `git commit --fixup` + `git rebase -i --autosquash`
7. **合并**：
   - Create merge commit（保留历史）
   - Squash & merge（压平）
   - Rebase & merge（线性）
8. **删除分支**

### 11.2 编写好的 PR

- 小：**一个 PR 只做一件事**（< 400 行 diff 最好）
- 有意义的标题（Conventional Commits 风格）
- 描述：背景、实现、测试、截图（UI）、相关 issue
- 自我审查一遍再请别人看
- CI 绿了再标 ready for review

### 11.3 `gh` CLI 基本用法

```bash
gh auth login
gh repo create my-repo --public --source=. --push
gh pr create --base main --title "feat: xxx" --body "..."
gh pr list
gh pr view 123
gh pr checkout 123
gh pr merge 123 --squash --delete-branch
gh issue list
```

### 11.4 分支保护

在 GitHub/GitLab 中常用规则：

- 禁止直接 push main
- PR 必须经 N 人评审
- 必须通过 CI
- 必须 signed commits / up-to-date

### 📝 笔试题 11-1：Merge / Squash / Rebase 合并 PR，优缺点？

| 策略 | 优点 | 缺点 |
|------|------|------|
| **Merge commit** | 保留全部历史，可追溯 | 历史非线性，commit 多 |
| **Squash** | 主干干净，一 feature 一 commit | 丢失细粒度历史 |
| **Rebase & merge** | 线性历史且保留每个 commit | commit hash 改变，易误导引用 |

团队偏好是主要决策因素；互联网公司多用 **Squash**。

---

## 12. 钩子、子模块与高级特性

### 12.1 Git Hooks

存放在 `.git/hooks/`。常用：

- `pre-commit`：提交前（lint、格式化）
- `commit-msg`：校验 message 规范
- `pre-push`：推送前（跑测试）
- `post-merge`、`post-checkout` 等

推荐用 **husky / pre-commit / lefthook** 管理，方便跨成员同步：

```json
// package.json
"husky": {
  "hooks": {
    "pre-commit": "lint-staged",
    "commit-msg": "commitlint -E HUSKY_GIT_PARAMS"
  }
}
```

### 12.2 Submodule（子模块）

把另一个 Git 仓库作为子目录引入：

```bash
git submodule add git@github.com:org/lib.git vendor/lib
git submodule update --init --recursive       # 首次克隆后初始化
git submodule update --remote                 # 拉取子模块最新
```

**缺点**：心智成本高，切分支、合并冲突、CI 缓存都更复杂。**优先考虑 monorepo 或包管理器**。

### 12.3 Worktree（工作树）

同一仓库同时检出多个分支到不同目录：

```bash
git worktree add ../repo-hotfix hotfix/urgent
git worktree list
git worktree remove ../repo-hotfix
```

适合不想反复 stash/switch 的场景。

### 12.4 Bisect：二分定位 bug

```bash
git bisect start
git bisect bad                     # 当前是坏的
git bisect good v1.0               # v1.0 是好的
# Git 自动切到中间点；你验证后：
git bisect good / bad
# 反复直到定位
git bisect reset
```

支持自动化：`git bisect run ./test.sh`。

### 12.5 Sparse-checkout（稀疏检出）

仅检出大仓库的一部分：

```bash
git sparse-checkout init --cone
git sparse-checkout set src docs
```

### 12.6 LFS（大文件）

大文件（图片、视频、二进制）不适合 Git：

```bash
git lfs install
git lfs track "*.psd"
git add .gitattributes
```

实际上传到 LFS 存储服务，Git 里只留指针。

### 📝 笔试题 12-1：`pre-commit` 钩子如何被团队共享？

默认 `.git/hooks/` 不入库。通过 **husky**（Node）或 **pre-commit**（Python）等工具，把 hook 脚本放入仓库并在安装时自动链接到 `.git/hooks/`。新人 `npm install` 或 `pre-commit install` 后自动启用。

---

## 13. 常见冲突与排错

### 13.1 merge 冲突

典型场景：合并前未先同步主干。

**预防**：

- 勤 `pull --rebase`，让分支基于最新主干
- 功能分支保持小而短寿

**解决**：编辑器手动解决 → `git add` → `git commit`（merge 场景）或 `git rebase --continue`（rebase 场景）。

### 13.2 `error: failed to push some refs`

远端有新提交，本地落后。

```bash
git pull --rebase
# 解决冲突...
git push
```

### 13.3 `detached HEAD`

`git checkout <commit>` 会进入游离状态，此时 commit 不属于任何分支，`switch` 走就**丢失**。

```bash
git switch -c new-branch            # 立即挂到新分支
```

### 13.4 换行符（CRLF / LF）

```bash
# Windows 上开发 Unix 项目
git config --global core.autocrlf input

# 项目内强制
# .gitattributes
* text=auto eol=lf
*.bat text eol=crlf
```

### 13.5 大小写敏感

macOS/Windows 默认**不区分**大小写，Linux CI 区分，容易出现"本地正常、CI 挂"。

```bash
git config core.ignorecase false
git mv README.md readme.md          # 真正重命名
```

### 13.6 常见错误速查

| 错误 | 原因 | 解决 |
|------|------|------|
| `non-fast-forward` | 远端比本地新 | `pull --rebase` 后 push |
| `Refusing to merge unrelated histories` | 两库无共祖 | `pull --allow-unrelated-histories` |
| `fatal: not a git repository` | 不在仓库内 | `cd` 进仓库或 `git init` |
| `Your local changes would be overwritten` | 切分支但有未提交变更 | `stash` 或 `commit` 后再切 |
| `cannot lock ref` | 并发/锁文件残留 | 删除 `.git/refs/.../*.lock` |
| `Author identity unknown` | 未配置 user | `git config user.email/name` |

### 📝 笔试题 13-1：合并时出现"无相关历史"错误怎么办？

两个仓库没有共同祖先（例如你 `git init` 了一个项目，又要合并别人独立仓库）：

```bash
git pull origin main --allow-unrelated-histories
```

随后手动处理可能的冲突。

---

## 14. VS Code 中的源代码管理

VS Code 内建 Git 支持（基于命令行 Git），并有丰富扩展。

### 14.1 侧边栏 Source Control（Ctrl/Cmd+Shift+G）

- **Changes**：工作区修改列表
- **Staged Changes**：已暂存
- 点击文件：查看 diff（左旧右新）
- 行号左侧的**✓**：stage 单行；**↩**：discard 单行
- 文件旁 **+**：stage 整个文件；**-**：unstage；**↩**：discard

### 14.2 提交、推送、拉取

- 顶部输入框写 commit message → `Ctrl/Cmd+Enter` 提交
- 点击 `...` 菜单：`Stage All` / `Commit All` / `Push` / `Pull` / `Sync`
- 状态栏底部：**分支名 + 同步箭头**（显示 `↓2 ↑1` 表示落后 2 领先 1），点击即 pull/push

### 14.3 分支操作

- 底部状态栏点击分支名 → 快速切换 / 创建新分支 / checkout remote
- `Ctrl+Shift+P` → `Git: ...` 可调用所有 Git 功能

### 14.4 解决冲突

冲突文件在编辑器中以**内联 diff** 方式显示，带按钮：

- **Accept Current Change**
- **Accept Incoming Change**
- **Accept Both Changes**
- **Compare Changes**（三方对比）

解决后 `git add`（侧栏 + 号），再 commit。

### 14.5 历史与比较

- **Timeline**（底部或右侧）：文件级历史，可点击某版本看 diff
- 需要更强可装扩展：
  - **GitLens**：行内 blame、文件历史、分支对比、交互式 rebase 最佳
  - **Git Graph**：可视化分支图，支持 cherry-pick / reset / rebase
  - **Git History**：简洁 log 视图

### 14.6 常用快捷键（默认）

| 操作 | 快捷键 |
|------|--------|
| 打开 Source Control | `Ctrl/Cmd+Shift+G` |
| 提交 | 在消息框按 `Ctrl/Cmd+Enter` |
| 命令面板 | `Ctrl/Cmd+Shift+P` → `Git: xxx` |
| 上/下一个变更 | `Alt+F3` / `Alt+F5`（需配合扩展） |

### 14.7 推荐扩展

- **GitLens** — Git Supercharged
- **Git Graph** — 直观分支图
- **GitHub Pull Requests and Issues** — 官方 GitHub 集成
- **GitLab Workflow** — GitLab 官方
- **Conventional Commits** — 辅助写规范 message

### 14.8 GitHub PR 工作流（在 VS Code 内）

装上 **GitHub Pull Requests** 扩展后：

1. 登录 GitHub
2. 侧栏可见 Pull Requests 面板
3. 创建 PR、填描述、指派审核
4. 在 PR 详情视图逐个文件 review、写评论、标 approve
5. 合并与关闭都无需离开编辑器

### 📝 笔试题 14-1：VS Code 中 stage 单行怎么做？

在 diff 视图或编辑器中，**把光标放到目标行/hunk 上**，使用命令面板 `Git: Stage Selected Ranges`，或点击左侧 gutter 上的 **+**。对应 CLI：`git add -p`。

---

## 15. JetBrains (IntelliJ/PyCharm 等) 中的 VCS

JetBrains IDE 家族（IntelliJ IDEA、PyCharm、WebStorm、GoLand、Rider、Android Studio 等）共用同一套 VCS/Git 集成，功能强大。

### 15.1 启用 Git 支持

- 打开项目，IDE 自动识别 `.git`
- 若未识别：`VCS` → `Enable Version Control Integration` → Git
- 设置路径：`Settings / Preferences` → `Version Control` → `Git`

### 15.2 主要视图

- **Git 工具窗口**（`Alt+9` / `Cmd+9`）
  - **Log**：分支图 + commit 列表 + diff 预览
  - **Changes**：当前工作区变更，按 changelist 分组
  - **Branches**：分支树，可切换/合并/变基/删除
- **Commit 工具窗口**（`Alt+0` / `Cmd+0`）：专门的提交面板
- **编辑器 Gutter**：行号左边颜色条标识新增/修改/删除（点击可查看 diff 并回滚单行）

### 15.3 提交流程

1. `Ctrl+K` / `Cmd+K` 打开 Commit 面板
2. 勾选要 stage 的文件 / hunk
3. 输入 message（支持模板、commitlint 插件、自动 issue 号）
4. 可选项：
   - **Reformat code**
   - **Rearrange code**
   - **Optimize imports**
   - **Run tests / Analyze code**
5. 点击 **Commit** 或 **Commit and Push**（`Ctrl+Alt+K` / `Cmd+Alt+K`）

IDE 提供 **Changelists**：把变更分组到不同"待提交集合"，适合并行处理多任务。

### 15.4 分支操作

右下角状态栏点击当前分支名，弹出强大的菜单：

- **New Branch** / **New Branch from Selected**
- **Checkout** / **Rename** / **Delete**
- **Compare with Current**
- **Merge into Current** / **Rebase Current onto Selected**
- **Cherry-pick** / **Push / Pull**
- 远端分支同界面可见

### 15.5 解决冲突

IDE 弹出**三方合并窗口**：

```
左侧：你的修改      中间：合并结果       右侧：对方修改
```

每个冲突块可一键 **Apply**，也可手动编辑中间面板。解决完点 **Apply**。体验远优于 CLI + 编辑器。

### 15.6 历史与对比

- **Log**：过滤作者、分支、分支图、`Cherry-pick`/`Reset`/`Revert`/`Create Patch`
- 文件右键 → **Git** → **Show History** / **Annotate**（逐行 blame，hover 看 commit）
- 比较两个 commit、两个分支、与 HEAD 的对比：`Show Diff` 或 `Compare with...`

### 15.7 Rebase / Interactive Rebase

- `Git` → `Rebase Current onto Selected`
- 交互式 rebase：Log 视图多选 commit → 右键 **Interactively Rebase from Here**
  - 可视化拖拽 reorder、`pick`/`squash`/`fixup`/`reword`/`drop`
- 冲突走三方合并窗口

### 15.8 Stash / Shelve

JetBrains 有两个概念：

- **Shelve**（IDE 概念，跨分支/工程都可用）
- **Stash**（Git 原生）

`VCS` → `Shelf` / `Stash Changes`。Shelve 存于 IDE 管理目录，更便于跨工程迁移；Stash 直接走 Git。

### 15.9 GitHub / GitLab / Bitbucket 集成

- `Settings` → `Version Control` → `GitHub/GitLab` 添加账号
- 直接在 IDE 创建 PR / MR、review、合并
- 内置 **Pull Requests** 工具窗口（新版 IDE）

### 15.10 实用技巧

- `Ctrl+K` 提交，`Ctrl+Shift+K` 推送
- `Annotate` 快速定位某行来源 commit
- **Local History**（不是 Git）：IDE 自动保存本地修改历史，即便未 commit 也能恢复
- `Compare with Branch` / `Compare with Revision`
- **Git Blame** Gutter 可开启：`Settings` → `Version Control` → `Git` → `Annotations`

### 📝 笔试题 15-1：JetBrains 中的 Local History 和 Git 的区别？

- **Local History**：IDE 层面，**每次文件保存**都记录一个快照，覆盖所有编辑，不需要 `git add`/`commit`
- **Git**：版本控制，需显式 commit；可协作、可推送

Local History 是"本地后悔药"，不是版本控制替代品；崩溃/误删后先试它往往能救回。

---

## 16. Visual Studio 的 Git 集成

Visual Studio 2019+ 已用全新 Git 体验替代老的 Team Explorer。

### 16.1 入口

- 顶部菜单 **Git**：Clone / New Repository / Manage Branches / Fetch / Pull / Push / Sync
- 状态栏：**当前分支**、**同步箭头**（↑ ↓）
- **Git Changes** 窗口（`View` → `Git Changes`）：暂存、提交、分支切换

### 16.2 提交

- `Git Changes` 窗口输入 message → `Commit Staged` 或 `Commit All`
- `Amend` / `Sign off` 通过 commit 按钮旁下拉选择
- 对单个 hunk stage：在 diff 中右键 → **Stage Chunk**

### 16.3 分支和历史

- **Git Repository** 窗口（`View` → `Git Repository`）：
  - 左：本地/远端分支列表
  - 中：提交图
  - 右：commit 详情 + diff
- 双击分支即 checkout；右键：rename / delete / merge / rebase / cherry-pick

### 16.4 合并冲突

VS 弹出 **Merge Editor**（三面板：Incoming / Current / Result），单击 Take 即可合并某侧变更，或手动编辑结果面板。

### 16.5 GitHub / Azure DevOps

原生集成 GitHub 账号和 Azure DevOps，支持 PR 创建、评审、合并。

### 📝 笔试题 16-1：VS 和 Visual Studio Code 的 Git 功能差在哪？

- **VS**：重量级 IDE，内建完整 Git UI + Merge Editor + Repository 视图，无需扩展
- **VS Code**：基础 Git 开箱，**强化需装扩展**（GitLens / Git Graph / GitHub PR），更轻量灵活

---

## 17. GUI 工具：SourceTree / GitKraken / GitHub Desktop

除 IDE 外，独立 GUI 工具适合不爱命令行的同学。

| 工具 | 平台 | 特点 | 许可 |
|------|------|------|------|
| **SourceTree** | Win / macOS | Atlassian 出品，免费，功能全 | 免费 |
| **GitKraken** | Win / macOS / Linux | 界面美观，分支图直观，内置 GitFlow 按钮 | 免费 / 付费 |
| **GitHub Desktop** | Win / macOS | 轻量、GitHub 最佳集成，适合新手 | 免费 |
| **Fork** | Win / macOS | 响应快，交互好 | 小额付费 |
| **Tower** | Win / macOS | 专业级，丰富快捷 | 付费 |
| **gitg / GitAhead / SmartGit** | 跨平台 | 其他选择 | 免费/付费 |

共性功能：

- 分支图可视化
- 暂存 / 提交 / 推送 / 拉取
- 合并 / 变基 / cherry-pick
- 冲突解决的三方视图
- 支持 Git Flow

**建议**：日常在 IDE 里完成 95% 操作；复杂历史整理时用专门 GUI（SourceTree / GitKraken）效率更高。

---

## 18. 最佳实践与 .gitignore

### 18.1 提交习惯

- 一次 commit 只做**一件事**（易于 review 与 revert）
- Message 用**Conventional Commits**，含动词原形（祈使语气）
- 避免"WIP"、"fix"、"update" 这种无信息量的 message
- commit 前 `git diff --staged` 自查

### 18.2 分支命名

- `feature/user-login`
- `fix/rounding-bug`
- `hotfix/crash-on-start`
- `chore/upgrade-deps`
- `release/1.4.0`
- 带 issue 号：`feature/PROJ-123-add-login`

### 18.3 .gitignore 范例

```gitignore
# OS
.DS_Store
Thumbs.db

# Editor
.idea/
.vscode/
*.swp
*.swo

# Build
node_modules/
dist/
build/
target/
*.pyc
__pycache__/
.venv/
.env
.env.local

# Logs
*.log
logs/

# Coverage
coverage/
*.lcov

# Secrets
*.pem
*.key
secrets/
```

**技巧**：

- 通用模板 https://github.com/github/gitignore
- 已追踪的文件加到 gitignore 不生效，需先 `git rm --cached <file>`
- `.gitignore` 本身应被提交
- `.gitignore_global` 可用于全局个人偏好

### 18.4 .gitattributes

用于指定文件属性，例如换行符、diff 驱动、合并策略、LFS 跟踪：

```
* text=auto eol=lf
*.bat text eol=crlf
*.png binary
*.psd filter=lfs diff=lfs merge=lfs -text
CHANGELOG.md merge=union
```

### 18.5 保护敏感数据

- 永远**不要**把 `.env`、密钥、证书、数据库密码提交
- 一旦误提交：立即**轮换凭证**；清理历史用 `git filter-repo`（新版替代 `filter-branch`）
- 用 **git-secrets / trufflehog / gitleaks** 做 pre-commit 扫描

### 18.6 性能建议

- 大仓库：启用 `feature.manyFiles`、`core.fsmonitor`（2.37+）
- 二进制：走 **Git LFS**
- 只需最新代码：`git clone --depth 1`
- 只需某子目录：`git sparse-checkout`

---

## 19. 综合笔试练习

### 19.1 选择题

**Q1** `git add` 的作用是？
A. 提交修改  B. 把修改放入暂存区  C. 推送到远端  D. 创建分支

<details><summary>答案</summary>B。</details>

**Q2** 下列命令中会**改写历史**的是？
A. `git commit`  B. `git merge`  C. `git rebase`  D. `git pull`

<details><summary>答案</summary>C。</details>

**Q3** 以下哪个命令用于撤销最近一次未推送的 commit，保留工作区改动？
A. `git reset --hard HEAD~1`
B. `git reset --soft HEAD~1`
C. `git revert HEAD`
D. `git checkout HEAD~1`

<details><summary>答案</summary>B。</details>

**Q4** `git pull` 等价于？
A. `git fetch + git merge`
B. `git clone + git push`
C. `git commit + git push`
D. `git stash + git pop`

<details><summary>答案</summary>A（或配置 `pull.rebase=true` 时为 `fetch + rebase`）。</details>

**Q5** 已推送分支的 commit 想撤销，最安全的做法？
A. `git reset --hard`  B. `git revert`  C. `git rebase -i`  D. 直接删仓库重建

<details><summary>答案</summary>B。</details>

**Q6** `git stash pop` 与 `git stash apply` 的区别？
A. 无区别
B. `pop` 应用后自动删除 stash，`apply` 保留
C. `apply` 能解决冲突，`pop` 不能
D. `pop` 只在本地分支有效

<details><summary>答案</summary>B。</details>

**Q7** 关于 `--force-with-lease`：
A. 与 `--force` 完全等价
B. 会在强推前检查远端是否被他人推过新提交
C. 仅用于 fetch
D. 会自动合并冲突

<details><summary>答案</summary>B。</details>

**Q8** `.gitignore` 对已经被追踪的文件：
A. 立即停止追踪
B. 无效，需先 `git rm --cached`
C. 自动删除文件
D. 仅对新文件生效，旧的需手动取消追踪

<details><summary>答案</summary>B/D 都对。官方答案：B（需显式 `git rm --cached`）。</details>

### 19.2 判断题

1. Git 每次提交保存的是文件差异。 ❌（快照，按内容寻址）
2. `git checkout` 仅用于切换分支。 ❌（还能检出文件、commit）
3. `rebase` 会生成新的 commit hash。 ✅
4. 公共分支可以安全地做 rebase。 ❌
5. `git reflog` 能恢复 `reset --hard` 丢失的 commit。 ✅（在过期前）
6. 分支本质是对 commit 的指针。 ✅
7. `git clone --depth 1` 得到的仓库不能正常 push。 ❌（可以，但历史不完整）
8. Git tag 推送要加 `--tags`。 ✅

### 19.3 简答题

**Q1** 描述一次 `git commit` 背后做的事情。

1. 对暂存区所有文件生成 blob 对象（相同内容复用）
2. 生成 tree 对象描述目录结构
3. 生成 commit 对象，包含 tree、父 commit、作者、时间、message
4. 更新当前分支引用指向新 commit
5. 更新 HEAD（指向分支）

**Q2** 合并（merge）和变基（rebase）各适合什么场景？

- **merge**：公共分支或多人协作，**不改写历史**；保留合并痕迹
- **rebase**：本地/私有分支清理历史；主干保持线性；**不要对已推送分支做**

**Q3** 误删了一个本地分支 `feature`，如何恢复？

```bash
git reflog                 # 找到 feature 最后一次 commit hash
git checkout -b feature <hash>
```

**Q4** 团队成员不小心把 `main` 指向了旧 commit 并强推覆盖了远端，如何恢复？

- **本地有最新的同事**在本地 `git reflog` / `git log origin/main@{1}` 找到被覆盖的 commit
- 用该 commit `git push --force-with-lease origin main` 还原
- 之后在 GitHub/GitLab 开启**分支保护**，禁止强推 `main`

### 19.4 操作题

**Q1** 写出把当前分支最近三个 commit 合并为一个的命令序列。

```bash
git rebase -i HEAD~3
# 在编辑器里把后两个 pick 改成 squash 或 fixup
# 保存 -> 解决冲突（如有）-> 完成
git push --force-with-lease          # 若已推送
```

**Q2** 团队主干工作流：新功能从 main 分出到合入的完整命令序列。

```bash
git switch main
git pull --rebase

git switch -c feature/checkout-flow
# 修改、提交多次
git commit -m "feat(checkout): add payment step"

# 同步主干
git fetch origin
git rebase origin/main

git push -u origin feature/checkout-flow
# 创建 PR（gh pr create 或网页）
# 评审通过合并后：
git switch main
git pull --rebase
git branch -d feature/checkout-flow
```

**Q3** 把 `release/1.3` 分支的某次修复 commit（hash 为 `abc1234`）拣到 `main`。

```bash
git switch main
git pull --rebase
git cherry-pick abc1234
# 若冲突：解决 -> git add -> git cherry-pick --continue
git push
```

**Q4** 已 `commit` 了一个含 1GB 视频文件的变更，尚未 push，如何正确处理？

```bash
# 1. 撤回最近 commit 但保留改动
git reset --soft HEAD~1
# 2. 从暂存区移除大文件
git restore --staged big-video.mp4
# 3. 把它纳入 LFS 或放到别处
git lfs install
git lfs track "*.mp4"
git add .gitattributes big-video.mp4
git commit -m "feat: add demo video via LFS"
# 4. 推送
git push
```

---

## 📚 学习建议

1. **CLI 是硬通货**：GUI 出问题时仍能救场；至少掌握 add/commit/branch/merge/rebase/reset/revert/log/stash/reflog
2. **日常在 IDE**：90% 操作走 IDE 可提效——GitLens、JetBrains VCS、VS Git Changes 都很强
3. **pro git 免费书**：<https://git-scm.com/book> 官方权威
4. **多用 `reflog`**：它是 Git 的后悔药，记住它就不怕 reset/rebase
5. **团队规范先行**：分支命名、commit 信息、保护规则，胜过任何花哨技巧
6. **练习 dangerous 操作**：在废仓库里练 `reset --hard`、`filter-repo`、`reflog` 恢复，真出事才不慌

> 祝你的 commit 常绿，history 常清。
