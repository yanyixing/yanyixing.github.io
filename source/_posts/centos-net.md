---
title: Centos7 修改网卡为ethX
date: 2017-10-10 17:57:57
tags:
---
Centos7之后默认的网卡名变成enp0s3这种样子，不再是ethX

通过下面的步骤可以把网卡名变回ethX

+ vim /etc/default/grub

	在GRUB_CMDLINE_LINUX的最后，加上 <span style="color:red">net.ifnames=0 biosdevname=0</span> 的参数
	
	```
	GRUB_TIMEOUT=5
	GRUB_DISTRIBUTOR=”$(sed ‘s, release .*$,,g’ /etc/system-release)”
	GRUB_DEFAULT=saved
	GRUB_DISABLE_SUBMENU=true
	GRUB_TERMINAL_OUTPUT=”console”
	GRUB_CMDLINE_LINUX=”rd.lvm.lv=rootvg/usrlv rd.lvm.lv=rootvg/swaplv crashkernel=auto vconsole.keymap=us rd.lvm.lv=rootvg/rootlv vconsole.font=latarcyrheb-sun16 rhgb quiet net.ifnames=0 biosdevname=0”
	GRUB_DISABLE_RECOVERY=”true”
	```
	
+ grub2-mkconfig -o /boot/grub2/grub.cfg
+ mv /etc/sysconfig/network-scripts/ifcfg-enp0s3  /etc/sysconfig/network-scripts/ifcfg-eth0
+ reboot


 