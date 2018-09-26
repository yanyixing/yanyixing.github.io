---
title: 通过pacemaker管理mysql
date: 2018-03-05 17:59:24
tags: ['linux','mysql','ha']
---

在上一篇blog中[mysql-replication](https://yanyixing.github.io/2018/03/02/mysql-replication/)介绍了如何配置mysql(mariadb)主从复制。我们还可以添加从节点到主节点的复制关系，这样就达到了mysql(mariadb)双向复制。

下面的配置通过pacemaker管理一个双向复制的mysql(mariadb)和一个VIP来实现mysql(mariadb)在双节点上的高可用。

## 环境

OS: CentOS Linux release 7.3.1611 (Core)

DB: mysql  Ver 15.1 Distrib 5.5.56-MariaDB, for Linux (x86_64) using readline 5.1

host1(master): 172.16.143.171

host2(slave): 172.16.143.172


## 安装


```bash
yum install -y mariadb-server
```

安装完成后可以按照上一篇blog来配置双向复制。


```bash
systemctl stop mariadb

systemctl disable mariadb
```

安装配置pacemaker

```bash
yum install pcs pacemaker corosync fence-angets-all resource-angets -y

systemctl start pcsd

systemctl enable pcsd

systemctl start pacemaker

systemctl enable pacemaker

systemctl start corosync

systemctl enable corosync

passwd hacluster

pcs cluster auth ceph01 ceph02

pcs cluster setup --start --name cluster_mysql ceph01 ceph02

pcs cluster start --all

pcs property set stonith-enabled=false

pcs property set no-quorum-policy=ignore
```

## 配置资源

```bash
pcs resource create virtual_ip ocf:heartbeat:IPaddr2 \
ip=172.16.143.200 cidr_netmask=24 nic=eth0 op monitor interval=30s on-fail=restart

pcs resource create mysql ocf:heartbeat:mysql  \
binary="/usr/bin/mysqld_safe"   config="/etc/my.cnf" \
  datadir="/var/lib/mysql"   pid="/var/lib/mysql/mysql.pid" \
  socket="/var/lib/mysql/mysql.sock"  \
 additional_parameters="--bind-address=0.0.0.0"   op start timeout=60s \
  op stop timeout=60s   op monitor interval=20s timeout=30s \
on-fail=standby

pcs resource clone mysql clone-max=2 clone-node-max=1
```

## 结果

```bash
[root@ceph01 ~]# pcs status
Cluster name: cluster_mysql
Stack: corosync
Current DC: ceph01 (version 1.1.16-12.el7_4.7-94ff4df) - partition with quorum
Last updated: Mon Mar  5 22:07:52 2018
Last change: Mon Mar  5 22:06:22 2018 by root via cibadmin on cephlcm

2 nodes configured
3 resources configured

Online: [ ceph01 ceph02 ]

Full list of resources:

 vip	(ocf::heartbeat:IPaddr2):	Started ceph01
 Clone Set: mysql-clone [mysql]
     Started: [ ceph01 ceph02 ]

Daemon Status:
  corosync: active/enabled
  pacemaker: active/enabled
  pcsd: active/enabled
```

