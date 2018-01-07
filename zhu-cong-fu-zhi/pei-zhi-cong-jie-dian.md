```
sed -i '/\[mysqld\]/a\relay-log-index=/var/lib/mysql/relay-bin.index' /etc/my.cnf
sed -i '/\[mysqld\]/a\relay-log=/var/lib/mysql/relay-bin' /etc/my.cnf
sed -i '/\[mysqld\]/a\server-id=102' /etc/my.cnf                   # server-id 要唯一，各个节点不能够重复，而且必须为数值

systemctl restart mysqld   # 重启服务
```

```
# 配置主从
change master to master_host='192.168.183.101',master_port=3306,master_user='userslave1',master_password='asdF',master_log_file='mysql-bin.000001',master_log_pos=0;
# 会有两个警告，show warnings;可以查看，警告，先不处理吧。
# | Note  | 1759 | Sending passwords in plain text without SSL/TLS is extremely insecure.
# | Note  | 1760 | Storing MySQL user name or password information in the master info repository is not secure and is therefore not recommended. Please consider using the USER and PASSWORD connection options for START SLAVE; see the 'START SLAVE Syntax' in the MySQL Manual for more information. |


start slave;
show slave status\G   # 查看主从的状态
```



遇到一个问题，主从同步停止了，这个问题的原因是，在master alter了userslave1用户的密码为asdF，而从库没有修改密码安全策略，这个密码太简单，因此这条sql语句在从库执行报错了，报错信息如下：

```
mysql> show slave status\G
*************************** 1. row ***************************
               Slave_IO_State: Waiting for master to send event
                  Master_Host: 192.168.183.101
                  Master_User: userslave1
                  Master_Port: 3306
                Connect_Retry: 60
              Master_Log_File: mysql-bin.000006
          Read_Master_Log_Pos: 154
               Relay_Log_File: relay-bin.000006
                Relay_Log_Pos: 367
        Relay_Master_Log_File: mysql-bin.000005
             Slave_IO_Running: Yes
            Slave_SQL_Running: No
              Replicate_Do_DB: 
          Replicate_Ignore_DB: 
           Replicate_Do_Table: 
       Replicate_Ignore_Table: 
      Replicate_Wild_Do_Table: 
  Replicate_Wild_Ignore_Table: 
                   Last_Errno: 1396
                   Last_Error: Error 'Operation ALTER USER failed for 'userslave1'@'192.168.183.%'' on query. Default database: ''. Query: 'ALTER USER 'userslave1'@'192.168.183.%' IDENTIFIED WITH 'mysql_native_password' AS '*ABA990061718A9097EA66815915693F747D60316''
                 Skip_Counter: 0
          Exec_Master_Log_Pos: 154
              Relay_Log_Space: 2727
              Until_Condition: None
               Until_Log_File: 
                Until_Log_Pos: 0
           Master_SSL_Allowed: No
           Master_SSL_CA_File: 
           Master_SSL_CA_Path: 
              Master_SSL_Cert: 
            Master_SSL_Cipher: 
               Master_SSL_Key: 
        Seconds_Behind_Master: NULL
Master_SSL_Verify_Server_Cert: No
                Last_IO_Errno: 0
                Last_IO_Error: 
               Last_SQL_Errno: 1396
               Last_SQL_Error: Error 'Operation ALTER USER failed for 'userslave1'@'192.168.183.%'' on query. Default database: ''. Query: 'ALTER USER 'userslave1'@'192.168.183.%' IDENTIFIED WITH 'mysql_native_password' AS '*ABA990061718A9097EA66815915693F747D60316''
  Replicate_Ignore_Server_Ids: 
             Master_Server_Id: 101
                  Master_UUID: 68605903-eea6-11e7-89e4-525400e9451c
             Master_Info_File: /var/lib/mysql/master.info
                    SQL_Delay: 0
          SQL_Remaining_Delay: NULL
      Slave_SQL_Running_State: 
           Master_Retry_Count: 86400
                  Master_Bind: 
      Last_IO_Error_Timestamp: 
     Last_SQL_Error_Timestamp: 180107 22:42:09
               Master_SSL_Crl: 
           Master_SSL_Crlpath: 
           Retrieved_Gtid_Set: 
            Executed_Gtid_Set: 
                Auto_Position: 0
         Replicate_Rewrite_DB: 
                 Channel_Name: 
           Master_TLS_Version: 
1 row in set (0.00 sec)

mysql> 
```



mysql主从复制，经常会遇到错误而导致slave端复制中断，这个时候一般就需要人工干预，跳过错误才能继续

跳过错误有两种方式：

**1.跳过指定数量的事件：**

当使用这个语句的时候，需要理解二进制日志实际上是作为一系列的事件组。每个事件组包含一系列的事件。对于事务表，一个事件组对应一个事务。对于非事务表，一个事件组对应一条单独的SQL语句。需要注意的是，一个单独的事务可能既包含事务表，也包含非事务表。

当使用SET GLOBAL sql\_slave\_skip\_counter跳过事件时，slave节点会处于事务组的中间，它会继续跳过一些事件直到它到达一个事务组的结束位置，然后slave节点会从下一个事件组开始执行。

这个参数的默认值是0

mysql&gt;set global sql\_slave\_skip\_counter=1        \#  跳过一个事件

mysql&gt; stop slave;

mysql&gt; start slave;



**2.修改mysql的配置文件，通过slave\_skip\_errors参数来跳所有错误或指定类型的错误**

vi /etc/my.cnf

\[mysqld\]

\#slave-skip-errors=1062,1053,1146          \# 跳过指定error no类型的错误

\#slave-skip-errors=all                                  \# 跳过所有错误



```
mysql> show slave status\G
*************************** 1. row ***************************
               Slave_IO_State: Waiting for master to send event
                  Master_Host: 192.168.183.101
                  Master_User: userslave1
                  Master_Port: 3306
                Connect_Retry: 60
              Master_Log_File: mysql-bin.000006
          Read_Master_Log_Pos: 154
               Relay_Log_File: relay-bin.000012
                Relay_Log_Pos: 320
        Relay_Master_Log_File: mysql-bin.000006
             Slave_IO_Running: Yes                   # 如果这两个为Yes就表示，主从同步工作正常。
            Slave_SQL_Running: Yes                   # 如果这两个为Yes就表示，主从同步工作正常。

```



