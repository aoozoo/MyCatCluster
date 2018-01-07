# **MySQL的几种日志文件（可能不是全部的）：**

1. error log      \# 错误日志
2. general log     \# 普通的查询日志
3. slow query log  \# 慢查询日志
4. binary log \# 慢查询日志
5. event log \# 事务日志

# 开启二进制日志

> yun安装的MySQL默认是没有开启二进制日志的

```
mysql> show variables like '%log_bin%';
+---------------------------------+-------+
| Variable_name                   | Value |
+---------------------------------+-------+
| log_bin                         | OFF   |
| log_bin_basename                |       |
| log_bin_index                   |       |
| log_bin_trust_function_creators | OFF   |
| log_bin_use_v1_row_events       | OFF   |
| sql_log_bin                     | ON    |
+---------------------------------+-------+
6 rows in set (0.00 sec)

mysql>
```

> 开启的方法有两种：
>
> 1. 修改配置文件
> 2. 修改MySQL的变量

**改配置文件的方法**

```
sed -i '/\[mysqld\]/a\log-bin=/var/lib/mysql/mysql-bin' /etc/my.cnf     # mysql-bin 是log_bin_basename
sed -i '/\[mysqld\]/a\server_id=mysqlnode_101' /etc/my.cnf         

systemctl restart mysqld    # 重启mysqld
```

# 验证已经开启二级制日志

```
[root@node_101_192.168.183.101 ~]#  ls -1 /var/lib/mysql/mysql-bin.*
/var/lib/mysql/mysql-bin.000001
/var/lib/mysql/mysql-bin.index                      # 这个是二级制日志的索引文件
[root@node_101_192.168.183.101 ~]#

mysql> show variables like '%log_bin%';                # 现在看到二级制日志已经打开
ERROR 2006 (HY000): MySQL server has gone away
No connection. Trying to reconnect...
Connection id:    3
Current database: *** NONE ***

+---------------------------------+--------------------------------+
| Variable_name                   | Value                          |
+---------------------------------+--------------------------------+
| log_bin                         | ON                             |
| log_bin_basename                | /var/lib/mysql/mysql-bin       |
| log_bin_index                   | /var/lib/mysql/mysql-bin.index |
| log_bin_trust_function_creators | OFF                            |
| log_bin_use_v1_row_events       | OFF                            |
| sql_log_bin                     | ON                             |
+---------------------------------+--------------------------------+
6 rows in set (0.01 sec)

mysql>
```

> 每次重启mysqld都会调用 flush logs 创建一个新的二进制日志文件

# 查看二级制日志文件

```
mysql> show master logs;       # 列出日志文件名称
#  cat /var/lib/mysql/mysql-bin.index

mysql> show master status;     # 查看当前的二进制日志文件，及其position
```

# 查看二进制日志内容

```
mysqlbinlog /var/lib/mysql/mysql-bin.000001 -vv

mysql> show binlog events;
mysql> show master status;
mysql> show binlog events in 'mysql-bin.000001';
```

# 二进制日志中的事件

> 事件由事件头和事件体两部分组成

```
# at 154
#180104  1:29:27 server id 0  end_log_pos 219 CRC32 0xf44a8387     Anonymous_GTID    last_committed=0    sequence_number=1    rbr_only=no
SET @@SESSION.GTID_NEXT= 'ANONYMOUS'/*!*/;
# at 219
#180104  1:29:27 server id 0  end_log_pos 313 CRC32 0x16f7a6b9     Query    thread_id=3    exec_time=0    error_code=0
SET TIMESTAMP=1515000567/*!*/;
SET @@session.pseudo_thread_id=3/*!*/;
SET @@session.foreign_key_checks=1, @@session.sql_auto_is_null=0, @@session.unique_checks=1, @@session.autocommit=1/*!*/;
SET @@session.sql_mode=1436549152/*!*/;
SET @@session.auto_increment_increment=1, @@session.auto_increment_offset=1/*!*/;
/*!\C utf8 *//*!*/;
SET @@session.character_set_client=33,@@session.collation_connection=33,@@session.collation_server=8/*!*/;
SET @@session.lc_time_names=0/*!*/;
SET @@session.collation_database=DEFAULT/*!*/;
create database test
/*!*/;
```

# 滚动二进制日志文件

```
mysql> flush logs;
```

# 清空所有二进制日志文件

```
reset master;
```



