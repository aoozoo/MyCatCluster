```
grant replication slave on *.* to 'userslave1'@'192.168.183.%' identified by 'asdF@123';    # 创建用户

# 开启二进制日志
sed -i '/\[mysqld\]/a\log-bin=/var/lib/mysql/mysql-bin' /etc/my.cnf     # mysql-bin 是log_bin_basename
sed -i '/\[mysqld\]/a\server-id=101' /etc/my.cnf         # 需要注意的是server-id要唯一，各个节点不能够重复，而且必须为数值

systemctl restart mysqld    # 重启mysqld
```



