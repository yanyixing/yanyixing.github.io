---
title: mysql-replication
date: 2018-03-02 17:23:27
tags: ['linux','mysql','ha']
---

## 环境
OS：Centos 7.3

DB version： mysql  Ver 15.1 Distrib 5.5.56-MariaDB, for Linux (x86_64) using readline 5.1

host1(master): 172.16.143.171

host2(slave): 172.16.143.172

## 安装

两个节点都执行下面的步骤

```bash
yum install mariadb-server -y

systemctl start mariadb
```
执行db安全选项

```bash
mysql_secure_installation
NOTE: RUNNING ALL PARTS OF THIS SCRIPT IS RECOMMENDED FOR ALL MariaDB
      SERVERS IN PRODUCTION USE!  PLEASE READ EACH STEP CAREFULLY!

In order to log into MariaDB to secure it, we'll need the current
password for the root user.  If you've just installed MariaDB, and
you haven't set the root password yet, the password will be blank,
so you should just press enter here.

Enter current password for root (enter for none):
OK, successfully used password, moving on...

Setting the root password ensures that nobody can log into the MariaDB
root user without the proper authorisation.

You already have a root password set, so you can safely answer 'n'.

Change the root password? [Y/n] Y
New password:
Re-enter new password:
Password updated successfully!
Reloading privilege tables..
 ... Success!


By default, a MariaDB installation has an anonymous user, allowing anyone
to log into MariaDB without having to have a user account created for
them.  This is intended only for testing, and to make the installation
go a bit smoother.  You should remove them before moving into a
production environment.

Remove anonymous users? [Y/n] Y
 ... Success!

Normally, root should only be allowed to connect from 'localhost'.  This
ensures that someone cannot guess at the root password from the network.

Disallow root login remotely? [Y/n] Y
 ... Success!

By default, MariaDB comes with a database named 'test' that anyone can
access.  This is also intended only for testing, and should be removed
before moving into a production environment.

Remove test database and access to it? [Y/n] Y
 - Dropping test database...
 ... Success!
 - Removing privileges on test database...
 ... Success!

Reloading the privilege tables will ensure that all changes made so far
will take effect immediately.

Reload privilege tables now? [Y/n] Y
 ... Success!

Cleaning up...

All done!  If you've completed all of the above steps, your MariaDB
installation should now be secure.

Thanks for using MariaDB!

```

### master节点配置

在/etc/my.cnf文件的[mysqld]中添加如下内容

```vim
server-id=1
log-bin = /var/lib/mysql/mysql-bin
binlog-ignore-db=mysql
binlog-ignore-db=information_schema
binlog-ignore-db=performance_schema
auto-increment-increment = 2
auto-increment-offset = 1
```

重启db

```bash
systemctl restart mariadb
```

```bash
mysql -u root -p

GRANT REPLICATION SLAVE ON *.* TO 'replic_user'@'%' IDENTIFIED BY 'password';

FLUSH PRIVILEGES;

SHOW MASTER STATUS;

QUIT;
```

如果 master 中已经存在需要复制的数据，则需要dump出来，然后在slave上重放

```bash
mysql -u root -p

mysql> FLUSH TABLES WITH READ LOCK;

mysql> SHOW MASTER STATUS;

```

```bash
mysqldump -u root -p --databases [database-1] [database-2] ...  > /root/db_dump.sql

mysql -u root -p
mysql> UNLOCK TABLES;

```

### slave节点配置

如果master节点上dump了数据，则需要执行下面的步骤进行重放。

```bash
scp root@172.16.143.171:/root/db_dump.sql /root/db_dump.sql
mysql -u root -p < /root/db_dump.sql
```

在/etc/my.cnf文件的[mysqld]中添加如下内容

```vim
server-id=2
og-bin = /var/lib/mysql/mysql-bin
binlog-ignore-db=mysql
binlog-ignore-db=information_schema
binlog-ignore-db=performance_schema
auto-increment-increment = 2
auto-increment-offset = 2
```
重启db

```bash
systemctl restart mariadb
```

```bash
mysql -u root -p

mysql> CHANGE MASTER TO MASTER_HOST='172.16.143.171',MASTER_USER='replic_user', MASTER_PASSWORD='password', MASTER_LOG_FILE='mysql-bin.000001', MASTER_LOG_POS=107;
```
上面命令中<b>MASTER_LOG_FILE</b> 和 <b>MASTER_LOG_POS</b> 需要根据master节点中```show master status```命令结果调整。

```bash
START SLAVE;

SHOW SLAVE STATUS\G
```

## 测试
在master节点中创建db，也可以在slave节点中看到。
