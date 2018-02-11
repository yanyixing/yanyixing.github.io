---
title: ceph-ansible
date: 2018-02-06 23:13:09
tags:
---

通过本文了解如果通过ceph-ansible来安装ceph

前提条件

- 配置免密
- 关闭防火墙
- 关闭selinux

安装ansible

    pip install ansible==2.3.1.0
    
    mkdir /etc/ansible
    
    touch /etc/ansible/hosts

下载ceph-ansible

    git clone https://github.com/ceph/ceph-ansible.git
    
    cd ceph-ansible
    
    git checkout -b stable-3.0 origin/stable-3.0

修改配置

规划好要部署的服务器及组件

例如，需要在ceph0节点上部署mon，mgr和osd，则在/etc/ansible/hosts文件中添加如下内容

    [mons]
    ceph0
    
    [osds]
    ceph0
    
    [mgrs]
    ceph0
    

修改ceph-ansible配置

    cd ceph-ansible/group_vars
    cp all.yml.sample all.yml
    cp mons.yml.sample mons.yml
    cp mgrs.yml.sample mgrs.yml
    cp osds.yml.sample osds.yml
    

把all.yml文件改成如下内容

    dummy:
    ceph_origin: repository
    ceph_repository: community
    ceph_mirror: http://mirrors.ustc.edu.cn/ceph
    ceph_stable_key: http://mirrors.ustc.edu.cn/ceph/keys/release.asc
    ceph_stable_release: luminous
    ceph_stable_redhat_distro: el7
    monitor_interface: ens160
    monitor_address: 0.0.0.0
    public_network: 123.59.153.0/24
    cluster_network: 10.10.20.0/24
    osd_objectstore: bluestore
    handler_health_osd_check: false

具体配置含义及其他一些配置内容可以参考相应的all.yml.sample

monitor_interface, public_network 和 cluster_network 根据实际情况调整

osd_objectstore可以选择bluestore和filestore

假设ceph0节点上有3块硬盘（sdb,sdc,sddd）用户部署OSD

把osds.yml 改成如下内容

    ---
    dummy:
    devices:
      - /dev/sdb
      - /dev/sdc
      - /dev/sdd
    osd_scenario: collocated

osds.yml.sample中定了多种osd部署方式，可以根据需要灵活调整。

上面这个例子中，会把一块硬盘分区两个分区，一个分区100M，一个占满磁盘的剩余空间。

100M的分区会作为ceph block, ceph block.db, ceph block.wal, 另一个分区作为 ceph data。

部署ceph

    cd ceph-ansible
    cp site.yml.sample site.yml
    
    ansible-playbook site.yml 
    

修改crush rule

部署完成后默认的crush rule的故障域是host，如果ceph是部署在一个server上，在创建多副本pool的情况下会存在问题。

通过修改crush rule的故障域为osd，可以解决这个问题。

    ceph osd getcrushmap -o crushmap
    
    crushtool -d crushmap -o crushmap.txt
    

修改crushmap.txt 文件中内容如下，把故障域从host改成osd

    rule replicated_rule {
            id 0
            type replicated
            min_size 1
            max_size 10
            step take default
            step chooseleaf firstn 0 type osd
            step emit
    }
    

导入新的crushmap

    crushtool -c crushmap.txt -o crushmap
    ceph osd setcrushmap -i crushmap
    

创建POOL

根据需要创建POOL

    ceph osd pool create testpool 32
    

清理ceph

    cd ceph-ansible
    
    cp infrastructure-playbooks/purge-cluster.yml .
    
    ansible-playbook purge-cluster.yml


