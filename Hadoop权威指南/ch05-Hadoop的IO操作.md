##### 1. 数据完整性
* HDFS数据完整性
  * HDFS会对写入的所有数据计算校验和，并在读取时计算校验和
  * datanode负责在收到数据后存储该数据及其校验和之前对数据进行验证，它在收到客户端的数据或复制其他datanode的数据时执行这个操作
  * 不只是客户端读取数据时会验证校验和，每个datanode也会在后台线程运行一个DataBlockScanner，从而定期验证存储在这个datanode上的所有数据块
  * HDFS存储有每个块的复本，因此可以通过数据复本来修复损坏的数据块

##### 2. 压缩
* 文件压缩两大好处：减少存储文件所需的磁盘空间，加速数据在网络和磁盘上的传输速度
* Hadoop中的压缩格式

  压缩格式|工具|算法|文件扩展名|是否可切分
  -|-|-|-|-
  DEFLATE|无|DEFLATE|.deflate|否
  gzip|gzip|DEFLATE|.gz|否
  bzip2|bzip2|bzip2|.bz2|是
  LZO|lzop|LZO|.lzo|否
  LZ4|无|LZ4|.lz4|否
  Snappy|无|Snappy|.snappy|否

  * 上表中所有压缩工具都提供9个选项来控制压缩必须考虑的压缩速度和压缩空间的权衡。-1为优化压缩速度，-9为优化压缩空间
* codec
  * codec是压缩-解压缩算法的一种实现

    压缩格式|HadoopCompressionCodec
    -|-
    DEFLATE|org.apache.hadoop.io.compress.DefaultCodec
    DEFLATE|org.apache.hadoop.io.compress.GzipCodec
    DEFLATE|org.apache.hadoop.io.compress.Bzip2Codec
    DEFLATE|com.hadoop.compression.lzo.LzoCodec
    DEFLATE|org.apache.hadoop.io.compress.lz4Codec
    DEFLATE|org.apache.hadoop.io.compress.SnappyCodec

  * 原生类库：为了提高性能最好使用原生类库来实现压缩和解压缩，比java实现更快

    压缩格式|是否有Java实现|是否有原生实现
    -|-|-
    DEFLATE|是|是
    gzip|是|是
    bzip2|是|否
    LZO|否|是
    LZ4|否|是
    Snappy|否|是

  * CodecPool：如果使用的是原生类库并且需要在应用中执行大量压缩和解压缩操作，可以使用CodecPool
* 压缩和输入分片：主要针对使用哪种压缩格式以提高Hadoop执行效率，因为不是所有压缩格式都支持数据切分
  1. 使用容器文件格式，例如顺序文件、Avro数据文件、ORCFiles或Parquet文件，这些文件都同时支持压缩和切分
  2. 使用支持切分的压缩格式如bzip2，或者使用通过索引实现切分的压缩格式如LZO
  3. 在应用中将文件先切分为数据块再分别对数据块进行压缩，这样就可使用任意压缩文件。但是要注意合理切分数据块以确保压缩后数据块大小接近HDFS块的大小
* 在MapReduce中使用压缩
  1. 对MapReduce作业的输出进行压缩，以减少写入HDFS系统磁盘的数据量

     属性名称|类型|默认值|描述
     -|-|-|-
     mapreduce.output.fileooutputformat.compress|boolean|false|是否压缩输 出
     mapreduce.output.fileoutputformat.compress.codec|类名称|org.apache.hadoop.io.compress.DefaultCodec|map输出所用的压缩codec
     mapreduce.output.fileoutputformat.compress.type|String|RECORD|顺序文 件输出可以使用的压缩类型：NONE/RECORD/BLOCK

  2. 对map任务输出进行压缩，以减少传输到reducer的数据量，提升带宽和磁盘使用速度
     属性名称|类型|默认值|描述
     -|-|-|-
     mapreduce.map.output.compress|boolean|false|是否对map任务输出进行压缩
     mapreduce.map.output.compress.codec|Class|org.apache.hadoop.io.compress.DefaultCodec|map输出所用的压缩codec
    
##### 3. 序列化
* 序列化是指将结构化对象转化为字节流以便在网上传输或写到磁盘进行永久存储的过程。反序列化是指将字节流转回结构化对象的逆过程
* 序列化用于分布式数据处理的两大领域：进程间通信和永久存储
* Hadoop系统中多个节点进程间通信是通过“远程过程调用”(RPC,remote procedure call)实现的。RPC协议将消息序列化成二进制流后发送到远程节点，远程节点将二进制流反序列化为原始消息
* Hadoop序列化格式要求
  1. 紧凑：紧凑格式充分利用网络带宽
  2. 快速：要尽量减少序列化和反序列化的性能开销
  3. 可扩展：为了满足新需求，协议不断变化
  4. 支持互操作：希望能以不同语言写的客户端与服务器交互
* Hadoop使用自己的序列化格式Writable，紧凑且速度快，但是不易用Java以外的语言扩展或使用
* 序列化框架：大多数MapReduce程序使用的都是Writable类型的键和值，但是并不是强制要求，可使用其他的序列化框架。为了支持这一机制Hadoop有针对可替换序列化框架的API，用Serialization实现
* 序列化IDL(Interface Description Language，接口定义语言)：不依赖于具体的语言，能够为其他语言生成类型，有效提高互操作能力。例如Apache Thrift、Google Buffers和Avro等

##### 4. 基于文件的数据结构
