# Git Tag

{{< figure src="/images/git-tag01.jpg" title="git tag (figure)" >}}
# 🚀git tag 标签管理
* 可以在版本库中打一个标签，确定打标签的版本
* 标签就是版本库的快照
<!--more-->
## 创建标签
> 切换到需要打标签的分支上
```bash
$ git branch
* dev
  master
$ git checkout master
Switched to branch 'master'
```
> 打一个新标签
```bash
git tag v1.0
```
> 查看所有标签
```bash
git tag
v1.0
```
* 默认标签是打在最新提交的commit上
* 找到历史提交的commit id，然后打
```bash
git log --pretty=oneline --abbrev-commit
12a631b (HEAD -> master, tag: v1.0, origin/master) merged bug fix 101
4c805e2 fix bug 101
e1e9c68 merge with no-ff
f52c633 add merge
cf810e4 conflict fixed
```
* 要对add merge这次提交打标签
```bash
git tag v0.9 f52c633
```
> 再次查看标签
```bash
git tag
v0.9
v1.0
```
> 标签不是按时间顺序列出，而是按字母排序的。可以用git show <tagname>查看标签信息
```bash
git show v0.9
commit f52c63349bc3c1593499807e5c8e972b82c8f286 (tag: v0.9)
Author: Michael Liao <askxuefeng@gmail.com>
Date:   Fri May 18 21:56:54 2018 +0800

    add merge

diff --git a/readme.txt b/readme.txt
```

> 可以创建带有说明的标签，用-a指定标签名，-m指定说明文字
```bash
git tag -a v0.1 -m "version 0.1 released" 1094adb
```

## 操作标签
> 删除标签
```bash
git tag -d v0.1
```

>  推送某个标签到远程
```bash
git push origin v1.0
```
* 推送所有标签
```bash
git push origin --tags
```

> 删除远程标签
```bash
## 本地删除标签
git tag -d v0.9
## 远程删除
git push origin :refs/tags/v0.9
```

## 命令总结
* git tag <tagname>用于新建一个标签，默认为HEAD，也可以指定一个commit id；
* git tag -a <tagname> -m "blablabla..."可以指定标签信息；
* git tag可以查看所有标签
* git push origin <tagname>可以推送一个本地标签；
* git push origin --tags可以推送全部未推送过的本地标签；
* git tag -d <tagname>可以删除一个本地标签；
* git push origin :refs/tags/<tagname>可以删除一个远程标签

参考链接
https://www.liaoxuefeng.com/wiki/896043488029600/900788941487552
