首先配置好MySQL 主从，MyCat 不负责数据同步问题



vim /usr/local/mycat/conf/schema.xml

```
        <dataHost name="localhost1" maxCon="1000" minCon="10" balance="0"
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



balance 为0 表示读操作只会去readhost，为1表示读操作会去readhost和配置的冗余的writehost。



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
mysql> explain  insert into company (id,name) values (null,'lll');
+-----------+---------------------------------------------------+
| DATA_NODE | SQL                                               |
+-----------+---------------------------------------------------+
| dn1       | insert into company (id,name) values (null,'lll') |
| dn2       | insert into company (id,name) values (null,'lll') |
| dn3       | insert into company (id,name) values (null,'lll') |
+-----------+---------------------------------------------------+
3 rows in set (0.00 sec)

mysql>
```

tail -f /usr/local/mycat/logs/wrapper.log

```
INFO   | jvm 1    | 2018/01/28 11:27:01 | 2018-01-28 11:27:01,681 [DEBUG][$_NIOREACTOR-1-RW] ServerConnection [id=1, schema=TESTDB, host=192.168.183.101, user=root,txIsolation=3, autocommit=true, schema=TESTDB] explain  insert into company (id,name) values (null,'lll')  (io.mycat.net.FrontendConnection:FrontendConnection.java:288) 
INFO   | jvm 1    | 2018/01/28 11:27:01 | 2018-01-28 11:27:01,682 [DEBUG][$_NIOREACTOR-1-RW] ServerConnection [id=1, schema=TESTDB, host=192.168.183.101, user=root,txIsolation=3, autocommit=true, schema=TESTDB]explain  insert into company (id,name) values (null,'lll')  (io.mycat.server.ServerQueryHandler:ServerQueryHandler.java:57) 

```



```
mysql> explain select * from company;
+-----------+---------------------------------+
| DATA_NODE | SQL                             |
+-----------+---------------------------------+
| dn2       | SELECT * FROM company LIMIT 100 |
+-----------+---------------------------------+
1 row in set (0.01 sec)

mysql>
```

tail -f /usr/local/mycat/logs/wrapper.log

```
INFO   | jvm 1    | 2018/01/28 11:29:31 | 2018-01-28 11:29:31,510 [DEBUG][$_NIOREACTOR-1-RW] ServerConnection [id=1, schema=TESTDB, host=192.168.183.101, user=root,txIsolation=3, autocommit=true, schema=TESTDB] explain select * from company  (io.mycat.net.FrontendConnection:FrontendConnection.java:288) 
INFO   | jvm 1    | 2018/01/28 11:29:31 | 2018-01-28 11:29:31,510 [DEBUG][$_NIOREACTOR-1-RW] ServerConnection [id=1, schema=TESTDB, host=192.168.183.101, user=root,txIsolation=3, autocommit=true, schema=TESTDB]explain select * from company  (io.mycat.server.ServerQueryHandler:ServerQueryHandler.java:57) 
INFO   | jvm 1    | 2018/01/28 11:29:31 | 2018-01-28 11:29:31,510 [DEBUG][$_NIOREACTOR-1-RW] SQLRouteCache  miss cache ,key:TESTDBselect * from company  (io.mycat.cache.impl.EnchachePool:EnchachePool.java:77) 

```



```
mysql> insert into company (id,name) values (null,'lll');
Query OK, 1 row affected (0.12 sec)

mysql> 
```

tail -f /usr/local/mycat/logs/wrapper.log

```
INFO   | jvm 1    | 2018/01/28 11:30:45 | 2018-01-28 11:30:45,317 [DEBUG][$_NIOREACTOR-1-RW] ServerConnection [id=1, schema=TESTDB, host=192.168.183.101, user=root,txIsolation=3, autocommit=true, schema=TESTDB] insert into company (id,name) values (null,'lll')  (io.mycat.net.FrontendConnection:FrontendConnection.java:288) 
INFO   | jvm 1    | 2018/01/28 11:30:45 | 2018-01-28 11:30:45,318 [DEBUG][$_NIOREACTOR-1-RW] ServerConnection [id=1, schema=TESTDB, host=192.168.183.101, user=root,txIsolation=3, autocommit=true, schema=TESTDB]insert into company (id,name) values (null,'lll')  (io.mycat.server.ServerQueryHandler:ServerQueryHandler.java:57) 
INFO   | jvm 1    | 2018/01/28 11:30:45 | 2018-01-28 11:30:45,324 [DEBUG][$_NIOREACTOR-1-RW] ServerConnection [id=1, schema=TESTDB, host=192.168.183.101, user=root,txIsolation=3, autocommit=true, schema=TESTDB]insert into company (id,name) values (null,'lll'), route={
INFO   | jvm 1    | 2018/01/28 11:30:45 |    1 -> dn1{insert into company (id,name) values (null,'lll')}
INFO   | jvm 1    | 2018/01/28 11:30:45 |    2 -> dn2{insert into company (id,name) values (null,'lll')}
INFO   | jvm 1    | 2018/01/28 11:30:45 |    3 -> dn3{insert into company (id,name) values (null,'lll')}
INFO   | jvm 1    | 2018/01/28 11:30:45 | } rrs   (io.mycat.server.NonBlockingSession:NonBlockingSession.java:110) 
INFO   | jvm 1    | 2018/01/28 11:30:45 | 2018-01-28 11:30:45,337 [DEBUG][$_NIOREACTOR-1-RW] execute mutinode query insert into company (id,name) values (null,'lll')  (io.mycat.backend.mysql.nio.handler.MultiNodeQueryHandler:MultiNodeQueryHandler.java:101) 
INFO   | jvm 1    | 2018/01/28 11:30:45 | 2018-01-28 11:30:45,338 [DEBUG][$_NIOREACTOR-1-RW] rrs.getRunOnSlave()-null  (io.mycat.backend.mysql.nio.handler.MultiNodeQueryHandler:MultiNodeQueryHandler.java:170) 
INFO   | jvm 1    | 2018/01/28 11:30:45 | 2018-01-28 11:30:45,338 [DEBUG][$_NIOREACTOR-1-RW] node.getRunOnSlave()1-null  (io.mycat.backend.mysql.nio.handler.MultiNodeQueryHandler:MultiNodeQueryHandler.java:180) 
INFO   | jvm 1    | 2018/01/28 11:30:45 | 2018-01-28 11:30:45,338 [DEBUG][$_NIOREACTOR-1-RW] node.getRunOnSlave()2-null  (io.mycat.backend.mysql.nio.handler.MultiNodeQueryHandler:MultiNodeQueryHandler.java:182) 
INFO   | jvm 1    | 2018/01/28 11:30:45 | 2018-01-28 11:30:45,338 [DEBUG][$_NIOREACTOR-1-RW] rrs.getRunOnSlave() null  (io.mycat.backend.datasource.PhysicalDBNode:PhysicalDBNode.java:96) 
INFO   | jvm 1    | 2018/01/28 11:30:45 | 2018-01-28 11:30:45,338 [DEBUG][$_NIOREACTOR-1-RW] rrs.getRunOnSlave() null  (io.mycat.backend.datasource.PhysicalDBNode:PhysicalDBNode.java:127) 
INFO   | jvm 1    | 2018/01/28 11:30:45 | 2018-01-28 11:30:45,339 [DEBUG][$_NIOREACTOR-1-RW] con need syn ,total syn cmd 1 commands SET names utf8;schema change:false con:MySQLConnection [id=1, lastTime=1517110245338, user=root, schema=db1, old shema=db1, borrowed=true, fromSlaveDB=false, threadId=384, charset=utf8, txIsolation=3, autocommit=true, attachment=dn1{insert into company (id,name) values (null,'lll')}, respHandler=io.mycat.backend.mysql.nio.handler.MultiNodeQueryHandler@6f20d37d, host=192.168.183.102, port=3306, statusSync=null, writeQueue=0, modifiedSQLExecuted=true]  (io.mycat.backend.mysql.nio.MySQLConnection:MySQLConnection.java:448) 
INFO   | jvm 1    | 2018/01/28 11:30:45 | 2018-01-28 11:30:45,340 [DEBUG][$_NIOREACTOR-1-RW] node.getRunOnSlave()1-null  (io.mycat.backend.mysql.nio.handler.MultiNodeQueryHandler:MultiNodeQueryHandler.java:180) 
INFO   | jvm 1    | 2018/01/28 11:30:45 | 2018-01-28 11:30:45,340 [DEBUG][$_NIOREACTOR-1-RW] node.getRunOnSlave()2-null  (io.mycat.backend.mysql.nio.handler.MultiNodeQueryHandler:MultiNodeQueryHandler.java:182) 
INFO   | jvm 1    | 2018/01/28 11:30:45 | 2018-01-28 11:30:45,340 [DEBUG][$_NIOREACTOR-1-RW] rrs.getRunOnSlave() null  (io.mycat.backend.datasource.PhysicalDBNode:PhysicalDBNode.java:96) 
INFO   | jvm 1    | 2018/01/28 11:30:45 | 2018-01-28 11:30:45,340 [DEBUG][$_NIOREACTOR-1-RW] rrs.getRunOnSlave() null  (io.mycat.backend.datasource.PhysicalDBNode:PhysicalDBNode.java:127) 
INFO   | jvm 1    | 2018/01/28 11:30:45 | 2018-01-28 11:30:45,340 [DEBUG][$_NIOREACTOR-1-RW] con need syn ,total syn cmd 1 commands SET names utf8;schema change:false con:MySQLConnection [id=4, lastTime=1517110245340, user=root, schema=db2, old shema=db2, borrowed=true, fromSlaveDB=false, threadId=381, charset=utf8, txIsolation=3, autocommit=true, attachment=dn2{insert into company (id,name) values (null,'lll')}, respHandler=io.mycat.backend.mysql.nio.handler.MultiNodeQueryHandler@6f20d37d, host=192.168.183.102, port=3306, statusSync=null, writeQueue=0, modifiedSQLExecuted=true]  (io.mycat.backend.mysql.nio.MySQLConnection:MySQLConnection.java:448) 
INFO   | jvm 1    | 2018/01/28 11:30:45 | 2018-01-28 11:30:45,340 [DEBUG][$_NIOREACTOR-1-RW] node.getRunOnSlave()1-null  (io.mycat.backend.mysql.nio.handler.MultiNodeQueryHandler:MultiNodeQueryHandler.java:180) 
INFO   | jvm 1    | 2018/01/28 11:30:45 | 2018-01-28 11:30:45,341 [DEBUG][$_NIOREACTOR-1-RW] node.getRunOnSlave()2-null  (io.mycat.backend.mysql.nio.handler.MultiNodeQueryHandler:MultiNodeQueryHandler.java:182) 
INFO   | jvm 1    | 2018/01/28 11:30:45 | 2018-01-28 11:30:45,341 [DEBUG][$_NIOREACTOR-1-RW] rrs.getRunOnSlave() null  (io.mycat.backend.datasource.PhysicalDBNode:PhysicalDBNode.java:96) 
INFO   | jvm 1    | 2018/01/28 11:30:45 | 2018-01-28 11:30:45,341 [DEBUG][$_NIOREACTOR-1-RW] rrs.getRunOnSlave() null  (io.mycat.backend.datasource.PhysicalDBNode:PhysicalDBNode.java:127) 
INFO   | jvm 1    | 2018/01/28 11:30:45 | 2018-01-28 11:30:45,341 [DEBUG][$_NIOREACTOR-1-RW] con need syn ,total syn cmd 1 commands SET names utf8;schema change:false con:MySQLConnection [id=7, lastTime=1517110245341, user=root, schema=db3, old shema=db3, borrowed=true, fromSlaveDB=false, threadId=387, charset=utf8, txIsolation=3, autocommit=true, attachment=dn3{insert into company (id,name) values (null,'lll')}, respHandler=io.mycat.backend.mysql.nio.handler.MultiNodeQueryHandler@6f20d37d, host=192.168.183.102, port=3306, statusSync=null, writeQueue=0, modifiedSQLExecuted=true]  (io.mycat.backend.mysql.nio.MySQLConnection:MySQLConnection.java:448) 
INFO   | jvm 1    | 2018/01/28 11:30:45 | 2018-01-28 11:30:45,342 [DEBUG][$_NIOREACTOR-1-RW] received ok response ,executeResponse:false from MySQLConnection [id=1, lastTime=1517110245338, user=root, schema=db1, old shema=db1, borrowed=true, fromSlaveDB=false, threadId=384, charset=utf8, txIsolation=3, autocommit=true, attachment=dn1{insert into company (id,name) values (null,'lll')}, respHandler=io.mycat.backend.mysql.nio.handler.MultiNodeQueryHandler@6f20d37d, host=192.168.183.102, port=3306, statusSync=io.mycat.backend.mysql.nio.MySQLConnection$StatusSync@10d29fa2, writeQueue=0, modifiedSQLExecuted=true]  (io.mycat.backend.mysql.nio.handler.MultiNodeQueryHandler:MultiNodeQueryHandler.java:231) 
INFO   | jvm 1    | 2018/01/28 11:30:45 | 2018-01-28 11:30:45,346 [DEBUG][$_NIOREACTOR-0-RW] received ok response ,executeResponse:false from MySQLConnection [id=4, lastTime=1517110245338, user=root, schema=db2, old shema=db2, borrowed=true, fromSlaveDB=false, threadId=381, charset=utf8, txIsolation=3, autocommit=true, attachment=dn2{insert into company (id,name) values (null,'lll')}, respHandler=io.mycat.backend.mysql.nio.handler.MultiNodeQueryHandler@6f20d37d, host=192.168.183.102, port=3306, statusSync=io.mycat.backend.mysql.nio.MySQLConnection$StatusSync@55711af0, writeQueue=0, modifiedSQLExecuted=true]  (io.mycat.backend.mysql.nio.handler.MultiNodeQueryHandler:MultiNodeQueryHandler.java:231) 
INFO   | jvm 1    | 2018/01/28 11:30:45 | 2018-01-28 11:30:45,346 [DEBUG][$_NIOREACTOR-1-RW] received ok response ,executeResponse:false from MySQLConnection [id=7, lastTime=1517110245338, user=root, schema=db3, old shema=db3, borrowed=true, fromSlaveDB=false, threadId=387, charset=utf8, txIsolation=3, autocommit=true, attachment=dn3{insert into company (id,name) values (null,'lll')}, respHandler=io.mycat.backend.mysql.nio.handler.MultiNodeQueryHandler@6f20d37d, host=192.168.183.102, port=3306, statusSync=io.mycat.backend.mysql.nio.MySQLConnection$StatusSync@79801dfe, writeQueue=0, modifiedSQLExecuted=true]  (io.mycat.backend.mysql.nio.handler.MultiNodeQueryHandler:MultiNodeQueryHandler.java:231) 
INFO   | jvm 1    | 2018/01/28 11:30:45 | 2018-01-28 11:30:45,360 [DEBUG][$_NIOREACTOR-0-RW] received ok response ,executeResponse:true from MySQLConnection [id=4, lastTime=1517110245338, user=root, schema=db2, old shema=db2, borrowed=true, fromSlaveDB=false, threadId=381, charset=utf8, txIsolation=3, autocommit=true, attachment=dn2{insert into company (id,name) values (null,'lll')}, respHandler=io.mycat.backend.mysql.nio.handler.MultiNodeQueryHandler@6f20d37d, host=192.168.183.102, port=3306, statusSync=null, writeQueue=0, modifiedSQLExecuted=true]  (io.mycat.backend.mysql.nio.handler.MultiNodeQueryHandler:MultiNodeQueryHandler.java:231) 
INFO   | jvm 1    | 2018/01/28 11:30:45 | 2018-01-28 11:30:45,361 [DEBUG][$_NIOREACTOR-0-RW] release connection MySQLConnection [id=4, lastTime=1517110245338, user=root, schema=db2, old shema=db2, borrowed=true, fromSlaveDB=false, threadId=381, charset=utf8, txIsolation=3, autocommit=true, attachment=dn2{insert into company (id,name) values (null,'lll')}, respHandler=io.mycat.backend.mysql.nio.handler.MultiNodeQueryHandler@6f20d37d, host=192.168.183.102, port=3306, statusSync=null, writeQueue=0, modifiedSQLExecuted=true]  (io.mycat.server.NonBlockingSession:NonBlockingSession.java:341) 
INFO   | jvm 1    | 2018/01/28 11:30:45 | 2018-01-28 11:30:45,361 [DEBUG][$_NIOREACTOR-0-RW] release channel MySQLConnection [id=4, lastTime=1517110245338, user=root, schema=db2, old shema=db2, borrowed=true, fromSlaveDB=false, threadId=381, charset=utf8, txIsolation=3, autocommit=true, attachment=null, respHandler=null, host=192.168.183.102, port=3306, statusSync=null, writeQueue=0, modifiedSQLExecuted=false]  (io.mycat.backend.datasource.PhysicalDatasource:PhysicalDatasource.java:442) 
INFO   | jvm 1    | 2018/01/28 11:30:45 | 2018-01-28 11:30:45,424 [DEBUG][$_NIOREACTOR-1-RW] received ok response ,executeResponse:true from MySQLConnection [id=1, lastTime=1517110245338, user=root, schema=db1, old shema=db1, borrowed=true, fromSlaveDB=false, threadId=384, charset=utf8, txIsolation=3, autocommit=true, attachment=dn1{insert into company (id,name) values (null,'lll')}, respHandler=io.mycat.backend.mysql.nio.handler.MultiNodeQueryHandler@6f20d37d, host=192.168.183.102, port=3306, statusSync=null, writeQueue=0, modifiedSQLExecuted=true]  (io.mycat.backend.mysql.nio.handler.MultiNodeQueryHandler:MultiNodeQueryHandler.java:231) 
INFO   | jvm 1    | 2018/01/28 11:30:45 | 2018-01-28 11:30:45,424 [DEBUG][$_NIOREACTOR-1-RW] release connection MySQLConnection [id=1, lastTime=1517110245338, user=root, schema=db1, old shema=db1, borrowed=true, fromSlaveDB=false, threadId=384, charset=utf8, txIsolation=3, autocommit=true, attachment=dn1{insert into company (id,name) values (null,'lll')}, respHandler=io.mycat.backend.mysql.nio.handler.MultiNodeQueryHandler@6f20d37d, host=192.168.183.102, port=3306, statusSync=null, writeQueue=0, modifiedSQLExecuted=true]  (io.mycat.server.NonBlockingSession:NonBlockingSession.java:341) 
INFO   | jvm 1    | 2018/01/28 11:30:45 | 2018-01-28 11:30:45,424 [DEBUG][$_NIOREACTOR-1-RW] release channel MySQLConnection [id=1, lastTime=1517110245338, user=root, schema=db1, old shema=db1, borrowed=true, fromSlaveDB=false, threadId=384, charset=utf8, txIsolation=3, autocommit=true, attachment=null, respHandler=null, host=192.168.183.102, port=3306, statusSync=null, writeQueue=0, modifiedSQLExecuted=false]  (io.mycat.backend.datasource.PhysicalDatasource:PhysicalDatasource.java:442) 
INFO   | jvm 1    | 2018/01/28 11:30:45 | 2018-01-28 11:30:45,425 [DEBUG][$_NIOREACTOR-1-RW] received ok response ,executeResponse:true from MySQLConnection [id=7, lastTime=1517110245338, user=root, schema=db3, old shema=db3, borrowed=true, fromSlaveDB=false, threadId=387, charset=utf8, txIsolation=3, autocommit=true, attachment=dn3{insert into company (id,name) values (null,'lll')}, respHandler=io.mycat.backend.mysql.nio.handler.MultiNodeQueryHandler@6f20d37d, host=192.168.183.102, port=3306, statusSync=null, writeQueue=0, modifiedSQLExecuted=true]  (io.mycat.backend.mysql.nio.handler.MultiNodeQueryHandler:MultiNodeQueryHandler.java:231) 
INFO   | jvm 1    | 2018/01/28 11:30:45 | 2018-01-28 11:30:45,425 [DEBUG][$_NIOREACTOR-1-RW] release connection MySQLConnection [id=7, lastTime=1517110245338, user=root, schema=db3, old shema=db3, borrowed=true, fromSlaveDB=false, threadId=387, charset=utf8, txIsolation=3, autocommit=true, attachment=dn3{insert into company (id,name) values (null,'lll')}, respHandler=io.mycat.backend.mysql.nio.handler.MultiNodeQueryHandler@6f20d37d, host=192.168.183.102, port=3306, statusSync=null, writeQueue=0, modifiedSQLExecuted=true]  (io.mycat.server.NonBlockingSession:NonBlockingSession.java:341) 
INFO   | jvm 1    | 2018/01/28 11:30:45 | 2018-01-28 11:30:45,425 [DEBUG][$_NIOREACTOR-1-RW] release channel MySQLConnection [id=7, lastTime=1517110245338, user=root, schema=db3, old shema=db3, borrowed=true, fromSlaveDB=false, threadId=387, charset=utf8, txIsolation=3, autocommit=true, attachment=null, respHandler=null, host=192.168.183.102, port=3306, statusSync=null, writeQueue=0, modifiedSQLExecuted=false]  (io.mycat.backend.datasource.PhysicalDatasource:PhysicalDatasource.java:442) 
INFO   | jvm 1    | 2018/01/28 11:30:46 | 2018-01-28 11:30:46,238 [DEBUG][Timer0] con query sql:select user() to con:MySQLConnection [id=8, lastTime=1517110246238, user=root, schema=db3, old shema=db3, borrowed=true, fromSlaveDB=false, threadId=383, charset=latin1, txIsolation=3, autocommit=true, attachment=null, respHandler=null, host=192.168.183.102, port=3306, statusSync=null, writeQueue=0, modifiedSQLExecuted=false]  (io.mycat.sqlengine.SQLJob:SQLJob.java:88) 
INFO   | jvm 1    | 2018/01/28 11:30:46 | 2018-01-28 11:30:46,239 [DEBUG][Timer0] con query sql:select user() to con:MySQLConnection [id=18, lastTime=1517110246239, user=root, schema=db3, old shema=db3, borrowed=true, fromSlaveDB=true, threadId=671, charset=latin1, txIsolation=3, autocommit=true, attachment=null, respHandler=null, host=192.168.183.103, port=3306, statusSync=null, writeQueue=0, modifiedSQLExecuted=false]  (io.mycat.sqlengine.SQLJob:SQLJob.java:88) 
INFO   | jvm 1    | 2018/01/28 11:30:46 | 2018-01-28 11:30:46,243 [DEBUG][$_NIOREACTOR-1-RW] release channel MySQLConnection [id=18, lastTime=1517110246236, user=root, schema=db3, old shema=db3, borrowed=true, fromSlaveDB=true, threadId=671, charset=latin1, txIsolation=3, autocommit=true, attachment=null, respHandler=null, host=192.168.183.103, port=3306, statusSync=null, writeQueue=0, modifiedSQLExecuted=false]  (io.mycat.backend.datasource.PhysicalDatasource:PhysicalDatasource.java:442) 
INFO   | jvm 1    | 2018/01/28 11:30:46 | 2018-01-28 11:30:46,244 [DEBUG][$_NIOREACTOR-0-RW] release channel MySQLConnection [id=8, lastTime=1517110246236, user=root, schema=db3, old shema=db3, borrowed=true, fromSlaveDB=false, threadId=383, charset=latin1, txIsolation=3, autocommit=true, attachment=null, respHandler=null, host=192.168.183.102, port=3306, statusSync=null, writeQueue=0, modifiedSQLExecuted=false]  (io.mycat.backend.datasource.PhysicalDatasource:PhysicalDatasource.java:442) 


```

> executeResponse:true from MySQLConnection......host=192.168.183.102

* 可以看出insert操作都去了master节点102



























