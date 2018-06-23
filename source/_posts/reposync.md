---
title: reposync
date: 2018-06-12 18:40:40
tags: ['linux']
---

在某些环境中，是不能上外网的，这就需要搭建一个内部的yum源，供内网的服务器安装软件。

那如何把公网的yum源下载下来能，这可以通过使用yum-utils工具包来完成

`
yum install yum-utils -y
`

安装完成之后就可以使用reposync命令来同步yum源了

通过下面的命令可以查看当前配置的yum源

`
yum repolist
`

找到需要同步的yum源，通过下面的命令进行同步

`
reposync --repoid=<$reponame> --download_path=<$path>
`

这样就把公网的yum源同步到本地磁盘了，以后就可以通过下载下来的内容搭建内网yum源
