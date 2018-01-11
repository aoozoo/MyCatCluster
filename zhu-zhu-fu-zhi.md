主主复制，是通过两台MySQL节点互为主备的方式来实现的，relay的SQL语句不会写入自己的二进制日志中。

原来从节点配置

```
grant replication slave on *.* to 'userslave1'@'192.168.183.%' identified by 'asdF';
sed -i '/\[mysqld\]/a\log-bin=/var/lib/mysql/mysql-bin' /etc/my.cnf
systemctl restart mysqld
```

原来主节点配置

```
sed -i '/\[mysqld\]/a\relay-log-index=/var/lib/mysql/relay-bin.index' /etc/my.cnf
sed -i '/\[mysqld\]/a\relay-log=/var/lib/mysql/relay-bin' /etc/my.cnf
systemctl restart mysqld

mysql> change master to master_host='192.168.183.102',master_port=3306,master_user='userslave1',master_password='asdF',master_log_file='mysql-bin.000001',master_log_pos=0;
mysql> start slave;
mysql> show slave status\G
```

MySQL 5.7解决了自增ID混乱的问题，如果是老版本的MySQL可在配置文件中配置如下两个参数来解决自增ID冲突的问题。

```
# 增长步长
auto-increment-increment=2
# 从一开始自增，另外一台需要配置从二开始自增。
auto-increment-offset=1
```

上面已经配置好了主从

在原来的从库插入数据

```
mysql> create database mytest;
mysql> create table t_order (id int primary key auto_increment,name varchar(10));
mysql> insert into t_order (id,name) values (null,'node102_1'),(null,'node102_2');
```

在原来的主库查询

```
mysql> select * from mytest.t_order;
+----+-----------+
| id | name      |
+----+-----------+
|  1 | node102_1 |
|  2 | node102_2 |
+----+-----------+
2 rows in set (0.00 sec)

mysql>
```

在原来的主库插入

```
mysql> insert into mytest.t_order (id,name) values (null,'node101_1'),(null,'node101_2');
```

在原来的从库查询

    mysql> select * from t_order;
    +----+-----------+
    | id | name      |
    +----+-----------+
    |  1 | node102_1 |
    |  2 | node102_2 |
    |  3 | node101_1 |
    |  4 | node101_2 |
    +----+-----------+
    4 rows in set (0.00 sec)

    mysql> 
    mysql> show create table t_order;
    +---------+----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
    | Table   | Create Table                                                                                                                                                                     |
    +---------+----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
    | t_order | CREATE TABLE `t_order` (
      `id` int(11) NOT NULL AUTO_INCREMENT,
      `name` varchar(10) DEFAULT NULL,
      PRIMARY KEY (`id`)
    ) ENGINE=InnoDB AUTO_INCREMENT=5 DEFAULT CHARSET=latin1 |
    +---------+----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
    1 row in set (0.00 sec)

    mysql> 
    # AUTO_INCREMENT=5 可以看到自增ID是同步自增的。



