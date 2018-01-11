# 安装HAProxy

```
wget http://www.haproxy.org/download/1.8/src/haproxy-1.8.3.tar.gz
tar zxf haproxy-1.8.3.tar.gz
cd haproxy-1.8.3
make TARGET=linux2628 USE_PCRE=1 USE_OPENSSL=1 USE_ZLIB=1 PREFIX=/data/programs/haproxy
[root@node_105_192.168.183.105 ~/haproxy-1.8.3]#  make install PREFIX=/data/programs/haproxy
install -d "/data/programs/haproxy/sbin"
install haproxy  "/data/programs/haproxy/sbin"
install -d "/data/programs/haproxy/share/man"/man1
install -m 644 doc/haproxy.1 "/data/programs/haproxy/share/man"/man1
install -d "/data/programs/haproxy/doc/haproxy"
for x in configuration management architecture peers-v2.0 cookie-options lua WURFL-device-detection proxy-protocol linux-syn-cookies network-namespaces DeviceAtlas-device-detection 51Degrees-device-detection netscaler-client-ip-insertion-protocol peers close-options SPOE intro; do \
    install -m 644 doc/$x.txt "/data/programs/haproxy/doc/haproxy" ; \
done
[root@node_105_192.168.183.105 ~/haproxy-1.8.3]#
```

```
cd /data/programs/haproxy/
mkdir conf
cd conf/
vim haproxy.cnf
[root@node_105_192.168.183.105 /data/programs/haproxy/conf]#  cat haproxy.cnf 
global
        daemon               # 后台方式运行
        nbproc 1
        pidfile /data/programs/haproxy/conf/haproxy.pid


defaults
        mode tcp               #默认的模式mode { tcp|http|health }，tcp是4层，http是7层，health只会返回OK
        retries 2               #两次连接失败就认为是服务器不可用，也可以通过后面设置
        option redispatch       #当serverId对应的服务器挂掉后，强制定向到其他健康的服务器
        option abortonclose     #当服务器负载很高的时候，自动结束掉当前队列处理比较久的链接
        maxconn 4096            #默认的最大连接数
        timeout connect 5000ms  #连接超时
        timeout client 30000ms  #客户端超时
        timeout server 30000ms  #服务器超时
        #timeout check 2000      #=心跳检测超时
        log 127.0.0.1 local0 err #[err warning info debug]


########test1配置#################
listen test1                         #这里是配置负载均衡，test1是名字，可以任意
        bind 0.0.0.0:3306            #这里是监听的IP地址和端口，端口号可以在0-65535之间，要避免端口冲突
        mode tcp                     #连接的协议，这里是tcp协议
        #maxconn 4086
        #log 127.0.0.1 local0 debug
        server s1 192.168.183.103:3306 #负载的机器
        server s2 192.168.183.104:3306 #负载的机器，负载的机器可以有多个，往下排列即可
[root@node_105_192.168.183.105 /data/programs/haproxy/conf]# 
echo 1 > haproxy.pid
/data/programs/haproxy/sbin/haproxy -f /data/programs/haproxy/conf/haproxy.cnf

```



# 启用HAProxy的监控功能



```
[root@node_105_192.168.183.105 /data/programs/haproxy]#  cat /data/programs/haproxy/conf/haproxy.cnf
global
        daemon               # 后台方式运行
        nbproc 1
        pidfile /data/programs/haproxy/conf/haproxy.pid


defaults
        mode tcp               #默认的模式mode { tcp|http|health }，tcp是4层，http是7层，health只会返回OK
        retries 2               #两次连接失败就认为是服务器不可用，也可以通过后面设置
        option redispatch       #当serverId对应的服务器挂掉后，强制定向到其他健康的服务器
        option abortonclose     #当服务器负载很高的时候，自动结束掉当前队列处理比较久的链接
        maxconn 4096            #默认的最大连接数
        timeout connect 5000ms  #连接超时
        timeout client 30000ms  #客户端超时
        timeout server 30000ms  #服务器超时
        #timeout check 2000      #=心跳检测超时
        log 127.0.0.1 local0 err #[err warning info debug]
	balance source


########test1配置#################
listen test1                         #这里是配置负载均衡，test1是名字，可以任意
        bind 0.0.0.0:3306            #这里是监听的IP地址和端口，端口号可以在0-65535之间，要避免端口冲突
        mode tcp                     #连接的协议，这里是tcp协议
        #maxconn 4086
        #log 127.0.0.1 local0 debug
        server s1 192.168.183.103:3306 check port 3306 #负载的机器
        server s2 192.168.183.104:3306 check port 3306 #负载的机器，负载的机器可以有多个，往下排列即可
listen admin_stats
	bind 0.0.0.0:8888
	mode http
	stats uri /my_haproxy
	stats auth admin:123123
[root@node_105_192.168.183.105 /data/programs/haproxy]#
[root@node_105_192.168.183.105 /data/programs/haproxy]#  ps -ef | grep haproxy | awk '{print $2}' | xargs kill
[root@node_105_192.168.183.105 /data/programs/haproxy]#  /data/programs/haproxy/sbin/haproxy -f /data/programs/haproxy/conf/haproxy.cnf
```

http://192.168.183.105:8888/my\_haproxy



