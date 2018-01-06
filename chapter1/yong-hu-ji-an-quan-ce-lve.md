MySQL默认的安全策略，要求在没有修改默认的root的密码之前，不能够对MySQL进行操作，会提示你修改密码

如下：

```shell
[root@node_101_192.168.183.101 ~]#  mysql -uroot -p$(fgrep password /var/log/mysqld.log | awk '{print $NF}' | head -1) -e "show variables like '%validate%';"
mysql: [Warning] Using a password on the command line interface can be insecure.
Please use --connect-expired-password option or invoke mysql in interactive mode.
[root@node_101_192.168.183.101 ~]#

[root@node_101_192.168.183.101 ~]#  mysql --connect-expired-password -uroot -p$(fgrep password /var/log/mysqld.log | awk '{print $NF}' | head -1) -e "show variables like '%validate%';"
mysql: [Warning] Using a password on the command line interface can be insecure.
ERROR 1820 (HY000) at line 1: You must reset your password using ALTER USER statement before executing this statement.
[root@node_101_192.168.183.101 ~]#
```

默认的密码策略,这个其实与validate\_password\_policy的值有关，RPM安装默认装载了validate\_password这个插件。

首先修改默认的root密码

```bash
[root@node_101_192.168.183.101 ~]#  mysql --connect-expired-password -uroot -p$(fgrep password /var/log/mysqld.log | awk '{print $NF}' | head -1) -e "alter user 'root'@'localhost' identified by 'Kinson@123';"
mysql: [Warning] Using a password on the command line interface can be insecure.
[root@node_101_192.168.183.101 ~]#
```

下面来看看validate\_password插件提供的参数

```bash
[root@node_101_192.168.183.101 ~]#  mysql -uroot -p'Kinson@123' -e "show variables like '%validate%';"
mysql: [Warning] Using a password on the command line interface can be insecure.
+--------------------------------------+--------+
| Variable_name                        | Value  |
+--------------------------------------+--------+
| query_cache_wlock_invalidate         | OFF    |
| validate_password_check_user_name    | OFF    |
| validate_password_dictionary_file    |        |
| validate_password_length             | 8      |
| validate_password_mixed_case_count   | 1      |
| validate_password_number_count       | 1      |
| validate_password_policy             | MEDIUM |
| validate_password_special_char_count | 1      |
+--------------------------------------+--------+
[root@node_101_192.168.183.101 ~]#


validate_password=ON/OFF/FORCE/FORCE_PLUS_PERMANENT: 决定是否使用该插件(及强制/永久强制使用)。

validate_password_dictionary_file：插件用于验证密码强度的字典文件路径。

validate_password_check_user_name：检查用户名。

validate_password_length：密码最小长度，默认为8，最小为4。

validate_password_mixed_case_count：密码至少要包含的小写字母个数和大写字母个数。

validate_password_number_count：密码至少要包含的数字个数。

validate_password_policy：密码强度检查等级，0/LOW（只检查长度）、1/MEDIUM（检查长度、数字、大小写、特殊字符）、2/STRONG（检查长度、数字、大小写、特殊字符字典文件）。

validate_password_special_char_count：密码至少要包含的特殊字符数。

如果修改了validate_password_number_count，validate_password_special_char_count，validate_password_mixed_case_count中任何一个值，则validate_password_length将进行动态修改。另外如果你显性指定validate_password_length的值小于4，尽管不会报错，但validate_password_length的值将设为4。


另外，需要注意的一点是从MySQL 5.7–>MySQL 5.7.11版本之间，默认的用户密码过期时间为360天，变量default_password_lifetime。
[root@node_101_192.168.183.101 ~]#  mysql -uroot -p'123456' -e "show global variables like 'default_password_lifetime';"
mysql: [Warning] Using a password on the command line interface can be insecure.
+---------------------------+-------+
| Variable_name             | Value |
+---------------------------+-------+
| default_password_lifetime | 0     |
+---------------------------+-------+
[root@node_101_192.168.183.101 ~]# 
所以如果你使用的是此版本一定需要注意，你可以把此值改为0，表示永不失效。从MySQL 5.7.11开始官方也意识到这是个巨坑，所以就默认把此值改为了0。
```

validate\_password\_policy有以下取值：

```bash
Policy            Tests Performed
0 or LOW    Length
1 or MEDIUM    Length; numeric, lowercase/uppercase, and special characters
2 or STRONG    Length; numeric, lowercase/uppercase, and special characters; dictionary file
```

默认是1，即MEDIUM，所以刚开始设置的密码必须符合长度，且必须含有数字，小写或大写字母，特殊字符。

有时候，只是为了自己测试，不想密码设置得那么复杂，譬如说，我只想设置root的密码为123456。

必须修改两个全局参数：

set global validate\_password\_policy=0;

这样，判断密码的标准就基于密码的长度了

密码长度由validate\_password\_length参数来决定

```bash
[root@node_101_192.168.183.101 ~]#  mysql -uroot -p'Kinson@123' -e "set global validate_password_policy=0;"
mysql: [Warning] Using a password on the command line interface can be insecure.
[root@node_101_192.168.183.101 ~]#  mysql -uroot -p'Kinson@123' -e "set global validate_password_length=1;"
mysql: [Warning] Using a password on the command line interface can be insecure.
[root@node_101_192.168.183.101 ~]# 

[root@node_101_192.168.183.101 ~]#  mysql -uroot -p'Kinson@123' -e "show variables like '%validate%';"
mysql: [Warning] Using a password on the command line interface can be insecure.
+--------------------------------------+-------+
| Variable_name                        | Value |
+--------------------------------------+-------+
| query_cache_wlock_invalidate         | OFF   |
| validate_password_check_user_name    | OFF   |
| validate_password_dictionary_file    |       |
| validate_password_length             | 4     |        # 最小要求是4位长度，所以最终生效的不是设置的一位长度
| validate_password_mixed_case_count   | 1     |
| validate_password_number_count       | 1     |
| validate_password_policy             | LOW   |
| validate_password_special_char_count | 1     |
+--------------------------------------+-------+
[root@node_101_192.168.183.101 ~]#
```

需要注意的一点是从MySQL 5.7–&gt;MySQL 5.7.11版本之间，默认的用户密码过期时间为360天，变量default\_password\_lifetime.

所以如果你使用的是此版本一定需要注意，你可以把此值改为0，表示永不失效。从MySQL 5.7.11开始官方也意识到这是个巨坑，所以就默认把此值改为了0.

