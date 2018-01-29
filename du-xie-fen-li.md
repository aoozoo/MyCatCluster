首先配置好MySQL 主从，MyCat 不负责数据同步问题

vim /usr/local/mycat/conf/schema.xml

```
        <dataHost name="localhost1" maxCon="1000" minCon="10" balance="3"
                          writeType="0" dbType="mysql" dbDriver="native" switchType="1"  slaveThreshold="100">
                <heartbeat>select user()</heartbeat>
                <!-- can have multi write hosts -->
                <writeHost host="hostM1" url="192.168.183.102:3306" user="root"
                                   password="123456">
                        <!-- can have multi read hosts -->
                        <readHost host="hostS2" url="192.168.183.103:3306" user="root" password="123456" />
                </writeHost>
                <!-- <writeHost host="hostS1" url="localhost:3316" user="root"
                                   password="123456" />   -->
                <!-- <writeHost host="hostM2" url="localhost:3316" user="root" password="123456"/> -->
        </dataHost>
```

balance 属性，负载均衡类型，目前的值有3种：

1. balance="0", 不开启读写分离机制，所有读操作都发送到当前可用的writeHost上。

2. balance="1"，全部的readHost与stand by writeHost参与select询句的负载均衡，简单的说，当双主双从模式\(M1-&gt;S1，M2-&gt;S2，并且M1与 M2互为主备\)，正常情况下，M2,S1,S2都参不select语句的负载均衡。

3. balance="2"，所有读操作都随机的在writeHost、readhost上分发。

4. balance="3"，所有读请求随机的分发到wiriterHost对应的readhost执行，writerHost不负担读压力，注意balance=3

参考系列文档：[http://blog.csdn.net/dream\_broken/article/details/77866342](http://blog.csdn.net/dream_broken/article/details/77866342)

* 启动MyCat 通过console的方式

> /usr/local/mycat/bin/mycat console

```
mysql> explain  insert into company (id,name) values (null,'m');
+-----------+-------------------------------------------------+
| DATA_NODE | SQL                                             |
+-----------+-------------------------------------------------+
| dn1       | insert into company (id,name) values (null,'m') |
| dn2       | insert into company (id,name) values (null,'m') |
| dn3       | insert into company (id,name) values (null,'m') |
+-----------+-------------------------------------------------+
3 rows in set (0.01 sec)

mysql>     # 三个datanode都写如，这个取决于表的策略配置
mysql> explain select * from company;
+-----------+---------------------------------+
| DATA_NODE | SQL                             |
+-----------+---------------------------------+
| dn1       | SELECT * FROM company LIMIT 100 |
+-----------+---------------------------------+
1 row in set (0.00 sec)

mysql>
```

**为了查看select 语句从哪个节点读取，打开日志级别为debug**

/usr/local/mycat/conf/log4j2.xml

level="info"  level="debug"

#### 重启MyCat

/usr/local/mycat/bin/mycat stop

/usr/local/mycat/bin/mycat start

```
mysql> explain insert into company (id,name) values (null,'bbggg');
+-----------+-----------------------------------------------------+
| DATA_NODE | SQL                                                 |
+-----------+-----------------------------------------------------+
| dn1       | insert into company (id,name) values (null,'bbggg') |
| dn2       | insert into company (id,name) values (null,'bbggg') |
| dn3       | insert into company (id,name) values (null,'bbggg') |
+-----------+-----------------------------------------------------+
3 rows in set (0.03 sec)

mysql> 
```

tail -f /usr/local/mycat/logs/wrapper.log

```
INFO   | jvm 1    | 2018/01/29 23:37:58 | 2018-01-29 23:37:58,917 [DEBUG][$_NIOREACTOR-1-RW] ServerConnection [id=1, schema=TESTDB, host=192.168.183.101, user=root,txIsolation=3, autocommit=true, schema=TESTDB] explain insert into company (id,name) values (null,'bbggg')  (io.mycat.net.FrontendConnection:FrontendConnection.java:288) 
INFO   | jvm 1    | 2018/01/29 23:37:58 | 2018-01-29 23:37:58,917 [DEBUG][$_NIOREACTOR-1-RW] ServerConnection [id=1, schema=TESTDB, host=192.168.183.101, user=root,txIsolation=3, autocommit=true, schema=TESTDB]explain insert into company (id,name) values (null,'bbggg')  (io.mycat.server.ServerQueryHandler:ServerQueryHandler.java:57)
```

```
mysql> explain select * from company;
+-----------+---------------------------------+
| DATA_NODE | SQL                             |
+-----------+---------------------------------+
| dn3       | SELECT * FROM company LIMIT 100 |
+-----------+---------------------------------+
1 row in set (0.01 sec)

mysql>
```

tail -f /usr/local/mycat/logs/wrapper.log

```
INFO   | jvm 1    | 2018/01/29 23:38:38 | 2018-01-29 23:38:38,730 [DEBUG][$_NIOREACTOR-1-RW] ServerConnection [id=1, schema=TESTDB, host=192.168.183.101, user=root,txIsolation=3, autocommit=true, schema=TESTDB] explain select * from company  (io.mycat.net.FrontendConnection:FrontendConnection.java:288) 
INFO   | jvm 1    | 2018/01/29 23:38:38 | 2018-01-29 23:38:38,730 [DEBUG][$_NIOREACTOR-1-RW] ServerConnection [id=1, schema=TESTDB, host=192.168.183.101, user=root,txIsolation=3, autocommit=true, schema=TESTDB]explain select * from company  (io.mycat.server.ServerQueryHandler:ServerQueryHandler.java:57) 
INFO   | jvm 1    | 2018/01/29 23:38:38 | 2018-01-29 23:38:38,732 [DEBUG][$_NIOREACTOR-1-RW] SQLRouteCache  miss cache ,key:TESTDBselect * from company  (io.mycat.cache.impl.EnchachePool:EnchachePool.java:77) 

```

```
mysql> insert into company (id,name) values (null,'bbggg');
Query OK, 1 row affected (0.07 sec)

mysql> 
```

tail -f /usr/local/mycat/logs/wrapper.log

```
INFO   | jvm 1    | 2018/01/29 23:39:30 | 2018-01-29 23:39:30,911 [DEBUG][$_NIOREACTOR-1-RW] ServerConnection [id=1, schema=TESTDB, host=192.168.183.101, user=root,txIsolation=3, autocommit=true, schema=TESTDB] insert into company (id,name) values (null,'bbggg')  (io.mycat.net.FrontendConnection:FrontendConnection.java:288) 
INFO   | jvm 1    | 2018/01/29 23:39:30 | 2018-01-29 23:39:30,912 [DEBUG][$_NIOREACTOR-1-RW] ServerConnection [id=1, schema=TESTDB, host=192.168.183.101, user=root,txIsolation=3, autocommit=true, schema=TESTDB]insert into company (id,name) values (null,'bbggg')  (io.mycat.server.ServerQueryHandler:ServerQueryHandler.java:57) 
INFO   | jvm 1    | 2018/01/29 23:39:30 | 2018-01-29 23:39:30,913 [DEBUG][$_NIOREACTOR-1-RW] ServerConnection [id=1, schema=TESTDB, host=192.168.183.101, user=root,txIsolation=3, autocommit=true, schema=TESTDB]insert into company (id,name) values (null,'bbggg'), route={
INFO   | jvm 1    | 2018/01/29 23:39:30 |    1 -> dn1{insert into company (id,name) values (null,'bbggg')}
INFO   | jvm 1    | 2018/01/29 23:39:30 |    2 -> dn2{insert into company (id,name) values (null,'bbggg')}
INFO   | jvm 1    | 2018/01/29 23:39:30 |    3 -> dn3{insert into company (id,name) values (null,'bbggg')}
INFO   | jvm 1    | 2018/01/29 23:39:30 | } rrs   (io.mycat.server.NonBlockingSession:NonBlockingSession.java:110) 
INFO   | jvm 1    | 2018/01/29 23:39:30 | 2018-01-29 23:39:30,927 [DEBUG][$_NIOREACTOR-1-RW] execute mutinode query insert into company (id,name) values (null,'bbggg')  (io.mycat.backend.mysql.nio.handler.MultiNodeQueryHandler:MultiNodeQueryHandler.java:101) 
INFO   | jvm 1    | 2018/01/29 23:39:30 | 2018-01-29 23:39:30,927 [DEBUG][$_NIOREACTOR-1-RW] rrs.getRunOnSlave()-null  (io.mycat.backend.mysql.nio.handler.MultiNodeQueryHandler:MultiNodeQueryHandler.java:170) 
INFO   | jvm 1    | 2018/01/29 23:39:30 | 2018-01-29 23:39:30,928 [DEBUG][$_NIOREACTOR-1-RW] node.getRunOnSlave()1-null  (io.mycat.backend.mysql.nio.handler.MultiNodeQueryHandler:MultiNodeQueryHandler.java:180) 
INFO   | jvm 1    | 2018/01/29 23:39:30 | 2018-01-29 23:39:30,928 [DEBUG][$_NIOREACTOR-1-RW] node.getRunOnSlave()2-null  (io.mycat.backend.mysql.nio.handler.MultiNodeQueryHandler:MultiNodeQueryHandler.java:182) 
INFO   | jvm 1    | 2018/01/29 23:39:30 | 2018-01-29 23:39:30,928 [DEBUG][$_NIOREACTOR-1-RW] rrs.getRunOnSlave() null  (io.mycat.backend.datasource.PhysicalDBNode:PhysicalDBNode.java:96) 
INFO   | jvm 1    | 2018/01/29 23:39:30 | 2018-01-29 23:39:30,928 [DEBUG][$_NIOREACTOR-1-RW] rrs.getRunOnSlave() null  (io.mycat.backend.datasource.PhysicalDBNode:PhysicalDBNode.java:127) 
INFO   | jvm 1    | 2018/01/29 23:39:30 | 2018-01-29 23:39:30,928 [DEBUG][$_NIOREACTOR-1-RW] con need syn ,total syn cmd 1 commands SET names utf8;schema change:false con:MySQLConnection [id=4, lastTime=1517240370928, user=root, schema=db1, old shema=db1, borrowed=true, fromSlaveDB=false, threadId=568, charset=utf8, txIsolation=3, autocommit=true, attachment=dn1{insert into company (id,name) values (null,'bbggg')}, respHandler=io.mycat.backend.mysql.nio.handler.MultiNodeQueryHandler@3989b923, host=192.168.183.102, port=3306, statusSync=null, writeQueue=0, modifiedSQLExecuted=true]  (io.mycat.backend.mysql.nio.MySQLConnection:MySQLConnection.java:448) 
INFO   | jvm 1    | 2018/01/29 23:39:30 | 2018-01-29 23:39:30,929 [DEBUG][$_NIOREACTOR-1-RW] node.getRunOnSlave()1-null  (io.mycat.backend.mysql.nio.handler.MultiNodeQueryHandler:MultiNodeQueryHandler.java:180) 
INFO   | jvm 1    | 2018/01/29 23:39:30 | 2018-01-29 23:39:30,929 [DEBUG][$_NIOREACTOR-1-RW] node.getRunOnSlave()2-null  (io.mycat.backend.mysql.nio.handler.MultiNodeQueryHandler:MultiNodeQueryHandler.java:182) 
INFO   | jvm 1    | 2018/01/29 23:39:30 | 2018-01-29 23:39:30,929 [DEBUG][$_NIOREACTOR-1-RW] rrs.getRunOnSlave() null  (io.mycat.backend.datasource.PhysicalDBNode:PhysicalDBNode.java:96) 
INFO   | jvm 1    | 2018/01/29 23:39:30 | 2018-01-29 23:39:30,929 [DEBUG][$_NIOREACTOR-1-RW] rrs.getRunOnSlave() null  (io.mycat.backend.datasource.PhysicalDBNode:PhysicalDBNode.java:127) 
INFO   | jvm 1    | 2018/01/29 23:39:30 | 2018-01-29 23:39:30,929 [DEBUG][$_NIOREACTOR-1-RW] con need syn ,total syn cmd 1 commands SET names utf8;schema change:false con:MySQLConnection [id=3, lastTime=1517240370929, user=root, schema=db2, old shema=db2, borrowed=true, fromSlaveDB=false, threadId=566, charset=utf8, txIsolation=3, autocommit=true, attachment=dn2{insert into company (id,name) values (null,'bbggg')}, respHandler=io.mycat.backend.mysql.nio.handler.MultiNodeQueryHandler@3989b923, host=192.168.183.102, port=3306, statusSync=null, writeQueue=0, modifiedSQLExecuted=true]  (io.mycat.backend.mysql.nio.MySQLConnection:MySQLConnection.java:448) 
INFO   | jvm 1    | 2018/01/29 23:39:30 | 2018-01-29 23:39:30,930 [DEBUG][$_NIOREACTOR-1-RW] node.getRunOnSlave()1-null  (io.mycat.backend.mysql.nio.handler.MultiNodeQueryHandler:MultiNodeQueryHandler.java:180) 
INFO   | jvm 1    | 2018/01/29 23:39:30 | 2018-01-29 23:39:30,930 [DEBUG][$_NIOREACTOR-1-RW] node.getRunOnSlave()2-null  (io.mycat.backend.mysql.nio.handler.MultiNodeQueryHandler:MultiNodeQueryHandler.java:182) 
INFO   | jvm 1    | 2018/01/29 23:39:30 | 2018-01-29 23:39:30,930 [DEBUG][$_NIOREACTOR-1-RW] rrs.getRunOnSlave() null  (io.mycat.backend.datasource.PhysicalDBNode:PhysicalDBNode.java:96) 
INFO   | jvm 1    | 2018/01/29 23:39:30 | 2018-01-29 23:39:30,930 [DEBUG][$_NIOREACTOR-1-RW] rrs.getRunOnSlave() null  (io.mycat.backend.datasource.PhysicalDBNode:PhysicalDBNode.java:127) 
INFO   | jvm 1    | 2018/01/29 23:39:30 | 2018-01-29 23:39:30,930 [DEBUG][$_NIOREACTOR-1-RW] con need syn ,total syn cmd 1 commands SET names utf8;schema change:false con:MySQLConnection [id=9, lastTime=1517240370930, user=root, schema=db3, old shema=db3, borrowed=true, fromSlaveDB=false, threadId=571, charset=utf8, txIsolation=3, autocommit=true, attachment=dn3{insert into company (id,name) values (null,'bbggg')}, respHandler=io.mycat.backend.mysql.nio.handler.MultiNodeQueryHandler@3989b923, host=192.168.183.102, port=3306, statusSync=null, writeQueue=0, modifiedSQLExecuted=true]  (io.mycat.backend.mysql.nio.MySQLConnection:MySQLConnection.java:448) 
INFO   | jvm 1    | 2018/01/29 23:39:31 | 2018-01-29 23:39:30,933 [DEBUG][$_NIOREACTOR-0-RW] received ok response ,executeResponse:false from MySQLConnection [id=4, lastTime=1517240370917, user=root, schema=db1, old shema=db1, borrowed=true, fromSlaveDB=false, threadId=568, charset=utf8, txIsolation=3, autocommit=true, attachment=dn1{insert into company (id,name) values (null,'bbggg')}, respHandler=io.mycat.backend.mysql.nio.handler.MultiNodeQueryHandler@3989b923, host=192.168.183.102, port=3306, statusSync=io.mycat.backend.mysql.nio.MySQLConnection$StatusSync@6171d136, writeQueue=0, modifiedSQLExecuted=true]  (io.mycat.backend.mysql.nio.handler.MultiNodeQueryHandler:MultiNodeQueryHandler.java:231) 
INFO   | jvm 1    | 2018/01/29 23:39:31 | 2018-01-29 23:39:30,934 [DEBUG][$_NIOREACTOR-1-RW] received ok response ,executeResponse:false from MySQLConnection [id=3, lastTime=1517240370917, user=root, schema=db2, old shema=db2, borrowed=true, fromSlaveDB=false, threadId=566, charset=utf8, txIsolation=3, autocommit=true, attachment=dn2{insert into company (id,name) values (null,'bbggg')}, respHandler=io.mycat.backend.mysql.nio.handler.MultiNodeQueryHandler@3989b923, host=192.168.183.102, port=3306, statusSync=io.mycat.backend.mysql.nio.MySQLConnection$StatusSync@192e6bd6, writeQueue=0, modifiedSQLExecuted=true]  (io.mycat.backend.mysql.nio.handler.MultiNodeQueryHandler:MultiNodeQueryHandler.java:231) 
INFO   | jvm 1    | 2018/01/29 23:39:31 | 2018-01-29 23:39:30,936 [DEBUG][$_NIOREACTOR-1-RW] received ok response ,executeResponse:false from MySQLConnection [id=9, lastTime=1517240370917, user=root, schema=db3, old shema=db3, borrowed=true, fromSlaveDB=false, threadId=571, charset=utf8, txIsolation=3, autocommit=true, attachment=dn3{insert into company (id,name) values (null,'bbggg')}, respHandler=io.mycat.backend.mysql.nio.handler.MultiNodeQueryHandler@3989b923, host=192.168.183.102, port=3306, statusSync=io.mycat.backend.mysql.nio.MySQLConnection$StatusSync@2666be51, writeQueue=0, modifiedSQLExecuted=true]  (io.mycat.backend.mysql.nio.handler.MultiNodeQueryHandler:MultiNodeQueryHandler.java:231) 
INFO   | jvm 1    | 2018/01/29 23:39:31 | 2018-01-29 23:39:30,949 [DEBUG][$_NIOREACTOR-1-RW] received ok response ,executeResponse:true from MySQLConnection [id=3, lastTime=1517240370917, user=root, schema=db2, old shema=db2, borrowed=true, fromSlaveDB=false, threadId=566, charset=utf8, txIsolation=3, autocommit=true, attachment=dn2{insert into company (id,name) values (null,'bbggg')}, respHandler=io.mycat.backend.mysql.nio.handler.MultiNodeQueryHandler@3989b923, host=192.168.183.102, port=3306, statusSync=null, writeQueue=0, modifiedSQLExecuted=true]  (io.mycat.backend.mysql.nio.handler.MultiNodeQueryHandler:MultiNodeQueryHandler.java:231) 
INFO   | jvm 1    | 2018/01/29 23:39:31 | 2018-01-29 23:39:30,950 [DEBUG][$_NIOREACTOR-1-RW] release connection MySQLConnection [id=3, lastTime=1517240370917, user=root, schema=db2, old shema=db2, borrowed=true, fromSlaveDB=false, threadId=566, charset=utf8, txIsolation=3, autocommit=true, attachment=dn2{insert into company (id,name) values (null,'bbggg')}, respHandler=io.mycat.backend.mysql.nio.handler.MultiNodeQueryHandler@3989b923, host=192.168.183.102, port=3306, statusSync=null, writeQueue=0, modifiedSQLExecuted=true]  (io.mycat.server.NonBlockingSession:NonBlockingSession.java:341) 
INFO   | jvm 1    | 2018/01/29 23:39:31 | 2018-01-29 23:39:30,951 [DEBUG][$_NIOREACTOR-1-RW] release channel MySQLConnection [id=3, lastTime=1517240370917, user=root, schema=db2, old shema=db2, borrowed=true, fromSlaveDB=false, threadId=566, charset=utf8, txIsolation=3, autocommit=true, attachment=null, respHandler=null, host=192.168.183.102, port=3306, statusSync=null, writeQueue=0, modifiedSQLExecuted=false]  (io.mycat.backend.datasource.PhysicalDatasource:PhysicalDatasource.java:442) 
INFO   | jvm 1    | 2018/01/29 23:39:31 | 2018-01-29 23:39:30,961 [DEBUG][$_NIOREACTOR-0-RW] received ok response ,executeResponse:true from MySQLConnection [id=4, lastTime=1517240370917, user=root, schema=db1, old shema=db1, borrowed=true, fromSlaveDB=false, threadId=568, charset=utf8, txIsolation=3, autocommit=true, attachment=dn1{insert into company (id,name) values (null,'bbggg')}, respHandler=io.mycat.backend.mysql.nio.handler.MultiNodeQueryHandler@3989b923, host=192.168.183.102, port=3306, statusSync=null, writeQueue=0, modifiedSQLExecuted=true]  (io.mycat.backend.mysql.nio.handler.MultiNodeQueryHandler:MultiNodeQueryHandler.java:231) 
INFO   | jvm 1    | 2018/01/29 23:39:31 | 2018-01-29 23:39:30,964 [DEBUG][$_NIOREACTOR-0-RW] release connection MySQLConnection [id=4, lastTime=1517240370917, user=root, schema=db1, old shema=db1, borrowed=true, fromSlaveDB=false, threadId=568, charset=utf8, txIsolation=3, autocommit=true, attachment=dn1{insert into company (id,name) values (null,'bbggg')}, respHandler=io.mycat.backend.mysql.nio.handler.MultiNodeQueryHandler@3989b923, host=192.168.183.102, port=3306, statusSync=null, writeQueue=0, modifiedSQLExecuted=true]  (io.mycat.server.NonBlockingSession:NonBlockingSession.java:341) 
INFO   | jvm 1    | 2018/01/29 23:39:31 | 2018-01-29 23:39:30,964 [DEBUG][$_NIOREACTOR-0-RW] release channel MySQLConnection [id=4, lastTime=1517240370917, user=root, schema=db1, old shema=db1, borrowed=true, fromSlaveDB=false, threadId=568, charset=utf8, txIsolation=3, autocommit=true, attachment=null, respHandler=null, host=192.168.183.102, port=3306, statusSync=null, writeQueue=0, modifiedSQLExecuted=false]  (io.mycat.backend.datasource.PhysicalDatasource:PhysicalDatasource.java:442) 
INFO   | jvm 1    | 2018/01/29 23:39:31 | 2018-01-29 23:39:30,965 [DEBUG][$_NIOREACTOR-1-RW] received ok response ,executeResponse:true from MySQLConnection [id=9, lastTime=1517240370917, user=root, schema=db3, old shema=db3, borrowed=true, fromSlaveDB=false, threadId=571, charset=utf8, txIsolation=3, autocommit=true, attachment=dn3{insert into company (id,name) values (null,'bbggg')}, respHandler=io.mycat.backend.mysql.nio.handler.MultiNodeQueryHandler@3989b923, host=192.168.183.102, port=3306, statusSync=null, writeQueue=0, modifiedSQLExecuted=true]  (io.mycat.backend.mysql.nio.handler.MultiNodeQueryHandler:MultiNodeQueryHandler.java:231) 
INFO   | jvm 1    | 2018/01/29 23:39:31 | 2018-01-29 23:39:30,965 [DEBUG][$_NIOREACTOR-1-RW] release connection MySQLConnection [id=9, lastTime=1517240370917, user=root, schema=db3, old shema=db3, borrowed=true, fromSlaveDB=false, threadId=571, charset=utf8, txIsolation=3, autocommit=true, attachment=dn3{insert into company (id,name) values (null,'bbggg')}, respHandler=io.mycat.backend.mysql.nio.handler.MultiNodeQueryHandler@3989b923, host=192.168.183.102, port=3306, statusSync=null, writeQueue=0, modifiedSQLExecuted=true]  (io.mycat.server.NonBlockingSession:NonBlockingSession.java:341) 
INFO   | jvm 1    | 2018/01/29 23:39:31 | 2018-01-29 23:39:30,965 [DEBUG][$_NIOREACTOR-1-RW] release channel MySQLConnection [id=9, lastTime=1517240370917, user=root, schema=db3, old shema=db3, borrowed=true, fromSlaveDB=false, threadId=571, charset=utf8, txIsolation=3, autocommit=true, attachment=null, respHandler=null, host=192.168.183.102, port=3306, statusSync=null, writeQueue=0, modifiedSQLExecuted=false]  (io.mycat.backend.datasource.PhysicalDatasource:PhysicalDatasource.java:442) 

```

> executeResponse:true from MySQLConnection......host=192.168.183.102

* 可以看出insert操作都去了master节点102



```
mysql> select * from company;
+----+---------+
| id | name    |
+----+---------+
|  1 | aa      |
|  2 | kk      |
|  3 | m       |
|  4 | lll     |
|  5 | pppp    |
|  6 | vvvvv   |
|  7 | wwww111 |
|  8 | ee2222  |
|  9 | bbggg   |
+----+---------+
9 rows in set (0.02 sec)

mysql> 
```

tail -f /usr/local/mycat/logs/wrapper.log

```
INFO   | jvm 1    | 2018/01/29 23:42:17 | 2018-01-29 23:42:17,593 [DEBUG][$_NIOREACTOR-1-RW] ServerConnection [id=1, schema=TESTDB, host=192.168.183.101, user=root,txIsolation=3, autocommit=true, schema=TESTDB] select * from company  (io.mycat.net.FrontendConnection:FrontendConnection.java:288) 
INFO   | jvm 1    | 2018/01/29 23:42:17 | 2018-01-29 23:42:17,594 [DEBUG][$_NIOREACTOR-1-RW] ServerConnection [id=1, schema=TESTDB, host=192.168.183.101, user=root,txIsolation=3, autocommit=true, schema=TESTDB]select * from company  (io.mycat.server.ServerQueryHandler:ServerQueryHandler.java:57) 
INFO   | jvm 1    | 2018/01/29 23:42:17 | 2018-01-29 23:42:17,595 [DEBUG][$_NIOREACTOR-1-RW] SQLRouteCache  miss cache ,key:TESTDBselect * from company  (io.mycat.cache.impl.EnchachePool:EnchachePool.java:77) 
INFO   | jvm 1    | 2018/01/29 23:42:17 | 2018-01-29 23:42:17,597 [DEBUG][$_NIOREACTOR-1-RW] ServerConnection [id=1, schema=TESTDB, host=192.168.183.101, user=root,txIsolation=3, autocommit=true, schema=TESTDB]select * from company, route={
INFO   | jvm 1    | 2018/01/29 23:42:17 |    1 -> dn2{SELECT *
INFO   | jvm 1    | 2018/01/29 23:42:17 | FROM company
INFO   | jvm 1    | 2018/01/29 23:42:17 | LIMIT 100}
INFO   | jvm 1    | 2018/01/29 23:42:17 | } rrs   (io.mycat.server.NonBlockingSession:NonBlockingSession.java:110) 
INFO   | jvm 1    | 2018/01/29 23:42:17 | 2018-01-29 23:42:17,597 [DEBUG][$_NIOREACTOR-1-RW] rrs.getRunOnSlave() null  (io.mycat.backend.mysql.nio.handler.SingleNodeHandler:SingleNodeHandler.java:166) 
INFO   | jvm 1    | 2018/01/29 23:42:17 | 2018-01-29 23:42:17,598 [DEBUG][$_NIOREACTOR-1-RW] node.getRunOnSlave() null  (io.mycat.backend.mysql.nio.handler.SingleNodeHandler:SingleNodeHandler.java:168) 
INFO   | jvm 1    | 2018/01/29 23:42:17 | 2018-01-29 23:42:17,598 [DEBUG][$_NIOREACTOR-1-RW] node.getRunOnSlave() null  (io.mycat.backend.mysql.nio.handler.SingleNodeHandler:SingleNodeHandler.java:177) 
INFO   | jvm 1    | 2018/01/29 23:42:17 | 2018-01-29 23:42:17,598 [DEBUG][$_NIOREACTOR-1-RW] node.getRunOnSlave() null  (io.mycat.backend.mysql.nio.handler.SingleNodeHandler:SingleNodeHandler.java:179) 
INFO   | jvm 1    | 2018/01/29 23:42:17 | 2018-01-29 23:42:17,598 [DEBUG][$_NIOREACTOR-1-RW] rrs.getRunOnSlave() null  (io.mycat.backend.datasource.PhysicalDBNode:PhysicalDBNode.java:96) 
INFO   | jvm 1    | 2018/01/29 23:42:17 | 2018-01-29 23:42:17,598 [DEBUG][$_NIOREACTOR-1-RW] rrs.getRunOnSlave() null  (io.mycat.backend.datasource.PhysicalDBNode:PhysicalDBNode.java:127) 
INFO   | jvm 1    | 2018/01/29 23:42:17 | 2018-01-29 23:42:17,598 [DEBUG][$_NIOREACTOR-1-RW] select read source hostS2 for dataHost:localhost1  (io.mycat.backend.datasource.PhysicalDBPool:PhysicalDBPool.java:456) 
INFO   | jvm 1    | 2018/01/29 23:42:17 | 2018-01-29 23:42:17,598 [DEBUG][$_NIOREACTOR-1-RW] con need syn ,total syn cmd 1 commands SET names utf8;schema change:false con:MySQLConnection [id=11, lastTime=1517240537598, user=root, schema=db2, old shema=db2, borrowed=true, fromSlaveDB=true, threadId=829, charset=utf8, txIsolation=3, autocommit=true, attachment=dn2{SELECT *
INFO   | jvm 1    | 2018/01/29 23:42:17 | FROM company
INFO   | jvm 1    | 2018/01/29 23:42:17 | LIMIT 100}, respHandler=SingleNodeHandler [node=dn2{SELECT *
INFO   | jvm 1    | 2018/01/29 23:42:17 | FROM company
INFO   | jvm 1    | 2018/01/29 23:42:17 | LIMIT 100}, packetId=0], host=192.168.183.103, port=3306, statusSync=null, writeQueue=0, modifiedSQLExecuted=false]  (io.mycat.backend.mysql.nio.MySQLConnection:MySQLConnection.java:448) 
INFO   | jvm 1    | 2018/01/29 23:42:17 | 2018-01-29 23:42:17,608 [DEBUG][$_NIOREACTOR-1-RW] release connection MySQLConnection [id=11, lastTime=1517240537595, user=root, schema=db2, old shema=db2, borrowed=true, fromSlaveDB=true, threadId=829, charset=utf8, txIsolation=3, autocommit=true, attachment=dn2{SELECT *
INFO   | jvm 1    | 2018/01/29 23:42:17 | FROM company
INFO   | jvm 1    | 2018/01/29 23:42:17 | LIMIT 100}, respHandler=SingleNodeHandler [node=dn2{SELECT *
INFO   | jvm 1    | 2018/01/29 23:42:17 | FROM company
INFO   | jvm 1    | 2018/01/29 23:42:17 | LIMIT 100}, packetId=13], host=192.168.183.103, port=3306, statusSync=io.mycat.backend.mysql.nio.MySQLConnection$StatusSync@528f1c09, writeQueue=0, modifiedSQLExecuted=false]  (io.mycat.server.NonBlockingSession:NonBlockingSession.java:341) 
INFO   | jvm 1    | 2018/01/29 23:42:17 | 2018-01-29 23:42:17,609 [DEBUG][$_NIOREACTOR-1-RW] release channel MySQLConnection [id=11, lastTime=1517240537595, user=root, schema=db2, old shema=db2, borrowed=true, fromSlaveDB=true, threadId=829, charset=utf8, txIsolation=3, autocommit=true, attachment=null, respHandler=null, host=192.168.183.103, port=3306, statusSync=null, writeQueue=0, modifiedSQLExecuted=false]  (io.mycat.backend.datasource.PhysicalDatasource:PhysicalDatasource.java:442)
```

> release connection MySQLConnection......host=192.168.183.103

* 可以看出读操作都去了103



确认操作也可以打开MySQL的general\_log，通过查看mysql的general\_log来确认。

```
mysql> SHOW VARIABLES LIKE "general_log%";
mysql> SET GLOBAL general_log = 'ON';
mysql> SET GLOBAL general_log_file = '/var/log/mysql/general_log.log';
```

tail -f /var/log/mysql/general\_log.log



