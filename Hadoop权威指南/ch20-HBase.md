##### 1. HBase基础
* HBase是一个在HDFS上开发的面向列的分布式数据库
* HBase自底向上进行构建，能够简单的通过增加节点来达到线性扩展。HBase是非关系型数据库，不支持SQL
* HBase和传统RDBMS区别

  项目|HBase|RDBMS
  -|-|-
  硬件架构|类似于Hadoop的分布式集群，硬件成本低|传统的多核系统，硬件成本高
  容错性|由软件架构实现，多个节点组成|一般需要额外硬件设备实现HA机制
  数据库大小|PB|GB,TB
  数据排布方式|稀疏的、分布的多维map|以行和列形式组织
  数据类型|Bytes|丰富的数据类型
  事务支持|ACID只支持单个Row级别|全面ACID支持，对Row和表
  查询语言|只支持Java API，除非与其他框架一起使用，比如Hive、Phoenix|SQL
  索引|只支持Row-key，除非与其他技术一起使用|支持
  吞吐量| 百万查询/秒|数千查询/秒

##### 2. 概念
1. 数据模型
   * 数据存放在带标签的表中，表的单元格(cell)是有版本的，默认为HBase插入单元格的时间戳。单元格内容是未解释的字节数组
   * 表中行的键也是字节数组。理论上任何东西都可以通过表示成字符串或将二进制形式转化为长整型或直接对数据结构进行序列化来作为键值。表中的行根据行的主键进行排序，对所有表的访问都要通过表的主键
   * 行中的列被分成“列族”(column family)，同一个列族的成员具有相同的前缀。列族的前缀必须由可打印的字符组成，修饰性的结尾字符即列族修饰符(Column Qualifier)可以为任意字节，列族和修饰符之间始终以冒号分隔。一个列族对应一个HFile
   * 一个表的列族必须作为表模式定义的一部分预先给出，但是新的列族成员可以随时按需要加入
   * 物理上所有列族成员都一起存放在文件系统中，所以HBase是一个面向列族的存储器。调优和存储都是在列族这个层次上进行的，所以最好使所有列都有相同的访问模式和大小特征

     Row-key|CF:Qualifier|Timestamp|Value
     -|-|-|-
     1|info:fn|123456789|三
     1|info:ln|123456789|张
     2|info:fn|123456789|四
     2|info:ln|123456789|李

   * Region：HBase自动把表水平划分成区域(region)。每个region由表中行的子集构成，每个区域由它所属的表、它所包含的第一行及最后一行来表示。表很大在超出设定阈值大小时会分成多个region，分散到集群上，Region 是 HBase 可用性和分布式的基本单位。如果当一个表格很大，并由多个 CF 组成时，那么表的数据将存放在多个 Region 之间，并且在每个 Region 中会关联多个存储的单元(Store)

2. 实现原理
   * HBase采用用一个Master节点协调管理一个或多个regionserver从属机器的模式。HBase依赖于ZooKeeper来管理才能运行
   ![HBase组成模块](https://www.ibm.com/developerworks/cn/analytics/library/ba-cn-bigdata-hbase/image002.png)
   * regionserver从属节点列表列在HBase的conf/regionservers文件中。启动和结束使用类似Hadoop的脚本，基于SSH来传递命令。集群配置文件有conf/hbase-site.xml和hbase-env.sh文件，也类似Hadoop配置
   ![HBase原理](https://www.ibm.com/developerworks/cn/analytics/library/ba-cn-bigdata-hbase/image003.png)
   * HBase内部保留名为hbase:meta的特殊目录表(catalog table)，维护当前集群上所有区域的列表、状态和位置。hbase:meta表中的项使用区域名作为键，区域名由所属的表名、区域的起始行、区域的创建时间以及对其整体进行的MD5哈希值组成
   * 具体运行原理：HBase 的集群是通过 Zookeeper 来进行机器之前的协调，也就是说 HBase Master 与 Region Server 之间的关系是依赖 Zookeeper 来维护。当一个 Client 需要访问 HBase 集群时，Client 需要先和 Zookeeper 来通信，然后才会找到对应的 Region Server。每一个 Region Server 管理着很多个 Region。对于 HBase 来说，Region 是 HBase 并行化的基本单元。因此，数据也都存储在 Region 中。这里我们需要特别注意，每一个 Region 都只存储一个 Column Family 的数据，并且是该 CF 中的一段（按 Row 的区间分成多个 Region）。Region 所能存储的数据大小是有上限的，当达到该上限时（Threshold），Region 会进行分裂，数据也会分裂到多个 Region 中，这样便可以提高数据的并行化，以及提高数据的容量。每个 Region 包含着多个 Store 对象。每个 Store 包含一个 MemStore，和一个或多个 HFile。MemStore 便是数据在内存中的实体，并且一般都是有序的。当数据向 Region 写入的时候，会先写入 MemStore。当 MemStore 中的数据需要向底层文件系统倾倒（Dump）时（例如 MemStore 中的数据体积到达 MemStore 配置的最大值），Store 便会创建 StoreFile，而 StoreFile 就是对 HFile 一层封装。所以 MemStore 中的数据会最终写入到 HFile 中，也就是磁盘 IO。由于 HBase 底层依靠 HDFS，因此 HFile 都存储在 HDFS 之中
   * HFile：HFile 由很多个数据块（Block）组成，并且有一个固定的结尾块。其中的数据块是由一个 Header 和多个 Key-Value 的键值对组成。在结尾的数据块中包含了数据相关的索引信息，系统也是通过结尾的索引信息找到 HFile 中的数据。HFile 中的数据块大小默认为 64KB。如果访问 HBase 数据库的场景多为有序的访问，那么建议将该值设置的大一些。如果场景多为随机访问，那么建议将该值设置的小一些。一般情况下，通过调整该值可以提高 HBase 的性能
   ![HFile结构](https://www.ibm.com/developerworks/cn/analytics/library/ba-cn-bigdata-hbase/image004.png)

##### 3. 安装与配置
* 安装：解压安装压缩包到指定位置即可
* 配置
  1. conf/hbase-env.sh环境变量配置
     ```
     export JAVA_HOME=Java路径 # 配置Java路径
     export HBASE_MANAGES_ZK=false # 是否使用HBase自带的ZooKeeper集群
     ```
     * 特别注意：JDK8+需要注释掉以下几行
       ```
       # Configure PermSize. Only needed in JDK7. You can safely remove it for JDK8+
       # export HBASE_MASTER_OPTS="$HBASE_MASTER_OPTS-XX:PermSize=128m-XX:MaxPermSize=128m-XX:ReservedCodeCacheSize=256m"
       # export HBASE_REGIONSERVER_OPTS="$HBASE_REGIONSERVER_OPTS-XX:PermSize=128m-XX:MaxPermSize=128m-XX:ReservedCodeCacheSize=256m"
       ```
  1. conf/hbase-site.xml配置文件简单配置，更多配置见官方文档
     ```
     <configuration>
     <!-- RegionServer 的共享目录，用来持久化 HBase，一般HDFS地址是要跟 Hadoop的core-site.xml配置文件fs.defaultFS属性的HDFS地址或者域名、端口一致 -->
     <property>
     　　<name>hbase.rootdir</name>
     　　<value>hdfs://namenodeIP:9000/hbase</value>
     </property>
     <!-- HBase的运行模式。false表示单机模式，true表示分布式模式 -->
     <property>
     　　<name>hbase.cluster.distributed</name>
     　　<value>true</value>
     </property>
     <!-- Zookeeper集群的地址列表，用逗号分割 -->
     <property>
     　　<name>hbase.zookeeper.quorum</name>
     　　<value>服务器1,服务器2,服务器3</value>
     </property>
     <!-- HBase WEB UI界面端口，默认16010 -->
     <property>
         <name>hbase.master.info.port</name>
         <value>16010</value>
     </property>
     </configuration>
     ```
  2. conf/regionservers配置文件，包含在HBase集群中运行RegionSever的主机名或IP列表，每行一个
* 在web页面 IP:16010可以查看HBase信息