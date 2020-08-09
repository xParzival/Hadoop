# hbase
## 搭建hbase
### 安装hbase
hbase前提是有zookeeper
* 安装包解压到/opt/module，根目录/opt/module/hbase
* 添加系统环境变量

### 配置hbase
* /opt/module/hbase/conf/hbase-env.sh
  ```
  export JAVA_HOME=对应目录
  export HADOOP_HOME=对应目录
  export HBASE_HOME=对应目录
  export HBASE_MANAGES_ZK=false
  ```
* /opt/module/hbase/conf/hbase-site.xml
  ```
  <!--hbase在hdfs上存储数据时的目录-->
  <property>
      <name>hbase.rootdir</name>
   	  <value>hdfs://hadoop101:9000/hbase</value>
  </property>
  <!--是否开启集群-->
  <property>
      <name>hbase.cluster.distributed</name>
      <value>true</value>
  </property>
  <property>
	  <name>hbase.tmp.dir</name>
	  <value>/opt/module/hbase/tmp</value>
  </property>
  <!--配置Zookeeper-->
  <property>
      <name>hbase.zookeeper.quorum</name>
      <value>hadoop101,hadoop102,hadoop103</value>
  </property>
  <property>
      <name>hbase.zookeeper.property.clientPort</name>
      <value>2181</value>
  </property>
  <!--Zookeeper的dataDir目录-->
  <property>
      <name>hbase.zookeeper.property.dataDir</name>
      <value>/opt/module/zookeeper/data</value>
  </property>
  <property>
      <name>zookeeper.znode.parent</name>
      <value>/hbase</value>
  </property>
  <property>
      <name>hbase.unsafe.stream.capability.enforce</name>
      <value>false</value>
  </property>
  ```
* /opt/module/hbase/conf/regionservers
  ```
  hadoop101
  hadoop102
  hadoop103
  hadoop104
  ```

### 启动hbase
```start-hbase.sh```

### 进入hbase
```hbase shell```

### hbase与hive集成
Hive与HBase整合的实现是利用两者本身对外的API接口互相通信来完成的，其具体工作交由Hive的lib目录中的hive-hbase-handler-*.jar工具类来实现
Hive整合HBase后的使用场景：
1. 通过Hive把数据加载到HBase中，数据源可以是文件也可以是Hive中的表
2. 通过整合，让HBase支持JOIN、GROUP等SQL查询语法
3. 通过整合，不仅可完成HBase的数据实时查询，也可以使用Hive查询HBase中的数据完成复杂的数据分析

#### 配置
* 配置hive，向hive-site.xml添加
  ```
  <property>
      <name>hbase.zookeeper.quorum</name>
      <value>hadoop102,hadoop103,hadoop104</value>
  </property>
  ```
* 拷贝jar文件
  * 把Hbase的lib目录下面的jar文件全部拷贝到Hive的lib目录下
  * 把Hive的lib目录下面的hive-hbase-handler-0.13.1.jar拷贝到Hbase的lib目录下面

#### 使用方法
* hive创建hbase中不存在的表
  ```
  CREATE TABLE hive表名(
      hive列名1 数据类型 comment "描述1", 
      hive列名2 数据类型 comment "描述2",
      ...)
  STORED BY 'org.apache.hadoop.hive.hbase.HBaseStorageHandler'   
  WITH SERDEPROPERTIES (
      "hbase.columns.mapping" = 
      ":key, # 作为row_key的列，不一定是第一列，符合对应位置的列为row_key
      hbase列族1:列名1,
      hbase列族1:列名2,
      hbase列族2:列名3,
      ...")   
  TBLPROPERTIES("hbase.table.name" = "hbase中的表名");
  ```
   其中数据类型包括string、int、float等，comment可省略
   这个方法创建的hbase表，当用hive删除该表时，hbase对应的表也会删除，因为是内部表
* hive关联hbase中已存在的表
  1. 先用hbase命令创建一个表
     ```
     create '表名','列族1','列族2',...
     ```
  2. 用hive创建外部表关联hbase中已存在的表
     ```
     CREATE EXTERNAL TABLE hive表名(
         hive列名1 数据类型 comment "描述1", 
         hive列名2 数据类型 comment "描述2",
         ...)   
     STORED BY 'org.apache.hadoop.hive.hbase.HBaseStorageHandler'   
     WITH SERDEPROPERTIES (
         "hbase.columns.mapping" = 
         ":key,
         hbase列族1:列名1,
         hbase列族1:列名2,
         hbase列族2:列名3,
         ...")   
     TBLPROPERTIES("hbase.table.name" = "hbase中的表名");
     ```
* 利用hive的map类型映射hbase的列族，这样做hive的表中每一列即为一个列族的字典
  ```
  CREATE [EXTERNAL] TABLE 表名(
     hive列名1 map<数据类型,数据类型> comment "描述1", 
     hive列名2 map<数据类型,数据类型> comment "描述2",
     ...)   
  STORED BY 'org.apache.hadoop.hive.hbase.HBaseStorageHandler'   
  WITH SERDEPROPERTIES (
     "hbase.columns.mapping" = 
     ":key,
     hbase列族1:,
     hbase列族2:,
     ...")   
  TBLPROPERTIES("hbase.table.name" = "hbase中的表名");
  ```
  当创建内部表时，插入数据时按照map对应关系插入
  ```
  INSERT INTO/OVERWRITE TABLE hive表名 select key列,列族1 map(hive列名1，hive列名2) from hive表;
  ```