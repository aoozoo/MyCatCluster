# 安装keepalived

```
wget http://www.keepalived.org/software/keepalived-1.4.0.tar.gz
tar zxf keepalived-1.4.0.tar.gz
cd keepalived-1.4.0
yum -y install openssl-devel libnl3-devel ipset-devel iptables-devel libnfnetlink-devel net-snmp-devel glib2-devel json-c-devel
./configure && make && make install
```

# 配置文件

```
[root@node_105_192.168.183.105 ~]#  cat /usr/local/etc/keepalived/keepalived.conf 
! Configuration File for keepalived

global_defs {
   router_id LVS_node105   # 这个名字必须唯一
}

vrrp_instance VI_1 {
    state MASTER
    interface eth0
    virtual_router_id 51   # 这个每个节点必须相同
    priority 100
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
        # 192.168.200.16
        # 192.168.200.17
        192.168.183.200
    }
}
[root@node_105_192.168.183.105 ~]# 

[root@node_106_192.168.183.106 ~/keepalived-1.4.0]#  cat /usr/local/etc/keepalived/keepalived.conf 
! Configuration File for keepalived 

global_defs { 
   router_id LVS_node106
} 
 
vrrp_instance VI_1 { 
    state BACKUP
    interface eth0 
    virtual_router_id 51 
    priority 90 
    advert_int 1 
    authentication { 
        auth_type PASS 
        auth_pass 1111 
    } 
    virtual_ipaddress { 
        # 192.168.200.16 
        # 192.168.200.17 
        192.168.183.200 
    } 
}
[root@node_106_192.168.183.106 ~/keepalived-1.4.0]#
```

# 启动

```
[root@node_106_192.168.183.106 ~]#  /usr/local/sbin/keepalived -D -f /usr/local/etc/keepalived/keepalived.conf
[root@node_105_192.168.183.105 ~]#  /usr/local/sbin/keepalived -D -f /usr/local/etc/keepalived/keepalived.conf

[root@node_105_192.168.183.105 ~]#  ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN qlen 1
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP qlen 1000
    link/ether 52:54:00:db:c6:9c brd ff:ff:ff:ff:ff:ff
    inet 192.168.183.105/24 brd 192.168.183.255 scope global eth0
       valid_lft forever preferred_lft forever
    inet 192.168.183.200/32 scope global eth0
       valid_lft forever preferred_lft forever
[root@node_105_192.168.183.105 ~]#
```



