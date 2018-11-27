---
title: 通过ceph-deploy部署ceph
date: 2018-11-27 14:58:11
tags: ceph
---

之前一直使用ceph-ansible来部署ceph，ceph-ansible在大规模部署的情况下比较合适，而且支持各种部署方式。  
现在遇到的场景是是集群需要动态的调整，一开始是一个小规模的集群，后续需要动态增删服务来动态调整集群。  
ceph-ansible并没有单独添加删除某个服务的脚本，并不适合这种情况；而ceph-deploy可以比较方便的支持服务的添加和删除，可以满足这种场景。 
 
下面通过实验来验证ceph-deploy部署ceph集群的各种服务。

# 环境信息

本次实验共有4台虚拟机，具体信息如下

| Hostname | OS | Public network | Cluster network | Role |
| ----- | ----- | ----- | ----- | ----- |
| ceph001 | CentOS Linux release 7.5.1804 (Core) | 172.16.143.151 | 172.16.140.151| ceph-deply |
| ceph002 | CentOS Linux release 7.5.1804 (Core) | 172.16.143.152 | 172.16.140.152| mon osd mgr rgw mds |
| ceph003 | CentOS Linux release 7.5.1804 (Core) | 172.16.143.153 | 172.16.140.153| mon osd mgr rgw mds |
| ceph004 | CentOS Linux release 7.5.1804 (Core) | 172.16.143.154 | 172.16.140.154| mon osd mgr rgw mds |

# 配置
* 关闭防火墙和SELinux
* 配置ceph001到ceph002~4的免密登录
* 配置ceph的国内源

ceph源配置，本次实验安装的ceph版本是**luminous**

```
[ceph_stable]
baseurl = http://mirrors.ustc.edu.cn/ceph/rpm-luminous/el7/$basearch
gpgcheck = 1
gpgkey = http://mirrors.ustc.edu.cn/ceph/keys/release.asc
name = Ceph Stable repo

[ceph_noarch]
name=Ceph noarch packages
baseurl=http://mirrors.ustc.edu.cn/ceph/rpm-luminous/el7/noarch/
gpgcheck=1
gpgkey=http://mirrors.ustc.edu.cn/ceph/keys/release.asc
```

# 安装

## 安装软件
在ceph001节点安装ceph-deploy，并且创建ceph集群

```
yum install ceph-deploy -y
ceph-deploy install --release luminous ceph002 ceph003 ceph004
ceph-deploy new --cluster-network 172.16.140.0/24 --public-network 172.16.143.0/24 ceph002 ceph003 ceph004

```

当前目录下会生成ceph.conf和 ceph.mon.keying

ceph.conf

```
[global]
fsid = 9ef68d6d-5117-4064-8969-39f51e91557e
public_network = 172.16.143.0/24
cluster_network = 172.16.140.0/24
mon_initial_members = ceph002, ceph003, ceph004
mon_host = 172.16.143.152,172.16.143.153,172.16.143.154
auth_cluster_required = cephx
auth_service_required = cephx
auth_client_required = cephx
```


## 部署MON

```
ceph-deploy mon create ceph002 ceph003 ceph004
```

## 收集key

```
ceph-deploy gatherkeys ceph002 ceph003 ceph004
```

## 创建OSD

```
ceph-deploy osd create ceph002 --data /dev/sdb
ceph-deploy osd create ceph003 --data /dev/sdb
ceph-deploy osd create ceph004 --data /dev/sdb
```

## 部署MGR

```
ceph-deploy mgr create ceph002 ceph003 ceph004
```

## 允许管理员执行ceph命令

```
ceph-deploy admin ceph002 ceph003 ceph004
```

## 部署RGW

```
ceph-deploy rgw create ceph002 ceph003 ceph004
```

## 部署MDS

```
ceph-deploy mds create ceph002 ceph003 ceph004
```

```
ceph osd pool create cephfs_metadata 8 8
ceph osd pool create cephfs_data 8 8
ceph fs new cephfs_demo cephfs_metadata cephfs_data
```

# 截图

{% asset_img cephstatus.png 集群状态 %}

{% asset_img cephfs.png cephfs状态 %}


# 总结

ceph-deploy适合小规模集群的部署，并且可以满足集群的动态调整。  
另外，当前版本的ceph-deploy已经使用ceph-volume替换ceph-disk。

