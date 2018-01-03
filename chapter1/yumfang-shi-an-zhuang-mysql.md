## 安装MySQL的**yum仓库**

[http://dev.mysql.com/downloads/repo/yum/](http://dev.mysql.com/downloads/repo/yum/)

选择相应的版本进行下载 

wget [https://repo.mysql.com//mysql57-community-release-el7-11.noarch.rpm](https://repo.mysql.com//mysql57-community-release-el7-11.noarch.rpm)

安装

yum -y install mysql57-community-release-el7-11.noarch.rpm

如果需要安装5.6或者5.5版本的MySQL，修改

/etc/yum.repos.d/mysql-community.repo

文件，打开相应的仓库即可

如果之前安装过mariadb，清除一下，否则可能会报错。

yum -y remove mariadb-libs MariaDB-common\*  MariaDB\*

然后删除/var/lib/mysql 目录

安装MySQL5.7

yum -y install mysql-community-server

启动mysqld

systemctl start mysqld

mysqld 第一次启动的时候，会为root用户创建一个默认的密码

密码保存在log-error文件中

```shell
[root@node_101_192.168.183.101 ~]#  fgrep log-error /etc/my.cnf
log-error=/var/log/mysqld.log
```

查看mysqld root用户的默认密码

fgrep password /var/log/mysqld.log \| awk '{print $NF}'

登录mysql

mysql -uroot -p$\(fgrep password /var/log/mysqld.log \| awk '{print $NF}' \| head -1\)

使用mysql执行命令

mysql --connect-expired-password -uroot -p$\(fgrep password /var/log/mysqld.log \| awk '{print $NF}' \| head -1\) -e "alter user 'root'@'localhost' identified by 'Kinson@123';"

