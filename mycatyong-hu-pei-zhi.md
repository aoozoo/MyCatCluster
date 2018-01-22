```
#  vim /usr/local/mycat/conf/server.xml

        <user name="root"> 
                <property name="password">123456</property> 
                <property name="schemas">TESTDB</property> 

                <!-- 表级 DML 权限设置 --> 
                <!--             
                <privileges check="false"> 
                        <schema name="TESTDB" dml="0110" > 
                                <table name="tb01" dml="0000"></table> 
                                <table name="tb02" dml="1111"></table> 
                        </schema> 
                </privileges>            
                 --> 
        </user> 

        <user name="user"> 
                <property name="password">user</property> 
                <property name="schemas">TESTDB</property> 
                <property name="readOnly">true</property> 
        </user>
```



