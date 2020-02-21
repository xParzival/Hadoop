* Hadoop相较关系型数据库优缺点比较
  1. 数据访问模式包含大量硬盘寻址Hadoop更好，数据库系统更新小部分数据关系型数据库更好
  2. MapReduce适合需要以批处理方式分析整个数据集，RDBMS适合索引后数据集的点查询和更新
  3. MapReduce适合一次写入、多次读取的数据，RDBMS适合持续更新的数据集
  4. Hadoop对非结构化或半结构化数据更有效，因为它是在处理数据时才对数据进行解释（“读时模式”），RDBMS数据是规范的
  5. MapReduce以及Hadoop中其他的处理模型是可以随着数据规模线性伸缩的，RDBMS一般不具备这种特性
  6. RDBMS已经开始吸收Hadoop的一些思想，而诸如Hive的Hadoop系统更具交互性（从MapReduce脱离出来），而且增加了索引和事务这样的特性，看上去更像关系型数据库
  
     项目|传统的关系型数据库|MapReduce
     -|-|-
     数据大小|GB|PB
     数据存取|交互式和批处理|批处理
     更新|多次读/写|一次写入，多次读取
     事务|ACID|无
     结构|写时模式|读时模式
     完整性|高|低
     横向扩展|非线性的|线性的

* Hadoop优势
  1. 高可靠性：Hadoop底层维护多个数据副本，所以即使某个计算元素或存储出现故障，也不会导致丢失
  2. 高扩展性：在集群之间分配任务数据，可方便的扩展数以千计的节点
  3. 高效性：在MapReduce的思想下，在Hadoop并行工作，加快处理速度
  4. 高容错性：能够将失败任务重新分配
* Hadoop1.x和Hadoop2.x区别
  * Hadoop1.x组成：MapReduce(计算+资源调度)、HDFS(数据存储)+common(辅助工具)
  * Hadoop2.x组成：MapReduce(计算)、Yarn(资源调度)、HDFS(数据存储)+common(辅助工具)

* HDFS中的block、packet、chunk
  1. block：文件上传前需要分块，这个块就是block，一般为128MB，可以修改，但是不推荐。因为块太小：寻址时间占比过高。块太大：Map任务数太少，作业执行速度变慢。它是最大的一个单位
  2. packet：packet是第二大的单位，它是client端向DataNode，或DataNode的PipLine之间传数据的基本单位，默认64KB
  3. chunk：chunk是最小的单位，它是client向DataNode，或DataNode的PipLine之间进行数据校验的基本单位，默认512Byte，因为用作校验，故每个chunk需要带有4Byte的校验位。所以实际每个chunk写入packet的大小为516Byte。由此可见真实数据与校验值数据的比值约为128 : 1。（即64*1024 / 512）