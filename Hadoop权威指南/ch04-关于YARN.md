##### 1. YARN(Yet Another Resource Negotiator)
* Apache YARN是Hadoop的集群资源管理系统，从Hadoop 2开始引入
* YARN通过两类守护进程提供自己的核心服务
  1. 管理集群上资源使用的资源管理器(resource manager)
  2. 运行在集群节点中所有节点上且能够启动和监控容器(container)的节点管理器(node manager)。容器用于执行特定应用程序的进程，每个容器都有资源限制(内存、CPU等)
* YARN运行机制
  1. 客户端联系resource manager要求运行一个application master进程
  2. resource manager找到一个能够在container中启动application master的node manager
  3. application master一旦运行起来后能够做些什么依赖于应用本身。可以是在所处container中简单的运行一个计算返回客户端结果；或是向resource manager请求更多的container以进行分布式计算
* YARN和MapReduce1对比
  1. 可扩展性：与MapReduce1相比YARN可以在更大规模的集群上运行。MapReduce1的jobtracker必须同时管理作业和任务，而YARN将resource manager和application master分离
  2. 可用性：当服务守护进程失败时，通过为另一个守护进程复制接管工作所需的状态以便继续提供服务，YARN可以做到但是MapReduce1不行
  3. 利用率：YARN中资源管理更精细，一个应用可按需请求资源而不是像MapReduce1请求固定的slot，资源利用率更高
  4. 多租户：YARN向MapReduce以外的其他类型的分布式应用开放了Hadoop，MapReduce仅仅是许多Hadoop应用中的一个

##### 2. YARN中的调度
* 三种调度器
  1. FIFO调度器(FIFO Schedule)  
    将应用放置在一个队列中，然后按照提交的顺序(先进先出)运行应用。优点是简单易用不需配置，缺点是不适合共享集群
  2. 容量调度器(Capacity Schedule)  
    一个专门队列保证小作业一提交就可以启动。这种策略以整个集群的利用率为代价，大作业运行时间较长
  3. 公平调度器(Fair Schedule)  
    不需要预留一定量的资源，调度器会在所有作业之间动态平衡资源。既得到较高集群利用率，又保证小作业能及时完成
