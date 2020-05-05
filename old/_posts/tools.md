---
title: tools
date: 2019-08-01 18:27:02
tags:
---

1 日志系统
===

[kibana展示es集群计算logbash收集并http发送至集群](https://blog.csdn.net/vic_qxz/article/details/80941015)

2 git
===
>- git工作区、暂存区、版本库
>- 工作区修改后，git add将修改内容加载到暂存区（版本库中的index索引区，objects中存具体内容，index中只存相应的文件或者tree的索引）
>- git commit将暂存区内容加载到版本库中（版本库目录结构)
>- git reset HEAD更改暂存区内容，不更改工作区
>- git checkout 更改工作区&暂存区
>- git rm --cached直接从暂存区删除文件，工作区不做变更
>- git diff --cached查看暂存区与版本库所在分支区别（即将被提交的内容）
[git回滚](https://segmentfault.com/a/1190000015792394)
[git工作区暂存区版本库](https://www.runoob.com/git/git-workspace-index-repo.html)

3 proto
===
>- [proto文件生成go文件](https://www.jianshu.com/p/b78478c1710b)