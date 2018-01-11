# 安装keepalived

```
wget http://www.keepalived.org/software/keepalived-1.4.0.tar.gz
cd keepalived-1.4.0
yum -y install openssl-devel libnl3-devel ipset-devel iptables-devel libnfnetlink-devel net-snmp-devel glib2-devel json-c-devel
./configure && make && make install

```



