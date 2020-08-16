# hive
## 搭建hive环境
### 安装mysql
版本mysql5.7.30，作为hive的metastore
* 安装包解压到/opt/module/，以下称根目录为/opt/module/mysql
* 创建/opt/module/mysql/data作为mysql存储数据的文件夹
* 创建系统账户mysql，并赋予相应权限，用于运行mysql服务
  ```
  # useradd -r mysql
  # chown -R mysql:mysql /opt/module/mysql
  # chmod 777 /opt/module/mysql
  ```
* 修改mysql配置文件/etc/my.cnf，如果有mysqld_safe的配置需要注释掉
  ```
  # vim /opt/etc/my.cnf

  [mysqld]
  basedir=/opt/module/mysql
  datadir=/opt/module/mysql/data
  socket=/tmp/mysql.sock
  port=3306
  character_set_server=latin1 # 如果为utf-8可能会出现删除表时卡死
  [mysqld_safe]
  # 注释掉mysqld_safe的配置
  ```

* 初始化mysql，注意记录系统给的初始密码，用于第一次登陆
  ```
  # /opt/module/mysql/bin/mysqld --user=mysql --basedir=/opt/module/mysql --datadir=/opt/module/mysql/data --initialize
  ```
* 将mysql服务设为开机启动
  ```
  # cp /opt/module/mysql/support-files/mysql.server /etc/init.d/
  # chmod +x /etc/init.d/mysql.server
  # chkconfig --add mysql.server
  ```
* 启动mysql服务
  ```
  # systemctl start mysql.server
  ```
* 将mysql根目录添加到系统环境变量，并第一次运行mysql，修改root密码
  ```
  # mysql -u root -p
  password:输入刚才记录的初始密码
  mysql> ALTER USER 'root'@'localhost' IDENTIFIED WITH mysql_native_password BY '新密码';
  ```
* 修改root用户的权限，使其可以远程登录
  ```
  mysql> use mysql
  # 查看现有设置
  mysql> select user,host from user where user='root';
  +-----+---------+
  |user |host     |
  +-----+---------+
  |root |localhost|
  +-----|---------+
  # 设置root用户可远程登录而不是只在本地
  mysql> UPDATE user SET host='%' WHERE user='root';
  # 查看新设置
  mysql> mysql> select user,host from user where user='root';
  +-----+-----+
  |user |host |
  +-----+-----+
  |root |%    |
  +-----|-----+
  ```

### 安装hive
#### 安装
* 安装包解压到/opt/module/hive
* 如果hadoop有/opt/module/hadoop/share/hadoop/common/lib/slf4j-log4j12-x.x.x.jar包，hive也有/opt/module/hive/lib/log4j-slf4j-impl-x.x.x.jar包，两个包只能保留一个。否则启动hive会报错，二者冲突
* 如果/opt/module/hadoop/share/hadoop/common/lib/guava-x.x-jre.jar和l/opt/module/hive/lib/guava-x.x.jar版本不一致，删除低版本的包，将高版本的包复制过去。否则启动hive会报错
* 因为hive需要jdbc才能使用mysql，因此要将mysql的jdbc驱动jar包放进/opt/module/hive/lib文件夹
#### 配置mysql
* 添加数据仓库hive专用存储metastore
  ```
  mysql> CREATE DATABASE hive;
  mysql> alter database hive character set latin1; # 不修改编码会出现hive无法删除表的情况

  # 退出mysql后重启mysql服务让以上设置生效
  # systemctl restart mysql.server
  ```
* 添加远程用户hive，供hive使用
  ```
  mysql> CREATE USER 'hive'@'%' IDENTIFIED BY 'hive';
  mysql> GRANT ALL ON hive.* TO 'hive'@'%' WITH GRANT OPTION;
  mysql> flush privileges
  ```
#### 配置hive
hive配置文件分为服务端和客户端来说明
1. HDFS的存储位置
   ```
   # hdfs dfs -mkdir -p /hive/warehouse
   # hdfs dfs -mkdir -p /hive/tmp
   # hdfs dfs -mkdir -p /hive/log
   # hdfs dfs -chmod g+w /hive/warehouse
   # hdfs dfs -chmod g+w /hive/tmp
   # hdfs dfs -chmod g+w /hive/log
   ```
2. 对于服务端和客户端都要做
   * 复制配置文件
     ```
     # cd /opt/module/hive-3.1.1/conf/
     # cp hive-log4j2.properties.template hive-log4j2.properties
     # cp hive-default.xml.template hive-default.xml
     # cp hive-default.xml.template hive-site.xml
     # cp hive-env.sh.template hive-env.sh
     ```
   * 配置hive-env.sh
     ```
     HADOOP_HOME=/opt/module/hadoop
     export HIVE_CONF_DIR=/opt/module/hive/conf
     export HIVE_AUX_JARS_PATH=/opt/module/hive/lib
     ```
   * 配置hive-log4j.properties
     先创建log存放的文件夹/opt/module/hive/logs，再修改配置文件
     ```
     # vim /opt/module/hive/conf/hive-log4j.properties

     hive.log.dir=/opt/module/hive/logs
     ```
3. 配置hive-site.xml
   * 服务端配置
     ```
     <?xml version="1.0" encoding="UTF-8" standalone="no"?>
     <?xml-stylesheet type="text/xsl" href="configuration.xsl"?>
     <configuration>
         <property>
             <name>javax.jdo.option.ConnectionURL</name>
             # 你的mysqlIP和port
             <value>jdbc:mysql://mysqlIP:port/hive?allowMultiQueries=true&amp;useSSL=false&amp;verifyServerCertificate=false&amp;allowPublicKeyRetrieval=true</value>
         </property>
         <property>
             <name>javax.jdo.option.ConnectionDriverName</name>
             <value>com.mysql.jdbc.Driver</value>
         </property>
         <property>
             <name>javax.jdo.option.ConnectionUserName</name>
             # 创建给hive使用的mysql账户名
             <value>hive</value>
         </property>
         <property>
             <name>javax.jdo.option.ConnectionPassword</name>
             # 创建给hive使用的mysql账户密码
             <value>hive</value>
         </property>
         <property>
             <name>hive.metastore.local</name>
             # 非本地模式运行
             <value>false</value>
         </property>
         # 当metastore服务器同时也是一台客户端时需要加上这个属性
         <property>
             <name>hive.metastore.uris</name>
             <value>thrift://192.168.184.104:9083</value>
         </property>
         # 对应前面创建的三个hdfs目录的配置
         <property>
             <name>hive.metastore.warehouse.dir</name>
             <value>/hive/warehouse</value>
             <description>location of default database for the warehouse</description>
         </property>
         <property>
             <name>hive.exec.scratchdir</name>
             <value>/hive/tmp</value>
         </property>
         <property>
             <name>hive.querylog.location</name>
             <value>/hive/log</value>
         </property>
         # 设置hive显示表头
         <property>
             <name>hive.cli.print.header</name>
             <value>true</value>
         </property>
         # 增加hive结果可读性，不显示表名
         <property>
             <name>hive.resultset.use.unique.column.names</name>
             <value>false</value>
         </property>
         <property>
             <name>hive.metastore.schema.verification</name>
             <value>false</value>
         </property>
         <property>
             <name>datanucleus.schema.autoCreateAll</name>
             <value>true</value>
         </property>
         <property>
             <name>datanucleus.readOnlyDatastore</name>
             <value>false</value>
         </property>
         <property>
             <name>datanucleus.fixedDatastore</name>
             <value>false</value>
         </property>
         <property>
             <name>datanucleus.autoCreateSchema</name>
             <value>true</value>
         </property>
         <property>
             <name>datanucleus.autoCreateTables</name>
             <value>true</value>
         </property>
         <property>
             <name>datanucleus.autoCreateColumns</name>
             <value>true</value>
         </property>
     </configuration>
     ```
   * 客户端配置
     ```
     <?xml version="1.0" encoding="UTF-8" standalone="no"?>
     <?xml-stylesheet type="text/xsl" href="configuration.xsl"?>
     <configuration>
         <property>
             <name>hive.metastore.local</name>
             # 非本地模式运行
             <value>false</value>
         </property>
         # 当metastore服务器的地址
         <property>
             <name>hive.metastore.uris</name>
             <value>thrift://192.168.184.104:9083</value>
         </property>
         # 对应前面创建的三个hdfs目录的配置
         <property>
             <name>hive.metastore.warehouse.dir</name>
             <value>/hive/warehouse</value>
             <description>location of default database for the warehouse</description>
         </property>
         <property>
             <name>hive.exec.scratchdir</name>
             <value>/hive/tmp</value>
         </property>
         <property>
             <name>hive.querylog.location</name>
             <value>/hive/log</value>
         </property>
     </configuration>
     ```
* 初始化数据库
  ```# schematool  -initSchema -dbType mysql```
* 启动metastore远程数据库
  ```# nohup hive --service metastore &```
* 此后即可使用各种方式启动hive

#### 配置和启动hiveserver2
* 在准备运行hiveserver2服务器上的hive-site.xml添加配置，不需要在每台服务器都设置
  ```
  <property>
      <name>hive.server2.thrift.port</name>
      <value>10000</value>
  </property>
  <property>
      <name>hive.server2.thrift.http.port</name>
      <value>10001</value>
      <description>Port number of HiveServer2 Thrift interface when hive.server2.transport.mode is 'http'.</description>
  </property>
  <property>
      <name>hive.server2.thrift.bind.host</name>
      <description>准备运行hiveserver2服务的主机地址，无需在每台服务器上都设置</description>
      <value>hadoop101</value>
  </property>
  <property>
      <name>hive.server2.thrift.client.user</name>
      <value>hadoop</value>
      <description>Username to use against thrift client</description>
  </property>
  <property>
      <name>hive.server2.thrift.client.password</name>
      <value>hadoop</value>
  </property>
  ```
* 在hadoop配置文件core-site.xml添加配置，并分发到hadoop所有机器
  ```
  <property>
      <name>hadoop.proxyuser.hadoop.hosts</name>
      <value>*</value>
  </property>
  <property>
      <name>hadoop.proxyuser.hadoop.groups</name>
      <value>*</value>
  </property>
  ``` 
  注意hadoop.proxyuser.$superuser.hosts中的superuser就是指hadoop的NameNode用户，另一个同理。比如我的用户名是hadoop，因此此处就是hadoop
  配置完hadoop记得重启服务使配置生效
* 在配置指定的服务器启动hiveserver2，后台运行
  ```# nohup hive --service hiveserver2 &```
  
  启动后查看jps，有RunJar服务就说明已启动
* 使用beeline连接hiveserver2
  ```
  # beeline
  # beeline> !connect jdbc:hive2://配置指定的hiveserver2服务器IP:10000
  ```
  输入指定账号密码即可登录，即可使用hivesql查询
  ```!quit```命令退出beeline