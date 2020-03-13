##### 1. 运行Hive
* 配置Hive：Hive用XML文件配置，conf/hive-site.xml
  * 可以通过--config选项指定配置文件目录。是指包含配置文件的目录而不是hive-site.xml本身.也可以通过环境变量HIVE_CONF_DIR指定配置目录
    ```hive --config 指定目录```
  * fs.defaultFS属性指定文件系统，yarn.resourcemanager.address指定资源管理器，即对应的Hadoop的配置
  * Hive支持用-hiveconf选项为单个会话设置属性
    ```
    hive -hiveconf fs.defaultFS=hdfs://localhost \
         -hiveconf yarn.resourcemanager.address=locahost:8032
    ```
  * 可以在一个会话中使用SET更改设置，这有利于对特定查询修改Hive设置。只带属性名的SET命令可查看指定属性的值。不带参数SET命令列出Hive所有设置属性值但不包括Hadoop默认值。SET -v 命令可包含Hadoop默认值
    ```hive> SET hive.enforce.bucketing=true;```
  * 设置属性优先级层次
    1. Hive SET命令
    2. 命令行-hiveconf选项
    3. hive-site.xml和Hadoop站点文件
    4. Hive默认值和Hadoop默认文件
* 执行引擎：Hive默认以MapReduce作为执行引擎，可由属性hive.execution.engine来控制，默认为mr，目前可选的还有Tez，对Spark的支持正在开发中
* Hive服务：Hive的shell环境只是hive命令提供的一种。可以在运行时用--service选项指定服务，--service help获取可用服务的列表。常用服务如下：
  1. cli：Hive命令行接口(shell环境)，即默认服务
  2. hiveserver2：让Hive以提供Thrift服务的服务器形式运行，允许用不同的语言编写的客户端进行访问。可使用Thrift、JDBC、ODBC连接器和Hive进行通信。hive.server2.thrift.port指定监听的端口号
  3. beeline：以嵌入方式工作的Hive命令行接口，或者使用JDBC连接Hiveserver2进程
  4. hwi：Hive的web接口。在没有安装任何客户端软件的情况下可以使用。Hue是一个功能更全面的Hadoop web接口，也可以运行Hive查询和浏览hive metastore的应用
  5. jar：与Hadoop jar等价
  6. metastore：默认情况下，metastore和Hive服务运行在同一个进程。使用这个服务可以使metastore作为一个单独的进程运行。通过设置METASTORE_PORT环境变量或者使用-p命令行选项可以指定服务器监听的端口号，默认为9083
* Hive客户端：如果以服务器方式运行Hive，即上述第2种方式，可以在应用程序中以不同机制连接到服务器
  * Thrift客户端：Hive服务器提供Thrift服务，任何支持Thrift的编程语言都可以与之交互
  * JDBC驱动：Hive提供了Type4(纯Java)的JDBC驱动，定义在org.apache.hadoop.jdbc.HiveDriver类中。在以jdbc:hive2://host:port/dbname形式配置JDBC URI后，Java应用程序可以在指定主机和端口连接到在另外一个进程中运行的Hive服务器
  * ODBC驱动：Hive的ODBC驱动允许支持ODBC的协议的应用程序连接到Hive
  ![Hive体系结构](http://static.zybuluo.com/BrandonLin/vu7slcsvxp2dkiyyt422hx2k/image_1aorqv69b184sbetjp6sni19e11g.png)
* metastore：metastore是Hive元数据的集中存放地，包括两部分：服务和后台数据的存储
  ![metastore三种配置](http://attachbak.dataguru.cn/attachments/forum/201212/01/2312139q13aqllkvzfffk2.png)
  * 内嵌metastore(embedded metastore)配置：默认情况下metastore服务和Hive服务运行在同一个JVM中，称为内嵌metastore配置。每次只有一个内嵌Derby数据库可以访问某个磁盘上的数据库文件，即一次只能为每个metastore打开一个Hive会话
  * 本地metastore(local metastore)配置：支持多会话需要一个独立的数据库，这种配置称为本地metastore配置。对于独立的metastore，一般使用MySQL
  * 远程metastore配置(remote metastore)：一个或多个metastore服务器和Hive服务运行在不同进程内
  * metastore配置属性

    属性名称|类型|默认值|说明
    -|-|-|-
    hive.metastore.warehouse.dir|URI|/user/hive/warehouse|相对于fs.default.name的目录，托管表就存储在这里
    hive.metastore.uris|逗号分隔的URI||默认使用当前metastore，否则连接到指定的远程metasore服务器
    javax.jdo.option.ConnectionURL|URI|jdbc:derby:;databaseName=metastoredb;create=true|metastore数据库的JDBC URL
    javax.jdo.option.ConnectionDriverName|string|org.apache.derby.jdbc.EmbeddedDriver|JDBC驱动器类名
    javax.jdo.option.ConnectionUserName|string|APP|JDBC用户名
    javax.jdo.option.ConnectionPassword|string|mine|JDBC密码
    
##### 2. Hive与传统数据库对比
* 读时模式和写时模式
  * 传统数据库表的模式是在数据加载时强制确定的。数据库在写入数据时对照模式进行检查，这一设计被称为“写时模式”(schema on write)；Hive对数据的验证并不在加载数据时进行，而是在查询时进行，称为“读时模式”(schema on read)
  * 这两种模式需要权衡。读时模式可以使数据加载非常迅速，它不需要读取数据进行解析(parse)再进行序列化并以数据库内部格式写入磁盘，同一数据对不同的分析任务也可能有不同模式；写时模式有利于提升查询性能，但是加载数据会花更长时间
* 更新、事务和索引
  * HDFS不提供就地更新文件，插入、更新和删除引起的一切变化都被保存在一个较小的增量文件中，metastore在后台运行的MapReduce作业会定期讲这些增量文件合并到基表文件中。上述操作必须有事务才能发挥作用
  * hive索引分为紧凑索引和位图索引
* 其他SQL-on-Hadoop技术
  * Cloudera Impala：性能比Hive强
  * Facebook Presto、Apache Drill、Spark SQL