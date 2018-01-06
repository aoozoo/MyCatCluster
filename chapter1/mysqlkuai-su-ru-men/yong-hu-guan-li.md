# 创建用户

```
create user 'jason.pan'@'%' identified by '123123';
```

# 查看用户

```sql
desc mysql.user;    # 查看用户表结构
select user,host from mysql.user;   # 查看当前的所有用户,字段名字不区分大小写
```

# 修改用户

```
alter user 'jason.pan'@'%' identified by '1234';
```

# 给用户授权

```
grant replication slave on *.* to 'Jason.Pan'@'%' identified by '123123' with grant option;
grant all on *.* to 'Jason.Pan'@'%' identified by '123123' with grant option;
```

# 回收用户权限

```
revoke all on *.* from 'Jason.Pan'@'%';
```

# 删除用户

```
drop user 'Jason.Pan'@'%';
```



