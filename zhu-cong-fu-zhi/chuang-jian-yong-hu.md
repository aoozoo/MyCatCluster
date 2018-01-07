# 创建slave用户

```
grant replication slave on *.* to 'slaveuser'@'192.168.183.%' identified by '123123';   # 创建用户并且授权
select user,host from mysql.user\G                                  # 查看用户表
```



