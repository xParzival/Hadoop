##### 1. MapReduce编程模型
* map
  * Hadoop将作业分成若干个任务(task)来执行，其中包括两类任务：map任务和reduce任务。这些任务运行在集群的节点上，并通过YARN进行调度。如果一个任务失败将在另一个不同的节点上自动重新调度运行
  * Hadoop将MapReduce的输入数据划分成等长的小数据块，称为输入分片(input split)，然后分别为每个分片构建一个map任务
  * Hadoop在存储有输入数据的节点上运行map任务即可获得最佳性能，因为无需使用宽带资源，称为“数据本地化优化”
  * map任务将其输出写入本地硬盘而不是HDFS，该输出交由reduce任务输出最终结果
  * 若运行map任务的节点在将输出传送给reduce任务前失败，Hadoop将在另一个节点重新运行这个map任务
* reduce
  * reduce任务不具备数据本地化，单个reduce任务的输入通常是所有map任务的输出
  * map输出通过网络传输到reduce任务节点，reduce任务最终输出到HDFS上
  * reduce任务的数量不是由输入数据大小指定，而是独立制定的。如果有多个reduce任务，每个map会针对输出进行分区(partition)，每个分区有许多键值对。默认通过hash函数来控制分区
  * map任务和reduce任务之间的数据流称为shuffle(混洗)
  ![Hadoop数据流](https://img-blog.csdn.net/20141201195403671?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvc3VpZmVuZzMwNTE=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)
* combiner函数
  * Hadoop允许针对map任务的输出指定一个combiner，combiner的输出作为reduce的输入