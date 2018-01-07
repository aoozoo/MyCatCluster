```
sed -i '/\[mysqld\]/a\relay-log-index=/var/lib/mysql/relay-bin.index' /etc/my.cnf
sed -i '/\[mysqld\]/a\relay-log=/var/lib/mysql/relay-bin' /etc/my.cnf
sed -i '/\[mysqld\]/a\server-id=102' /etc/my.cnf                   # server-id 要唯一，各个节点不能够重复，而且必须为数值

systemctl restart mysqld   # 重启服务
```



