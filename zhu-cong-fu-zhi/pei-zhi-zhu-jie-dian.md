```
grant replication slave on *.* to 'userslave1'@'192.168.183.%' identified by 'asdF@123';    # 创建用户
 
# 开启二进制日志
sed -i '/\[mysqld\]/a\log-bin=/var/lib/mysql/mysql-bin' /etc/my.cnf     # mysql-bin 是log_bin_basename
sed -i '/\[mysqld\]/a\server-id=mysqlnode_101' /etc/my.cnf         

systemctl restart mysqld    # 重启mysqld
```



