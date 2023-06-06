# Git Skill

{{< figure src="/images/git-skill01.png" title="git (figure)" >}}
# 🚀git 常用命令和技巧
* 列举一些常用的git命令和一些小技巧
>  统一概念
* **工作区**： 该动（增删文件和内容）
* **暂存区**： 输入命令，git add 改动的文件名，此次改动就放到了 ‘暂存区’
* **本地仓库**： 输入命令：git commit 此次修改的描述，此次改动就放到了本地仓库，每个 commit，我叫它为一个版本
* **远程仓库**： 输入命令：git push 远程仓库，此次改动就放到了远程仓库（GitHub 等)
* **commit-id**：输出命令：git log，最上面那行 commit xxxxxx，后面的字符串就是 commit-id
<!--more-->


## git 基础介绍

> 创建新仓库
```bash
# 创建新文件夹， 打开， 然后执行
git init
```

> 检出仓库
```bash
# 执行命令以创建一个本地仓库的克隆版本
git clone /path/to/repository
# 远端服务器的仓库
git clone username@host:/path/to/repository
```

> 工作流
* 本地仓库有git维护三颗树组成
1. **工作目录**： 持有实际的文件
2. **暂存区** ： 缓存区域，临时保存你的改动
3. **HEAD**： 指向你最后一次提交的结果

> 添加和提交
```bash
## 提出更改（把它们添加到暂存区）
git add <filename>
git add *
## 改动已经提交到了 HEAD
git commit -m "代码提交信息"
```

> 推送改动
```bash
## 将改动提交到 远端仓库
git push origin master
## 将你的仓库连接到某个远程服务器，你可以使用如下命令添加
git remote add origin <server>
```

> 分支
```bash
## 创建一个分支
git checkout -b feature_x
## 切换回主分支
git checkout master
## 把新建的分支删除
git branch -d feature_x
## 除非你将分支推送到远端仓库，不然该分支就是 不为他人所见的
git push origin <branch>
```

>  更新与合并
```bash
## 更新你的本地仓库至最新改动
git pull
## 合并其他分支到你的当前分支
git merge <branch>
## 在这两种情况下，git 都会尝试去自动合并改动。遗憾的是，这可能并非每次都成功，并可能出现冲突（conflicts）。 这时候就需要你修改这些文件来手动合并这些冲突（conflicts）。改完之后，你需要执行如下命令以将它们标记为合并成功
git add <filename>
## 在合并改动之前，你可以使用如下命令预览差异
git diff <source_branch> <target_branch>
```

> 标签
```bash
## 创建一个 1.00 的标签
git tag 1.0.0 1b2e1d63ff
## 1b2e1d63ff 是你想要标记的提交 ID 的前 10 位字符。可以使用下列命令获取提交 ID
git log
```

> log
```bash
## 本地仓库的历史记录
git log
## 只看某人的提交记录
git log --author=bob
## 一个压缩后的每一条提交记录只占一行输出
git log --pretty=oneline
## 通过 ASCII 艺术的树形结构来展示所有的分支
git log --graph --oneline --decorate --all
## 查看那些文件改变了
git log --name-status
```

> 替换本地改动
```bash
## 替换掉本地的改动
git checkout -- <filename>
## 丢弃本地所有的改动和提交，可以到服务器上获取最新的版本历史，并将你本地主分支指向它
git fetch origin
git reset --hard origin/master
```

## 回到远程仓库的状态
* 抛弃本地所有的修改，回到远程仓库的状态的
```bash
git fetch --all && git reset --hard origin/master
```

## 重设第一个 commit
* 把所有的修改都重新放回工作区， 并**清空所有的 commit**， 这样可以重新提交第一个commit
```bash
git update-ref -d HEAD
```

## 查看冲突文件列表
* 展示工作区的冲突文件列表
```bash
git diff --name-only --diff-filter=U
```

## 展示工作区和暂存区的不同
```bash
## 输出**工作区**和**暂存区**的 different
git diif
## 还可以展示本地仓库中任意两个 commit 之间的文件变动
git diff <commit-id> <commit-id>
```

## 展示暂存区和最近版本的不同
```bash
## 输出暂存区和本地最近的版本 (commit) 的 different (不同)
git diff --cached
```

## 快速切换到上一个分支
```bash
git checkout -
```

## 删除已经合并到 master 的分支
```bash
git branch --merged master | grep -v '^\*\|  master' | xargs -n 1 git branch -d
```

## 展示本地分支关联远程仓库的情况
```bash
git branch -vv
```

## 关联远程分支
```bash
## 关联之后，git branch -vv 就可以展示关联的远程分支名了，同时推送到远程仓库直接：git push，不需要指定远程仓库
git branch -u origin/mybranch
## 或者在 push 时加上 -u 参数
git push origin/mybranch -u
```

## 列出所有的远程分支
```bash
## -r 参数相当于：remote
git branch -r
```

## 列出本地和远程分支
```bash
## -a 参数相当于：all
git branch -a
```

## 查看远程分支和本地分支的对应关系
```bash
git remote show origin
```

## 远程删除了分支本地也想删除
```bash
git remote prune origin
```

## 创建并切换到本地分支
```bash
git checkout -b <branch-name>
```

## 从远程分支中创建并切换到本地分支
```bash
git checkout -b <branch-name> origin/<branch-name>
```

## 删除本地分支
```bash
git branch -d <local-branchname>
```

## 删除远程分支
```bash
git push origin --delete <remote-branchname>
## 或者
git push origin :<remote-branchname>
```

## 重命名本地分支
```bash
git branch -m <new-branch-name>
```

## 查看标签
```bash
git tag
## 展示当前分支的最近的 tag
git describe --tags --abbrev=0
```

## 查看标签的详细信息
```bash
git tag -ln
```

## 本地创建标签
```bash
git tag <version-number>
## 默认 tag 是打在最近的一次 commit 上，如果需要指定 commit 打 tag：
$ git tag -a <version-number> -m "v1.0 发布(描述)" <commit-id>
```

## 推送标签到远程仓库
```bash
## 首先要保证本地创建好了标签才可以推送标签到远程仓库：
git push origin <local-version-number>
## 一次性推送所有标签，同步到远程仓库：
git push origin --tags
```

## 删除本地标签
```bash
git tag -d <tag-name>
```

## 删除远程标签
```bash
git push origin --delete tag <tagname>
```

## 切回到某个标签
```bash
## 一般上线之前都会打 tag，就是为了防止上线后出现问题，方便快速回退到上一版本。下面的命令是回到某一标签下的状态
git checkout -b branch_name tag_name
```

## 放弃工作区的修改
```bash
git checkout <file-name>
## 或者放弃所有修改
git checkout .
```

## 恢复删除的文件
```bash
#得到 deleting_commit
git rev-list -n 1 HEAD -- <file_path> 
#回到删除文件 deleting_commit 之前的状态
git checkout <deleting_commit>^ -- <file_path>
```

## 以新增一个 commit 的方式还原某一个 commit 的修改
```bash
git revert <commit-id>
```

## 回到某个 commit 的状态，并删除后面的 commit
```bash
## 和 revert 的区别：reset 命令会抹去某个 commit id 之后的所有 commit
git reset <commit-id>  #默认就是-mixed参数。

git reset --mixed HEAD^  #回退至上个版本，它将重置HEAD到另外一个commit,并且重置暂存区以便和HEAD相匹配，但是也到此为止。工作区不会被更改。

git reset --soft HEAD~3  #回退至三个版本之前，只回退了commit的信息，暂存区和工作区与回退之前保持一致。如果还要提交，直接commit即可  

git reset --hard <commit-id>  #彻底回退到指定commit-id的状态，暂存区和工作区也会变为指定commit-id版本的内容
```

## 修改上一个 commit 的描述
```bash
## 如果暂存区有改动，同时也会将暂存区的改动提交到上一个 commit
git commit --amend
```

## 查看 commit 历史
```bash
git log
```

## 查看某段代码是谁写的
```bash
## blame 的意思为‘责怪’，你懂的。
git blame <file-name>
```

## 显示本地更新过 HEAD 的 git 命令记录
```bash
## 每次更新了 HEAD 的 git 命令比如 commit、amend、cherry-pick、reset、revert 等都会被记录下来（不限分支），就像 shell 的 history 一样。 这样你可以 reset 到任何一次更新了 HEAD 的操作之后，而不仅仅是回到当前分支下的某个 commit 之后的状态
git reflog
```

## 修改作者名
```bash
git commit --amend --author='Author Name <email@address.com>'
```

## 修改远程仓库的 url
```bash
git remote set-url origin <URL>
```

## 增加远程仓库
```bash
git remote add origin <remote-url>
```

## 列出所有远程仓库
```bash
git remote
```

## 查看两个星期内的改动
```bash
git whatchanged --since='2 weeks ago'
```

## 把 A 分支的某一个 commit，放到 B 分支上
```bash
## 这个过程需要 cherry-pick 命令
git checkout <branch-name> && git cherry-pick <commit-id>
```

## 给 git 命令起别名
```bash
git config --global alias.<handle> <command>

比如：git status 改成 git st，这样可以简化命令

git config --global alias.st status
```

## 存储当前的修改，但不用提交 commit
详解可以参考廖雪峰老师的 git 教程 
https://www.liaoxuefeng.com/wiki/896043488029600/900388704535136
```bash
git stash
```

## 保存当前状态，包括 untracked 的文件
```bash
## untracked 文件：新建的文件
git stash -u
## 展示所有 stashes
git stash list
## 回到某个 stash 的状态
git stash apply <stash@{n}>
## 回到最后一个 stash 的状态，并删除这个 stash
git stash pop
## 删除所有的 stash
git stash clear
## 从 stash 中拿出某个文件的修改
git checkout <stash@{n}> -- <file-path>
## 展示所有 tracked 的文件
git ls-files -t
## 展示所有 untracked 的文件
git ls-files --others
```

## 展示所有忽略的文件
```bash
git ls-files --others -i --exclude-standard
```

## 展示简化的 commit 历史
```bash
git log --pretty=oneline --graph --decorate --all
```

## 把某一个分支导出成一个文件
```bash
git bundle create <file> <branch-name>
```

## 从包中导入分支
```bash
## 新建一个分支，分支内容就是上面 git bundle create 命令导出的内容
git clone repo.bundle <repo-dir> -b <branch-name>
```

## 从远程仓库根据 ID，拉下某一状态，到本地分支
```bash
git fetch origin pull/<id>/head:<branch-name>
```

## 清除 gitignore 文件中记录的文件
```bash
git clean -X -f
```

## 展示所有 alias 和 configs
```bash
## 注意： config 分为：当前目录（local）和全局（golbal）的 config，默认为当前目录的 config
git config --local --list (当前目录)
git config --global --list (全局)
```

## 展示忽略的文件
```bash
git status --ignored
```

## 删除全局设置
```bash
git config --global --unset <entry-name>    
```

## 新建并切换到新分支上，同时这个分支没有任何 commit
```bash
## 相当于保存修改，但是重写 commit 历史
git checkout --orphan <branch-name>
```

## git checkout --orphan <branch-name>
```bash
git show <branch-name>:<file-name>
```

## clone 下来指定的单一分支
```bash
git clone -b <branch-name> --single-branch https://github.com/user/repo.git
```

## 忽略文件的权限变化
```bash
## 不再将文件的权限变化视作改动
git config core.fileMode false
```

## 在 commit log 中查找相关内容
```bash
## 通过 grep 查找，given-text：所需要查找的字段
git log --all --grep='<given-text>'
```

## 把暂存区的指定 file 放到工作区中
```bash
## 不添加参数，默认是 -mixed
git reset <file-name>
```

## 强制推送
```bash
git push -f <remote-name> <branch-name>
```
