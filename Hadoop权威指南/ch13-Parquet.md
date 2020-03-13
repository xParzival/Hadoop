* Apache Parquet是一种能够有效存储嵌套数据的列式存储格式，它以扁平的列式存储结构和很小的额外开销来存储嵌套的结构。很多工具都支持这种格式

##### 1. 数据模型
* Parquet定义了少数几个原子数据类型

  类型|描述
  -|-
  boolean|二进制值
  int32|32位有符号整数
  int64|64位带符号整数
  int96|96位有符号整数
  float|单精度(32位)IEEE754浮点数	
  double|双精度(64位)IEEE754浮点数
  binary|8位无符号字节序列
  fixed_len_byte_array|固定数量的8位无符号字节

* 保存在Parquet文件中的数据通过模式进行描述，模式的根为message，其中包含一组字段，每个字段有一个重复数(required,optional或repeated)、一个数据类型和一个字段名称组成，例如
  ```
  message AddressBook {
    required binary owner (UTF8); # required表示必须且唯一
    repeated binary ownerPhoneNumbers (UTF8); # repeated表示可以有零个或多个
    repeated group contacts {
      required binary name (UTF8);
      optional binary phoneNumber (UTF8); # optional表示可有可无
      }
  }
  ```
* Parquet原子类型不包括字符串。Parquet定义了一些逻辑类型，这些逻辑类型指出应当如何对原子类型进行解读，从而使得序列化的表示与特定应用的语义相互独立

  逻辑类型注解|描述|模式示例
  -|-|-
  UTF8|由UTF-8字符组成的字符串，可用于注解binary|message m {required binary a (UTF8)}
  ENUM|命名值的集合，可用于注解binary|message m {required binary a (ENUM)}
  DEMICAL(precision,scale)|任意精度的有符号小数，可用于注解int32，int64，binary或fixed_len_byte_array|message m {required int32 a (DEMICAL(5,2))}
  DATE|不带时间的日期，可用于注解int32|message m {required int32 a (DATE)}
  LIST|一组有序的值，可用于注解group|message m {<br>required group a (LIST) {<br>required group list {<br>required int32 element<br>}}}
  MAP|一组无序的键值对，可用于注解group|message m {<br>required group a (MAP) {<br>repeated group_key_value {<br>required binary key (UTF8);<br>optional int32 value;<br>}}}

* Parquet利用group类型来构造复杂类型，它可以增加一级嵌套。没有注解的group就是一个简单的嵌套记录
* 嵌套编码
  * 对于没有嵌套没有重复的列式存储结构，可以直观判断每个值属于哪一行。对于有嵌套的和重复的结构还需要对嵌套的结构进行编码
  * Parquet使用Dremel编码方式。模式中每个原子类型字段都单独存储为一列，且每个值都要通过使用两个整数来对其结构进行编码，这两个整数分别是列定义深度(definition level)和列元素重复次数(repetition)。这样做对于任意一列(即使是嵌套列)数据的读取都不涉及其他列，从而极大提升读取性能
  
##### 2. Parquet文件格式
* Parquet文件由一个文件头(header)、一个或多个紧随其后的文件块(block)，以及一个用于结尾的文件尾(footer)构成。
  * header：仅包含一个称为PAR1的4字节数字(Magic Number)，用来识别整个Parquet文件格式
  * footer：保存文件的所有元数据，包括文件格式的的版本信息、模式信息、额外的键值以及所有块的元数据信息。文件尾的最后两个字段分别是一个4字节字段(包含文件尾中元数据长度的编码)和一个PAR1(与文件头中的相同)
  * 读取一个Parquet文件时，先读取Footer的metadata。顺序文件和Avro数据文件都是把元数据保存在文件头，并且使用sync marker分割文件块。Parquet格式文件不需要读取sync markers这样的标记分割查找，因为metadata的写入是在所有blocks块写入完成之后的，所以所有block的位置信息都存在于内存直到文件close
  * 通过读取footer的metadata可以定位文件块，因此Parquet文件是可分割的切可并行处理的
* Parquet文件中每个文件块负责存储一个行组(row group)，行组由列块(column chunk)构成，切一个列块负责存储一列数据，每个列块中的数据以页(page)为单位存储
  ![Parquet文件内部结构](https://images2015.cnblogs.com/blog/820234/201606/820234-20160606221152449-1498604387.jpg)
  * 每页包含的值都属于同一列，因此极有可能相差不大，所以一般以页为压缩单位。在写文件时Parquet会根据列的类型自动选择适当的编码方式进行压缩。也可以以页为单位利用标准压缩算法对编码后的数据进行第二次压缩

##### 3. Parquet的配置
* Parquet文件的属性在写操作时设置，下表列出的文件属性适用于MapReduce、Crunch、Pig或Hive创建的Parquet文件
  
  属性名称|类型|默认值|描述
  parquet.block.size|int|134217728(128MB)|文件块大小，以字节为单位
  parquet.page.size|int|1048576(1MB)|页大小，以字节为单位
  parquet.dictionary.page.size|int|1048576(1MB)|一个页可允许的最大字典，以字节为单位。若字典超过这个尺寸，将退回到无格式编码
  parquet.enable.dictionary|boolean|true|是否使用字典编码
  parquet.compression|string|UNCOMPRESSED|Parquet文件使用的压缩类型

* Parquet文件块的大小不能超过其HDFS块大小，这样才能使每个Parquet文件块仅需读一个HDFS块，一般设为相同的值
* 页是Parquet文件最小存储单元，页越小单行查找要读取的值越少效率越高。但是页越小页个数就越多，额外的元数据页就越多