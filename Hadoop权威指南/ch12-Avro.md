* Avro是一种独立于编程语言的数据序列化系统，旨在解决Hadoop中Writable类型的不足：缺乏语言的可移植性
##### 1. Avro数据类型和模式
* Avro基本类型

  类型|描述|模式实例
  -|-|-
  null|空值|"null"
  boolean|二进制值|"boolean"
  int|32位带符号整数|"int"
  long|64位带符号整数|"long"
  float|单精度(32位)IEEE754浮点数|"float"
  double|双精度(64位)IEEE754浮点数|"double"
  bytes|8位无符号字节序列|"bytes"
  string|Unicode字符序列|"string"

* Avro复杂类型

  类型|描述|模式实例
  -|-|-
  array|一个排过序的对象集合。特定数组中的所有对象必须模式相同|{"type": "array",<br>"items": "long"}
  map|未排过序的键值对。键必须是字符串，值可以是任何一种类型，一个特定的map中所有值模式必须相同|{"type": "map",<br>"values": "string"}
  record|一个任意类型的命名字段集合|{"type": "record",<br>"name": "WeatherRecord",<br>"doc": "A weather reading.",<br>"fields":[{"name": "year", "type": "int"},<br>{"name": "temperature", "type": "int"},<br>{"name": "stationId", "type": "string"}]}
  enum|一个命名值集合|{"type": "enum",<br>"name": "Cutlery",<br>"doc": "An eating utensil.",<br>"symbols": ["KNIGHT", "FORK", "SPOON"]}
  fixed|一组固定数量的8位无符号字节|{"type": "fixed",<br>"name": "Md5Hash",<br>"size": 16}
  union|模式的并集。并集可用json数组表示，其中每个元素为一个模式|["null",<br>"string",<br>{"type": "map", "values": "string"}]

##### 2. 内存中的序列化和反序列化
* Avro为序列化和反序列化提供了API，如果想把Avro集成到现有系统就可以使用这些API。其他情况可考虑使用Avro数据文件格式

##### 3. Avro数据文件
* Avro的对象容器文件格式主要用于存储Avro对象序列
* Avro数据文件的头部含有元数据，包括一个Avro模式和一个sync marker(同步标识)，紧接着是一系列包含序列化Avro对象的数据块(压缩可选)。数据块通过sync marker分隔，而sync marker对该文件来说是唯一的并允许在文件中搜索到任意位置之后通过块边界快速重新进行同步。因此Avro数据文件是可切分的，适合MapReduce快速处理

##### 4. 互操作性
* Avro有多种不同语言的API

##### 5. 模式解析
* 可以选择使用不同于写入数据的模式来读取数据，这样可以进行模式的演化
* 别名：别名允许在读取Avro数据的模式与写入Avro数据模式时使用不同的字段名称

##### 6. 排列顺序
* Avro定义了对象的排列顺序。除了record之外，所有类型均按照Avro规范中预先定义的规则来排序。对于record可以通过order属性来控制排列顺序,有三个值：ascending(默认值)descending或ignore
* Avro实现了高效的二进制比较，不需要将二进制对象反序列化为对象即可实现比较，因为它可以直接对字节流进行操作

##### 7. 关于Avro MapReduce
* 为了方便在Avro数据上运行MapReduce程序，Avro在org.apache.avro.mapreduce包中有MapReduce的API
