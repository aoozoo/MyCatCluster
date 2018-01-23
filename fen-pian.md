* mysql -h192.168.183.105 -uroot -p -P8066    \# 连接到mycat服务器   创建 employee  表，employee  表分片了。

```
mysql> explain create table employee (id int primary key auto_increment,name varchar(10));
+-----------+----------------------------------------------------------------------------+
| DATA_NODE | SQL                                                                        |
+-----------+----------------------------------------------------------------------------+
| dn1       | create table employee (id int primary key auto_increment,name varchar(10)) |
| dn2       | create table employee (id int primary key auto_increment,name varchar(10)) |
+-----------+----------------------------------------------------------------------------+
2 rows in set (0.01 sec)

mysql>

mysql> create table employee (id int primary key auto_increment,name varchar(10));
Query OK, 0 rows affected (0.15 sec)

mysql> explain insert into employee (name) values ('aa');
ERROR 1064 (HY000): bad insert sql (sharding column:SHARDING_ID not provided,INSERT INTO employee (name)
VALUES ('aa')
mysql> insert into employee (name) values ('aa');
ERROR 1064 (HY000): bad insert sql (sharding column:SHARDING_ID not provided,INSERT INTO employee (name)
VALUES ('aa')
mysql> alter table employee add column sharding_id int not null;
Query OK, 0 rows affected (0.21 sec)
Records: 0  Duplicates: 0  Warnings: 0

mysql> explain insert into employee (name,sharding_id) values ('aa',1);
ERROR 1064 (HY000): can't find any valid datanode :EMPLOYEE -> SHARDING_ID -> 1
mysql> explain insert into employee (name,sharding_id) values ('aa',10000);
+-----------+-------------------------------------------------------------+
| DATA_NODE | SQL                                                         |
+-----------+-------------------------------------------------------------+
| dn1       | insert into employee (name,sharding_id) values ('aa',10000) |
+-----------+-------------------------------------------------------------+
1 row in set (0.01 sec)

mysql> 
mysql> insert into employee (name,sharding_id) values ('aa',10000);
Query OK, 1 row affected (0.05 sec)

mysql>
```

employee 表  分片的配置

schema.xml

```
<table name="employee" primaryKey="ID" dataNode="dn1,dn2"
rule="sharding-by-intfile" />
```

rule.xml

```
        <tableRule name="sharding-by-intfile">
                <rule>
                        <columns>sharding_id</columns>
                        <algorithm>hash-int</algorithm>
                </rule>
        </tableRule>
```

partition-hash-int.txt

```
[root@node_105_192.168.183.105 /usr/local/mycat/conf]#  cat partition-hash-int.txt
10000=0
10010=1
```

partition-hash-int.txt  中定义了分片的匹配条件

