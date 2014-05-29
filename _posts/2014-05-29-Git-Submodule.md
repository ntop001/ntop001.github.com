---
layout: post
title: "Git submodule 使用"
description: ""
category: 
tags: []
---
{% include JB/setup %}



##建立：

分别在 github 上建立 main 工程和 sub 工程
把 main 工程 clone到本地

```
git clone xxx-main.git
```

把 sub 工程添加到 submodule

```
git submodule add xxx-sub.git
```

添加submodule 之后会自动clone submodule 的代码到本地

对于main来说 submodule 的添加就像添加一个文件，所以需要 add 并 commit 

```
git add .
git commit -m "set up submodule"
git push origin maser //传送到服务器
```

##Clone :
其他开发人员可以直接Clone代码到本地

```
git clone xxx-main.git
```

但是默认情况下，git不会同步submodule 的代码,只是建立了一个空的文件夹，这时候需要调用 

```
git submodule init //把 submodule 的git 工程初始化
git submodule update // 同步最新的 submodule 代码
```

更新 submodule 是非常重要的操作，每次同步代码之后务必使用 git submodule 命令查看当前 submodule 状态，
如果 commit id 前有+ 号，表示当前parent分支下面的 submodule 不是最新的，这时同样需要调用 git submodule update 
更新当前分支. 否则会把旧的 submodule 重新提交到主线分支，使新的 submodule 代码得不到更新。

如果 main.git 还有其他的分支，比如 dev , 并且对应的 submodule 也有改动，同样需要调用 git submodule update 同步最新的submodule状态.

```
git checkout dev
git submodule    //查看 submodule 是否是最新的（有没有+号）
git submodule update //更新子模块
```

##改动并提交Submodule代码

假如你已经修改了submodule 并想把这些改动提交，并希望其他人可以引用。

需要单独把 submodule 的代码提交，把 submodule 当做一个独立的 git 工程对待。

```
cd xxx-sub // 进入 submodule 目录
git add .
git commit -m "xxxx"
git push origin master	//提交submodule代码
```

返回main工程，

```
git add .
git commit -m "update submodule"
git push origin master	//提交main 代码
```

这样才能够把 submodule 的改动提交到 main 工程。


###为什么分别提交了两次 ？

Git 对submodule 的管理是这样的，在 main 工程中，并不递归记录 submodule 中文件的变动，仅仅记住 submodule 的分支和Commit Id ，这样足够从一个 submodule 中取出需要的代码，而 submodule的管理是完全独立的。所以需要先把submodule的改动提交了，再在 main 工程更新submodule 的分支和commit id 变化。

如果这时有人pull 了你的 main 工程代码，他也需要 git submodule update 把分支的更新同步到本地，就像前面说的那样。


PS：<br/>
第一次Clone代码的时候可以使用 git clone --recursive xxx-main.git 一步到位的同步main和submodule工程（仅限于master分支）















