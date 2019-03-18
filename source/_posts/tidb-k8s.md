---
title: tidb-operator
date: 2019-03-18 11:22:15
tags: ['k8s']	
---

# 概述
tidb是一个分布式的数据库，tidb-operator可以让tidb跑在k8s集群上面。  
本文主要验证tidb使用local pv方式部署在k8s集群上面。

# 安装

实验环境的k8s环境共有三个节点

```
[root@kube01 ~]# kubectl get nodes
NAME     STATUS   ROLES    AGE    VERSION
kube01   Ready    master   3d5h   v1.13.4
kube02   Ready    master   3d5h   v1.13.4
kube03   Ready    master   3d5h   v1.13.4

```


tidb安装过程中需要用到6个pv，pd需要用到3个1G的pv，tikv需要用到3个10G的pv。  
环境中有一块20G的空闲磁盘vdc，我们把vdc分成两个分区，一个2G，一个18G。

```
[root@kube01 ~]# sgdisk -n 1:0:+2G /dev/vdc
Creating new GPT entries.
The operation has completed successfully.
[root@kube01 ~]# sgdisk -n 2:0:0 /dev/vdc
The operation has completed successfully.

```

tidb-operator默认会把挂载在/mnt/disks/vol$i 的分区作为一个PV，所以把我们vdc1和vdc2这两个分区挂载在vol0和vol1下。  
tidb推荐使用ext4文件系统

```
mkfs.ext4 /dev/vdc1
mkfs.ext4 /dev/vdc2
mount /dev/vdc1 /mnt/disks/vol0/
mount /dev/vdc1 /mnt/disks/vol1/
```

通过tidb-opertor提供的local-volumen-provisioner可以把前面的分区变成local pv

```
[root@kube01 local-dind]# kubectl apply -f local-volume-provisioner.yaml
storageclass.storage.k8s.io/local-storage changed
configmap/local-provisioner-config changed
daemonset.extensions/local-volume-provisioner changed
serviceaccount/local-storage-admin changed
clusterrolebinding.rbac.authorization.k8s.io/local-storage-provisioner-pv-binding changed
clusterrole.rbac.authorization.k8s.io/local-storage-provisioner-node-clusterrole changed
clusterrolebinding.rbac.authorization.k8s.io/local-storage-provisioner-node-binding changed

```

通过helm安装tidb-operator

需要修改charts/tidb-operator/values.yaml文件中  
kubeSchedulerImage: gcr.io/google-containers/hyperkube:v1.13.4  
镜像的版本和kubelet的版本一致。

```

[root@kube01 manifests]# kubectl apply -f crd.yaml
customresourcedefinition.apiextensions.k8s.io/tidbclusters.pingcap.com created

[root@kube01 tidb-operator]# kubectl get customresourcedefinitions
NAME                                CREATED AT
tidbclusters.pingcap.com            2019-03-18T05:53:22Z


[root@kube01 tidb-operator]# helm install charts/tidb-operator --name=tidb-operator --namespace=tidb-admin
NAME:   tidb-operator
LAST DEPLOYED: Mon Mar 18 14:46:32 2019
NAMESPACE: tidb-admin
STATUS: DEPLOYED

RESOURCES:
==> v1/ConfigMap
NAME                   DATA  AGE
tidb-scheduler-policy  1     0s

==> v1/Pod(related)
NAME                                      READY  STATUS             RESTARTS  AGE
tidb-controller-manager-7c56fb85dd-h6k5m  0/1    ContainerCreating  0         0s
tidb-scheduler-7f8b69d57b-xx8jq           0/2    ContainerCreating  0         0s

==> v1/ServiceAccount
NAME                     SECRETS  AGE
tidb-controller-manager  1        0s
tidb-scheduler           1        0s

==> v1beta1/ClusterRole
NAME                                   AGE
tidb-operator:tidb-controller-manager  0s
tidb-operator:tidb-scheduler           0s

==> v1beta1/ClusterRoleBinding
NAME                                   AGE
tidb-operator:tidb-controller-manager  0s
tidb-operator:tidb-scheduler           0s

==> v1beta1/Deployment
NAME                     READY  UP-TO-DATE  AVAILABLE  AGE
tidb-controller-manager  0/1    1           0          0s
tidb-scheduler           0/1    1           0          0s


NOTES:
1. Make sure tidb-operator components are running
   kubectl get pods --namespace tidb-admin -l app.kubernetes.io/instance=tidb-operator
2. Install CRD
   kubectl apply -f https://raw.githubusercontent.com/pingcap/tidb-operator/master/manifests/crd.yaml
   kubectl get customresourcedefinitions
3. Modify tidb-cluster/values.yaml and create a TiDB cluster by installing tidb-cluster charts
   helm install tidb-cluster


[root@kube01 tidb-operator]# kubectl get pods --namespace tidb-admin -l app.kubernetes.io/instance=tidb-operator
NAME                                       READY   STATUS    RESTARTS   AGE
tidb-controller-manager-7c56fb85dd-h6k5m   1/1     Running   0          5m11s
tidb-scheduler-7f8b69d57b-xx8jq            2/2     Running   0          5m11s
```

通过helm安装tidb-cluster

```

[root@kube01 tidb-operator]# helm install charts/tidb-cluster --name=tidb-cluster --namespace=tidb
NAME:   tidb-cluster
LAST DEPLOYED: Mon Mar 18 14:57:35 2019
NAMESPACE: tidb
STATUS: DEPLOYED

RESOURCES:
==> v1/ConfigMap
NAME          DATA  AGE
demo-monitor  3     0s
demo-pd       2     0s
demo-tidb     2     0s
demo-tikv     2     0s

==> v1/Job
NAME                       COMPLETIONS  DURATION  AGE
demo-monitor-configurator  0/1          0s        0s
demo-tidb-initializer      0/1          0s        0s

==> v1/Pod(related)
NAME                             READY  STATUS             RESTARTS  AGE
demo-discovery-5468c7c556-8dcs5  0/1    ContainerCreating  0         0s
demo-monitor-84446b7957-rbdl9    0/2    Init:0/1           0         0s
demo-monitor-configurator-vz25r  0/1    ContainerCreating  0         0s
demo-tidb-initializer-dh8v2      0/1    ContainerCreating  0         0s

==> v1/Secret
NAME          TYPE    DATA  AGE
demo-monitor  Opaque  2     0s
demo-tidb     Opaque  1     0s

==> v1/Service
NAME             TYPE       CLUSTER-IP      EXTERNAL-IP  PORT(S)                         AGE
demo-discovery   ClusterIP  172.30.201.100  <none>       10261/TCP                       0s
demo-grafana     NodePort   172.30.122.104  <none>       3000:32242/TCP                  0s
demo-prometheus  NodePort   172.30.59.221   <none>       9090:32099/TCP                  0s
demo-tidb        NodePort   172.30.181.223  <none>       4000:30344/TCP,10080:32651/TCP  0s

==> v1/ServiceAccount
NAME            SECRETS  AGE
demo-discovery  1        0s
demo-monitor    1        0s

==> v1alpha1/TidbCluster
NAME  AGE
demo  0s


==> v1beta1/Deployment
NAME            READY  UP-TO-DATE  AVAILABLE  AGE
demo-discovery  0/1    1           0          0s
demo-monitor    0/1    1           0          0s

==> v1beta1/Role
NAME            AGE
demo-discovery  0s
demo-monitor    0s

==> v1beta1/RoleBinding
NAME            AGE
demo-discovery  0s
demo-monitor    0s


NOTES:
1. Watch tidb-cluster up and running
   watch kubectl get pods --namespace tidb -l app.kubernetes.io/instance=tidb-cluster -o wide
2. List services in the tidb-cluster
   kubectl get services --namespace tidb -l app.kubernetes.io/instance=tidb-cluster
3. Wait until tidb-initializer pod becomes completed
   watch kubectl get po --namespace tidb  -l app.kubernetes.io/component=tidb-initializer
4. Get the TiDB password
   PASSWORD=$(kubectl get secret -n tidb demo-tidb -o jsonpath="{.data.password}" | base64 --decode | awk '{print $6}')
   echo ${PASSWORD}
5. Access tidb-cluster using the MySQL client
   kubectl port-forward -n tidb svc/demo-tidb 4000:4000 &
   mysql -h 127.0.0.1 -P 4000 -u root -D test -p
6. View monitor dashboard for TiDB cluster
   kubectl port-forward -n tidb svc/demo-grafana 3000:3000
   Open browser at http://localhost:3000. The default username and password is admin/admin.


[root@kube01 tidb-operator]# kubectl get tidbcluster -n tidb
NAME   AGE
demo   1m

[root@kube01 tidb-operator]# kubectl get statefulset -n tidb
NAME        READY   AGE
demo-pd     3/3     5m41s
demo-tidb   2/2     3m16s
demo-tikv   3/3     5m

[root@kube01 tidb-operator]# kubectl get service -n tidb
NAME              TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                          AGE
demo-discovery    ClusterIP   172.30.25.248    <none>        10261/TCP                        6m
demo-grafana      NodePort    172.30.103.60    <none>        3000:31597/TCP                   6m
demo-pd           ClusterIP   172.30.122.100   <none>        2379/TCP                         6m
demo-pd-peer      ClusterIP   None             <none>        2380/TCP                         6m
demo-prometheus   NodePort    172.30.147.201   <none>        9090:30429/TCP                   6m
demo-tidb         NodePort    172.30.244.61    <none>        4000:30512/TCP,10080:31051/TCP   6m
demo-tidb-peer    ClusterIP   None             <none>        10080/TCP                        3m34s
demo-tikv-peer    ClusterIP   None             <none>        20160/TCP                        5m18s


[root@kube01 tidb-operator]# kubectl get configmap -n tidb
NAME           DATA   AGE
demo-monitor   3      6m20s
demo-pd        2      6m20s
demo-tidb      2      6m20s
demo-tikv      2      6m20s


[root@kube01 tidb-operator]# kubectl get pod -n tidb
NAME                              READY   STATUS      RESTARTS   AGE
demo-discovery-5468c7c556-l9xl7   1/1     Running     0          6m41s
demo-monitor-84446b7957-4zsnd     2/2     Running     0          6m41s
demo-monitor-configurator-rnklq   0/1     Completed   0          6m41s
demo-pd-0                         1/1     Running     0          6m40s
demo-pd-1                         1/1     Running     0          6m40s
demo-pd-2                         1/1     Running     1          6m40s
demo-tidb-0                       1/1     Running     0          4m15s
demo-tidb-1                       1/1     Running     0          4m15s
demo-tidb-initializer-nkng4       0/1     Completed   0          6m41s
demo-tikv-0                       2/2     Running     0          5m59s
demo-tikv-1                       2/2     Running     0          5m59s
demo-tikv-2                       2/2     Running     0          5m59s


```

通过mysql client进行验证

```
[root@kube01 tidb-operator]# kubectl port-forward svc/demo-tidb 4000:4000 --namespace=tidb
Forwarding from 127.0.0.1:4000 -> 4000


[root@kube01 ~]# PASSWORD=$(kubectl get secret -n tidb demo-tidb -ojsonpath="{.data.password}" | base64 --decode | awk '{print $6}')
[root@kube01 ~]# echo ${PASSWORD}
'IwDSSjpq89'
[root@kube01 ~]# mysql -h 127.0.0.1 -P 4000 -u root -p
Enter password:
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MySQL connection id is 3
Server version: 5.7.10-TiDB-v2.1.4 MySQL Community Server (Apache License 2.0)

Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

MySQL [(none)]> show databases;
+--------------------+
| Database           |
+--------------------+
| INFORMATION_SCHEMA |
| PERFORMANCE_SCHEMA |
| mysql              |
| test               |
+--------------------+
4 rows in set (0.00 sec)

MySQL [(none)]>

```

# 参考

* [https://github.com/pingcap/tidb-operator/blob/master/docs/local-dind-tutorial.md](https://github.com/pingcap/tidb-operator/blob/master/docs/local-dind-tutorial.md)
* [https://github.com/pingcap/tidb-operator/blob/master/docs/setup.md](https://github.com/pingcap/tidb-operator/blob/master/docs/setup.md)