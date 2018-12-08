---
title: 通过kubeadm搭建单节点k8s环境
date: 2018-12-08 15:15:00
tags: k8s
---


本次实验，通过kubeadm来安装一个单节点的k8s环境。

本次实验是在虚拟机上进行，虚拟机的配置如下：

| OS | CPU | 内存 | IP |
| ----- | ----- | ----- | ----- |
| CentOS Linux release 7.6.1810 (Core) | 2 vCPU | 2G | 172.16.143.171 |

# 环境准备

安装docker  

```
[kube@kube ~]$ yum update
[kube@kube ~]$ yum install docker -y
[kube@kube ~]$ systemctl start docker
[kube@kube ~]$ systemctl enable docker

[kube@kube ~]$ docker --version
Docker version 1.13.1, build 07f3374/1.13.1

```


关闭Swap

```
swapoff -a

```

# 安装k8s

配置kubernetes阿里云源  

```
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
exclude=kube*
EOF

```

关闭selinux  

```
setenforce 0
sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config
```
安装kubeadm, kubelet和kubectl

```
yum install -y kubelet kubeadm kubectl --disableexcludes=kubernetes

systemctl enable kubelet && systemctl start kubelet
```

在centos系统上设置iptables  

```
cat <<EOF >  /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF

sysctl --system
```

kubeadm默认会从k8s.gcr.io上下载kube的images，但是在国内环境是访问不了这些镜像的，所以可以从aliyun的registry上下载相应的image，然后修改tag，瞒过kubeadm。  

先通过下面的命令查看当前需要哪些image

```
[root@kube ~]# kubeadm config images list
I1208 16:30:01.471171   30018 version.go:94] could not fetch a Kubernetes version from the internet: unable to get URL "https://dl.k8s.io/release/stable-1.txt": Get https://storage.googleapis.com/kubernetes-release/release/stable-1.txt: net/http: request canceled while waiting for connection (Client.Timeout exceeded while awaiting headers)
I1208 16:30:01.471304   30018 version.go:95] falling back to the local client version: v1.13.0
k8s.gcr.io/kube-apiserver:v1.13.0
k8s.gcr.io/kube-controller-manager:v1.13.0
k8s.gcr.io/kube-scheduler:v1.13.0
k8s.gcr.io/kube-proxy:v1.13.0
k8s.gcr.io/pause:3.1
k8s.gcr.io/etcd:3.2.24
k8s.gcr.io/coredns:1.2.6

```
通过下面的方式可以从阿里云的registry上下载镜像并修改tag

```
images=(
    kube-apiserver:v1.13.0
    kube-controller-manager:v1.13.0
    kube-scheduler:v1.13.0
    kube-proxy:v1.13.0
    pause:3.1
    etcd:3.2.24
    coredns:1.2.6
)

for imageName in ${images[@]} ; do
    docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/$imageName
    docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/$imageName k8s.gcr.io/$imageName
done
```

初始化集群

```
[root@kube ~]# kubeadm init
I1208 16:38:21.289892   31021 version.go:94] could not fetch a Kubernetes version from the internet: unable to get URL "https://dl.k8s.io/release/stable-1.txt": Get https://storage.googleapis.com/kubernetes-release/release/stable-1.txt: net/http: request canceled while waiting for connection (Client.Timeout exceeded while awaiting headers)
I1208 16:38:21.289971   31021 version.go:95] falling back to the local client version: v1.13.0
[init] Using Kubernetes version: v1.13.0
[preflight] Running pre-flight checks
[preflight] Pulling images required for setting up a Kubernetes cluster
[preflight] This might take a minute or two, depending on the speed of your internet connection
[preflight] You can also perform this action in beforehand using 'kubeadm config images pull'
[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet-start] Activating the kubelet service
[certs] Using certificateDir folder "/etc/kubernetes/pki"
[certs] Generating "front-proxy-ca" certificate and key
[certs] Generating "front-proxy-client" certificate and key
[certs] Generating "etcd/ca" certificate and key
[certs] Generating "etcd/peer" certificate and key
[certs] etcd/peer serving cert is signed for DNS names [kube localhost] and IPs [172.16.143.171 127.0.0.1 ::1]
[certs] Generating "apiserver-etcd-client" certificate and key
[certs] Generating "etcd/healthcheck-client" certificate and key
[certs] Generating "etcd/server" certificate and key
[certs] etcd/server serving cert is signed for DNS names [kube localhost] and IPs [172.16.143.171 127.0.0.1 ::1]
[certs] Generating "ca" certificate and key
[certs] Generating "apiserver" certificate and key
[certs] apiserver serving cert is signed for DNS names [kube kubernetes kubernetes.default kubernetes.default.svc kubernetes.default.svc.cluster.local] and IPs [10.96.0.1 172.16.143.171]
[certs] Generating "apiserver-kubelet-client" certificate and key
[certs] Generating "sa" key and public key
[kubeconfig] Using kubeconfig folder "/etc/kubernetes"
[kubeconfig] Writing "admin.conf" kubeconfig file
[kubeconfig] Writing "kubelet.conf" kubeconfig file
[kubeconfig] Writing "controller-manager.conf" kubeconfig file
[kubeconfig] Writing "scheduler.conf" kubeconfig file
[control-plane] Using manifest folder "/etc/kubernetes/manifests"
[control-plane] Creating static Pod manifest for "kube-apiserver"
[control-plane] Creating static Pod manifest for "kube-controller-manager"
[control-plane] Creating static Pod manifest for "kube-scheduler"
[etcd] Creating static Pod manifest for local etcd in "/etc/kubernetes/manifests"
[wait-control-plane] Waiting for the kubelet to boot up the control plane as static Pods from directory "/etc/kubernetes/manifests". This can take up to 4m0s
[apiclient] All control plane components are healthy after 24.003474 seconds
[uploadconfig] storing the configuration used in ConfigMap "kubeadm-config" in the "kube-system" Namespace
[kubelet] Creating a ConfigMap "kubelet-config-1.13" in namespace kube-system with the configuration for the kubelets in the cluster
[patchnode] Uploading the CRI Socket information "/var/run/dockershim.sock" to the Node API object "kube" as an annotation
[mark-control-plane] Marking the node kube as control-plane by adding the label "node-role.kubernetes.io/master=''"
[mark-control-plane] Marking the node kube as control-plane by adding the taints [node-role.kubernetes.io/master:NoSchedule]
[bootstrap-token] Using token: 7ywghw.pbq0fkwpz3c5jozk
[bootstrap-token] Configuring bootstrap tokens, cluster-info ConfigMap, RBAC Roles
[bootstraptoken] configured RBAC rules to allow Node Bootstrap tokens to post CSRs in order for nodes to get long term certificate credentials
[bootstraptoken] configured RBAC rules to allow the csrapprover controller automatically approve CSRs from a Node Bootstrap Token
[bootstraptoken] configured RBAC rules to allow certificate rotation for all node client certificates in the cluster
[bootstraptoken] creating the "cluster-info" ConfigMap in the "kube-public" namespace
[addons] Applied essential addon: CoreDNS
[addons] Applied essential addon: kube-proxy

Your Kubernetes master has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

You can now join any number of machines by running the following on each node
as root:

  kubeadm join 172.16.143.171:6443 --token 7ywghw.pbq0fkwpz3c5jozk --discovery-token-ca-cert-hash sha256:b30445a8098e1e025ce703e7c1bae82567e0f892039489630d39608e77a41b89
```

从上面的结果可以看出k8s master已经初始化成功，k8s推进使用非root用户使用集群，所以下面我们创建一个kube的用户，并配置sudo权限。

```
useradd kube
passwd kube
```

通过visudo给kube用户配置sudo权限  
```
kube    ALL=(ALL)       ALL
```

下面的步骤把k8s的配置拷贝到用户的.kube目录下

```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

要使用k8s集群，还需要安装网络插件，k8s支持很多网络插件，比如calico，flannel，weave等，下面我们就安装weave网络插件。

配置Weave Net  
```
kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')"
```

默认情况下，master node是不会运行容器的，由于本次实验只有一个节点，所以需要设置master node运行容器。

允许Master Node 运行容器  
```
[kube@kube ~]$ kubectl taint nodes --all node-role.kubernetes.io/master-
node/kube untainted
```

这样一个简单的k8s集群就算搭建完成了，通过下面的命令可以看到当前集群中的节点，当前集群中运行的pod。

```
[kube@kube ~]$ kubectl get node
NAME   STATUS   ROLES    AGE   VERSION
kube   Ready    master   41m   v1.13.0
[kube@kube ~]$ kubectl get pods --all-namespaces
NAMESPACE     NAME                           READY   STATUS    RESTARTS   AGE
kube-system   coredns-86c58d9df4-89xn5       1/1     Running   0          41m
kube-system   coredns-86c58d9df4-mb96l       1/1     Running   0          41m
kube-system   etcd-kube                      1/1     Running   2          40m
kube-system   kube-apiserver-kube            1/1     Running   2          40m
kube-system   kube-controller-manager-kube   1/1     Running   1          40m
kube-system   kube-proxy-tgk25               1/1     Running   0          41m
kube-system   kube-scheduler-kube            1/1     Running   1          40m
kube-system   weave-net-csgmh                2/2     Running   0          27m
```

# 参考
* [https://kubernetes.io/docs/setup/independent/install-kubeadm/](https://kubernetes.io/docs/setup/independent/install-kubeadm/)
* [https://zhuanlan.zhihu.com/p/46341911](https://zhuanlan.zhihu.com/p/46341911) 