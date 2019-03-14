---
title: Ceph对象存储使用纠删码存储池
date: 2019-03-13 15:05:34
tags: ceph
---

# 概述
本文主要验证ceph对象存储使用纠删码的情况  
本文中纠删码的配置K+M为 4+2，理论上可以容忍M个osd的故障ceph


# 配置

创建erasure-code-profile和crush rule

```
[root@ceph04 ~]# ceph osd erasure-code-profile set rgw_ec_profile k=4 m=2 crush-root=root_rgw plugin=isa crush-failure-domain=host
[root@ceph04 ~]# ceph osd erasure-code-profile get rgw_ec_profile
crush-device-class=
crush-failure-domain=host
crush-root=root_rgw
k=4
m=2
plugin=isa
technique=reed_sol_van
```

```
[root@ceph04 ~]# ceph osd crush rule create-erasure rgw_ec_rule rgw_ec_profile
created rule rgw_ec_rule at 2
[root@ceph04 ~]# ceph osd crush rule dump
[
    {
        "rule_id": 0,
        "rule_name": "replicated_rule",
        "ruleset": 0,
        "type": 1,
        "min_size": 1,
        "max_size": 10,
        "steps": [
            {
                "op": "take",
                "item": -1,
                "item_name": "default"
            },
            {
                "op": "chooseleaf_firstn",
                "num": 0,
                "type": "host"
            },
            {
                "op": "emit"
            }
        ]
    },
    {
        "rule_id": 1,
        "rule_name": "rule_rgw",
        "ruleset": 1,
        "type": 1,
        "min_size": 1,
        "max_size": 10,
        "steps": [
            {
                "op": "take",
                "item": -13,
                "item_name": "root_rgw"
            },
            {
                "op": "chooseleaf_firstn",
                "num": 0,
                "type": "host"
            },
            {
                "op": "emit"
            }
        ]
    },
    {
        "rule_id": 2,
        "rule_name": "rgw_ec_rule",
        "ruleset": 2,
        "type": 3,
        "min_size": 3,
        "max_size": 6,
        "steps": [
            {
                "op": "set_chooseleaf_tries",
                "num": 5
            },
            {
                "op": "set_choose_tries",
                "num": 100
            },
            {
                "op": "take",
                "item": -13,
                "item_name": "root_rgw"
            },
            {
                "op": "chooseleaf_indep",
                "num": 0,
                "type": "host"
            },
            {
                "op": "emit"
            }
        ]
    }
]
```
由于实验环境只有3个节点，需要调整crush rule，先选择3个host，再在每个host选择两个osd

```
ceph osd getcrushmap -o crushmap

crushtool -d crushmap -o crushmap.txt
```

```
rule rgw_ec_rule {
        id 2
        type erasure
        min_size 3
        max_size 6
        step set_chooseleaf_tries 5
        step set_choose_tries 100
        step take root_rgw
        step choose indep 3 type host
        step choose indep 2 type osd
        step emit
}
```

```
crushtool -c crushmap.txt -o crushmap
ceph osd setcrushmap -i crushmap
```

由于环境中还没有任何数据，我们先停止rgw，然后把默认的default.rgw.buckets.data存储池删掉，再创建一个纠删码的default.rgw.buckets.data存储池

```
[root@ceph04 ~]# ceph osd pool create default.rgw.buckets.data 64 64 erasure rgw_ec_profile rgw_ec_rule
pool 'default.rgw.buckets.data' created

[root@ceph04 ~]# ceph osd pool application enable default.rgw.buckets.data rgw
enabled application 'rgw' on pool 'default.rgw.buckets.data'

[root@ceph04 ~]# ceph -s
  cluster:
    id:     57df615e-6dec-4f69-84f0-72a2ba76d4d7
    health: HEALTH_WARN
            noout flag(s) set
            too few PGs per OSD (21 < min 30)
            clock skew detected on mon.ceph04

  services:
    mon: 3 daemons, quorum ceph06,ceph05,ceph04
    mgr: ceph04(active), standbys: ceph06, ceph05
    osd: 18 osds: 18 up, 18 in
         flags noout

  data:
    pools:   1 pools, 64 pgs
    objects: 0 objects, 0 bytes
    usage:   19351 MB used, 339 GB / 358 GB avail
    pgs:     64 active+clean

```

可以看到默认创建的存储池的size是k+m=6, min_size=k-m+1=5, 当存储池的当前size小于min_size的时候，pg会出现incomplete的情况，所以在还需要调整存储池的min_size为4，这样就可以容忍2个osd节点故障。

```
[root@ceph04 ~]# ceph osd pool ls detail
pool 33 'default.rgw.buckets.data' erasure size 6 min_size 5 crush_rule 2 object_hash rjenkins pg_num 64 pgp_num 64 last_change 466 flags hashpspool stripe_width 16384 application rgw
pool 34 '.rgw.root' replicated size 3 min_size 2 crush_rule 0 object_hash rjenkins pg_num 8 pgp_num 8 last_change 405 flags hashpspool stripe_width 0 application rgw
pool 35 'default.rgw.meta' replicated size 3 min_size 2 crush_rule 0 object_hash rjenkins pg_num 8 pgp_num 8 last_change 407 flags hashpspool stripe_width 0 application rgw
pool 36 'default.rgw.log' replicated size 3 min_size 2 crush_rule 0 object_hash rjenkins pg_num 8 pgp_num 8 last_change 409 flags hashpspool stripe_width 0 application rgw
pool 37 'default.rgw.buckets.index' replicated size 3 min_size 2 crush_rule 0 object_hash rjenkins pg_num 8 pgp_num 8 last_change 412 flags hashpspool stripe_width 0 application rgw


[root@ceph04 ~]# ceph osd pool set default.rgw.buckets.data min_size 4
set pool 33 min_size to 4
[root@ceph04 ~]# ceph osd pool ls detail
pool 33 'default.rgw.buckets.data' erasure size 6 min_size 4 crush_rule 2 object_hash rjenkins pg_num 64 pgp_num 64 last_change 483 flags hashpspool stripe_width 16384 application rgw
pool 34 '.rgw.root' replicated size 3 min_size 2 crush_rule 0 object_hash rjenkins pg_num 8 pgp_num 8 last_change 405 flags hashpspool stripe_width 0 application rgw
pool 35 'default.rgw.meta' replicated size 3 min_size 2 crush_rule 0 object_hash rjenkins pg_num 8 pgp_num 8 last_change 407 flags hashpspool stripe_width 0 application rgw
pool 36 'default.rgw.log' replicated size 3 min_size 2 crush_rule 0 object_hash rjenkins pg_num 8 pgp_num 8 last_change 409 flags hashpspool stripe_width 0 application rgw
pool 37 'default.rgw.buckets.index' replicated size 3 min_size 2 crush_rule 0 object_hash rjenkins pg_num 8 pgp_num 8 last_change 412 flags hashpspool stripe_width 0 application rgw

```

# 验证

创建对象存储用户，并用s3cmd进行验证

```
[root@ceph04 ~]# radosgw-admin user create --uid=test --display-name=test
{
    "user_id": "test",
    "display_name": "test",
    "email": "",
    "suspended": 0,
    "max_buckets": 1000,
    "auid": 0,
    "subusers": [],
    "keys": [
        {
            "user": "test",
            "access_key": "DE8EBP0W6WSO9SGYGV66",
            "secret_key": "AG6ufkpWuO4pUtJLOKKimfZNvVmwVsMkeXZmmBgi"
        }
    ],
    "swift_keys": [],
    "caps": [],
    "op_mask": "read, write, delete",
    "default_placement": "",
    "placement_tags": [],
    "bucket_quota": {
        "enabled": false,
        "check_on_raw": false,
        "max_size": -1,
        "max_size_kb": 0,
        "max_objects": -1
    },
    "user_quota": {
        "enabled": false,
        "check_on_raw": false,
        "max_size": -1,
        "max_size_kb": 0,
        "max_objects": -1
    },
    "temp_url_keys": [],
    "type": "rgw"
}
```

```
[root@ceph04 ~]# s3cmd ls s3://
[root@ceph04 ~]# s3cmd mb s3://test
Bucket 's3://test/' created
```

停掉2个osd，pg也没有出现incomplete的状态, 通过s3cmd也可以正常上传下载

```
[root@ceph04 ~]# systemctl stop ceph-osd@0
[root@ceph04 ~]# systemctl stop ceph-osd@1
[root@ceph04 ~]# ceph -s
  cluster:
    id:     57df615e-6dec-4f69-84f0-72a2ba76d4d7
    health: HEALTH_WARN
            noout flag(s) set
            2 osds down
            Degraded data redundancy: 471/3570 objects degraded (13.193%), 12 pgs degraded
            too few PGs per OSD (26 < min 30)
            clock skew detected on mon.ceph05, mon.ceph04

  services:
    mon: 3 daemons, quorum ceph06,ceph05,ceph04
    mgr: ceph04(active), standbys: ceph06, ceph05
    osd: 18 osds: 16 up, 18 in
         flags noout
    rgw: 1 daemon active

  data:
    pools:   5 pools, 96 pgs
    objects: 1184 objects, 15056 kB
    usage:   19554 MB used, 339 GB / 358 GB avail
    pgs:     471/3570 objects degraded (13.193%)
             43 active+undersized
             41 active+clean
             12 active+undersized+degraded

  io:
    client:   170 B/s rd, 0 B/s wr, 0 op/s rd, 0 op/s wr
```


# 参考
* [http://www.zphj1987.com/2018/06/12/ceph-erasure-default-min-size/](http://www.zphj1987.com/2018/06/12/ceph-erasure-default-min-size/)
