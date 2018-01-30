> MySQL的高可用方案可以参考mycat的官方文档。



* vim /usr/local/mycat/conf/schema.xml

```
        <dataHost name="localhost1" maxCon="1000" minCon="10" balance="3"
                          writeType="0" dbType="mysql" dbDriver="native" switchType="1"  slaveThreshold="100">
                <heartbeat>select user()</heartbeat>
                <!-- can have multi write hosts -->
                <writeHost host="hostM1" url="192.168.183.102:3306" user="root"
                                   password="123456">
                        <!-- can have multi read hosts -->
                        <readHost host="hostS2" url="192.168.183.103:3306" user="root" password="123456" />
                </writeHost>
                <!-- <writeHost host="hostS1" url="localhost:3316" user="root"
                                   password="123456" />   -->
                <writeHost host="hostM2" url="192.168.183.101:3306" user="root" password="123456"/>
        </dataHost>

```

* 重启MyCat

> /usr/local/mycat/bin/mycat stop
>
> /usr/local/mycat/bin/mycat start

* 测试

> 打开MySQL的general\_log，然后tail查看MySQL的general\_log，开始所有的写操作都，写入到了102节点，把102节点的MySQL服务停止之后，就在101节点写入了。



