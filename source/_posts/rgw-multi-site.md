---
title: rgw_multi_site
date: 2019-03-09 20:04:30
tags: ceph
---

# 概述
本文主要介绍如何配置Ceph RGW的异步复制功能，通过这个功能可以实现跨数据中心的灾备功能。  
RGW多活方式是在同一zonegroup的多个zone之间进行，即同一zonegroup中多个zone之间的数据是完全一致的，用户可以通过任意zone读写同一份数据。 但是，对元数据的操作，比如创建桶、创建用户，仍然只能在master zone进行。对数据的操作，比如创建桶中的对象，访问对象等，可以在任意zone中 处理.

# 环境
实验环境是两个ceph集群，信息如下：  
集群ceph101  

```
[root@ceph101 ~]# ceph -s
  cluster:
    id:     d6e42188-9871-471b-9db0-957f47893902
    health: HEALTH_OK

  services:
    mon: 1 daemons, quorum ceph101
    mgr: ceph101(active)
    osd: 3 osds: 3 up, 3 in
    rgw: 1 daemon active

  data:
    pools:   4 pools, 32 pgs
    objects: 187 objects, 1.09KiB
    usage:   3.01GiB used, 56.7GiB / 59.7GiB avail
    pgs:     32 active+clean
```


集群ceph102  

```
[root@ceph102 ~]# ceph -s
  cluster:
    id:     2e80de18-e95f-463f-9eb0-531fd3254f0b
    health: HEALTH_OK

  services:
    mon: 1 daemons, quorum ceph102
    mgr: ceph102(active)
    osd: 3 osds: 3 up, 3 in
    rgw: 1 daemon active

  data:
    pools:   4 pools, 32 pgs
    objects: 187 objects, 1.09KiB
    usage:   3.01GiB used, 56.7GiB / 59.7GiB avail
    pgs:     32 active+clean
```

这两个ceph集群都一个rgw服务，本次实验就通过这两个ceph集群验证rgw multi site的配置，已经功能的验证。  
本次实验已第一个集群(ceph101)做为主集群，ceph102作为备集群。

# Multi Site 配置

在主集群创建一个名为realm100的realm

```
[root@ceph101 ~]# radosgw-admin realm create --rgw-realm=realm100 --default
{
    "id": "337cd1c3-1ad0-4975-b220-e021a7f2b3eb",
    "name": "realm100",
    "current_period": "bd6ecbd6-3a28-46d7-a806-22e9ea001ca3",
    "epoch": 1
}
```

创建master zonegroup

```
[root@ceph101 ~]# radosgw-admin zonegroup create --rgw-zonegroup=cn --endpoints=http://172.16.143.201:8080 --rgw-realm=realm100 --master --default
{
    "id": "2b3f1ea0-73e7-4f17-b4f9-1acf5d5c6033",
    "name": "cn",
    "api_name": "cn",
    "is_master": "true",
    "endpoints": [
        "http://172.16.143.201:8080"
    ],
    "hostnames": [],
    "hostnames_s3website": [],
    "master_zone": "",
    "zones": [],
    "placement_targets": [],
    "default_placement": "",
    "realm_id": "ba638e8a-8a33-4607-8fe3-13aa69dd1758"
}
```

创建master zone

```
[root@ceph101 ~]# radosgw-admin zone create --rgw-zonegroup=cn --rgw-zone=shanghai --master --default --endpoints=http://172.16.143.201:8080
{
    "id": "9c655173-6346-47e7-9759-5e5d32aa017d",
    "name": "shanghai",
    "domain_root": "shanghai.rgw.meta:root",
    "control_pool": "shanghai.rgw.control",
    "gc_pool": "shanghai.rgw.log:gc",
    "lc_pool": "shanghai.rgw.log:lc",
    "log_pool": "shanghai.rgw.log",
    "intent_log_pool": "shanghai.rgw.log:intent",
    "usage_log_pool": "shanghai.rgw.log:usage",
    "reshard_pool": "shanghai.rgw.log:reshard",
    "user_keys_pool": "shanghai.rgw.meta:users.keys",
    "user_email_pool": "shanghai.rgw.meta:users.email",
    "user_swift_pool": "shanghai.rgw.meta:users.swift",
    "user_uid_pool": "shanghai.rgw.meta:users.uid",
    "system_key": {
        "access_key": "",
        "secret_key": ""
    },
    "placement_pools": [
        {
            "key": "default-placement",
            "val": {
                "index_pool": "shanghai.rgw.buckets.index",
                "data_pool": "shanghai.rgw.buckets.data",
                "data_extra_pool": "shanghai.rgw.buckets.non-ec",
                "index_type": 0,
                "compression": ""
            }
        }
    ],
    "metadata_heap": "",
    "tier_config": [],
    "realm_id": "ba638e8a-8a33-4607-8fe3-13aa69dd1758"
}
```

更新period

```
[root@ceph101 ~]# radosgw-admin period update --commit
{
    "id": "064c5b5e-eb87-462c-aa37-c420ddd68b23",
    "epoch": 1,
    "predecessor_uuid": "1abfe2d0-7453-4c01-881b-3db7f53f15c1",
    "sync_status": [],
    "period_map": {
        "id": "064c5b5e-eb87-462c-aa37-c420ddd68b23",
        "zonegroups": [
            {
                "id": "2b3f1ea0-73e7-4f17-b4f9-1acf5d5c6033",
                "name": "cn",
                "api_name": "cn",
                "is_master": "true",
                "endpoints": [
                    "http://172.16.143.201:8080"
                ],
                "hostnames": [],
                "hostnames_s3website": [],
                "master_zone": "9c655173-6346-47e7-9759-5e5d32aa017d",
                "zones": [
                    {
                        "id": "9c655173-6346-47e7-9759-5e5d32aa017d",
                        "name": "shanghai",
                        "endpoints": [
                            "http://172.16.143.201:8080"
                        ],
                        "log_meta": "false",
                        "log_data": "false",
                        "bucket_index_max_shards": 0,
                        "read_only": "false",
                        "tier_type": "",
                        "sync_from_all": "true",
                        "sync_from": []
                    }
                ],
                "placement_targets": [
                    {
                        "name": "default-placement",
                        "tags": []
                    }
                ],
                "default_placement": "default-placement",
                "realm_id": "ba638e8a-8a33-4607-8fe3-13aa69dd1758"
            }
        ],
        "short_zone_ids": [
            {
                "key": "9c655173-6346-47e7-9759-5e5d32aa017d",
                "val": 1556250220
            }
        ]
    },
    "master_zonegroup": "2b3f1ea0-73e7-4f17-b4f9-1acf5d5c6033",
    "master_zone": "9c655173-6346-47e7-9759-5e5d32aa017d",
    "period_config": {
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
        }
    },
    "realm_id": "ba638e8a-8a33-4607-8fe3-13aa69dd1758",
    "realm_name": "realm100",
    "realm_epoch": 2
}
```

创建同步用户

```
[root@ceph101 ~]# radosgw-admin user create --uid="syncuser" --display-name="Synchronization User" --system
{
    "user_id": "syncuser",
    "display_name": "Synchronization User",
    "email": "",
    "suspended": 0,
    "max_buckets": 1000,
    "auid": 0,
    "subusers": [],
    "keys": [
        {
            "user": "syncuser",
            "access_key": "LPTHGKYO5ULI48Q88AWF",
            "secret_key": "jGFoORTVt72frRYsmcnPOpGXnz652Dl3C2IeBLN8"
        }
    ],
    "swift_keys": [],
    "caps": [],
    "op_mask": "read, write, delete",
    "system": "true",
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

修改zone的key，并更新period

```
[root@ceph101 ~]# radosgw-admin zone modify --rgw-zone=shanghai --access-key=LPTHGKYO5ULI48Q88AWF --secret=jGFoORTVt72frRYsmcnPOpGXnz652Dl3C2IeBLN8

[root@ceph101 ~]# radosgw-admin period update --commit
{
    "id": "064c5b5e-eb87-462c-aa37-c420ddd68b23",
    "epoch": 2,
    "predecessor_uuid": "1abfe2d0-7453-4c01-881b-3db7f53f15c1",
    "sync_status": [],
    "period_map": {
        "id": "064c5b5e-eb87-462c-aa37-c420ddd68b23",
        "zonegroups": [
            {
                "id": "2b3f1ea0-73e7-4f17-b4f9-1acf5d5c6033",
                "name": "cn",
                "api_name": "cn",
                "is_master": "true",
                "endpoints": [
                    "http://172.16.143.201:8080"
                ],
                "hostnames": [],
                "hostnames_s3website": [],
                "master_zone": "9c655173-6346-47e7-9759-5e5d32aa017d",
                "zones": [
                    {
                        "id": "9c655173-6346-47e7-9759-5e5d32aa017d",
                        "name": "shanghai",
                        "endpoints": [
                            "http://172.16.143.201:8080"
                        ],
                        "log_meta": "false",
                        "log_data": "false",
                        "bucket_index_max_shards": 0,
                        "read_only": "false",
                        "tier_type": "",
                        "sync_from_all": "true",
                        "sync_from": []
                    }
                ],
                "placement_targets": [
                    {
                        "name": "default-placement",
                        "tags": []
                    }
                ],
                "default_placement": "default-placement",
                "realm_id": "ba638e8a-8a33-4607-8fe3-13aa69dd1758"
            }
        ],
        "short_zone_ids": [
            {
                "key": "9c655173-6346-47e7-9759-5e5d32aa017d",
                "val": 1556250220
            }
        ]
    },
    "master_zonegroup": "2b3f1ea0-73e7-4f17-b4f9-1acf5d5c6033",
    "master_zone": "9c655173-6346-47e7-9759-5e5d32aa017d",
    "period_config": {
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
        }
    },
    "realm_id": "ba638e8a-8a33-4607-8fe3-13aa69dd1758",
    "realm_name": "realm100",
    "realm_epoch": 2
}
```

删除默认zone和zonegroup

```
[root@ceph101 ~]# radosgw-admin zonegroup remove --rgw-zonegroup=default --rgw-zone=default
[root@ceph101 ~]# radosgw-admin period update --commit
[root@ceph101 ~]# radosgw-admin zone delete --rgw-zone=default
[root@ceph101 ~]# radosgw-admin period update --commit
[root@ceph101 ~]# radosgw-admin zonegroup delete --rgw-zonegroup=default
[root@ceph101 ~]# radosgw-admin period update --commit
```

删除默认pool

```
[root@ceph101 ~]# ceph osd pool delete default.rgw.control default.rgw.control --yes-i-really-really-mean-it
pool 'default.rgw.control' removed
[root@ceph101 ~]# ceph osd pool delete default.rgw.meta default.rgw.meta --yes-i-really-really-mean-it
pool 'default.rgw.meta' removed
[root@ceph101 ~]# ceph osd pool delete default.rgw.log default.rgw.log --yes-i-really-really-mean-it
pool 'default.rgw.log' removed
```

修改rgw配置, 增加rgw_zone = shanghai

```
[root@ceph101 ~]# vim /etc/ceph/ceph.conf

[global]
fsid = d6e42188-9871-471b-9db0-957f47893902



mon initial members = ceph101
mon host = 172.16.143.201

public network = 172.16.143.0/24
cluster network = 172.16.140.0/24


mon allow pool delete = true


[client.rgw.ceph101]
host = ceph101
keyring = /var/lib/ceph/radosgw/ceph-rgw.ceph101/keyring
log file = /var/log/ceph/ceph-rgw-ceph101.log
rgw frontends = civetweb port=172.16.143.201:8080 num_threads=100
rgw_zone = shanghai
```


重启rgw，并查看pool是否创建

```
[root@ceph101 ~]# systemctl restart ceph-radosgw@rgw.ceph101
[root@ceph101 ~]# ceph osd pool ls
.rgw.root
shanghai.rgw.control
shanghai.rgw.meta
shanghai.rgw.log
```

在secondy zone节点进行如下配置：  

同步realm, 并设置realm100为默认的realm

```
[root@ceph102 ~]# radosgw-admin realm pull --url=http://172.16.143.201:8080 --access-key=LPTHGKYO5ULI48Q88AWF --secret=jGFoORTVt72frRYsmcnPOpGXnz652Dl3C2IeBLN8
2019-03-09 21:25:10.377485 7fa55aca2dc0  1 found existing latest_epoch 2 >= given epoch 2, returning r=-17
{
    "id": "ba638e8a-8a33-4607-8fe3-13aa69dd1758",
    "name": "realm100",
    "current_period": "064c5b5e-eb87-462c-aa37-c420ddd68b23",
    "epoch": 2
}

[root@ceph102 ~]# radosgw-admin realm default --rgw-realm=realm100
```

更新period

```
[root@ceph102 ~]# radosgw-admin period pull --url=http://172.16.143.201:8080 --access-key=LPTHGKYO5ULI48Q88AWF --secret=jGFoORTVt72frRYsmcnPOpGXnz652Dl3C2IeBLN8
{
    "id": "064c5b5e-eb87-462c-aa37-c420ddd68b23",
    "epoch": 2,
    "predecessor_uuid": "1abfe2d0-7453-4c01-881b-3db7f53f15c1",
    "sync_status": [],
    "period_map": {
        "id": "064c5b5e-eb87-462c-aa37-c420ddd68b23",
        "zonegroups": [
            {
                "id": "2b3f1ea0-73e7-4f17-b4f9-1acf5d5c6033",
                "name": "cn",
                "api_name": "cn",
                "is_master": "true",
                "endpoints": [
                    "http://172.16.143.201:8080"
                ],
                "hostnames": [],
                "hostnames_s3website": [],
                "master_zone": "9c655173-6346-47e7-9759-5e5d32aa017d",
                "zones": [
                    {
                        "id": "9c655173-6346-47e7-9759-5e5d32aa017d",
                        "name": "shanghai",
                        "endpoints": [
                            "http://172.16.143.201:8080"
                        ],
                        "log_meta": "false",
                        "log_data": "false",
                        "bucket_index_max_shards": 0,
                        "read_only": "false",
                        "tier_type": "",
                        "sync_from_all": "true",
                        "sync_from": []
                    }
                ],
                "placement_targets": [
                    {
                        "name": "default-placement",
                        "tags": []
                    }
                ],
                "default_placement": "default-placement",
                "realm_id": "ba638e8a-8a33-4607-8fe3-13aa69dd1758"
            }
        ],
        "short_zone_ids": [
            {
                "key": "9c655173-6346-47e7-9759-5e5d32aa017d",
                "val": 1556250220
            }
        ]
    },
    "master_zonegroup": "2b3f1ea0-73e7-4f17-b4f9-1acf5d5c6033",
    "master_zone": "9c655173-6346-47e7-9759-5e5d32aa017d",
    "period_config": {
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
        }
    },
    "realm_id": "ba638e8a-8a33-4607-8fe3-13aa69dd1758",
    "realm_name": "realm100",
    "realm_epoch": 2
}
```

创建secondy zone

```
[root@ceph102 ~]# radosgw-admin zone create --rgw-zonegroup=cn --rgw-zone=beijing --endpoints=http://172.16.143.202:8080 --access-key=LPTHGKYO5ULI48Q88AWF --secret=jGFoORTVt72frRYsmcnPOpGXnz652Dl3C2IeBLN8
2019-03-09 21:30:24.323527 7f2441e45dc0  0 failed reading obj info from .rgw.root:zone_info.9c655173-6346-47e7-9759-5e5d32aa017d: (2) No such file or directory
2019-03-09 21:30:24.323577 7f2441e45dc0  0 WARNING: could not read zone params for zone id=9c655173-6346-47e7-9759-5e5d32aa017d name=shanghai
{
    "id": "bf59d999-1561-4f7e-a874-9de718d4c31b",
    "name": "beijing",
    "domain_root": "beijing.rgw.meta:root",
    "control_pool": "beijing.rgw.control",
    "gc_pool": "beijing.rgw.log:gc",
    "lc_pool": "beijing.rgw.log:lc",
    "log_pool": "beijing.rgw.log",
    "intent_log_pool": "beijing.rgw.log:intent",
    "usage_log_pool": "beijing.rgw.log:usage",
    "reshard_pool": "beijing.rgw.log:reshard",
    "user_keys_pool": "beijing.rgw.meta:users.keys",
    "user_email_pool": "beijing.rgw.meta:users.email",
    "user_swift_pool": "beijing.rgw.meta:users.swift",
    "user_uid_pool": "beijing.rgw.meta:users.uid",
    "system_key": {
        "access_key": "LPTHGKYO5ULI48Q88AWF",
        "secret_key": "jGFoORTVt72frRYsmcnPOpGXnz652Dl3C2IeBLN8"
    },
    "placement_pools": [
        {
            "key": "default-placement",
            "val": {
                "index_pool": "beijing.rgw.buckets.index",
                "data_pool": "beijing.rgw.buckets.data",
                "data_extra_pool": "beijing.rgw.buckets.non-ec",
                "index_type": 0,
                "compression": ""
            }
        }
    ],
    "metadata_heap": "",
    "tier_config": [],
    "realm_id": "ba638e8a-8a33-4607-8fe3-13aa69dd1758"
}
```

删除默认default zone， defaul zonegroup和 default存储池

```
[root@ceph102 ~]# radosgw-admin zone delete --rgw-zone=default
2019-03-09 21:31:23.577581 7f2e6c528dc0  0 zone id ce963a6e-7d58-426d-b07a-a2af2983379a is not a part of zonegroup cn
[root@ceph102 ~]# radosgw-admin zonegroup list
{
    "default_info": "2b3f1ea0-73e7-4f17-b4f9-1acf5d5c6033",
    "zonegroups": [
        "cn",
        "default"
    ]
}

[root@ceph102 ~]# radosgw-admin zonegroup delete --rgw-zonegroup=default

[root@ceph102 ~]# ceph osd pool delete default.rgw.control default.rgw.control --yes-i-really-really-mean-it
pool 'default.rgw.control' removed
[root@ceph102 ~]# ceph osd pool delete default.rgw.meta default.rgw.meta --yes-i-really-really-mean-it
pool 'default.rgw.meta' removed
[root@ceph102 ~]# ceph osd pool delete default.rgw.log default.rgw.log --yes-i-really-really-mean-it
pool 'default.rgw.log' removed
```

修改rgw配置, 增加rgw_zone = beijing

```
[root@ceph101 ~]# vim /etc/ceph/ceph.conf

[global]
fsid = d6e42188-9871-471b-9db0-957f47893902



mon initial members = ceph101
mon host = 172.16.143.201

public network = 172.16.143.0/24
cluster network = 172.16.140.0/24


mon allow pool delete = true


[client.rgw.ceph101]
host = ceph101
keyring = /var/lib/ceph/radosgw/ceph-rgw.ceph101/keyring
log file = /var/log/ceph/ceph-rgw-ceph101.log
rgw frontends = civetweb port=172.16.143.201:8080 num_threads=100
rgw_zone = shanghai
```


重启rgw，并查看pool是否创建

```
[root@ceph102 ~]# systemctl restart ceph-radosgw@rgw.ceph102
[root@ceph102 ~]# ceph osd pool ls
.rgw.root
beijing.rgw.control
beijing.rgw.meta
beijing.rgw.log
```

更新period

```
[root@ceph102 ~]# radosgw-admin period update --commit
2019-03-09 21:36:44.462809 7f23c756bdc0  1 Cannot find zone id=bf59d999-1561-4f7e-a874-9de718d4c31b (name=beijing), switching to local zonegroup configuration
Sending period to new master zone 9c655173-6346-47e7-9759-5e5d32aa017d
{
    "id": "064c5b5e-eb87-462c-aa37-c420ddd68b23",
    "epoch": 3,
    "predecessor_uuid": "1abfe2d0-7453-4c01-881b-3db7f53f15c1",
    "sync_status": [],
    "period_map": {
        "id": "064c5b5e-eb87-462c-aa37-c420ddd68b23",
        "zonegroups": [
            {
                "id": "2b3f1ea0-73e7-4f17-b4f9-1acf5d5c6033",
                "name": "cn",
                "api_name": "cn",
                "is_master": "true",
                "endpoints": [
                    "http://172.16.143.201:8080"
                ],
                "hostnames": [],
                "hostnames_s3website": [],
                "master_zone": "9c655173-6346-47e7-9759-5e5d32aa017d",
                "zones": [
                    {
                        "id": "9c655173-6346-47e7-9759-5e5d32aa017d",
                        "name": "shanghai",
                        "endpoints": [
                            "http://172.16.143.201:8080"
                        ],
                        "log_meta": "false",
                        "log_data": "true",
                        "bucket_index_max_shards": 0,
                        "read_only": "false",
                        "tier_type": "",
                        "sync_from_all": "true",
                        "sync_from": []
                    },
                    {
                        "id": "bf59d999-1561-4f7e-a874-9de718d4c31b",
                        "name": "beijing",
                        "endpoints": [],
                        "log_meta": "false",
                        "log_data": "true",
                        "bucket_index_max_shards": 0,
                        "read_only": "false",
                        "tier_type": "",
                        "sync_from_all": "true",
                        "sync_from": []
                    }
                ],
                "placement_targets": [
                    {
                        "name": "default-placement",
                        "tags": []
                    }
                ],
                "default_placement": "default-placement",
                "realm_id": "ba638e8a-8a33-4607-8fe3-13aa69dd1758"
            }
        ],
        "short_zone_ids": [
            {
                "key": "9c655173-6346-47e7-9759-5e5d32aa017d",
                "val": 1556250220
            },
            {
                "key": "bf59d999-1561-4f7e-a874-9de718d4c31b",
                "val": 3557185953
            }
        ]
    },
    "master_zonegroup": "2b3f1ea0-73e7-4f17-b4f9-1acf5d5c6033",
    "master_zone": "9c655173-6346-47e7-9759-5e5d32aa017d",
    "period_config": {
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
        }
    },
    "realm_id": "ba638e8a-8a33-4607-8fe3-13aa69dd1758",
    "realm_name": "realm100",
    "realm_epoch": 2
}
```

查看同步状态

```
[root@ceph101 ~]# radosgw-admin sync status
          realm ba638e8a-8a33-4607-8fe3-13aa69dd1758 (realm100)
      zonegroup 2b3f1ea0-73e7-4f17-b4f9-1acf5d5c6033 (cn)
           zone 9c655173-6346-47e7-9759-5e5d32aa017d (shanghai)
  metadata sync no sync (zone is master)
      data sync source: 0a2f706f-cc81-44f3-adf6-0d79c3e362ac (beijing)
                        syncing
                        full sync: 0/128 shards
                        incremental sync: 128/128 shards
                        data is caught up with source
```

```
[root@ceph102 ~]# radosgw-admin sync status
          realm ba638e8a-8a33-4607-8fe3-13aa69dd1758 (realm100)
      zonegroup 2b3f1ea0-73e7-4f17-b4f9-1acf5d5c6033 (cn)
           zone 0a2f706f-cc81-44f3-adf6-0d79c3e362ac (beijing)
  metadata sync syncing
                full sync: 0/64 shards
                incremental sync: 64/64 shards
                metadata is caught up with master
      data sync source: 9c655173-6346-47e7-9759-5e5d32aa017d (shanghai)
                        syncing
                        full sync: 0/128 shards
                        incremental sync: 128/128 shards
                        data is caught up with source
```

# 验证
在master zone 创建一个test用户，在secondy zone 查看信息

```
[root@ceph101 ~]# radosgw-admin user create --uid test --display-name="test user"
2019-03-09 22:16:27.795146 7f022820cdc0  0 WARNING: can't generate connection for zone 0a2f706f-cc81-44f3-adf6-0d79c3e362ac id beijing: no endpoints defined
{
    "user_id": "test",
    "display_name": "test user",
    "email": "",
    "suspended": 0,
    "max_buckets": 1000,
    "auid": 0,
    "subusers": [],
    "keys": [
        {
            "user": "test",
            "access_key": "9R9YENBPPDZ88TQGP32D",
            "secret_key": "B5wtnLKNloIVxmSDISut6scU8MClv52dfInb3Omh"
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

secondy zone 查看用户

```
[root@ceph102 ~]# radosgw-admin user list
[
    "syncuser",
    "test"
]
```

在secondy zone 创建一个test2用户，在master zone 查看信息

```
[root@ceph102 ~]# radosgw-admin user create --uid test@2 --display-name="test2 user"
{
    "user_id": "test@2",
    "display_name": "test2 user",
    "email": "",
    "suspended": 0,
    "max_buckets": 1000,
    "auid": 0,
    "subusers": [],
    "keys": [
        {
            "user": "test@2",
            "access_key": "9413B8F517W6LTEKN0H6",
            "secret_key": "lg9vzVhpS2gYdXDnEOEUv3uCwkz0wkyuuBSnh8dl"
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

[root@ceph102 ~]# radosgw-admin user list
[
    "syncuser",
    "test@2",
    "test"
]
```

在master zone 查看

```
[root@ceph101 ~]# radosgw-admin user list
[
    "syncuser",
    "test"
]
```

可以看到在master zone创建的用户，在secondy zone也可以看到  
而在secondy zone创建的用户，在master zone看不到


通过test用户，在master zone 创建名为bucket1的bucket

```
[root@ceph101 ~]# s3cmd mb s3://bucket1
Bucket 's3://bucket1/' created
[root@ceph101 ~]# radosgw-admin bucket list
[
    "bucket1"
]
```

在secondy zone查看

```
[root@ceph102 ~]# radosgw-admin bucket list
[
    "bucket1"
]
```


通过test用户，在secondy zone 创建名为bucket2的bucket

```
[root@ceph102 ~]# s3cmd mb s3://bucket2
Bucket 's3://bucket2/' created
[root@ceph102 ~]# radosgw-admin bucket list
[
    "bucket1",
    "bucket2"
]
```

在master zone 查看

```
[root@ceph101 ~]# radosgw-admin bucket list
[
    "bucket1",
    "bucket2"
]
```

在master zone 上传文件

```
[root@ceph101 ~]# s3cmd put Python-3.4.9.tgz s3://bucket1/python3.4.9.tgz
upload: 'Python-3.4.9.tgz' -> 's3://bucket1/python3.4.9.tgz'  [part 1 of 2, 15MB] [1 of 1]
 15728640 of 15728640   100% in    2s     6.26 MB/s  done
upload: 'Python-3.4.9.tgz' -> 's3://bucket1/python3.4.9.tgz'  [part 2 of 2, 3MB] [1 of 1]
 3955465 of 3955465   100% in    0s    32.73 MB/s  done
 
[root@ceph101 ~]# md5sum Python-3.4.9.tgz
c706902881ef95e27e59f13fabbcdcac  Python-3.4.9.tgz
```

在secondy zone 查看信息

```
[root@ceph102 ~]# s3cmd ls s3://bucket1
2019-03-09 15:34  19684105   s3://bucket1/python3.4.9.tgz
[root@ceph102 ~]# s3cmd get s3://bucket1/python3.4.9.tgz Python3.4.9.tgz
download: 's3://bucket1/python3.4.9.tgz' -> 'Python3.4.9.tgz'  [1 of 1]
 19684105 of 19684105   100% in    0s    51.81 MB/s  done
[root@ceph102 ~]# md5sum Python3.4.9.tgz
c706902881ef95e27e59f13fabbcdcac  Python3.4.9.tgz
```

在secondy zone 上传文件

```
[root@ceph102 ~]# s3cmd put anaconda-ks.cfg s3://bucket2/anaconda-ks.cfg
upload: 'anaconda-ks.cfg' -> 's3://bucket2/anaconda-ks.cfg'  [1 of 1]
 1030 of 1030   100% in    0s     8.28 kB/s  done
[root@ceph102 ~]# md5sum anaconda-ks.cfg
1627b1ce985ce5befdd0d1cb0e6164ae  anaconda-ks.cfg
```

在master zone 查看

```
[root@ceph101 tmp]# s3cmd ls s3://bucket2
2019-03-09 15:37      1030   s3://bucket2/anaconda-ks.cfg
[root@ceph101 tmp]# s3cmd get s3://bucket2/anaconda-ks.cfg anaconda-ks.cfg
download: 's3://bucket2/anaconda-ks.cfg' -> 'anaconda-ks.cfg'  [1 of 1]
 1030 of 1030   100% in    0s    23.32 kB/s  done
[root@ceph101 tmp]# md5sum anaconda-ks.cfg
1627b1ce985ce5befdd0d1cb0e6164ae  anaconda-ks.cfg
```

停止master zone的grup，然后在secondy zone上创建存储桶

```
[root@ceph101 tmp]# systemctl stop ceph-radosgw@rgw.ceph101
```

```
[root@ceph102 ~]# s3cmd mb s3://bucket4
WARNING: Retrying failed request: / (503 (ServiceUnavailable))
WARNING: Waiting 3 sec...
WARNING: Retrying failed request: / (503 (ServiceUnavailable))
WARNING: Waiting 6 sec...
WARNING: Retrying failed request: / (503 (ServiceUnavailable))
WARNING: Waiting 9 sec...
```

可以看到在secondy zone上并不能创建bucket，之前在secondy zone上创建bucket，也是把请求转到master zone上。

反之，停止secondy zone的rgw，在master zone也是可以创建存储桶

```
[root@ceph102 ~]# systemctl stop ceph-radosgw@ceph102
```

```
[root@ceph101 tmp]# s3cmd mb s3://bucket5
Bucket 's3://bucket5/' created
```

# 参考
* http://docs.ceph.com/docs/luminous/radosgw/multisite/
* https://access.redhat.com/documentation/en-us/red_hat_ceph_storage/2/html/object_gateway_guide_for_red_hat_enterprise_linux/multi_site
* https://www.jianshu.com/p/31a6f8df9a8f
* http://stor.51cto.com/art/201807/578337.htm
