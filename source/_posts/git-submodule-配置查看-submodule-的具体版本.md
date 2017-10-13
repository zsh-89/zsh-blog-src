---
title: 'git submodule: 配置查看 submodule 的具体版本'
date: 2017-08-10 10:57:53
tags: 
  - git
  - git submodule
---

## 背景
git submodule 存在的价值, 基本的教程可以查看[这里](https://git-scm.com/book/en/v2/Git-Tools-Submodules).
比较遗憾的是, 目前看到的教程都没有说清: 
+ 如何配置 submodule 的具体版本 (具体哪一个 commit)
+ 在哪里查看当前的配置信息

某个[stackoverflow 回答(第二个回答)](https://stackoverflow.com/questions/1777854/git-submodules-specify-a-branch-tag)提供了完全错误的信息.


## 为什么需要配置 submodule 的具体版本 ( commit sha-1 )
从工程的角度看, 配置主项目的 submodule 时, 
我们必然是将主 git 项目和 dependency git 项目的某个 stable release 版本关联在一起;
毕竟我们所有的测试都是针对特定版本的; 和版本 1 一起工作良好, 换版本 2 可能编译都不通过.


## 实例
以 "个人 blog" 作为实例.
可以在 [这里](https://github.com/zsh-89/zsh-blog-src) 看到我的个人 blog 的源码; 可以看到 themes/next 关联到了另一个 git repo.

第一次用 `git submodule add https://github.com/iissnan/hexo-theme-next.git themes/next` 命令添加 submodule 时,
命令帮我们将 hexo-theme-next.git 的 master branch 的最新版本 (假设这个版本的 commit sha-1 是 123456) checkout 到本地.
这时, 我们如果直接 commit 主 project,
git 就 **隐式地指定了** 主 project 采用 hexo-theme-next.git project 中 123456 这个 commit 版本作为 submodule (在 github Web 上可以看到 track 的 sha-1).

"submodule 的版本的 sha-1" 可以在进入 "themes/next" 文件夹后, 运行 `git rev-parse HEAD` 查看; 也可以通过 `git status` 查看状态;
可以看到, 这个目录相当于是一个独立的 git repo 目录.

如果不希望主 project 使用 subproject 的 master branch 的最新版, 
可以在 `git submodule add` 之后, 
进入 submodule 目录 (这里例子里, 是 themes/next 目录), 手动 checkout 到希望使用的版本 (如 `git checkout tags/<some_stable_release>`), 再 commit.


