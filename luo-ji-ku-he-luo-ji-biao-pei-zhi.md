schema\(逻辑库\)--&gt;逻辑表--&gt;dataNode

dataNode\(name\)--&gt;dataHost\(DBName\)

dataHost\(ip,user,pass\)--&gt;

* MyCat执行健康检查的命令 select user\(\)

* 修改mycat的配置文件，然后重启mycat生效

```xml
                <writeHost host="hostM1" url="192.168.183.102:3306" user="root"
                                   password="123456">
                        <!-- can have multi read hosts -->
                        <readHost host="hostS2" url="192.168.183.103:3306" user="root" password="123456" />
```

```bash
# 在 192.168.183.102 主数据库服务器，创建这三个库
mysql> create database db1;
Query OK, 1 row affected (0.02 sec)

mysql> create database db2;
Query OK, 1 row affected (0.01 sec)

mysql> create database db3;
Query OK, 1 row affected (0.01 sec)

mysql>
```

* mysql -h192.168.183.105 -uroot -p -P8066    \# 连接到mycat服务器

```
mysql> explain create table company (id int primary key auto_increment,name varchar(10));
+-----------+---------------------------------------------------------------------------+
| DATA_NODE | SQL                                                                       |
+-----------+---------------------------------------------------------------------------+
| dn1       | create table company (id int primary key auto_increment,name varchar(10)) |
| dn2       | create table company (id int primary key auto_increment,name varchar(10)) |
| dn3       | create table company (id int primary key auto_increment,name varchar(10)) |
+-----------+---------------------------------------------------------------------------+
3 rows in set (0.01 sec)

mysql>            # company 表一定要事先在mycat中定义   会同时在三个DataNode创建

mysql> create table company (id int primary key auto_increment,name varchar(10));
Query OK, 0 rows affected (0.32 sec)

mysql>
```

* 登录到103真实数据库服务器查看，每个库中都创建了 company 表 ，通过mycat查询只会查询到一条数据出来

```
mysql> use db1;
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Database changed
mysql> show tables;
+---------------+
| Tables_in_db1 |
+---------------+
| company       |
+---------------+
1 row in set (0.00 sec)

mysql> use db2;
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Database changed
mysql> show tables;
+---------------+
| Tables_in_db2 |
+---------------+
| company       |
+---------------+
1 row in set (0.00 sec)

mysql> use db3;
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Database changed
mysql> show tables;
+---------------+
| Tables_in_db3 |
+---------------+
| company       |
+---------------+
1 row in set (0.00 sec)

mysql>
```



