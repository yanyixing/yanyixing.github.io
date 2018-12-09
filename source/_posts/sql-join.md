---
title: SQL Join
date: 2018-12-09 15:35:21
tags: sql
---

现在有两张表，user和class，内容如下：

```
MariaDB [jointest]> select * from user;
+------+------+----------+
| id   | name | class_id |
+------+------+----------+
| 1    | aaa  | 1        |
| 2    | bbb  | 1        |
| 3    | ccc  | 1        |
| 4    | ddd  | 2        |
| 5    | eee  | 2        |
| 6    | fff  | 3        |
| 7    | ggg  | 7        |
| 8    | hhh  | 9        |
+------+------+----------+
8 rows in set (0.00 sec)

MariaDB [jointest]> select * from class;
+------+--------+
| id   | name   |
+------+--------+
| 1    | class1 |
| 2    | class2 |
| 3    | class3 |
| 4    | class4 |
| 5    | class5 |
+------+--------+
5 rows in set (0.00 sec)

```

# inner join

```
MariaDB [jointest]> select * from user inner join class on user.class_id=class.id;
+------+------+----------+------+--------+
| id   | name | class_id | id   | name   |
+------+------+----------+------+--------+
| 1    | aaa  | 1        | 1    | class1 |
| 2    | bbb  | 1        | 1    | class1 |
| 3    | ccc  | 1        | 1    | class1 |
| 4    | ddd  | 2        | 2    | class2 |
| 5    | eee  | 2        | 2    | class2 |
| 6    | fff  | 3        | 3    | class3 |
+------+------+----------+------+--------+
6 rows in set (0.00 sec)
```

# left join

```
MariaDB [jointest]> select * from user left join class on user.class_id=class.id;
+------+------+----------+------+--------+
| id   | name | class_id | id   | name   |
+------+------+----------+------+--------+
| 1    | aaa  | 1        | 1    | class1 |
| 2    | bbb  | 1        | 1    | class1 |
| 3    | ccc  | 1        | 1    | class1 |
| 4    | ddd  | 2        | 2    | class2 |
| 5    | eee  | 2        | 2    | class2 |
| 6    | fff  | 3        | 3    | class3 |
| 7    | ggg  | 7        | NULL | NULL   |
| 8    | hhh  | 9        | NULL | NULL   |
+------+------+----------+------+--------+
8 rows in set (0.00 sec)
```

# right join

```
MariaDB [jointest]> select * from user right join class on user.class_id=class.id;
+------+------+----------+------+--------+
| id   | name | class_id | id   | name   |
+------+------+----------+------+--------+
| 1    | aaa  | 1        | 1    | class1 |
| 2    | bbb  | 1        | 1    | class1 |
| 3    | ccc  | 1        | 1    | class1 |
| 4    | ddd  | 2        | 2    | class2 |
| 5    | eee  | 2        | 2    | class2 |
| 6    | fff  | 3        | 3    | class3 |
| NULL | NULL | NULL     | 4    | class4 |
| NULL | NULL | NULL     | 5    | class5 |
+------+------+----------+------+--------+
8 rows in set (0.00 sec)
```

# full join

mysql不知吃full join，不过可以通过union 合并left jion和right jion的结果来模拟full jion。

```
MariaDB [jointest]> select * from user left join class on user.class_id=class.id
    -> union
    -> select * from user right join class on user.class_id=class.id;
+------+------+----------+------+--------+
| id   | name | class_id | id   | name   |
+------+------+----------+------+--------+
| 1    | aaa  | 1        | 1    | class1 |
| 2    | bbb  | 1        | 1    | class1 |
| 3    | ccc  | 1        | 1    | class1 |
| 4    | ddd  | 2        | 2    | class2 |
| 5    | eee  | 2        | 2    | class2 |
| 6    | fff  | 3        | 3    | class3 |
| 7    | ggg  | 7        | NULL | NULL   |
| 8    | hhh  | 9        | NULL | NULL   |
| NULL | NULL | NULL     | 4    | class4 |
| NULL | NULL | NULL     | 5    | class5 |
+------+------+----------+------+--------+
10 rows in set (0.00 sec)
```

# cross join

user表一共有8条记录，class表一共有5条记录，cross join一同有8*5=40条结果。

```
MariaDB [jointest]> select * from user cross join class;
+------+------+----------+------+--------+
| id   | name | class_id | id   | name   |
+------+------+----------+------+--------+
| 1    | aaa  | 1        | 1    | class1 |
| 1    | aaa  | 1        | 2    | class2 |
| 1    | aaa  | 1        | 3    | class3 |
| 1    | aaa  | 1        | 4    | class4 |
| 1    | aaa  | 1        | 5    | class5 |
| 2    | bbb  | 1        | 1    | class1 |
| 2    | bbb  | 1        | 2    | class2 |
| 2    | bbb  | 1        | 3    | class3 |
| 2    | bbb  | 1        | 4    | class4 |
| 2    | bbb  | 1        | 5    | class5 |
| 3    | ccc  | 1        | 1    | class1 |
| 3    | ccc  | 1        | 2    | class2 |
| 3    | ccc  | 1        | 3    | class3 |
| 3    | ccc  | 1        | 4    | class4 |
| 3    | ccc  | 1        | 5    | class5 |
| 4    | ddd  | 2        | 1    | class1 |
| 4    | ddd  | 2        | 2    | class2 |
| 4    | ddd  | 2        | 3    | class3 |
| 4    | ddd  | 2        | 4    | class4 |
| 4    | ddd  | 2        | 5    | class5 |
| 5    | eee  | 2        | 1    | class1 |
| 5    | eee  | 2        | 2    | class2 |
| 5    | eee  | 2        | 3    | class3 |
| 5    | eee  | 2        | 4    | class4 |
| 5    | eee  | 2        | 5    | class5 |
| 6    | fff  | 3        | 1    | class1 |
| 6    | fff  | 3        | 2    | class2 |
| 6    | fff  | 3        | 3    | class3 |
| 6    | fff  | 3        | 4    | class4 |
| 6    | fff  | 3        | 5    | class5 |
| 7    | ggg  | 7        | 1    | class1 |
| 7    | ggg  | 7        | 2    | class2 |
| 7    | ggg  | 7        | 3    | class3 |
| 7    | ggg  | 7        | 4    | class4 |
| 7    | ggg  | 7        | 5    | class5 |
| 8    | hhh  | 9        | 1    | class1 |
| 8    | hhh  | 9        | 2    | class2 |
| 8    | hhh  | 9        | 3    | class3 |
| 8    | hhh  | 9        | 4    | class4 |
| 8    | hhh  | 9        | 5    | class5 |
+------+------+----------+------+--------+
40 rows in set (0.01 sec)
```


# mysqldump

```
-- MySQL dump 10.14  Distrib 5.5.60-MariaDB, for Linux (x86_64)
--
-- Host: localhost    Database: jointest
-- ------------------------------------------------------
-- Server version       5.5.60-MariaDB

/*!40101 SET @OLD_CHARACTER_SET_CLIENT=@@CHARACTER_SET_CLIENT */;
/*!40101 SET @OLD_CHARACTER_SET_RESULTS=@@CHARACTER_SET_RESULTS */;
/*!40101 SET @OLD_COLLATION_CONNECTION=@@COLLATION_CONNECTION */;
/*!40101 SET NAMES utf8 */;
/*!40103 SET @OLD_TIME_ZONE=@@TIME_ZONE */;
/*!40103 SET TIME_ZONE='+00:00' */;
/*!40014 SET @OLD_UNIQUE_CHECKS=@@UNIQUE_CHECKS, UNIQUE_CHECKS=0 */;
/*!40014 SET @OLD_FOREIGN_KEY_CHECKS=@@FOREIGN_KEY_CHECKS, FOREIGN_KEY_CHECKS=0 */;
/*!40101 SET @OLD_SQL_MODE=@@SQL_MODE, SQL_MODE='NO_AUTO_VALUE_ON_ZERO' */;
/*!40111 SET @OLD_SQL_NOTES=@@SQL_NOTES, SQL_NOTES=0 */;

--
-- Current Database: `jointest`
--

CREATE DATABASE /*!32312 IF NOT EXISTS*/ `jointest` /*!40100 DEFAULT CHARACTER SET latin1 */;

USE `jointest`;

--
-- Table structure for table `class`
--

DROP TABLE IF EXISTS `class`;
/*!40101 SET @saved_cs_client     = @@character_set_client */;
/*!40101 SET character_set_client = utf8 */;
CREATE TABLE `class` (
  `id` varchar(10) DEFAULT NULL,
  `name` varchar(20) DEFAULT NULL
) ENGINE=InnoDB DEFAULT CHARSET=latin1;
/*!40101 SET character_set_client = @saved_cs_client */;

--
-- Dumping data for table `class`
--

LOCK TABLES `class` WRITE;
/*!40000 ALTER TABLE `class` DISABLE KEYS */;
INSERT INTO `class` VALUES ('1','class1'),('2','class2'),('3','class3'),('4','class4'),('5','class5');
/*!40000 ALTER TABLE `class` ENABLE KEYS */;
UNLOCK TABLES;

--
-- Table structure for table `user`
--

DROP TABLE IF EXISTS `user`;
/*!40101 SET @saved_cs_client     = @@character_set_client */;
/*!40101 SET character_set_client = utf8 */;
CREATE TABLE `user` (
  `id` varchar(10) DEFAULT NULL,
  `name` varchar(20) DEFAULT NULL,
  `class_id` varchar(10) DEFAULT NULL
) ENGINE=InnoDB DEFAULT CHARSET=latin1;
/*!40101 SET character_set_client = @saved_cs_client */;

--
-- Dumping data for table `user`
--

LOCK TABLES `user` WRITE;
/*!40000 ALTER TABLE `user` DISABLE KEYS */;
INSERT INTO `user` VALUES ('1','aaa','1'),('2','bbb','1'),('3','ccc','1'),('4','ddd','2'),('5','eee','2'),('6','fff','3'),('7','ggg','7'),('8','hhh','9');
/*!40000 ALTER TABLE `user` ENABLE KEYS */;
UNLOCK TABLES;
/*!40103 SET TIME_ZONE=@OLD_TIME_ZONE */;

/*!40101 SET SQL_MODE=@OLD_SQL_MODE */;
/*!40014 SET FOREIGN_KEY_CHECKS=@OLD_FOREIGN_KEY_CHECKS */;
/*!40014 SET UNIQUE_CHECKS=@OLD_UNIQUE_CHECKS */;
/*!40101 SET CHARACTER_SET_CLIENT=@OLD_CHARACTER_SET_CLIENT */;
/*!40101 SET CHARACTER_SET_RESULTS=@OLD_CHARACTER_SET_RESULTS */;
/*!40101 SET COLLATION_CONNECTION=@OLD_COLLATION_CONNECTION */;
/*!40111 SET SQL_NOTES=@OLD_SQL_NOTES */;

-- Dump completed on 2018-12-09 17:54:20
```

# 参考
- https://www.oschina.net/translate/mysql-joins-on-vs-using-vs-theta-style
- https://www.cnblogs.com/BeginMan/p/3754322.html
- http://blog.bittiger.io/post198/
