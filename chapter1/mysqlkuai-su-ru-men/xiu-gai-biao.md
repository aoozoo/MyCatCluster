```
insert into userinfo (name) values ('hello');         # 往表中插入数据
select * from userinfo;
select * from userinfo where id = 1;
update userinfo set name = 'helloHi' where id = 1;    # 修改表中的数据      更新数据一定要带where条件，否则会更新所有的数据。
delete from userinfo where id = 1;                    # 删除表中的数据      删除数据一定要带where条件，否则会删除所有的数据。
```



