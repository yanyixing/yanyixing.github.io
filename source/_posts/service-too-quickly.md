---
title: service start too often
date: 2017-12-18 16:15:08
tags:
---

通过systemd来管理服务，经常会遇到这种错误"failed because start of the service wasattempted too often".

遇到这种情况可以到/etc/systemd/system目录下，修改相应的service文件

一般情况下在添加下面的内容即可：

`[Service]
StartLimitBurst=0`

保存退出后，执行reload

`
sudo systemctl daemon-reload
`

再启动相应服务即可。
