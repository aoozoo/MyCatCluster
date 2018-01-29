### 安装JDK

---

```
mkdir -p /data/programs/java
cd /data/programs/java
wget http://mirrors.linuxeye.com/jdk/jdk-8u144-linux-x64.tar.gz
tar zxf jdk-8u144-linux-x64.tar.gz
ln -s jdk1.8.0_144 latest
echo '
JAVA_HOME=/data/programs/java/latest
JRE_HOME=/data/programs/java/latest/jre
PATH=$PATH:$JAVA_HOME/bin:$JRE_HOME/bin
CLASSPATH=$JAVA_HOME/jre/lib/ext:$JAVA_HOME/lib/tools.jar:$JAVA_HOME/lib/dt.jar:$JRE_HOME/lib
export PATH JAVA_HOME JRE_HOME CLASSPATH
' > /etc/profile.d/java.sh
source /etc/profile.d/java.sh
java -version
```

### 安装MyCat

---

```
cd /usr/local/
tar zxf Mycat-server-1.6-RELEASE-20161028204710-linux.tar.gz

# 如果虚拟机的内存比较小，需要修改几个参数才能成功启动MyCat
vim mycat/conf/wrapper.conf
    wrapper.java.additional.5=-XX:MaxDirectMemorySize=512M
    wrapper.java.additional.10=-Xmx1G
    wrapper.java.additional.11=-Xms256M

/usr/local/mycat/bin/mycat start    # 启动mycat
netstat -lntpu | grep java          # mycat 侦听了8066端口
```

MyCat默认有个user的用户，密码也为user

root用户，密码为123456

端口为8066

![](/assets/2018-01-21_15h31_41.png)

#### 内存设置

[http://www.cnblogs.com/ivictor/p/5319833.html](http://www.cnblogs.com/ivictor/p/5319833.html)

vim /usr/local/mycat/conf/wrapper.conf

```
# Initial Java Heap Size (in MB)
#wrapper.java.initmemory=3

# Maximum Java Heap Size (in MB)
#wrapper.java.maxmemory=64

# 单位为MB，如果填写2G，那么会设置为2MB，字符G会被忽略
```

#### 启动报错

less /usr/local/mycat/logs/wrapper.log

```
Startup failed: Timed out waiting for a signal from the JVM.
```

启动如上报错，解决方法：

vim /usr/local/mycat/conf/wrapper.conf

```
wrapper.startup.timeout=7200
wrapper.ping.timeout=3600
```

wrapper.startup.timeout     \# 增加这行配置



#### dble对MyCat做的增强

https://github.com/actiontech/dble/wiki/dble%E5%AF%B9MyCat%E5%81%9A%E7%9A%84%E5%A2%9E%E5%BC%BA

