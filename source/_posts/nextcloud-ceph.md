---
title: nextcloud对接ceph
date: 2017-08-09 14:38:45
tags:
---
nextcloud是一个私有网盘解决方案，支持多种后端存储解决方案。
nextcloud也支持aws的s3, 而ceph的radosgw也是兼容s3协议的，所以nextcloud也可以使用ceph作为后端存储，具体的配置如下
{% asset_img nextcloud1.png 找到插件 %}
{% asset_img nextcloud2.png enable插件 %}
{% asset_img nextcloud3.png 添加ceph rgw %}
填写ceph rgw的server和port，access_key,secret_key,enable path style

{% asset_img nextcloud4.png 找到对应的文件夹 %}
{% asset_img nextcloud5.png 上传文件 %}
{% asset_img nextcloud6.png 在Ceph rgw中查看文件 %}
