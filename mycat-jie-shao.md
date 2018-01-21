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

MyCat默认有个user的用户，密码也为user。





