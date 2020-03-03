##### Hadoop相较关系型数据库优缺点比较
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

###### Hadoop优势
  1. 高可靠性：Hadoop底层维护多个数据副本，所以即使某个计算元素或存储出现故障，也不会导致丢失
  2. 高扩展性：在集群之间分配任务数据，可方便的扩展数以千计的节点
  3. 高效性：在MapReduce的思想下，在Hadoop并行工作，加快处理速度
  4. 高容错性：能够将失败任务重新分配
##### Hadoop1.x和Hadoop2.x区别
  * Hadoop1.x组成：MapReduce(计算+资源调度)、HDFS(数据存储)+common(辅助工具)
  * Hadoop2.x组成：MapReduce(计算)、Yarn(资源调度)、HDFS(数据存储)+common(辅助工具)

##### HDFS中的block、packet、chunk
  1. block：文件上传前需要分块，这个块就是block，一般为128MB，可以修改，但是不推荐。因为块太小：寻址时间占比过高。块太大：Map任务数太少，作业执行速度变慢。它是最大的一个单位
  2. packet：packet是第二大的单位，它是client端向DataNode，或DataNode的PipLine之间传数据的基本单位，默认64KB
  3. chunk：chunk是最小的单位，它是client向DataNode，或DataNode的PipLine之间进行数据校验的基本单位，默认512Byte，因为用作校验，故每个chunk需要带有4Byte的校验位。所以实际每个chunk写入packet的大小为516Byte。由此可见真实数据与校验值数据的比值约为128 : 1。（即64*1024 / 512）

##### Hadoop在linux系统中安装与配置
###### 1. java安装：
  * 检查系统中是否已有自带java，若有需要先卸载
    ```
    java -version # 查看java版本，若有信息则需要卸载自带java
    rpm -qa | grep java | xargs sudo rpm -e [--nodeps] # 卸载当前java，有依赖时--nodeps参数
    ```
  * 安装java包
    ```
    tar -zxvf /opt/software/jdk压缩包.tar.gz -C /opt/module # 把下载的jdk压缩文压到指定文件夹
    ```
  * 配置java环境变量
    ```
    vim /etc/profile # 打开环境变量配置文件
    export JAVA_HOME=/opt/module/解压后jdk文件夹
    export PATH=$PATH:$JAVA_HOME/bin  # 在配置文件写入上述命令，配置java环境变量
    source /etc/profile # 配置完后重新执行刚修改的初始化文件，使之立即生效
    ```
###### 2. Hadoop安装
  * 安装Hadoop包
    ```
    tar -zxvf /opt/software/Hadoop压缩包.tar.gz -C /opt/module # 把下载的Hadoop文件解压到指定文件夹
    ```
  * 配置Hadoop环境变量
    ```
    vim /etc/bashrc # 打开环境变量配置文件
    export HADOOP_HOME=/opt/module/解压后Hadoop文件夹
    export PATH=$PATH:$HADOOP_HOME/bin:$HADOOP_HOME/sbin  # 在配置文件写入上述命配置Hadoop环境变量
    source /etc/profile # 配置完后重新执行刚修改的初始化文件，使之立即生效
    ```

##### Hadoop三种运行模式
  1. 本地模式(Standalone Operation)：无需任何守护进程，所有程序都在同一个JVM上执行。该模式下测试和调试MapReduce程序很方便，开发阶段比较合适。
  2. 伪分布模式(Pseudo-Distributed Operation)：Hadoop守护进程运行在本地机器上，模拟一个小规模集群
  3. 全分布模式(Fully-Distributed Operation)：Hadoop守护进程运行在一个集群上，此模式为生产模式
  * 在特定模式下运行Hadoop需要关注两个因素：正确设置启动属性和启动Hadoop守护进程。本地模式下使用本地文件系统和本地MapReduce作业运行器;分布模式下启动HDFS和YARN守护进程，此外还需配置MapReduce以便能够使用YARN