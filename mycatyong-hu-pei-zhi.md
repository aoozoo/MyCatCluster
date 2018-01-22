```
#  vim /usr/local/mycat/conf/server.xml

        <user name="root"> 
                <property name="password">123456</property> 
                <property name="schemas">TESTDB</property> 

                <!-- 表级 DML 权限设置 --> 
                <!--             
                <privileges check="false"> 
                        <schema name="TESTDB" dml="0110" > 
                                <table name="tb01" dml="0000"></table> 
                                <table name="tb02" dml="1111"></table> 
                        </schema> 
                </privileges>            
                 --> 
        </user> 

        <user name="user"> 
                <property name="password">user</property> 
                <property name="schemas">TESTDB</property> 
                <property name="readOnly">true</property> 
        </user>
```

schema\(逻辑库\)--&gt;逻辑表--&gt;dataNode

dataNode\(name\)--&gt;dataHost\(DBName\)

dataHost\(ip,user,pass\)--&gt;



* MyCat指定健康检查的命令 select user\(\)



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

mysql>            # company 表一定要事先在mycat中定义

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



