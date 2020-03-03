##### 1. HDFS
* 永久性数据结构
  * namenode目录结构
    ```
    ${dfs.namenode.name.dir}
    |——current
    |    |——VERSION # 正在运行的HDFS版本信息(不是指Hadoop版本)
    |    |——edits_0000000000000000001-0000000000000000019
    |    |——edits_inprogress_0000000000000000020 # edits前缀的为编辑日志
    |    |——fsimage_0000000000000000000
    |    |——fsimage_0000000000000000000.md5
    |    |——fsimage_0000000000000000019
    |    |——fsimage_0000000000000000019.md5 # fsimage前缀的为映像文件
    |    |——seen_txid
    |——in_use.lock # 锁文件，为存储目录加锁
    ```
  * datanode目录结构
    ```
    ${dfs.datanode.data.dir}
    |-current
    |  |—BP-526805057-127.0.0.1-1411980876842
    |  |  |—current
    |  |  |—VERSION
    |  |  |—finalized
    |  |  |  |——blk_1073741825
    |  |  |  |——blk_1073741825_1001.meta
    |  |  |  |——blk_1073741826 # blk前缀的为HDFS数据块
    |  |  |  |——blk_1073741826_1002.meta # .mata后缀的与数据块相关联的元数据
    |  |  |—rbw
    |  |—VERSION
    |—in_use.lock
    ```

* 安全模式：namenode在启动时，首先将映像文件(fsimage)载入内存，并执行编辑日志(edits)中的各项编辑操作。在内存中成功建立系统元数据映像后则创建一个新的fsimage文件和一个空的编辑日志。在这个过程中namenode运行在安全模式，即文件系统对于客户端是只读的
  * 安全模式有关的配置属性

    属性名称|配置文件|类型|默认值|说明
    -|-|-|-|-
    dfs.namenode.replication.min|hdfs-site.xml|int|1|成功执行写操作所需要创建的最小复本数目(也称为最小复本级别)
    dfs.namenode.safemode.threshold-pct|hdfs-site.xml|float|0.999|在namemode退出安全模式前系统中满足最小复本级别的块的比例
    dfs.namenode.safemode.extension|hdfs-site.xml|int|30000|满足最小复本级别后namenode还要处于安全模式的时间(以毫秒为单位)
  
  * 安全模式选项
    ```
    hdfs dfsadmin -safemode get # 查看namenode是否处于安全模式
    hdfs dfsadmin -safemode wait # 在安全模式中暂停以读取或写入文件
    hdfs dfsadmin -safemode enter # 进入安全模式
    hdfs dfsadmin -safemode leave # 退出安全模式

* HDFS工具
  1. dfsadmin工具(需要超级用户权限)
    
     命令|说明
     -|-
     -help|显示指定命令的帮助，若没指定则显示所有命令的帮助
     -report|显示文件系统的统计信息以及所连接的各个datanode的信息
     -metasave|将某些信息储存到Hadoop日志目录的一个文件中
     -safemode|改变或查询安全模式
     -saveNamespace|将内存中的文件系统映像保存为fsimage文件，重置edits文件，仅在安全模式执行
     -fetchImage|从namenode获取最新的fsimage文件并保存为本地文件
     -refreshNodes|更新允许连接到namenode的datanode列表
     -upgradeProgress|获取有关HDFS升级的进度条信息或强制升级
     -finalizeUpgrade|移除datanode和namenode上存储的旧版本数据。一般在升级完成，新集群运行正常的情况下使用
     -setQuota|设置目录的文件或目录的数目配额
     -clrQuota|清理指定目录的配额
     -setSpaceQuota|设置目录空间的容量配额，以限制存储在目录树中的所有文件总规模
     -clrSpaceQuota|清理指定的空间配额
     -refreshServiceAcl|刷新namenode的服务级授权策略文件
     -allowSnapshot|允许为指定的目录创建快照
     -disallowSnapshot|禁止为指定的目录创建快照
  
  2. 文件系统检查工具fsck
     * fsck用来检查HDFS文件的健康状况，从指定路径开始循环遍历文件系统的命名空间，并检查它所找到的所有文件。fsck只从namenode获取信息，并不与任何datanode交互
     * fsck结果部分说明
       1. 过多复制的块(Over-replicated blocks)：复本数超过最小复本级别的块
       2. 仍需复制的块(Under-replicated blocks)：复本数低于最小复本级别的块
       3. 错误复制的块(Miss-replicated blocks)：违反块复本放置策略的块
       4. 损坏的块(Corrupt blocks)：所有复本均已损坏的块
       5. 缺失的副本(Missing replications)：在集群中没有任何副本的块
     * 损坏的块和缺失的块是最需要考虑的，默认情况下fsck不会对这些块做任何操作，可以使用某些选项指定操作
       1. fsck -move：将受影响的文件移动到HDFS的/lost+found目录
       2. fsck -delete：删除受影响的文件
     * 查找一个文件的数据块：
       1. fsck -files：查找指定文件的信息
       2. fsck -blocks：查找指定文件各个块的信息
       3. fsck -racks：指定文件各个块的机架位置和datanode地址
  3. datanode块扫描器
     * 各个datanode运行一个块扫描器，定期检测本节点上的所有块，在客户端读到坏块之前及时的检测和修复。默认隔三周检测块，由hdfs-site.xml文件的dfs.datanode.scan.period.hours属性设置，默认504小时
     * 用户可访问datanode的web端(http://datanode:50075/blockScannerReport)查看检测报告
  4. 均衡器
     * 集群使用一段时间后datanode会越来越不均衡，会降低MapReduce的本地性，导致性能下降
     * 均衡器是一个Hadoop守护进程，可从忙碌的datanode上将块移动到相对空闲的datanode，保证集群均衡，即每个datanode的使用率（该节点上已用空间与总空间容量之比）和集群的使用率非常接近。
     * 任何时刻集群中都只运行一个均衡器
     ```start-balancer.sh # 启动均衡器```
     * -threshold参数指定阈值以判定集群是否均衡，默认为10%
     * 均衡器在标准日志文件目录中创建一个日志文件，记录重新分配的过程
     * 均衡器在后台运行，在不同节点之间复制数据的带宽也受限。hdfs-site.xml文件的dfs.datanode.balance.bandwidthPerSec设置带宽，默认为1MB/s

##### 2. 维护
* 委任和解除节点：通常情况下节点同时运行datanode和node manager，因此两者一般同时被委任或者解除
  1. 委任新节点
     * 最简单但不安全的方式：配置hdfs-site.xml指向namenode，配置yarn-site.xml指向resource manager，然后启动datanode和node manager守护进程
     * 安全方式：
     1. 允许连接到namanode的所有datanode放在一个文件中，由hdfs-site.xml文件的dfs.hosts属性指定，该文件放在namenode本地文件系统中，每行对应一个datanode的网络地址。允许连接到resource manager的所有node manager放在一个文件中，由yarn-site.xml文件的yarn.resourcemanager.nodes.include-path属性指定。一般让两者指向同一个文件命名为include
     2. 运行以下命令，将审核过的一系列datanode集合更新至namenode
        ```hdfs dfsadmin -refreshNodes```
     3. 运行以下命令，将审核过的一系列node manager集合更新至resource manager
        ```hdfs rmamin -refreshNodes```
     4. 把新节点更新到slaves文件，使Hadoop控制脚本把新节点包括在未来操作之中
     5. 单节点启动新的datanode和node manager
     6. 检查新的datanode和node manager是否出现在web界面中
     7. 根据需求决定是否运行均衡器
     * dfs.hosts属性和yarn.resourcemanager.nodes.include-path属性指定的文件与slaves文件区别：前者供namenode和resource manager使用，用于决定可以连接哪些工作节点。后者供Hadoop控制脚本使用执行面向整个集群范围的操作，例如重启集群。Hadoop守护进程不使用slaves文件
  2. 解除旧节点
     * 不允许直接停止datanode，应该将需要退出的datanode通知namenode，保证在datanode停机之前将块复制到其他datanode
     * 解除步骤
       1. 不允许连接到namanode的所有datanode放在一个文件中，由hdfs-site.xml文件的dfs.hosts.exclude属性指定，该文件放在namenode本地文件系统中，每行对应一个datanode的网络地址。不允许连接到resource manager的所有node manager放在一个文件中，由yarn-site.xml文件的yarn.resourcemanager.nodes.exclude-path属性指定。一般让两者指向同一个文件命名为exclude
       2. 不更新include文件
       3. 运行以下命令，更新namenode和resource manager信息
          ```
          hdfs dfsadmin -refreshNodes
          hdfs rmamin -refreshNodes
          ```
       4. 在网页界面查看待解除datanode是否已经“正在解除“(Decommission In Progress)。此时相关datanode会把块复制到其他datanode
       5. 所有datanode变为“解除完毕”(Decommissioned)，所有块已复制完毕。关闭已经解除的节点
       6. 在include文件中移除这些节点，运行如下命令
          ```
          hdfs dfsadmin -refreshNodes
          hdfs rmamin -refreshNodes
          ```
       7. slaves文件中移除节点
