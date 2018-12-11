---
title: Ceph mgr启动restful插件
date: 2018-12-10 17:03:33
tags: ceph
---

# 概述
本文主要介绍如何开启ceph mgr restful插件，并通过这个restful接口获取ceph的数据。

环境信息如下:

```
[root@ceph11 ~]# ceph -s
  cluster:
    id:     f8b6141c-5039-464d-be1e-61816208a006
    health: HEALTH_OK

  services:
    mon:     3 daemons, quorum ceph12,ceph13,ceph14
    mgr:     ceph14(active), standbys: ceph12, ceph13
    mds:     cephfs-1/1/1 up  {0=umstor12=up:active}
    osd:     7 osds: 7 up, 7 in
    rgw:     3 daemons active
    rgw-nfs: 3 daemons active

  data:
    pools:   9 pools, 762 pgs
    objects: 8835 objects, 5352 MB
    usage:   24008 MB used, 6496 GB / 6519 GB avail
    pgs:     762 active+clean

  io:
    client:   127 B/s wr, 0 op/s rd, 1 op/s wr
    recovery: 71 B/s, 0 objects/s
    
```

# 启动插件

```
ceph mgr module enable restful

```

发现restful服务并没有启动，8003端口没有监听，要启动restful服务，还需要配置SSL cetificate(证书)。

下面的命令生产自签名证书：

```
ceph restful create-self-signed-cert
```

这个时候可以查看在active的mgr节点(ceph14)上，restful服务已经启动

```
[root@ceph14 ~]# netstat -nltp | grep 8003
tcp6       0      0 :::8003                 :::*                    LISTEN      3551433/ceph-mgr

```

默认情况下，当前active的ceph-mgr daemon将绑定主机上任何可用的IPv4或IPv6地址的8003端口

## 指定IP和PORT

```
ceph config-key set mgr/restful/server_addr $IP
ceph config-key set mgr/restful/server_port $PORT
```

如果没有配置IP，则restful将会监听全部ip  
如果没有配置Port，则restful将会监听在8003端口

上面的配置是针对全部mgr的，如果要针对某个mgr的配置，需要在配置中指定相应的mgr的hostname

```
ceph config-key set mgr/restful/$name/server_addr $IP
ceph config-key set mgr/restful/$name/server_port $PORT
```

## 创建用户

```
[root@ceph14 ~]# ceph restful create-key admin01
144276ee-1fdc-48ca-a358-0fb59bbb689f
```
后面的访问restful接口需要用到这个用户和密码

## 验证
启动restful插件后，可以通过浏览器进行访问并验证。

```
https://192.168.180.138:8003/

{
    "api_version": 1,
    "auth": "Use \"ceph restful create-key <key>\" to create a key pair, pass it as HTTP Basic auth to authenticate",
    "doc": "See /doc endpoint",
    "info": "Ceph Manager RESTful API server"
}
```

获取全部存储池的信息  

```
https://192.168.180.138:8003/pool

[
    {
        "application_metadata": {
            "rgw": {}
        },
        "auid": 0,
        "cache_min_evict_age": 0,
        "cache_min_flush_age": 0,
        "cache_mode": "none",
        "cache_target_dirty_high_ratio_micro": 600000,
        "cache_target_dirty_ratio_micro": 400000,
        "cache_target_full_ratio_micro": 800000,
        "crash_replay_interval": 0,
        "crush_rule": 1,
        "erasure_code_profile": "",
        "expected_num_objects": 0,
        "fast_read": false,
        "flags": 1,
        "flags_names": "hashpspool",
        "grade_table": [],
        "hit_set_count": 0,
        "hit_set_grade_decay_rate": 0,
        "hit_set_params": {
            "type": "none"
        },
        "hit_set_period": 0,
        "hit_set_search_last_n": 0,
        "last_change": "61",
        "last_force_op_resend": "0",
        "last_force_op_resend_preluminous": "0",
        "min_read_recency_for_promote": 0,
        "min_size": 2,
        "min_write_recency_for_promote": 0,
        "object_hash": 2,
        "options": {},
        "pg_num": 8,
        "pgp_num": 8,
        "pool": 1,
        "pool_name": ".rgw.root",
        "pool_snaps": [],
        "quota_max_bytes": 0,
        "quota_max_objects": 0,
        "read_tier": -1,
        "removed_snaps": "[]",
        "size": 3,
        "snap_epoch": 0,
        "snap_mode": "selfmanaged",
        "snap_seq": 0,
        "stripe_width": 0,
        "target_max_bytes": 0,
        "target_max_objects": 0,
        "tier_of": -1,
        "tiers": [],
        "type": 1,
        "use_gmt_hitset": false,
        "write_tier": -1
    },
    {
        "application_metadata": {
            "rgw": {}
        },
        "auid": -1,
        "cache_min_evict_age": 0,
        "cache_min_flush_age": 0,
        "cache_mode": "none",
        "cache_target_dirty_high_ratio_micro": 600000,
        "cache_target_dirty_ratio_micro": 400000,
        "cache_target_full_ratio_micro": 800000,
        "crash_replay_interval": 0,
        "crush_rule": 1,
        "erasure_code_profile": "",
        "expected_num_objects": 0,
        "fast_read": false,
        "flags": 1,
        "flags_names": "hashpspool",
        "grade_table": [],
        "hit_set_count": 0,
        "hit_set_grade_decay_rate": 0,
        "hit_set_params": {
            "type": "none"
        },
        "hit_set_period": 0,
        "hit_set_search_last_n": 0,
        "last_change": "64",
        "last_force_op_resend": "0",
        "last_force_op_resend_preluminous": "0",
        "min_read_recency_for_promote": 0,
        "min_size": 2,
        "min_write_recency_for_promote": 0,
        "object_hash": 2,
        "options": {},
        "pg_num": 8,
        "pgp_num": 8,
        "pool": 2,
        "pool_name": "default.rgw.control",
        "pool_snaps": [],
        "quota_max_bytes": 0,
        "quota_max_objects": 0,
        "read_tier": -1,
        "removed_snaps": "[]",
        "size": 3,
        "snap_epoch": 0,
        "snap_mode": "selfmanaged",
        "snap_seq": 0,
        "stripe_width": 0,
        "target_max_bytes": 0,
        "target_max_objects": 0,
        "tier_of": -1,
        "tiers": [],
        "type": 1,
        "use_gmt_hitset": true,
        "write_tier": -1
    },
    {
        "application_metadata": {
            "rgw": {}
        },
        "auid": -1,
        "cache_min_evict_age": 0,
        "cache_min_flush_age": 0,
        "cache_mode": "none",
        "cache_target_dirty_high_ratio_micro": 600000,
        "cache_target_dirty_ratio_micro": 400000,
        "cache_target_full_ratio_micro": 800000,
        "crash_replay_interval": 0,
        "crush_rule": 1,
        "erasure_code_profile": "",
        "expected_num_objects": 0,
        "fast_read": false,
        "flags": 1,
        "flags_names": "hashpspool",
        "grade_table": [],
        "hit_set_count": 0,
        "hit_set_grade_decay_rate": 0,
        "hit_set_params": {
            "type": "none"
        },
        "hit_set_period": 0,
        "hit_set_search_last_n": 0,
        "last_change": "67",
        "last_force_op_resend": "0",
        "last_force_op_resend_preluminous": "0",
        "min_read_recency_for_promote": 0,
        "min_size": 2,
        "min_write_recency_for_promote": 0,
        "object_hash": 2,
        "options": {},
        "pg_num": 8,
        "pgp_num": 8,
        "pool": 3,
        "pool_name": "default.rgw.meta",
        "pool_snaps": [],
        "quota_max_bytes": 0,
        "quota_max_objects": 0,
        "read_tier": -1,
        "removed_snaps": "[]",
        "size": 3,
        "snap_epoch": 0,
        "snap_mode": "selfmanaged",
        "snap_seq": 0,
        "stripe_width": 0,
        "target_max_bytes": 0,
        "target_max_objects": 0,
        "tier_of": -1,
        "tiers": [],
        "type": 1,
        "use_gmt_hitset": true,
        "write_tier": -1
    },
    {
        "application_metadata": {
            "rgw": {}
        },
        "auid": -1,
        "cache_min_evict_age": 0,
        "cache_min_flush_age": 0,
        "cache_mode": "none",
        "cache_target_dirty_high_ratio_micro": 600000,
        "cache_target_dirty_ratio_micro": 400000,
        "cache_target_full_ratio_micro": 800000,
        "crash_replay_interval": 0,
        "crush_rule": 1,
        "erasure_code_profile": "",
        "expected_num_objects": 0,
        "fast_read": false,
        "flags": 1,
        "flags_names": "hashpspool",
        "grade_table": [],
        "hit_set_count": 0,
        "hit_set_grade_decay_rate": 0,
        "hit_set_params": {
            "type": "none"
        },
        "hit_set_period": 0,
        "hit_set_search_last_n": 0,
        "last_change": "70",
        "last_force_op_resend": "0",
        "last_force_op_resend_preluminous": "0",
        "min_read_recency_for_promote": 0,
        "min_size": 2,
        "min_write_recency_for_promote": 0,
        "object_hash": 2,
        "options": {},
        "pg_num": 8,
        "pgp_num": 8,
        "pool": 4,
        "pool_name": "default.rgw.log",
        "pool_snaps": [],
        "quota_max_bytes": 0,
        "quota_max_objects": 0,
        "read_tier": -1,
        "removed_snaps": "[]",
        "size": 3,
        "snap_epoch": 0,
        "snap_mode": "selfmanaged",
        "snap_seq": 0,
        "stripe_width": 0,
        "target_max_bytes": 0,
        "target_max_objects": 0,
        "tier_of": -1,
        "tiers": [],
        "type": 1,
        "use_gmt_hitset": true,
        "write_tier": -1
    },
    {
        "application_metadata": {
            "rgw": {}
        },
        "auid": 0,
        "cache_min_evict_age": 0,
        "cache_min_flush_age": 0,
        "cache_mode": "none",
        "cache_target_dirty_high_ratio_micro": 600000,
        "cache_target_dirty_ratio_micro": 400000,
        "cache_target_full_ratio_micro": 800000,
        "crash_replay_interval": 0,
        "crush_rule": 1,
        "erasure_code_profile": "rule_rgw",
        "expected_num_objects": 0,
        "fast_read": false,
        "flags": 1,
        "flags_names": "hashpspool",
        "grade_table": [],
        "hit_set_count": 0,
        "hit_set_grade_decay_rate": 0,
        "hit_set_params": {
            "type": "none"
        },
        "hit_set_period": 0,
        "hit_set_search_last_n": 0,
        "last_change": "57",
        "last_force_op_resend": "0",
        "last_force_op_resend_preluminous": "0",
        "min_read_recency_for_promote": 0,
        "min_size": 2,
        "min_write_recency_for_promote": 0,
        "object_hash": 2,
        "options": {},
        "pg_num": 16,
        "pgp_num": 16,
        "pool": 5,
        "pool_name": "default.rgw.buckets.index",
        "pool_snaps": [],
        "quota_max_bytes": 0,
        "quota_max_objects": 0,
        "read_tier": -1,
        "removed_snaps": "[]",
        "size": 3,
        "snap_epoch": 0,
        "snap_mode": "selfmanaged",
        "snap_seq": 0,
        "stripe_width": 0,
        "target_max_bytes": 0,
        "target_max_objects": 0,
        "tier_of": -1,
        "tiers": [],
        "type": 1,
        "use_gmt_hitset": true,
        "write_tier": -1
    },
    {
        "application_metadata": {
            "rgw": {}
        },
        "auid": 0,
        "cache_min_evict_age": 0,
        "cache_min_flush_age": 0,
        "cache_mode": "none",
        "cache_target_dirty_high_ratio_micro": 600000,
        "cache_target_dirty_ratio_micro": 400000,
        "cache_target_full_ratio_micro": 800000,
        "crash_replay_interval": 0,
        "crush_rule": 1,
        "erasure_code_profile": "rule_rgw",
        "expected_num_objects": 0,
        "fast_read": false,
        "flags": 1,
        "flags_names": "hashpspool",
        "grade_table": [],
        "hit_set_count": 0,
        "hit_set_grade_decay_rate": 0,
        "hit_set_params": {
            "type": "none"
        },
        "hit_set_period": 0,
        "hit_set_search_last_n": 0,
        "last_change": "790",
        "last_force_op_resend": "0",
        "last_force_op_resend_preluminous": "788",
        "min_read_recency_for_promote": 0,
        "min_size": 2,
        "min_write_recency_for_promote": 0,
        "object_hash": 2,
        "options": {},
        "pg_num": 514,
        "pgp_num": 514,
        "pool": 6,
        "pool_name": "default.rgw.buckets.data",
        "pool_snaps": [],
        "quota_max_bytes": 0,
        "quota_max_objects": 0,
        "read_tier": -1,
        "removed_snaps": "[]",
        "size": 3,
        "snap_epoch": 0,
        "snap_mode": "selfmanaged",
        "snap_seq": 0,
        "stripe_width": 0,
        "target_max_bytes": 0,
        "target_max_objects": 0,
        "tier_of": -1,
        "tiers": [],
        "type": 1,
        "use_gmt_hitset": true,
        "write_tier": -1
    },
    {
        "application_metadata": {
            "cephfs": {}
        },
        "auid": 0,
        "cache_min_evict_age": 0,
        "cache_min_flush_age": 0,
        "cache_mode": "none",
        "cache_target_dirty_high_ratio_micro": 600000,
        "cache_target_dirty_ratio_micro": 400000,
        "cache_target_full_ratio_micro": 800000,
        "crash_replay_interval": 0,
        "crush_rule": 0,
        "erasure_code_profile": "",
        "expected_num_objects": 0,
        "fast_read": false,
        "flags": 1,
        "flags_names": "hashpspool",
        "grade_table": [],
        "hit_set_count": 0,
        "hit_set_grade_decay_rate": 0,
        "hit_set_params": {
            "type": "none"
        },
        "hit_set_period": 0,
        "hit_set_search_last_n": 0,
        "last_change": "136",
        "last_force_op_resend": "0",
        "last_force_op_resend_preluminous": "0",
        "min_read_recency_for_promote": 0,
        "min_size": 2,
        "min_write_recency_for_promote": 0,
        "object_hash": 2,
        "options": {},
        "pg_num": 128,
        "pgp_num": 128,
        "pool": 7,
        "pool_name": "fs_data",
        "pool_snaps": [],
        "quota_max_bytes": 0,
        "quota_max_objects": 0,
        "read_tier": -1,
        "removed_snaps": "[]",
        "size": 3,
        "snap_epoch": 0,
        "snap_mode": "selfmanaged",
        "snap_seq": 0,
        "stripe_width": 0,
        "target_max_bytes": 0,
        "target_max_objects": 0,
        "tier_of": -1,
        "tiers": [],
        "type": 1,
        "use_gmt_hitset": true,
        "write_tier": -1
    },
    {
        "application_metadata": {
            "cephfs": {}
        },
        "auid": 0,
        "cache_min_evict_age": 0,
        "cache_min_flush_age": 0,
        "cache_mode": "none",
        "cache_target_dirty_high_ratio_micro": 600000,
        "cache_target_dirty_ratio_micro": 400000,
        "cache_target_full_ratio_micro": 800000,
        "crash_replay_interval": 0,
        "crush_rule": 0,
        "erasure_code_profile": "",
        "expected_num_objects": 0,
        "fast_read": false,
        "flags": 1,
        "flags_names": "hashpspool",
        "grade_table": [],
        "hit_set_count": 0,
        "hit_set_grade_decay_rate": 0,
        "hit_set_params": {
            "type": "none"
        },
        "hit_set_period": 0,
        "hit_set_search_last_n": 0,
        "last_change": "136",
        "last_force_op_resend": "0",
        "last_force_op_resend_preluminous": "0",
        "min_read_recency_for_promote": 0,
        "min_size": 2,
        "min_write_recency_for_promote": 0,
        "object_hash": 2,
        "options": {},
        "pg_num": 64,
        "pgp_num": 64,
        "pool": 8,
        "pool_name": "fs_metadata",
        "pool_snaps": [],
        "quota_max_bytes": 0,
        "quota_max_objects": 0,
        "read_tier": -1,
        "removed_snaps": "[]",
        "size": 3,
        "snap_epoch": 0,
        "snap_mode": "selfmanaged",
        "snap_seq": 0,
        "stripe_width": 0,
        "target_max_bytes": 0,
        "target_max_objects": 0,
        "tier_of": -1,
        "tiers": [],
        "type": 1,
        "use_gmt_hitset": true,
        "write_tier": -1
    },
    {
        "application_metadata": {
            "rgw": {}
        },
        "auid": 0,
        "cache_min_evict_age": 0,
        "cache_min_flush_age": 0,
        "cache_mode": "none",
        "cache_target_dirty_high_ratio_micro": 600000,
        "cache_target_dirty_ratio_micro": 400000,
        "cache_target_full_ratio_micro": 800000,
        "crash_replay_interval": 0,
        "crush_rule": 0,
        "erasure_code_profile": "",
        "expected_num_objects": 0,
        "fast_read": false,
        "flags": 1,
        "flags_names": "hashpspool",
        "grade_table": [],
        "hit_set_count": 0,
        "hit_set_grade_decay_rate": 0,
        "hit_set_params": {
            "type": "none"
        },
        "hit_set_period": 0,
        "hit_set_search_last_n": 0,
        "last_change": "139",
        "last_force_op_resend": "0",
        "last_force_op_resend_preluminous": "0",
        "min_read_recency_for_promote": 0,
        "min_size": 2,
        "min_write_recency_for_promote": 0,
        "object_hash": 2,
        "options": {},
        "pg_num": 8,
        "pgp_num": 8,
        "pool": 9,
        "pool_name": "default.rgw.buckets.non-ec",
        "pool_snaps": [],
        "quota_max_bytes": 0,
        "quota_max_objects": 0,
        "read_tier": -1,
        "removed_snaps": "[]",
        "size": 3,
        "snap_epoch": 0,
        "snap_mode": "selfmanaged",
        "snap_seq": 0,
        "stripe_width": 0,
        "target_max_bytes": 0,
        "target_max_objects": 0,
        "tier_of": -1,
        "tiers": [],
        "type": 1,
        "use_gmt_hitset": true,
        "write_tier": -1
    }
]
```


# Python调用

可以通过requests来调用ceph mgr restful的接口，下面通过Python来获取全部存储池信息。

```
#! /usr/bin/env python3

import requests

r = requests.get('https://192.168.180.138:8003/pool',verify=False, auth=('admin','839df177-560e-421a-95fc-9f6a1c08236e'))
print(r.json())
```

# 参考
- https://lnsyyj.github.io/2017/11/27/CEPH-MANAGER-DAEMON-RESTful-plugin/