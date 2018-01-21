### 数据库文件

---

* InnoDB

> .frm         表的结构文件
>
> .idb         表的数据文件

* MyISAM

> .frm         表的结构文件
>
> .MYD       表的数据文件
>
> .MYI        表的索引文件

### 表空间

---

* 共享表空

> 所有的表都保存在一个idb文件中

* 独享表空间

> 每个表保存在一个单独的idb文件

### 表分区

---

> 表分区属于，物理分表。数据库通过策略，将一张表的数据，保存到多个idb文件中。
>
> 逻辑分表的策略是通过应用程序或者中间件来实现的，表分区的分表策略是通过数据库来实现的。
>
> 数据库支持四种策略来创建表分区：Range、List、Hash、Key
>
> **Range：**比如 ID 1-100万存在第一个文件，101-200万存在第二个文件中，201-300万存在第三个文件中
>
> **Lisit：**比如北京，上海，深圳保存到第一个文件，湖南，湖北，河南保存到第二个文件，Range指定的是范围，而List指定的就是一些值。
>
> **Hash：**Hash分区主要用来分散热点读，确保数据在预先确定个数的分区中尽可能平均分布。
>
> **Key：**类似Hash分区，Hash分区允许使用用户自定义的表达式，但Key分区不允许使用用户自定义的表达式。Hash仅支持整数分区，而Key分区支持除了Blob和text的其他类型的列作为分区键。

* 请参考：[http://www.notehub.cn/2016/03/20/dev/mysql\_partition/](http://www.notehub.cn/2016/03/20/dev/mysql_partition/)

### 使用range表分区

---

##### 创建一个400万条数据的表

```sql
mysql> usw mytest;
mysql> create table t_base(id int primary key auto_increment,name varchar(10));
mysql> insert into t_base(name) values ('Jason');
mysql> insert into t_base(name) select name from t_base;
mysql> insert into t_base(name) select name from t_base;
mysql> insert into t_base(name) select name from t_base;
...
mysql> insert into t_base(name) select name from t_base;
Query OK, 2105344 rows affected (31.35 sec)
Records: 2105344  Duplicates: 0  Warnings: 0

mysql> 
# 一直重复插入比较多的数据到表中。上面已经插入了420多万的数据
```

#### 创建range分区的表

```sql
mysql> set global innodb_file_per_table=1;          # 先设置使用独立的表空间
mysql> create table t_range(id int primary key auto_increment,name varchar(10)) partition by range(id) (partition p0 values less than (1000000),partition p1 values less than (2000000),partition p2 values less than (3000000),partition p3 values less than (4000000),partition p4 values less than maxvalue);
[root@node_101_192.168.183.101 ~]#  ls /var/lib/mysql/mytest/ -lh
total 137M
-rw-r----- 1 mysql mysql   65 Jan  8 05:54 db.opt
-rw-r----- 1 mysql mysql 8.4K Jan 16 05:04 t_base.frm
-rw-r----- 1 mysql mysql 136M Jan 16 05:12 t_base.ibd
-rw-r----- 1 mysql mysql 8.4K Jan  8 05:54 t_order.frm
-rw-r----- 1 mysql mysql  96K Jan  8 05:55 t_order.ibd
-rw-r----- 1 mysql mysql 8.4K Jan 16 06:13 t_range.frm
-rw-r----- 1 mysql mysql  96K Jan 16 06:13 t_range#P#p0.ibd
-rw-r----- 1 mysql mysql  96K Jan 16 06:13 t_range#P#p1.ibd
-rw-r----- 1 mysql mysql  96K Jan 16 06:13 t_range#P#p2.ibd
-rw-r----- 1 mysql mysql  96K Jan 16 06:13 t_range#P#p3.ibd
-rw-r----- 1 mysql mysql  96K Jan 16 06:13 t_range#P#p4.ibd
[root@node_101_192.168.183.101 ~]#

mysql> insert into t_range(name) select name from t_base;           # 将t_base表中的400多万条数据插入到表中

[root@node_101_192.168.183.101 ~]#  ls /var/lib/mysql/mytest/ -lh
total 311M
-rw-r----- 1 mysql mysql   65 Jan  8 05:54 db.opt
-rw-r----- 1 mysql mysql 8.4K Jan 16 05:04 t_base.frm
-rw-r----- 1 mysql mysql 136M Jan 16 05:12 t_base.ibd
-rw-r----- 1 mysql mysql 8.4K Jan  8 05:54 t_order.frm
-rw-r----- 1 mysql mysql  96K Jan  8 05:55 t_order.ibd
-rw-r----- 1 mysql mysql 8.4K Jan 16 06:13 t_range.frm
-rw-r----- 1 mysql mysql  40M Jan 16 06:20 t_range#P#p0.ibd        # 400万条数据保存到了不同的文件中
-rw-r----- 1 mysql mysql  40M Jan 16 06:20 t_range#P#p1.ibd
-rw-r----- 1 mysql mysql  40M Jan 16 06:21 t_range#P#p2.ibd
-rw-r----- 1 mysql mysql  40M Jan 16 06:21 t_range#P#p3.ibd
-rw-r----- 1 mysql mysql  14M Jan 16 06:21 t_range#P#p4.ibd
[root@node_101_192.168.183.101 ~]#
```

#### 验证range表分区中的数据

```sql
mysql> select * from t_range limit 3;
+----+-------+
| id | name  |
+----+-------+
|  1 | Jason |
|  2 | Jason |
|  3 | Jason |
+----+-------+
3 rows in set (0.00 sec)

mysql> alter table t_range drop partition p0;      # 删除partition p0中的数据，再查看数据从100万开始了，说明数据分区存放正确。
Query OK, 0 rows affected (0.13 sec)
Records: 0  Duplicates: 0  Warnings: 0

mysql> select * from t_range limit 3;
+---------+-------+
| id      | name  |
+---------+-------+
| 1000000 | Jason |
| 1000001 | Jason |
| 1000002 | Jason |
+---------+-------+
3 rows in set (0.00 sec)

mysql> 
[root@node_101_192.168.183.101 ~]#  ls /var/lib/mysql/mytest/ -lh
total 271M
-rw-r----- 1 mysql mysql   65 Jan  8 05:54 db.opt
-rw-r----- 1 mysql mysql 8.4K Jan 16 05:04 t_base.frm
-rw-r----- 1 mysql mysql 136M Jan 16 05:12 t_base.ibd
-rw-r----- 1 mysql mysql 8.4K Jan  8 05:54 t_order.frm
-rw-r----- 1 mysql mysql  96K Jan  8 05:55 t_order.ibd
-rw-r----- 1 mysql mysql 8.4K Jan 16 06:25 t_range.frm
-rw-r----- 1 mysql mysql  40M Jan 16 06:20 t_range#P#p1.ibd
-rw-r----- 1 mysql mysql  40M Jan 16 06:21 t_range#P#p2.ibd
-rw-r----- 1 mysql mysql  40M Jan 16 06:21 t_range#P#p3.ibd
-rw-r----- 1 mysql mysql  14M Jan 16 06:21 t_range#P#p4.ibd
[root@node_101_192.168.183.101 ~]#
```

### 使用list表分区

```sql
mysql> create table t_list(id int primary key auto_increment,province int) partition by list(id)(partition p0 values in(1,2,3),partition p1 values in(4,5,6),partition p2 values in(7,8,9));
Query OK, 0 rows affected (0.14 sec)

mysql>
mysql> insert into t_list(id,province) values (null,1),(null,2),(null,3);
Query OK, 3 rows affected (0.02 sec)
Records: 3  Duplicates: 0  Warnings: 0

mysql> insert into t_list(id,province) values (null,4),(null,5),(null,6),(null,7),(null,8),(null,9);
Query OK, 6 rows affected (0.02 sec)
Records: 6  Duplicates: 0  Warnings: 0

mysql> select * from t_list;
+----+----------+
| id | province |
+----+----------+
|  1 |        1 |
|  2 |        2 |
|  3 |        3 |
|  4 |        4 |
|  5 |        5 |
|  6 |        6 |
|  7 |        7 |
|  8 |        8 |
|  9 |        9 |
+----+----------+
9 rows in set (0.00 sec)

mysql>
# 可以看到有9条数据，保存在三个文件中
[root@node_101_192.168.183.101 ~]#  ls /var/lib/mysql/mytest/ -lh | grep t_list
-rw-r----- 1 mysql mysql 8.4K Jan 16 06:46 t_list.frm
-rw-r----- 1 mysql mysql  96K Jan 16 06:48 t_list#P#p0.ibd
-rw-r----- 1 mysql mysql  96K Jan 16 06:48 t_list#P#p1.ibd
-rw-r----- 1 mysql mysql  96K Jan 16 06:48 t_list#P#p2.ibd
[root@node_101_192.168.183.101 ~]#


mysql> alter table t_list drop partition p1;
Query OK, 0 rows affected (0.09 sec)
Records: 0  Duplicates: 0  Warnings: 0

mysql> select * from t_list;
+----+----------+
| id | province |
+----+----------+
|  1 |        1 |
|  2 |        2 |
|  3 |        3 |
|  7 |        7 |
|  8 |        8 |
|  9 |        9 |
+----+----------+
6 rows in set (0.00 sec)

mysql> 
[root@node_101_192.168.183.101 ~]#  ls /var/lib/mysql/mytest/ -lh | grep t_list
-rw-r----- 1 mysql mysql 8.4K Jan 16 06:49 t_list.frm
-rw-r----- 1 mysql mysql  96K Jan 16 06:48 t_list#P#p0.ibd
-rw-r----- 1 mysql mysql  96K Jan 16 06:48 t_list#P#p2.ibd
[root@node_101_192.168.183.101 ~]#    # 看到数据分区是正确的。
```

### 使用hash表分区

```sql
mysql> create table t_hash(id int primary key auto_increment,name varchar(10)) partition by hash(id) partitions 4;
Query OK, 0 rows affected (0.11 sec)
# 需要注意的是，使用hash分区的必须是主键而且是int类型
mysql>
[root@node_101_192.168.183.101 ~]#  ls /var/lib/mysql/mytest/ -lh | grep t_hash
-rw-r----- 1 mysql mysql 8.4K Jan 16 06:57 t_hash.frm
-rw-r----- 1 mysql mysql  96K Jan 16 06:57 t_hash#P#p0.ibd
-rw-r----- 1 mysql mysql  96K Jan 16 06:57 t_hash#P#p1.ibd
-rw-r----- 1 mysql mysql  96K Jan 16 06:57 t_hash#P#p2.ibd
-rw-r----- 1 mysql mysql  96K Jan 16 06:57 t_hash#P#p3.ibd
[root@node_101_192.168.183.101 ~]#

# 插入400万条数据
mysql> insert into t_hash (name) select name from t_base;

[root@node_101_192.168.183.101 ~]#  ls /var/lib/mysql/mytest/ -lh | grep t_hash
-rw-r----- 1 mysql mysql 8.4K Jan 16 06:57 t_hash.frm
-rw-r----- 1 mysql mysql 9.0M Jan 16 06:58 t_hash#P#p0.ibd
-rw-r----- 1 mysql mysql 9.0M Jan 16 06:58 t_hash#P#p1.ibd
-rw-r----- 1 mysql mysql 9.0M Jan 16 06:58 t_hash#P#p2.ibd
-rw-r----- 1 mysql mysql 9.0M Jan 16 06:58 t_hash#P#p3.ibd
[root@node_101_192.168.183.101 ~]#  ls /var/lib/mysql/mytest/ -lh | grep t_hash
-rw-r----- 1 mysql mysql 8.4K Jan 16 06:57 t_hash.frm
-rw-r----- 1 mysql mysql  11M Jan 16 06:58 t_hash#P#p0.ibd
-rw-r----- 1 mysql mysql  11M Jan 16 06:58 t_hash#P#p1.ibd
-rw-r----- 1 mysql mysql  11M Jan 16 06:58 t_hash#P#p2.ibd
-rw-r----- 1 mysql mysql  11M Jan 16 06:58 t_hash#P#p3.ibd
[root@node_101_192.168.183.101 ~]#     # 可以看到hash中的数据是同时增长的，hash能够将数据平稳的存放到各个表分区文件中
[root@node_101_192.168.183.101 ~]#  ls /var/lib/mysql/mytest/ -lh | grep t_hash
-rw-r----- 1 mysql mysql 8.4K Jan 16 06:57 t_hash.frm
-rw-r----- 1 mysql mysql  40M Jan 16 06:59 t_hash#P#p0.ibd
-rw-r----- 1 mysql mysql  40M Jan 16 06:59 t_hash#P#p1.ibd
-rw-r----- 1 mysql mysql  40M Jan 16 06:59 t_hash#P#p2.ibd
-rw-r----- 1 mysql mysql  40M Jan 16 06:59 t_hash#P#p3.ibd
[root@node_101_192.168.183.101 ~]#
```



