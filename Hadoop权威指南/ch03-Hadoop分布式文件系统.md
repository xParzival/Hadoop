##### 1. HDFS(Hadoop Distributed Filesystem)
* 数据块(block)：HDFS默认块大小为128MB，文件被划分为多个分块(chunk)作为独立的存储单元
  1. 文件的大小可以大于网络中任意一个磁盘的容量
  2. 使用抽象块而不是整个文件作为存储单元可以简化存储子系统的设计
  3. 块适合用于数据备份进而提供数据容错能力和提高可用性
* namenode和datanode
  * HDFS集群上有两类节点，以管理节点-工作节点模式运行：一个namenode(管理节点)和多个工作节点(datanode)
  * namenode：管理文件系统的命名空间。
    1. 维护文件系统树以及整棵树内所有的文件和目录，这些信息以文件的形式保存在本地磁盘：命名空间镜像文件和编辑日志文件
    2. namenode也记录每个块所在的数据节点信息，但并不永久保存块的位置信息，这些信息会在系统启动时根据数据节点信息重建 
    3. 没有namenode文件系统将无法使用
  * datanode：文件系统的工作节点。根据需要存储并检索数据块(受客户端或namenode调度)，并且定期向namenode发送它们所存储的块的列表
* namenode的两种容错机制
  1. 备份组成文件系统元数据持久状态的文件。可以通过配置使namenode在多个文件系统上保存元数据持久状态，一般是远程挂载的网络文件系统NFS
  2. 运行一个辅助namenode，定期与合并编辑日志与命名空间镜像。辅助namenode一般在另一台单独物理计算机上运行，保存合并后的命名空间副本，在namenode故障后启用
* 块缓存：通常datanode从磁盘中读取块，但对于访问频繁的文件，其对应的块可能显式缓存在datanode内存中，以堆外块缓存(off-heap block cache)形式存在
* 联邦HDFS：允许系统通过添加namenode实现扩展，每个namenode管理文件系统命名空间中的一部分
  * namenode在内存中保存文件系统中每个文件和每个数据块的引用关系，内存限制了系统的横向扩展
  * 联邦环境下每个namenode维护一个命名空间卷(namespace volume)，由命名空间的元数据和一个数据块池(block pool)组成，数据块池包含该命名空间下文件的所有数据块
  * 数据块池不再进行切分，因此集群中的datanode需要注册到每个namenode并且存储着来自多个数据块池中的数据块
  * HDFS高可用性
    * namenode存在单点失效(SPOF,single point of failure)
    * 备用namenode

##### 2. 数据流
* 客户端从HDFS读取文件数据
  1. 客户端请求HDFS读取文件
  2. 对于文件的每一个数据块，HDFS调用namenode返回存有该block副本的datanode地址信息，这些datanode根据它们与客户端的距离排序(距离计算根据集群的网络拓扑)。如果该客户端本身就是datanode，那么会从保存有相应block副本的本地datanode读取数据
  3. 客户端接收到文件block地址信息后连接与客户端距离最近的datanode将block传输到客户端，读取完毕后关闭连接
  4. 如果最近的datanode读取错误，会从次邻近的datanode读取block并记录故障信息，并保证后续不会反复读取该节点上的block，以此类推
  5. 读取完后系统会校验datanode传输的数据是否完整，若块损坏，同样会从次邻近的datanode读取block，同时将损坏block通知给namenode，以此类推
  6. 传输完成该文件的一个block之后寻找下一个block的最佳datanode并传输数据，如此循环直到所有文件块读取完成
  ![客户端读取HDFS数据](https://oscimg.oschina.net/oscnet/65d899d751b4a67047618035498da524a98.jpg)
* 客户端向HDFS写入文件数据
  1. 客户端请求HDFS新建文件
  2. HDFS调用namenode在命名空间中新建一个文件，此时该文件还没有相应的block。namenode执行各种检查确保这个文件不存在以及客户端有权限新建
  3. 客户端收到可以上传的响应后，会把待上传的文件切块（hadoop2.x默认块大小为128M）；然后再次给namenode发送请求，上传第一个block块
  4. 写入数据时数据被分为一个个数据包(packet,60KB)写入内部队列称为“数据队列”(data queue)，流处理对象挑出适合存储数据副本的一组datanode，据此要求namenode分配新数据块。
  5. 以3个数据副本为例，这一组3个datanode构成一个管线(Pipeline)，流处理对象将数据包流式传输到管线中第1个datanode，该datanode存储数据并发送到第2个datanode，以此类推到第3个
  6. 流处理对象有一个“确认队列”(ack queue)来接收datanode的确认信息。管道中所有datanode确认后数据包才会从确认队列删除
  7. 如果某个datanode发生故障，首先关闭管线，把确认队列中所有数据包添加到数据队列最前端以确保故障节点下游节点不会漏掉数据包。同时为存储在正常datanode的当前数据块指定新标识并传送给namenode以便故障节点恢复后删除部分数据块。从管线删除故障datanode，基于两个正常datanode构建一条新管线。余下的数据块写入管线中正常的datanode。当namenode发现块副本量不足时会在另一个节点创建新副本
  8. 客户端完成一个数据块写入后通知namenode并等待所有数据块依次写入
  ![客户端向HDFS写入数据](https://oscimg.oschina.net/oscnet/7ca6109abaa96492261659b014d209f24a6.jpg)
* 数据复本存放方法
  * 默认存放方法： 
    * 在运行客户端的节点上放第1个复本，若客户端在集群之外就会随机选取一个节点，  但是会避开存储太满或运行太忙的节点
    * 第2个复本放在与第1个不同且随机的另一个机架节点上
    * 第3个复本与第2个放在同一机架上，随机选择一个节点
  * 优势：提供很好的稳定性，并实现很好的负载均衡，包括写入带宽、读取性能和集群中块的均匀分布
* 通过distcp并行复制
  * distcp命令可以并行从Hadoop文件系统中复制大量数据
  * distcp的一种替代方法是hadoop fs -cp
  * distcp是作为一个MapReduce作业来实现的，复制是通过集群中并行运行的map来完成的，没有reducer。每个文件块通过map来复制，并且文件被划分为大致相等的块
  * distcp也可以用于在两个HDFS系统之间进行复制
  ```hadoop distcp file1 file2```