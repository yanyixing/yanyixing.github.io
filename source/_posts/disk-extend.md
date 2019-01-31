---
title: Centos 磁盘扩容
date: 2019-01-30 14:10:15
tags: centos
---

# 概述
本文主要介绍如何在vmware环境中给centos7虚拟机进行扩容。  
centos7默认磁盘用lvm管理，系统盘挂在一个xfs的lv上。

# 扩容前
先看一下扩容前的样子  
 {% asset_img disk1.png 扩容前 %}  
 虚拟机有一个40G的硬盘，根分区挂在了centos-root的lv上，大小是37.5G
 
# 扩容
先关闭虚拟机，通过vmware软件来给虚拟机的硬盘扩容。  
{% asset_img disk2.png 虚拟机配置 %}  
{% asset_img disk3.png 磁盘扩容 %}  
通过上面的步骤，磁盘的空间扩展到了50G

{% asset_img disk4.png 磁盘空间 %}  
可以看到，磁盘的空间已经是50G，接下来通过parted命令，把剩余的10G空间做出lvm分区。  
{% asset_img disk5.png 分区 %}  
{% asset_img disk6.png 调整分区类型 %}  
把新创建的分区作为PV，并且添加到VG中。  
{% asset_img disk7.png 增加VG %}  
扩大LV的空间  
{% asset_img disk8.png LV扩容 %}
通过xfs_growfs命令来动态调整xfs文件系统的容量  
{% asset_img disk9.png XFS扩容 %}  

最终我们可以看到根分区的文件系统扩大了10G。  





