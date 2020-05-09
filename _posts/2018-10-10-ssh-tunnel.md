---
layout: post
title: "ssh隧道及代理"
categories: ssh
tags: tunnel socks5
---  

* content
{:toc}

fork 了别人的仓库后，原作者又更新了仓库，如何将自己的代码和原仓库保持一致？本文将给你解答。





## 如何使用搜索引擎

其实这个问题并不难，我又被坑了。百度搜的东西不靠谱啊，以后这种问题一定要用**英文**在 [Google](http://www.google.com) 或者 [Bing](http://cn.bing.com/) 上搜索，这样才能搜到原汁原味的答案。就当是一个教训吧。   

搜索 fork sync，就可以看到 GitHub 自己的帮助文档 [Syncing a fork](https://help.github.com/articles/syncing-a-fork/) 点进去看这篇的时候，注意到有一个 Tip: Before you can sync your fork with an upstream repository, you must [configure a remote that points to the upstream repository](https://help.github.com/articles/configuring-a-remote-for-a-fork/) in Git.   

根据这两篇文章，问题迎刃而解！   

## 具体方法

### Configuring a remote for a fork
