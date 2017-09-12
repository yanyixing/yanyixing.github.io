---
title: timedatectl
date: 2017-09-12 18:00:19
tags:
---
CentOS 7 设置日期和时间

在CentOS 6版本，时间设置有date、hwclock命令，从CentOS 7开始，使用了一个新的命令timedatectl。

一、基本概念

1.1 GMT、UTC、CST、DST 时间

(1) UTC

整个地球分为二十四时区，每个时区都有自己的本地时间。在国际无线电通信场合，为了统一起见，使用一个统一的时间，称为通用协调时(UTC, Universal Time Coordinated)。

(2) GMT

格林威治标准时间 (Greenwich Mean Time)指位于英国伦敦郊区的皇家格林尼治天文台的标准时间，因为本初子午线被定义在通过那里的经线。(UTC与GMT时间基本相同，本文中不做区分)

(3) CST

中国标准时间 (China Standard Time)

1
GMT + 8 = UTC + 8 = CST
(4) DST

夏令时(Daylight Saving Time) 指在夏天太阳升起的比较早时，将时钟拨快一小时，以提早日光的使用。（中国不使用）

1.2 硬件时钟和系统时钟

(1) 硬件时钟

RTC(Real-Time Clock)或CMOS时钟，一般在主板上靠电池供电，服务器断电后也会继续运行。仅保存日期时间数值，无法保存时区和夏令时设置。

(2) 系统时钟

一般在服务器启动时复制RTC时间，之后独立运行，保存了时间、时区和夏令时设置。

二、timedatectl 命令

2.1 读取时间

```
timedatectl //等同于 timedatectl status
```
2.2 设置时间

```
timedatectl set-time "YYYY-MM-DD HH:MM:SS"
```

2.3 列出所有时区

```
timedatectl list-timezones
```

2.4 设置时区

```
timedatectl set-timezone Asia/Shanghai
```

2.5 是否NTP服务器同步

```
timedatectl set-ntp yes //yes或者no
```

2.6 将硬件时钟调整为与本地时钟一致

```
timedatectl set-local-rtc 1
hwclock --systohc --localtime //与上面命令效果一致
```
注意 硬件时钟默认使用UTC时间，因为硬件时钟不能保存时区和夏令时调整，修改后就无法从硬件时钟中读取出准确标准时间，因此不建议修改。修改后系统会出现警告。

2.6 硬件时间设置成 UTC：

```
timedatectl set-local-rtc 1
hwclock --systohc --utc //与上面命令效果一致
```

三、设置系统时间为中国时区并启用NTP同步

```
yum install ntp //安装ntp服务
systemctl enable ntpd //开机启动服务
systemctl start ntpd //启动服务
timedatectl set-timezone Asia/Shanghai //更改时区
timedatectl set-ntp yes //启用ntp同步
ntpq -p //同步时间
```
如需更改时间服务器, 修改 /etc/ntp.conf 文件中的服务器地址 server 即可.