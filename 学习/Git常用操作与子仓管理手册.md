# Git 常用操作与子仓管理手册

刚开始用 Git 的时候，大家往往记住了几条命令，但是遇到推送失败、切换分支报错，或者项目里又套着一个仓库，就不知道下一步应该做什么了。

这份手册按实验室里常见的使用场景整理：下载项目、修改代码或文章、同步远端、协作、恢复操作，以及管理第三方驱动和 SDK。先把日常流程用顺，再根据需要查看后面的内容。

> 命令写法参考 Git 官方文档，查阅日期为 2026 年 10 月 22 日。本机核对版本为 Git 2.41.0；其他版本和项目配置可能存在差异。示例以 PowerShell 为主，Git 命令大多也适用于其他终端。代码块按场景使用，不需要把整篇命令从头到尾连续执行。
>
> `main`、`origin`、文件路径和分支名称都以实际项目为准。`OWNER`、`ROBOT`、`COMMIT_ID` 等是待替换内容；命令中的 `#` 开头行为说明。执行会改动文件或历史的命令前，先看当前目录与 `git status`。

## 目录

1. [先理解几个位置与名字](#concepts)
2. [第一次使用：配置、初始化与克隆](#start)
3. [查看状态、差异和历史](#inspect)
4. [暂存、提交与文件操作](#commit)
5. [fetch、pull、push 与远端](#remote)
6. [分支与 switch](#branches)
7. [写到一半：stash](#stash)
8. [合并、变基与冲突](#integrate)
9. [撤销、回退与恢复](#undo)
10. [忽略文件与换行](#ignore)
11. [子仓到底是什么](#nested)
12. [submodule 子模块的完整操作](#submodule)
13. [worktree、多仓与其他组织方式](#multi)
14. [标签与版本定位](#tags)
15. [实验室协作流程与常见问题](#workflow)
16. [命令速查与官方资料](#reference)

<a id="concepts"></a>

## 一、先理解几个位置与名字

```text
工作区：正在编辑的文件
   │ git add
   ▼
暂存区：准备进入下一次提交的内容
   │ git commit
   ▼
本地仓库：本机已有的提交历史与分支
   │ git push
   ▼
远端仓库：GitHub、GitLab 或实验室服务器上的仓库
```

- **工作区（working tree）：** 编辑器里正在修改的实际文件。
- **暂存区（index / staging area）：** 下一次提交准备采用的文件状态。暂存后继续编辑，新增修改还需要再次暂存。
- **提交（commit）：** 对一个版本的记录，具有提交 ID、作者、说明和父提交等信息。
- **分支（branch）：** 指向一条开发历史最新提交的名字，例如 `main`、`feature/uart`。
- **HEAD：** 当前检出位置，通常指向当前分支；也可能直接指向某个提交。
- **origin：** 克隆时常用的远端名称，只是一个名字，可以更改。
- **origin/main：** 本地记录的远端跟踪引用，通过 fetch 等操作更新，不是随时联网显示的远端实时状态。
- **upstream：** 当前分支关联的上游分支；它和有人命名为 `upstream` 的远端不是同一概念。

保存文件、暂存、提交和推送是不同动作。`commit` 可以只在本地完成，`push` 才把相应提交发布到远端。

你提到的 **`state` 不是标准 Git 的内置子命令**。如果想看状态，一般是 `git status`；想暂存未完成的工作，一般是 `git stash`。某些环境可能自行定义了 `state` 别名或扩展，不能当作通用命令使用。可用 `git help -a` 查看命令，用 `git config --get-regexp '^alias\.'` 查看已配置的别名。

<a id="start"></a>

## 二、第一次使用：配置、初始化与克隆

### 1. 配置提交身份

```powershell
git --version
git config --global user.name "你的名字"
git config --global user.email "student@example.com"
```

把示例邮箱换成自己的邮箱或代码托管平台提供的隐私邮箱。这些是提交身份信息，不是远端登录凭据。

`--global` 作用于当前用户的多个仓库。如果只想为当前仓库设置身份，进入仓库后省略 `--global`：

```powershell
git config user.name "项目内使用的名字"
git config user.email "student@example.com"
git config --show-origin --get user.name
git config --show-origin --get user.email
```

官方说明：[git config](https://git-scm.com/docs/git-config)。

### 2. 下载已有仓库

```powershell
git clone https://github.com/D807Embedded-Lab/Embedder-systems-study-guide.git
cd Embedder-systems-study-guide
git status
git remote -v
```

克隆会取得仓库历史并创建工作目录，通常自动配置名为 `origin` 的远端。如果项目含子模块，查看后面的 `--recurse-submodules` 用法。

### 3. 把已有项目交给 Git 管理

以下用于尚未初始化的项目目录，不要对已有远端项目再机械地重复创建一套历史：

```powershell
cd "D:\Projects\robot"
git init -b main
# 先建立适合工程的 .gitignore，再选文件暂存
git add -- README.md src
git diff --cached
git commit -m "chore: 初始化项目"
```

远端仓库需另行创建。若远端已有独立提交，先确认两边历史关系；初学时通常更方便先 clone 再复制自己的项目文件。

<a id="inspect"></a>

## 三、查看状态、差异和历史

### 1. status：先弄清楚自己在哪里

```powershell
git status
git status -sb
git branch --show-current
git rev-parse --show-toplevel
```

分别查看完整状态、简短状态与分支信息、当前分支名，以及当前仓库根目录。父仓与子模块来回操作时，根目录尤其值得检查。

普通未冲突文件的 `git status --short` 有两列状态：第一列通常表示暂存区相对 HEAD 的变化，第二列表示工作区相对暂存区的变化。

```text
 M README.md       工作区修改，还未暂存
M  README.md       已暂存修改
MM README.md       已暂存后又继续修改
A  new-file.md     已暂存新增文件
 D old-file.md     工作区删除，删除还未暂存
?? draft.md        未跟踪文件
UU config.txt      双方修改造成的未解决冲突之一
```

`ahead` / `behind` 依据本地已有的跟踪信息计算。需要知道远端新变化时先 fetch。[官方：git status](https://git-scm.com/docs/git-status)

### 2. diff：分别看哪些修改

```powershell
git diff
git diff --cached
git diff HEAD
git diff --stat
git diff --check
```

| 命令 | 查看或检查的范围 |
| --- | --- |
| `git diff` | 工作区相对暂存区的变化，即尚未暂存的修改 |
| `git diff --cached` | 暂存区相对 HEAD 的变化，即下一次准备提交的内容 |
| `git diff HEAD` | 已跟踪文件在工作区相对 HEAD 的总体变化 |
| `git diff --stat` | 默认比较范围的变化统计 |
| `git diff --check` | 默认差异中的空白错误等问题，不代替编译和功能验证 |

普通 diff 不会自动展示未跟踪文件的完整内容。暂存后检查提交格式可以用 `git diff --cached --check`。

限制到某个路径时，建议使用 `--` 区分参数与路径：

```powershell
git diff -- "学习/嵌入式学习.md"
git diff --cached -- "学习/嵌入式学习.md"
```

### 3. log、show、blame：看改动从哪里来

```powershell
git log --oneline -10
git log --oneline --graph --decorate --all
git log --follow -- "README.md"
git show COMMIT_ID
git blame -- README.md
```

`log` 看历史，`show` 看某次提交等对象，`blame` 帮助定位每行最近的相关提交。找到提交后还要读上下文，不能只凭一行的作者判断问题原因。

<a id="commit"></a>

## 四、暂存、提交与文件操作

### 1. add 与 commit

```powershell
git add -- "学习/新文章.md"
git diff --cached
git commit -m "docs: 增加串口调试记录"
```

`git add -p` 可以交互式选择部分改动；`git add -A` 会把其作用范围内的新增、修改和删除纳入暂存，执行后应检查是否混入构建产物或无关文件。

`git commit -am "说明"` 可以暂存并提交已跟踪文件的修改和删除，但不会把普通未跟踪文件自动加入。提交消息说明“改了什么、为什么改”，之后排障更容易找到对应版本。

### 2. 移动、改名与删除

```powershell
git mv -- "资源/具身智能学习路线.md" "学习/具身智能学习路线.md"
git diff --cached --stat
git commit -m "docs: 调整具身智能学习路线的位置"
```

目标父目录应已存在。`git mv` 会更新工作区及暂存区；也可以用编辑器移动后再暂存删除与新增，Git 会在比较时判断是否构成重命名。移动 Markdown 后记得检查相对图片与文章链接。

`git rm -- path/to/file` 会删除工作区文件并暂存删除。`git rm --cached -- path/to/file` 只从暂存区移除跟踪，保留本地文件，常用于停止跟踪本地配置；两者不要混用。

### 3. Git 为什么不能直接提交空文件夹

Git 通常记录文件与树，不单独保留空目录。可以创建一个 `.gitkeep` 占位文件再提交。`.gitkeep` 只是约定名称，没有特殊的忽略或保护功能。

<a id="remote"></a>

## 五、fetch、pull、push 与远端

### 1. 先查看远端

```powershell
git remote -v
git branch -vv
git remote show origin
```

`git remote add origin URL` 添加远端，`git remote set-url origin URL` 修改已有远端地址。把 `URL` 换成实际的 HTTPS 或 SSH 地址，不要把令牌或密码写进共享命令、截图和远端 URL。

### 2. fetch：先拿到远端信息

```powershell
git fetch origin
git log --oneline HEAD..origin/main
git log --oneline origin/main..HEAD
```

第一组 log 查看 origin/main 上有、当前 HEAD 没有的提交；第二组方向相反。fetch 通常更新远端跟踪引用，不自动把远端改动合并进当前工作区。

`git fetch --prune origin` 可清理对应已删除远端分支的过期跟踪引用，不是删除同名的普通本地分支。[官方：git fetch](https://git-scm.com/docs/git-fetch)

### 3. pull：获取并整合到当前分支

在本地 `main` 上同步远端 `main`：

```powershell
git switch main
git pull --ff-only origin main
```

`--ff-only` 只允许快进。如果本地与远端已经分叉，它会拒绝整合，留给你检查后选择 merge 或 rebase。它不是覆盖本地代码的参数。

**`git pull origin main` 不会把你切换到 main。** 它把取得的 main 历史整合进当前分支，所以要先确认自己站在哪个分支。

| 写法 | 整合方式 |
| --- | --- |
| `git pull --ff-only` | 只接受快进，分叉时停止 |
| `git pull --rebase` | 获取后尝试把当前分支的相关本地提交变基到上游历史之上 |
| `git pull --no-rebase` | 获取后通过合并整合；具体合并形态还受选项和配置影响 |

不带策略的 pull 行为受 Git 版本和配置影响。团队确定一种方式后按约定使用；执行前先提交或妥善保存未完成工作。[官方：git pull](https://git-scm.com/docs/git-pull)

### 4. push：推送提交，不是上传当前所有文件

```powershell
git push origin main
git push -u origin feature/git-guide
```

第一条把本地 main 推到 origin 的 main；不一定是在推当前检出的分支。第二条推送指定分支，并在成功时为其建立相应上游关系。之后通常可用 `git push` / `git pull` 简写，但仍受当前分支和配置影响。

只编辑、没有 commit 的文件不会因为 push 自动上传。推送也可能被权限、受保护分支、检查规则等拒绝。[官方：git push](https://git-scm.com/docs/git-push)

### 5. push 被拒绝时先判断原因

遇到 `non-fast-forward`，先取得远端信息并查看两边提交：

```powershell
git fetch origin
git log --oneline --graph --decorate --all
```

若是协作者新增了提交，按团队约定整合、解决冲突、验证后再推。认证失败要检查账号与凭据；保护规则拒绝则通过规定流程提交 PR。不要把所有推送失败都用强制推送处理。

<a id="branches"></a>

## 六、分支与 switch

### 1. 创建与切换

```powershell
git branch
git branch -a
git switch main
git switch -c feature/uart
git switch -
```

`git branch feature/uart` 只创建分支；`git switch -c feature/uart` 创建并切换，默认从当前 HEAD 开始；`git switch -` 回到上一次检出位置。

如果希望从更新后的 main 建立分支，可以先同步 main，再执行 `switch -c`。切分支会更新检出的文件；未提交修改有时可以保留带过去，有时因为会被覆盖而被拒绝，不要使用强制选项绕过检查。

### 2. 检出远端已有分支

```powershell
git fetch origin
git switch --track origin/feature/uart
```

这要求远端分支存在、对应本地分支尚未创建。已有本地分支时直接切换；需要调整跟踪关系可以用 `git branch --set-upstream-to=origin/feature/uart feature/uart`。

### 3. checkout 和 switch 的区别

`checkout` 同时承担切换分支和恢复文件等职责；较新的 Git 用 `switch` 表达切换，用 `restore` 表达文件恢复，阅读时更容易分清意图。旧项目教程里的 checkout 并没有因此失效。

`switch` 与 `restore` 从 Git 2.23 起提供。参考：[git switch](https://git-scm.com/docs/git-switch)。

### 4. 删除分支

`git branch -d feature/uart` 删除已满足合并检查条件的本地分支；先切到其他分支。`-D` 是跳过相应合并检查的强制删除，可能使未合并提交失去该分支引用。

`git push origin --delete feature/uart` 删除远端分支，不是删除自己的本地分支；确认团队已不再需要再操作。

<a id="stash"></a>

## 七、写到一半：stash

有临时任务需要切分支，但当前修改还没准备提交时，可以暂时保存：

```powershell
git status
git stash push -u -m "临时保存串口调试修改"
git stash list
git switch main
```

默认 stash 处理已跟踪文件的修改，`-u` 还纳入未跟踪文件；忽略文件默认不包括在内。子模块内部的未提交工作也不能只指望父仓的一次 stash 保存，需分别检查。

回来以后先切回原分支，再查看和恢复：

```powershell
git switch feature/uart
git stash show -p -u "stash@{0}"
git stash apply "stash@{0}"
git status
```

- `apply` 恢复但保留 stash 记录，方便核对。
- `pop` 尝试恢复，成功后删除该记录；发生冲突时通常保留记录，需要自己处理。
- `apply --index` 还尝试恢复暂存区状态，能否成功取决于当前状态和冲突。
- `drop` 删除指定记录，`clear` 清空记录，确认不再需要才使用。

PowerShell 中将 `"stash@{0}"` 这类引用加引号。stash 是本地临时保存，不会自动随 push 分享，也不能替代正式提交与备份。[官方：git stash](https://git-scm.com/docs/git-stash)

<a id="integrate"></a>

## 八、合并、变基与冲突

### 1. merge：把另一条历史合进当前分支

例如在 main 合入已完成的功能：

```powershell
git switch main
git merge feature/uart
```

可能快进，也可能创建合并提交。多人协作仓库通常通过 PR 审查和合并，不一定允许本地直接合入再推 main。[官方：git merge](https://git-scm.com/docs/git-merge)

### 2. rebase：把提交重新应用到另一基础上

例如在功能分支整合更新后的 main：

```powershell
git fetch origin
git switch feature/uart
git rebase origin/main
```

被重放的提交通常产生新的提交 ID。适合团队约定允许的场景，尤其是整理自己尚未发布的提交；已经共享给其他人的分支历史，改写前要沟通。[官方：git rebase](https://git-scm.com/docs/git-rebase)

### 3. 发生冲突以后做什么

先运行 `git status`，打开冲突文件，理解两边改动，编辑为最终正确内容。不要不看语义就统一选择一边。常见标记包括：

```text
<<<<<<< HEAD
一侧内容
=======
另一侧内容
>>>>>>> 分支或提交说明
```

修改后删除冲突标记，暂存已解决文件，再继续对应操作：

| 当前操作 | 解决并暂存后 | 准备退出本次操作 |
| --- | --- | --- |
| merge | `git merge --continue` | `git merge --abort` |
| rebase | `git rebase --continue` | `git rebase --abort` |
| cherry-pick | `git cherry-pick --continue` | `git cherry-pick --abort` |
| revert | `git revert --continue` | `git revert --abort` |

合并前保持工作区清晰，更利于恢复；带着既有未提交修改开始操作时，abort 不保证替你完整还原所有混在一起的工作。rebase 冲突里的 ours / theirs 也不能简单按“我写的 / 别人写的”理解，要读实际内容。

`git cherry-pick COMMIT_ID` 可把指定提交的改动应用到当前分支，常用于移植一个独立修复。它不是自动合并整条分支。

<a id="undo"></a>

## 九、撤销、回退与恢复

先确认要撤销的是工作区修改、暂存动作、本地提交，还是已经公开的提交。

### 1. 只取消暂存，保留编辑结果

```powershell
git restore --staged -- README.md
git status
```

通常让该路径的暂存区恢复为 HEAD 版本，工作区中的编辑仍保留。需要仓库已有合适的源提交；刚初始化、没有首次提交时，可按提示使用 `git rm --cached` 等对应方式。

### 2. 丢弃某文件尚未暂存的修改

**这一步会覆盖工作区里该文件的未暂存改动。** 确认不需要，或已另行保存后才执行：

```powershell
git restore -- README.md
```

默认从暂存区恢复，不是永远从最近提交恢复。若文件已经暂存一部分修改，恢复后得到的是那份暂存内容。

`git restore --source=HEAD --staged --worktree -- README.md` 会把该路径的暂存区与工作区都恢复到 HEAD，丢弃这两个位置相对 HEAD 的修改。[官方：git restore](https://git-scm.com/docs/git-restore)

### 3. 最近一次本地提交需要调整

修改后暂存，再用：

```powershell
git commit --amend
```

会改写最近一次提交，可调整内容与提交说明。只想替换说明可用 `git commit --amend -m "新的说明"`。若提交已经发布，先考虑团队的历史管理约定。

### 4. reset 的几种常见方式

以下假设当前分支至少存在可回退的上一提交，仅用于清楚知道目标的本地历史调整：

| 命令 | 当前分支位置 | 暂存区 | 工作区 |
| --- | --- | --- | --- |
| `git reset --soft HEAD~1` | 回到上一提交 | 保留 | 保留 |
| `git reset --mixed HEAD~1` | 回到上一提交 | 按目标提交重置 | 保留 |
| `git reset --hard HEAD~1` | 回到上一提交 | 按目标提交重置 | 按目标提交重置 |

`--hard` 会丢弃受影响的已跟踪修改，并可能删除妨碍检出的未跟踪路径，不能当作日常“刷新项目”的命令。reset 还有针对路径的形式，不都是移动分支。[官方：git reset](https://git-scm.com/docs/git-reset)

### 5. 已经共享的错误提交，通常考虑 revert

```powershell
git revert COMMIT_ID
```

生成一个反向修改的新提交，保留原有历史。可能冲突；合并提交的撤销还涉及主线父提交选择，不能直接套用普通提交示例。[官方：git revert](https://git-scm.com/docs/git-revert)

### 6. reflog：查本机之前的位置

```powershell
git reflog
git branch rescue/uart COMMIT_ID
```

先从 reflog 找到目标提交，再给它建立分支引用，便于检查和恢复。reflog 是本机引用变化记录，有保留与清理机制，不是远端备份，也不能保证找回从未被 Git 保存的编辑内容。[官方：git reflog](https://git-scm.com/docs/git-reflog)

### 7. 强制推送与清理文件

- **强制推送：** `git push --force` 可能覆盖远端历史。确实需要发布自己分支的历史改写时，先与协作者确认，再了解 `--force-with-lease` 的检查条件。它不是“不会出错的 force”。
- **显式 lease：** `git push --force-with-lease=refs/heads/feature/uart:EXPECTED_REMOTE_COMMIT origin feature/uart:feature/uart` 要求远端该分支仍位于你指定的完整提交 ID；目标不符就拒绝。先核实该 ID，替换占位内容。仅用不带预期值的 lease，会依赖本地跟踪引用，后台 fetch 可能影响其保护效果。[官方说明](https://git-scm.com/docs/git-push)
- **清理未跟踪文件：** 先用 `git clean -nd` 预览。`git clean -fd` 会实际删除相应未跟踪文件和目录；`-x` 还会纳入忽略文件，可能包含本地配置和构建资料。确认具体路径并做好保存后再清理，不要在父目录不明时直接执行。[官方：git clean](https://git-scm.com/docs/git-clean)

<a id="ignore"></a>

## 十、忽略文件与换行

### 1. .gitignore 怎么用

以下只是一个示例，按项目调整，不要把需要交接的源文件与配置误排除：

```gitignore
/build/
*.o
*.elf
/config.local
.env
```

```powershell
git check-ignore -v -- build/firmware.elf
git ls-files -- config.local
```

第一条检查忽略规则，第二条检查文件是否已被跟踪。`.gitignore` 不会自动让已跟踪文件退出历史管理；需要停止跟踪时使用 `git rm --cached` 并提交规则与移除记录。

忽略规则也不会清除旧提交里的敏感内容。密钥或令牌误公开时，要处理凭据本身，不能只删除当前版本。[官方：gitignore](https://git-scm.com/docs/gitignore)

### 2. Windows 换行提示怎么看

`LF will be replaced by CRLF` 通常是在说明换行转换策略，未必意味着提交失败。可检查：

```powershell
git config --show-origin --get core.autocrlf
git ls-files --eol
```

团队可在 `.gitattributes` 中约定文本和二进制规则，例如：

```gitattributes
* text=auto
*.md text eol=lf
*.sh text eol=lf
*.png -text
```

不要为消除一个提示就随意改变所有项目的全局配置。新增或改变规则后，可能出现较多格式变化，需要审查后再提交。

<a id="nested"></a>

## 十一、子仓到底是什么

嵌入式项目里经常会有应用、BSP、RTOS、驱动库、算法库等内容，但“文件夹里面有另一个仓库”存在几种不同情况。

| 情况 | 由谁记录内容 | 典型特点 |
| --- | --- | --- |
| 普通子目录 | 父仓直接跟踪其中的普通文件 | 一套历史，操作简单 |
| 独立仓库放在另一个仓库目录内 | 内外各自有仓库边界 | 不能假设父仓会自动把内部文件完整带走 |
| submodule 子模块 | 子仓记录代码，父仓记录路径处所采用的提交 | 可以固定第三方依赖版本，但要分别管理两层提交 |
| worktree 工作树 | 同一仓库共享对象库与大部分引用 | 多个目录检出不同分支，不是第三方子仓 |

在父仓直接 `git add` 一个嵌套仓库，可能得到 embedded repository 提示，或记录一个 gitlink 而缺少完整子模块配置。不要忽略提示就提交，否则别人 clone 后可能拿不到内部代码。

**子模块的重点是固定一个提交。** 父仓提交某一版本时，记录“这个路径使用子仓的哪次提交”，不是自动复制子仓所有文件，也不是永远跟着子仓 main 最新版本变化。

常见目录关系：

```text
robot/                         父仓
├─ .gitmodules                 子模块路径、URL 等配置
├─ app/
└─ external/
   └─ driver-lib/              子模块，有自己的提交历史
      └─ .git                  现代常见布局中可能是指向实际 Git 目录的文件
```

`.gitmodules` 应随项目提交，内容应能让协作者访问；子仓的实际权限仍独立管理。父仓能访问，不代表私有子仓也自动有权限。[官方：子模块模型](https://git-scm.com/docs/gitsubmodules)、[.gitmodules 配置](https://git-scm.com/docs/gitmodules)

<a id="submodule"></a>

## 十二、submodule 子模块的完整操作

以下用 `external/driver-lib` 举例，父仓叫 robot。实际项目的路径、分支和 URL 要替换，操作前分别确认父仓与子仓的状态。

### 1. 克隆含子模块的项目

```powershell
git clone --recurse-submodules https://github.com/OWNER/ROBOT.git
```

已经普通 clone 过，进入父仓补全：

```powershell
git submodule update --init --recursive
git submodule status --recursive
```

`--init` 初始化配置，`--recursive` 处理嵌套子模块。默认 update 检出父仓所记录的子模块提交，通常处于 detached HEAD，不等于自动升级依赖。[官方：git submodule](https://git-scm.com/docs/git-submodule)

### 2. 给项目加入一个子模块

在父仓根目录执行，目标路径需符合命令要求、不要覆盖已有工程：

```powershell
git submodule add https://github.com/OWNER/driver-lib.git external/driver-lib
git status
git diff --cached --submodule=log
git commit -m "build: 加入 driver-lib 子模块"
```

通常会暂存 `.gitmodules` 与子模块路径记录。检查 URL、使用版本和协作者访问条件，再按正常方式推送父仓。

### 3. 更新父仓后，同步到它指定的子仓版本

在父仓对应分支完成更新后：

```powershell
git pull --ff-only
git submodule update --init --recursive
```

第一条更新当前父仓分支，第二条让子模块工作目录对应父仓记录。不要为了让目录“看起来新一点”就给所有子模块随意 pull，这可能破坏已验证的版本组合。

### 4. 子仓里也改了代码，应该先提交谁

先进入子仓，从当前使用的提交创建自己的修复分支：

```powershell
git -C external/driver-lib status
git -C external/driver-lib switch -c fix/uart-timeout
# 编辑并验证子仓代码
git -C external/driver-lib add -- src/uart.c
git -C external/driver-lib commit -m "fix: 修复 UART 超时处理"
git -C external/driver-lib push -u origin fix/uart-timeout
```

这要求你对该子仓远端有推送权限。没有权限时，需走子仓自己的 fork / PR 流程，并确保最终使用的提交在协作者可访问的远端上。子仓若已有其他未完成工作，先处理清楚，再创建与切换分支。

回到父仓，记录子模块的新提交：

```powershell
git diff --submodule=log -- external/driver-lib
git add -- external/driver-lib
git commit -m "build: 更新 driver-lib 的 UART 修复版本"
git push --recurse-submodules=check origin main
```

最后一条中的 `main` 要换成父仓实际准备推送的分支。`--recurse-submodules=check` 会检查相应子模块提交是否能在其远端跟踪分支中找到；它不会替你提交子仓工作区内容，也不会保证所有同学都有远端访问权限。

**常用顺序：子仓提交 → 子仓推送 → 父仓暂存新指针 → 父仓提交 → 父仓推送。** 只推父仓、把它指向尚未共享的子仓提交，其他人更新时就可能找不到对应版本。[官方示例](https://git-scm.com/book/en/v2/Git-Tools-Submodules)

### 5. 想主动升级到子仓上游的新版本

先确保子仓没有需要保留的未提交修改，再在父仓执行：

```powershell
git submodule update --remote -- external/driver-lib
git diff --submodule=log -- external/driver-lib
```

这次取的是子模块配置的远端跟踪分支状态，通常默认远端 HEAD，可通过配置指定分支；它与“恢复父仓锁定版本”的普通 update 不同。

检查提交、编译并验证后，才把新指针提交到父仓。若希望仓库共享跟踪分支配置，可在确认分支存在后用 `git submodule set-branch --branch main -- external/driver-lib`，并将 `.gitmodules` 的变化一起审查提交。`main` 不一定是每个子仓的默认分支。

### 6. 子仓路径显示 modified，究竟改了什么

```powershell
git status
git submodule status --recursive
git diff --submodule=log -- external/driver-lib
git -C external/driver-lib status
git -C external/driver-lib diff
```

父仓看到的变化可能是：子仓 HEAD 换了提交、子仓内部有修改、子仓有未跟踪文件等。父仓的一次 `git add external/driver-lib` 只能记录提交指针，不能把子仓的未提交源文件变成子仓提交。

`git submodule status` 常见前缀：空格表示对应记录版本，`-` 表示未初始化，`+` 表示当前提交与父仓暂存区记录不一致，`U` 表示相应合并冲突。发现 `+` 后应检查是否为自己有意升级。[官方输出说明](https://git-scm.com/docs/git-submodule)

### 7. detached HEAD 是不是出错了

对子模块默认检出固定提交而言，这是正常状态；它表示 HEAD 直接指向提交，没有挂在普通分支上。

需要在此基础上修改时，先创建分支保存工作。若已经在 detached HEAD 中提交了修改，确认仍位于该提交后，可以用 `git switch -c fix/save-work` 给它建立分支。若已离开，先用该子仓自己的 reflog 定位提交，再建立引用。

### 8. URL 修改、删除与冲突

- **URL 修改：** 按团队流程更改 `.gitmodules` 后，协作者可运行 `git submodule sync --recursive` 同步 URL 配置，再 update。sync 本身不负责切到新代码版本。
- **删除子模块：** 现代 Git 可在父仓使用 `git rm -- external/driver-lib`，检查路径删除和 `.gitmodules` 的对应变化后提交。它会移除检出的内容，先确认子仓工作已保存；不要自行递归删除 `.git/modules` 以求“干净”。
- **父仓子模块指针冲突：** 要决定采用哪一个子仓提交，必要时先在子仓完成代码合并并推送，再回父仓暂存最终提交指针。只解决父仓指针不等于已经解决子仓源码的冲突。
- **切父仓分支：** 不要假定子模块总会自动跟着切；检查状态，保存子仓工作，再按父仓记录运行 `git submodule update --init --recursive`。

<a id="multi"></a>

## 十三、worktree、多仓与其他组织方式

### 1. worktree：同一仓库同时打开不同分支

当前目录保留原分支，另开一个目录处理修复：

```powershell
git worktree list
git worktree add -b fix/uart ../robot-uart-fix main
```

上例要求 `fix/uart` 尚不存在，`main` 是有效起点，目标目录可用。在新目录中按正常方式编辑和提交。

工作区和暂存区各自独立，但共享仓库对象与大部分分支等引用；默认不允许把同一分支同时检出到多个 worktree。它解决的是同一仓库的并行工作，不是引入第三方代码。

完成并保存工作后，可从其他工作树执行 `git worktree remove ../robot-uart-fix`。移除工作树不自动删除对应分支；有未提交修改时通常会拒绝，先检查原因，不要直接强制删除。

含子模块的父仓需要额外注意：Git 官方仍说明多工作树对子模块的支持不完整，且包含子模块的工作树在移动、移除等操作上有限制。初次练习 worktree 时先用普通仓库；复杂父子仓项目不要直接照搬上面的清理步骤。[官方：git worktree](https://git-scm.com/docs/git-worktree)

### 2. 该用哪种组织方式

| 方式 | 适合什么情况 | 需要承担的管理工作 |
| --- | --- | --- |
| 单仓库普通目录 | 应用和模块由同一团队一起维护 | 清楚划分模块，统一提交与构建 |
| submodule | 依赖有独立历史，希望固定具体版本 | 子仓权限、提交可获取性、版本指针与递归更新 |
| subtree | 希望把外部项目内容纳入当前仓库历史，并按需要同步 | 理解导入与同步策略；工具在不同安装环境中的可用性需确认 |
| 多个独立仓库 | 固件、上位机、算法等可独立维护 | 在交接文档或清单中记录各仓 URL 与兼容提交 |
| manifest 工具 | 工程由较多仓库共同组成 | 学习工具的清单与版本锁定方法，如 Zephyr 的 west 或 Android 的 Repo |
| worktree | 同一仓库要同时处理多个分支 | 各工作树的分支、路径、构建产物与生命周期 |

后两类清单工具不是 Git 的同名内置命令。选择子模块也不意味着所有 SDK 都适合这样引入，要考虑许可证、现有构建方式、依赖更新频率和团队熟悉程度。

<a id="tags"></a>

## 十四、标签与版本定位

### 1. 给验证过的版本打标签

```powershell
git tag -a v0.8.1 -m "记录已验证的 Robot 0.8.1 版本"
git push origin v0.8.1
git tag --list
git show v0.8.1
```

执行前确认 HEAD 就是准备标记的提交，标签名称未被占用。标签不会因为普通的分支 push 就总是自动上传；这里只推指定标签，避免把所有本地标签都发布出去。

标签是 Git 引用，GitHub Release 则是托管平台上的发布功能，两者有关联但不是同一个对象。

### 2. 看旧版本，或从它继续修复

```powershell
git switch --detach v0.8.1
```

用于检查旧版本时会进入 detached HEAD。准备继续开发，可以改用 `git switch -c fix/v0.8.1 v0.8.1` 创建修复分支。切换前仍要妥善保存当前工作。

交接固件时，可以同时记录代码提交 ID、标签、工具链、构建配置和产物校验值。只留一个 HEX / BIN，通常不足以复现构建。

<a id="workflow"></a>

## 十五、实验室协作流程与常见问题

### 1. 给这个学习手册补一篇文章

假设你已 clone 仓库、工作区清晰，并具备创建远端分支的权限：

```powershell
git switch main
git pull --ff-only origin main
git switch -c docs/git-guide
# 编辑并校验文章、链接和图片
git status
git add -- "学习/Git常用操作与子仓管理手册.md"
git diff --cached
git diff --cached --check
git commit -m "docs: 增加 Git 常用操作与子仓管理手册"
git push -u origin docs/git-guide
```

这个例子假设你选择把文章放在该路径，并已实际创建文件。推送分支后，在 GitHub 创建 PR，供维护者审阅。PR 不是 Git 的提交命令；没有原仓推送权限时，可先 fork 到自己的账号，向自己的仓库推分支，再向原仓发起 PR。

### 2. 固件工程准备交接

先分别确认应用仓和各子仓工作已提交、对应提交已在可访问远端。检查子模块版本、构建配置和交接说明，再用一个新的干净克隆验证。不要只在自己一直使用、残留很多未跟踪依赖的目录里测试。

### 3. 常见现象速查

| 现象 | 先查什么 |
| --- | --- |
| `not a git repository` | 当前路径是否在仓库里，是否进入了预期的父仓或子仓 |
| `nothing to commit` | 文件是否保存、是否被忽略、是否在另一仓库；暂存区与工作区是否确实无变化 |
| commit 后网页没更新 | 是否 push，远端地址、远端分支和网页所选分支是否一致 |
| push 成功但新文件没出现 | 新文件是否已 add 并进入提交，是否推的是预期分支 |
| `no upstream branch` | 首次推送分支时用合适的 `git push -u origin 分支名` 建立关联 |
| `src refspec ... does not match any` | 分支是否存在、是否已有提交、名称是否拼错 |
| pull 无法快进 | 查看两边历史，确认是否分叉，再选 merge / rebase，不要直接 hard reset |
| switch 提示会覆盖修改 | 先提交、stash 或另行保存自己的修改，再切换 |
| 父仓显示子模块 modified | 分别检查指针变化、子仓已跟踪修改和未跟踪内容 |
| clone 后子模块目录缺内容 | 检查 init/update、URL、权限与记录提交是否已共享 |
| 子仓 update 找不到某次提交 | 检查父仓是否引用了未推送提交、URL 是否正确、历史是否被改写或删除 |
| 提示 dubious ownership | 核对目录所有权和来源，仅在确认可信后按环境要求处理；不要对所有目录一概信任 |
| 文本突然整篇变化 | 检查编码、换行、格式化设置和工具配置 |

### 4. 在独立练习仓库里试一遍

可以新建一个练习目录，放两三个文本文件，依次练习：修改但不暂存、暂存后再修改、取消暂存、创建分支、stash 恢复、制造一次冲突。每次都对照 `status` 和两种 diff 看变化。

子模块再单独做一个父仓与子仓练习，观察父仓提交里记录的指针如何变化。练习中的回退、删除和强制操作只在自己可丢弃的数据上做，不拿实验室共享历史试命令。

<a id="reference"></a>

## 十六、命令速查与官方资料

| 想做什么 | 常用命令 |
| --- | --- |
| 看当前状态 | `git status -sb` |
| 看未暂存修改 | `git diff` |
| 看准备提交的修改 | `git diff --cached` |
| 暂存指定文件 | `git add -- 文件路径` |
| 创建本地提交 | `git commit -m "说明"` |
| 获取远端变化 | `git fetch origin` |
| 快进同步当前分支上游 | `git pull --ff-only` |
| 首次推送指定分支并关联上游 | `git push -u origin 分支名` |
| 切分支 / 创建并切分支 | `git switch 分支名` / `git switch -c 新分支名` |
| 暂时保存未完成工作 | `git stash push -u -m "说明"` |
| 取消暂存 | `git restore --staged -- 文件路径` |
| 通过新提交撤销旧改动 | `git revert 提交ID` |
| 看本机引用变化记录 | `git reflog` |
| 按父仓记录补全子模块 | `git submodule update --init --recursive` |
| 查看子模块版本 | `git submodule status --recursive` |
| 在子仓执行命令 | `git -C 子仓路径 status` |
| 查看工作树 | `git worktree list` |

详细语义以对应版本的官方文档为准：

- 基础：[Git 命令索引](https://git-scm.com/docs)、[config](https://git-scm.com/docs/git-config)、[status](https://git-scm.com/docs/git-status)、[diff](https://git-scm.com/docs/git-diff)、[commit](https://git-scm.com/docs/git-commit)。
- 同步：[fetch](https://git-scm.com/docs/git-fetch)、[pull](https://git-scm.com/docs/git-pull)、[push](https://git-scm.com/docs/git-push)。
- 分支：[switch](https://git-scm.com/docs/git-switch)、[merge](https://git-scm.com/docs/git-merge)、[rebase](https://git-scm.com/docs/git-rebase)、[stash](https://git-scm.com/docs/git-stash)。
- 撤销：[restore](https://git-scm.com/docs/git-restore)、[reset](https://git-scm.com/docs/git-reset)、[revert](https://git-scm.com/docs/git-revert)、[reflog](https://git-scm.com/docs/git-reflog)、[clean](https://git-scm.com/docs/git-clean)。
- 工程：[gitignore](https://git-scm.com/docs/gitignore)、[gitattributes](https://git-scm.com/docs/gitattributes)、[submodule](https://git-scm.com/docs/git-submodule)、[子模块模型](https://git-scm.com/docs/gitsubmodules)、[.gitmodules](https://git-scm.com/docs/gitmodules)、[worktree](https://git-scm.com/docs/git-worktree)。

刚开始不需要记住所有参数。先习惯在操作前后看状态、提交前看暂存差异、遇到分叉时看历史；涉及子仓时，再多确认一次当前命令究竟在操作哪个仓库。
