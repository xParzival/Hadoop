##### 1. 剖析MapReduce作业运行机制
* Hadoop运行作业的整个过程
  1. 客户端提交MapReduce作业
  2. resource manager负责协调集群上计算机资源的分配
  3. node manager负责启动和监视集群中机器上的计算容器(container)
  4. MapReduce的application master负责协调运行MapReduce作业的任务。它和MapReduce任务在容器中运行，这些容器由资源管理器分配并由node manager管理
  5. HDFS与其他实体之间共享作业文件
  ![mapreduce运行机制](https://s3.51cto.com/wyfs02/M00/8A/77/wKioL1gxvd7h7PJTAAEC1lT2g9Q016.png)

##### 2.shuffle和排序
* MapReduce确保每个reducer的输入都是按键值排序的。系统执行排序。将mapper输出作为输入传给reducer的过程称为shuffle
  ![shuffle](https://img-blog.csdn.net/20150708232349407?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)
* map端
  1. map函数产生输出时用缓冲的方式写入内存，缓冲区默认为100MB，可用mapreduce.task.io.sort.mb属性调整。一旦缓冲区达到阈值(mapreduce.map.sort.spill.percent，默认0.8)，一个后台线程会将缓冲区数据溢出到磁盘，在此期间map输出继续写入缓冲区，如果期间缓冲区满，map会被阻塞直到磁盘写入完成
  2. 在写入磁盘之前，线程要根据数据最终要传的reducer数目进行分区(partition)，每个分区中数据按键进行内存中排序。如果有combiner函数，它就在排序后运行
  3. 每次内存缓冲区达到溢出阈值就会新建一个溢出文件，因此一个map任务会有多个溢出文件。任务完成以前溢出文件会合并成一个已分区且排序的输出文件。配置属性mapreduce.task.io.sort.factor控制一次能合并多少流，默认为10
  4. 可以将map输出写到磁盘的数据进行压缩以增加写入磁盘速度并减少传给reducer的数据量。mapreduce.map.output.compress控制是否压缩，默认为false
* reduce端
  1. map输出文件位于运行map任务节点的本地磁盘，每个map任务运行时间不同，reduce在有map任务完成时就开始复制输出。map任务完成时会用心跳机制通知application master，reduce也有线程定期询问application master已完成map任务的位置。mapreduce.reduce.shuffle.parallelcopies属性控制reduce并行复制的线程数，默认为5
  2. 如果复制的map输出很小会被复制到reduce任务的JVM内存，否则会复制到磁盘。一旦缓冲内存达到阈值或达到map输出阈值，则合并后溢出到磁盘。如果指定combiner则在合并期间运行以降低硬盘写入量
  3. 随着磁盘上副本增多，后台线程会将它们合并为更大的排序的文件。此前压缩的map输出必须被解压后才能合并。这个阶段是为了最后的合并节约时间
  4. 复制完所有map输出后，reduce任务进入合并阶段，合并map输出并维持其排序
  5. 合并过程是循环进行的，每一轮合并不一定合并平均数量的文件数，指导原则是使用整个合并过程中写入磁盘的数据量最小，为了达到这个目的，则需要最终一轮合并尽可能多的数据，因为最后一轮的数据直接作为reduce的输入，无需写入磁盘再读出。每一轮合并的文件数最大值即合并因子，通过mapreduce.task.io.sort.factor来配置，默认为10
  6. 对已排序的输出调用reduce函数直接写入HDFS
* 配置调优
  1. map端相关配置
     * 总的原则是给shuffle过程尽量多的内存空间。因此map函数和reduce函数要尽量少用内存
     * map端要避免多次溢出写入磁盘来提升性能

     属性名称|类型|默认值|说明
     -|-|-|-
     mapreduce.task.io.sort.mb|int|100|排序map输出时所使用的内存缓冲区大小，单位为MB
     mapreduce.map.sort.spill.percent|float|0.80|map输出内存缓冲区开始溢出写入磁盘的百分比阈值
     mapreduce.task.io.sort.factor|int|10|排序文件时一次最多合并的文件数，这个属性也适用于reduce
     mapreduce.map.combine.minspills|int|3|运行combiner所需的最少溢出文件
     mapreduce.map.output.compress|Boolean|false|是否压缩map输出
     mapreduce.map.output.compress.codec|Class name|org.apache.hadoop.io.compress.DefaultCodec|用于map输出的压缩解码器
     mapreduce.shuffle.max.threads|int|0|每个节点管理器的工作线程数，用于将map输出到reducer。默认为机器处理器数的两倍

  2. reduce端相关配置
     * reduce端中间数据全部驻留在内存时能获得最佳性能，reduce函数内存需求不大时可将数据全部存在内存而不溢出写入磁盘

    属性名称|类型|默认值|说明
    -|-|-|-
    mapreduce.reduce.shuffle.parallelcopies|int|5|用于把map输出复制到reducer的线程数
    mapreduce.reduce.shuffle.maxfetchfailures|int|10|在声明失败以前reducer获取一个map输出所花的最大时间
    mapreduce.task.io.sort.factor|int|10|排序文件时一次最多合并的文件数，这个属性也适用于mapreduce
    mapreduce.reduce.shuffle.input.buffer.percent|float|0.70|在shuffle复制阶段分配给map输出的缓冲区占堆空间百分比
    mapreduce.reduce.shuffle.input.merge.percent|float|0.66|map输出缓冲区开始溢出写入磁盘的阈值百分比
    mapreduce.reduce.merge.inmem.threshold|int|1000|启动合并输出和溢出写入磁盘的map输出阈值数
    mapreduce.reduce.input.buffer.percent|float|0.0|reduce函数开始运行时在内存中保存map输出的空间占整个堆空间的比例。reduce阶段开始时map输出大小不能大于该值
