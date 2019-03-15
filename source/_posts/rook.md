---
title: Rook
date: 2019-03-12 14:42:10
tags: ['ceph','k8s']
---

# 概述
本文主要介绍如何通过rook在k8s上部署一套ceph集群。  
测试的k8s集群一共三个节点：  

```
[root@kube01 ~]# kubectl get nodes
NAME     STATUS   ROLES    AGE    VERSION
kube01   Ready    master   3h3m   v1.13.4
kube02   Ready    master   175m   v1.13.4
kube03   Ready    master   172m   v1.13.4

```

# Rook部署

clone rook代码  

```
git clone https://github.com/rook/rook.git
cd rook/
git checkout v0.9.3
```
通过kubectl执行rook-ceph的operator

```
[root@kube01 rook]# kubectl apply -f cluster/examples/kubernetes/ceph/operator.yaml
namespace/rook-ceph-system created
customresourcedefinition.apiextensions.k8s.io/cephclusters.ceph.rook.io created
customresourcedefinition.apiextensions.k8s.io/cephfilesystems.ceph.rook.io created
customresourcedefinition.apiextensions.k8s.io/cephobjectstores.ceph.rook.io created
customresourcedefinition.apiextensions.k8s.io/cephobjectstoreusers.ceph.rook.io created
customresourcedefinition.apiextensions.k8s.io/cephblockpools.ceph.rook.io created
customresourcedefinition.apiextensions.k8s.io/volumes.rook.io created
clusterrole.rbac.authorization.k8s.io/rook-ceph-cluster-mgmt created
role.rbac.authorization.k8s.io/rook-ceph-system created
clusterrole.rbac.authorization.k8s.io/rook-ceph-global created
clusterrole.rbac.authorization.k8s.io/rook-ceph-mgr-cluster created
serviceaccount/rook-ceph-system created
rolebinding.rbac.authorization.k8s.io/rook-ceph-system created
clusterrolebinding.rbac.authorization.k8s.io/rook-ceph-global created
deployment.apps/rook-ceph-operator created
```

等待全部pod都running状态

```
[root@kube01 rook]# kubectl get pods -n rook-ceph-system -owide
NAME                                  READY   STATUS    RESTARTS   AGE     IP               NODE     NOMINATED NODE   READINESS GATES
rook-ceph-agent-7zgcv                 1/1     Running   0          2m15s   192.168.10.181   kube01   <none>           <none>
rook-ceph-agent-ww4sk                 1/1     Running   0          2m15s   192.168.10.129   kube02   <none>           <none>
rook-ceph-agent-xjgm4                 1/1     Running   0          2m15s   192.168.10.44    kube03   <none>           <none>
rook-ceph-operator-5f4ff4d57d-fm8s5   1/1     Running   0          3m41s   10.244.1.2       kube02   <none>           <none>
rook-discover-2mwpn                   1/1     Running   0          2m15s   10.244.0.10      kube01   <none>           <none>
rook-discover-nqhr8                   1/1     Running   0          2m15s   10.244.1.3       kube02   <none>           <none>
rook-discover-wgpng                   1/1     Running   0          2m15s   10.244.2.2       kube03   <none>           <none>
```

ceph会使用每个节点的vdb作为osd，所以需要修改cluster/examples/kubernetes/ceph/cluster.yaml的内容

```
  storage: # cluster level storage configuration and selection
    useAllNodes: true
    useAllDevices: false
    deviceFilter: "^vdb"
    location:
    config:
      # The default and recommended storeType is dynamically set to bluestore for devices and filestore for directories.
      # Set the storeType explicitly only if it is required not to use the default.
      storeType: bluestore
      #databaseSizeMB: "1024" # this value can be removed for environments with normal sized disks (100 GB or larger)
      #journalSizeMB: "1024"  # this value can be removed for environments with normal sized disks (20 GB or larger)
      osdsPerDevice: "1" # this value can be overridden at the node or device level
```

通过kubectl部署ceph集群

```
[root@kube01 rook]# kubectl apply -f cluster/examples/kubernetes/ceph/cluster.yaml
namespace/rook-ceph created
serviceaccount/rook-ceph-osd created
serviceaccount/rook-ceph-mgr created
role.rbac.authorization.k8s.io/rook-ceph-osd created
role.rbac.authorization.k8s.io/rook-ceph-mgr-system created
role.rbac.authorization.k8s.io/rook-ceph-mgr created
rolebinding.rbac.authorization.k8s.io/rook-ceph-cluster-mgmt created
rolebinding.rbac.authorization.k8s.io/rook-ceph-osd created
rolebinding.rbac.authorization.k8s.io/rook-ceph-mgr created
rolebinding.rbac.authorization.k8s.io/rook-ceph-mgr-system created
rolebinding.rbac.authorization.k8s.io/rook-ceph-mgr-cluster created
cephcluster.ceph.rook.io/rook-ceph created
```

等ceph部署完成，可以看到有三个mon，一个mgr和三个osd

```
[root@kube01 rook]# kubectl get pods -n rook-ceph
NAME                                 READY   STATUS      RESTARTS   AGE
rook-ceph-mgr-a-66db78887f-lmhcf     1/1     Running     0          4m13s
rook-ceph-mon-a-b6556df54-t7cd4      1/1     Running     0          4m53s
rook-ceph-mon-b-7f84c6d4b-nhjj6      1/1     Running     0          4m42s
rook-ceph-mon-c-868c5b476b-ghfqs     1/1     Running     0          4m32s
rook-ceph-osd-0-6fdf57bcb7-bf6f2     1/1     Running     0          54s
rook-ceph-osd-1-8c99b7447-h4mfd      1/1     Running     0          39s
rook-ceph-osd-2-66b76d6944-5df9j     1/1     Running     0          27s
rook-ceph-osd-prepare-kube01-dfrp2   0/2     Completed   0          3m50s
rook-ceph-osd-prepare-kube02-pnbsc   0/2     Completed   0          3m49s
rook-ceph-osd-prepare-kube03-4ztl9   0/2     Completed   0          3m48s
```

安装ceph-tool，登录到ceph-tools的pod，可以执行ceph相关的命令，查看ceph状态

```
[root@kube01 rook]# kubectl apply -f cluster/examples/kubernetes/ceph/toolbox.yaml
deployment.apps/rook-ceph-tools created
[root@kube01 rook]# kubectl get pods -n rook-ceph
NAME                                 READY   STATUS    RESTARTS   AGE
rook-ceph-mgr-a-66db78887f-lmhcf     1/1     Running   0          5m10s
rook-ceph-mon-a-b6556df54-t7cd4      1/1     Running   0          5m50s
rook-ceph-mon-b-7f84c6d4b-nhjj6      1/1     Running   0          5m39s
rook-ceph-mon-c-868c5b476b-ghfqs     1/1     Running   0          5m29s
rook-ceph-osd-0-6fdf57bcb7-bf6f2     1/1     Running   0          111s
rook-ceph-osd-1-8c99b7447-h4mfd      1/1     Running   0          96s
rook-ceph-osd-2-66b76d6944-5df9j     1/1     Running   0          84s
rook-ceph-osd-prepare-kube01-d2rrt   1/2     Running   0          49s
rook-ceph-osd-prepare-kube02-jxl6g   1/2     Running   0          47s
rook-ceph-osd-prepare-kube03-fz276   1/2     Running   0          45s
rook-ceph-tools-544fb656d-tddrx      1/1     Running   0          3s
```
登录到ceph-tools Pod

```
[root@kube01 rook]# kubectl exec -it rook-ceph-tools-544fb656d-tddrx bash -n rook-ceph
bash: warning: setlocale: LC_CTYPE: cannot change locale (en_US.UTF-8): No such file or directory
bash: warning: setlocale: LC_COLLATE: cannot change locale (en_US.UTF-8): No such file or directory
bash: warning: setlocale: LC_MESSAGES: cannot change locale (en_US.UTF-8): No such file or directory
bash: warning: setlocale: LC_NUMERIC: cannot change locale (en_US.UTF-8): No such file or directory
bash: warning: setlocale: LC_TIME: cannot change locale (en_US.UTF-8): No such file or directory
[root@kube03 /]# ceph -s
  cluster:
    id:     f9609ec9-62c7-4462-a4f2-35c4137c25ef
    health: HEALTH_OK

  services:
    mon: 3 daemons, quorum c,a,b
    mgr: a(active)
    osd: 3 osds: 3 up, 3 in

  data:
    pools:   0 pools, 0 pgs
    objects: 0  objects, 0 B
    usage:   3.0 GiB used, 57 GiB / 60 GiB avail
    pgs:

[root@kube03 /]# ceph osd tree
ID CLASS WEIGHT  TYPE NAME       STATUS REWEIGHT PRI-AFF
-1       0.05846 root default
-7       0.01949     host kube01
 2   hdd 0.01949         osd.2       up  1.00000 1.00000
-3       0.01949     host kube02
 0   hdd 0.01949         osd.0       up  1.00000 1.00000
-5       0.01949     host kube03
 1   hdd 0.01949         osd.1       up  1.00000 1.00000
 
```
 
部署好的ceph集群并没有rgw服务，通过下面的方式可以添加rgw服务
 
```
[root@kube01 rook]# kubectl apply -f cluster/examples/kubernetes/ceph/object.yaml
cephobjectstore.ceph.rook.io/my-store created

[root@kube01 rook]# kubectl get pods -n rook-ceph
NAME                                      READY   STATUS      RESTARTS   AGE
rook-ceph-mgr-a-66db78887f-lmhcf          1/1     Running     0          11m
rook-ceph-mon-a-b6556df54-t7cd4           1/1     Running     0          11m
rook-ceph-mon-b-7f84c6d4b-nhjj6           1/1     Running     0          11m
rook-ceph-mon-c-868c5b476b-ghfqs          1/1     Running     0          11m
rook-ceph-osd-0-f986cc57d-6xclh           1/1     Running     0          6m
rook-ceph-osd-1-765556c558-jwv2n          1/1     Running     0          5m40s
rook-ceph-osd-2-766db888c7-j7z8f          1/1     Running     0          5m48s
rook-ceph-osd-prepare-kube01-d2rrt        0/2     Completed   0          6m57s
rook-ceph-osd-prepare-kube02-jxl6g        0/2     Completed   0          6m55s
rook-ceph-osd-prepare-kube03-fz276        0/2     Completed   0          6m53s
rook-ceph-rgw-my-store-5b68744bc6-pc7g7   1/1     Running     0          21s
rook-ceph-tools-544fb656d-tddrx           1/1     Running     0          6m11s

[root@kube01 rook]# kubectl exec -it rook-ceph-tools-544fb656d-tddrx bash -n rook-ceph
bash: warning: setlocale: LC_CTYPE: cannot change locale (en_US.UTF-8): No such file or directory
bash: warning: setlocale: LC_COLLATE: cannot change locale (en_US.UTF-8): No such file or directory
bash: warning: setlocale: LC_MESSAGES: cannot change locale (en_US.UTF-8): No such file or directory
bash: warning: setlocale: LC_NUMERIC: cannot change locale (en_US.UTF-8): No such file or directory
bash: warning: setlocale: LC_TIME: cannot change locale (en_US.UTF-8): No such file or directory
[root@kube03 /]# ceph -s
  cluster:
    id:     f9609ec9-62c7-4462-a4f2-35c4137c25ef
    health: HEALTH_OK

  services:
    mon: 3 daemons, quorum c,a,b
    mgr: a(active)
    osd: 3 osds: 3 up, 3 in
    rgw: 1 daemon active

  data:
    pools:   6 pools, 600 pgs
    objects: 201  objects, 3.7 KiB
    usage:   3.0 GiB used, 57 GiB / 60 GiB avail
    pgs:     600 active+clean
    
    

```
通过下面的方式可以把rgw服务以NodePort的方式对外提供服务
 
```
[root@kube01 rook]# kubectl apply -f cluster/examples/kubernetes/ceph/rgw-external.yaml
service/rook-ceph-rgw-my-store-external created
 
[root@kube01 rook]# kubectl get services -n rook-ceph
NAME                              TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)        AGE
rook-ceph-mgr                     ClusterIP   172.30.117.80    <none>        9283/TCP       12m
rook-ceph-mgr-dashboard           ClusterIP   172.30.225.42    <none>        8443/TCP       12m
rook-ceph-mon-a                   ClusterIP   172.30.82.3      <none>        6790/TCP       13m
rook-ceph-mon-b                   ClusterIP   172.30.122.28    <none>        6790/TCP       13m
rook-ceph-mon-c                   ClusterIP   172.30.62.26     <none>        6790/TCP       13m
rook-ceph-rgw-my-store            ClusterIP   172.30.107.175   <none>        80/TCP         2m47s
rook-ceph-rgw-my-store-external   NodePort    172.30.131.185   <none>        80:32185/TCP   4m2s
 
```

下面的部署安装ceph mds 

```
[root@kube01 rook]# kubectl apply -f cluster/examples/kubernetes/ceph/filesystem.yaml
cephfilesystem.ceph.rook.io/myfs created

[root@kube01 rook]# kubectl get pods -n rook-ceph
NAME                                      READY   STATUS      RESTARTS   AGE
rook-ceph-mds-myfs-a-6bbc59cbc8-fp5px     1/1     Running     0          11s
rook-ceph-mds-myfs-b-658fc8fd66-5cd9g     1/1     Running     0          11s
rook-ceph-mgr-a-66db78887f-lmhcf          1/1     Running     0          23m
rook-ceph-mon-a-b6556df54-t7cd4           1/1     Running     0          24m
rook-ceph-mon-b-7f84c6d4b-nhjj6           1/1     Running     0          24m
rook-ceph-mon-c-868c5b476b-ghfqs          1/1     Running     0          23m
rook-ceph-osd-0-f986cc57d-6xclh           1/1     Running     0          18m
rook-ceph-osd-1-765556c558-jwv2n          1/1     Running     0          17m
rook-ceph-osd-2-766db888c7-j7z8f          1/1     Running     0          18m
rook-ceph-osd-prepare-kube01-d2rrt        0/2     Completed   0          19m
rook-ceph-osd-prepare-kube02-jxl6g        0/2     Completed   0          19m
rook-ceph-osd-prepare-kube03-fz276        0/2     Completed   0          19m
rook-ceph-rgw-my-store-5b68744bc6-pc7g7   1/1     Running     0          12m
rook-ceph-tools-544fb656d-tddrx           1/1     Running     0          18m

[root@kube01 rook]# kubectl exec -it rook-ceph-tools-544fb656d-tddrx bash -n rook-ceph
bash: warning: setlocale: LC_CTYPE: cannot change locale (en_US.UTF-8): No such file or directory
bash: warning: setlocale: LC_COLLATE: cannot change locale (en_US.UTF-8): No such file or directory
bash: warning: setlocale: LC_MESSAGES: cannot change locale (en_US.UTF-8): No such file or directory
bash: warning: setlocale: LC_NUMERIC: cannot change locale (en_US.UTF-8): No such file or directory
bash: warning: setlocale: LC_TIME: cannot change locale (en_US.UTF-8): No such file or directory
[root@kube03 /]# ceph -s
  cluster:
    id:     f9609ec9-62c7-4462-a4f2-35c4137c25ef
    health: HEALTH_OK

  services:
    mon: 3 daemons, quorum c,a,b
    mgr: a(active)
    mds: myfs-1/1/1 up  {0=myfs-b=up:active}, 1 up:standby-replay
    osd: 3 osds: 3 up, 3 in
    rgw: 1 daemon active

  data:
    pools:   8 pools, 800 pgs
    objects: 224  objects, 5.9 KiB
    usage:   3.0 GiB used, 57 GiB / 60 GiB avail
    pgs:     800 active+clean

  io:
    client:   852 B/s rd, 1 op/s rd, 0 op/s wr


```

cluster/examples/kubernetes/ceph/ 目录下还有其他yaml文件，可以对ceph集群进行其他操作，比如启用mgr dashboar，安装prometheus监控等

```
-rwxr-xr-x 1 root root  8139 Mar 15 15:20 cluster.yaml
-rw-r--r-- 1 root root   363 Mar 15 14:47 dashboard-external-https.yaml
-rw-r--r-- 1 root root   362 Mar 15 14:47 dashboard-external-http.yaml
-rw-r--r-- 1 root root  1487 Mar 15 14:47 ec-filesystem.yaml
-rw-r--r-- 1 root root  1538 Mar 15 14:47 ec-storageclass.yaml
-rw-r--r-- 1 root root  1375 Mar 15 14:47 filesystem.yaml
-rw-r--r-- 1 root root  1923 Mar 15 14:48 kube-registry.yaml
drwxr-xr-x 2 root root    85 Mar 15 16:07 monitoring
-rw-r--r-- 1 root root   160 Mar 15 14:47 object-user.yaml
-rw-r--r-- 1 root root  1813 Mar 15 14:47 object.yaml
-rwxr-xr-x 1 root root 12690 Mar 15 14:48 operator.yaml
-rw-r--r-- 1 root root   742 Mar 15 14:47 pool.yaml
-rw-r--r-- 1 root root   410 Mar 15 14:47 rgw-external.yaml
-rw-r--r-- 1 root root  1216 Mar 15 14:47 scc.yaml
-rw-r--r-- 1 root root   991 Mar 15 14:47 storageclass.yaml
-rw-r--r-- 1 root root  1544 Mar 15 14:48 toolbox.yaml
-rw-r--r-- 1 root root  6492 Mar 15 14:47 upgrade-from-v0.8-create.yaml
-rw-r--r-- 1 root root   874 Mar 15 14:47 upgrade-from-v0.8-replace.yaml
```

# 参考
* [http://www.yangguanjun.com/2018/12/22/rook-ceph-practice-part1/](http://www.yangguanjun.com/2018/12/22/rook-ceph-practice-part1/)
* [http://www.yangguanjun.com/2018/12/28/rook-ceph-practice-part2/](http://www.yangguanjun.com/2018/12/28/rook-ceph-practice-part2/)
 
