# HiveSQL
## 数据仓库操作
### 创建数据仓库
```
CREATE DATABASE [IF NOT EXISTS] 仓库名
[LOCATION 'hdfs目录path'] # 省略则按配置文件hive.metastore.warehouse.dir指定目录
[COMMENT '添加描述']
[WITH DBPROPERTIES ('property1' = 'value1', ...)];
```
### 查看仓库信息
```
DESCRIBE DATABASE [EXTENDED] 仓库名;
```
### 修改数据仓库
```
ALTER DATABASE 仓库名 SET DBPROPERTIES ('property1' = 'value1', ...);
```
## hive表操作
### 内部表（管理表）和外部表
hive的表分为内部表和外部表
* 内部表：hive控制其生命周期，hive删除该表时，也会删除表中数据
* 外部表不方便与其他工具共享数据，例如我们想用hive查询hbase的表，但是hive对这些数据没有所有权，这种表称为外部表，hive删除该表时，表本身的数据不会改变

### 创建表
```
CREATE [EXTERNAL] TABLE [IF NOT EXISTS] 表名 (
    列名1 数据类型1 [COMMENT '描述1'],
    列名2 数据类型2 [COMMENT '描述2'],
    ...
)
[COMMENT '表描述']
[TBLPROPERTIES ('property1' = 'value1', ...)]
[LOCATION 'hdfs目录path']
[ROW FORMAT 行格式]
[STORED AS 文件格式];
```
EXTERNAL参数即表示创建外部表
* 行格式：
  ```
  [DELIMITED # 分隔符设置开始语句
  [FIELDS TERMINATED BY '分隔符' [ESCAPED BY char]] # 字段分隔符
  [COLLECTION ITEMS TERMINATED BY char] # 设置一个复杂类型(array,struct)字段的各个item之间的分隔符
  [MAP KEYS TERMINATED BY char] # 设置一个复杂类型(Map)字段的key value之间的分隔
  [LINES TERMINATED BY char] # 行与行之间的分隔符
  [NULL DEFINED AS char]]# 空值设置
  [SERDE serde_name WITH SERDEPROPERTIE (
      property_name=property_value,
      property_name=property_value,
      ...)];
  ```
* 文件格式：默认TEXTFILE，其他支持的文件格式有RCFILE、ORC、PARQUET、AVRO等，需要用对应的serde属性来设置

例程：创建一个可导入csv文件的hive表，以','为分隔符，以"'"为字符串指示符
```
create table test(
  col1 int,
  col2 string,
  col3 string,
  ...
)
row format serde 'org.apache.hadoop.hive.serde2.OpenCSVSerde' # csv文件的serde解析器
with serde properties(
  "separatorChar" = ",", # 指定分隔符
  "quoteChar" = "\'" # 指定字符串指示符
)
stored as textfile;
```
### 复制表结构
```
CREATE TABLE 表名 LIKE 旧表名;
```
复制旧表的结构，但是不导入数据
### 查看表信息
```
DESCRIBE [EXTENDED] [FORMATTED] 表名;
```
### 修改表
```ALTER TABLE```语句。注意，该语句只修改表元数据，表本身不发生改变，需要自己确认修改和真实表数据是一致的
* 重命名
  ```ALTER TABLE 表名 RENAME TO 新表名;```
* 修改列信息
  ```
  ALTER TABLE 表名
  CHANGE COLUMN 列名 [新列名 新数据类型]
  [COMMENT '描述']
  [AFTER 列名] # 移动到某列之后
  [FIRST]; # 移动到最前
  ```
* 增加列
  ```
  ALTER TABLE 表名 ADD COLUMNS(
      列名1 数据类型 [COMMENT '描述1'],
      列名2 数据类型 [COMMENT '描述2'],
      ...
  );
  ```
* 删除列或替换列
  ```
  ALTER TABLE 表名 REPLACE COLUMN(
      列名1 新数据类型 [COMMENT '描述'],
      列名2 数据类型 [COMMENT '描述'],
      ...
  );
  ```
  如果指定列新数据类型和原数据类型不同，则替换为新列，如果相同则删除列
* 修改表属性
  ```
  ALTER TABLE 表名 SET TBLPROPERTIES (
      'property1' = 'value1',
      'property2' = 'value2',
      ...
  );
  ```
  可以增加属性，或修改已有属性，但是不能删除属性
* 修改存储属性
  1. 修改为内置文件格式
     ```
     ALTER TABLE 表名
     SET FILEFORMAT 文件格式;
     ```
  2. 修改为其他serde指定格式
     ```
     ALTER TABLE 表名
     SET SERDE 'SERDE名'
     WITH SERDEPROPERTIES (
         'property1' = 'value1',
         'property2' = 'value2',
         ...
     );
     ```

* 删除表
  1. 删除表中指定行的数据
     ```delete from 表名 where 指定条件;```
  2. 删除表中所有数据，保留表结构
     ```truncate table 表名;```
  3. 删除表
     ``` drop table [if exists] 表名;```

### 向hive表写入数据
1. 和普通sql语句一样insert into/overwrite
   ```
   INSERT INTO/OVERWRITE TABLE 表名 (
       列1,
       列2,
       ...) 
   VALUES (
       val1,
       val2,
       ...
   );
   ```
2. load语句加载文件中的数据，要注意文件类型和分隔符
   ```
   LOAD DATA [LOCAL] INPATH '文件路径'
   OVERWRITE INTO TABLE 表名;
   ```
   LOCAL代表本地路径，不加代表hdfs路径
3. 批量导入已存在的表数据
   ```
   FROM 已存在表名
   INSERT INTO TABLE 表名
     SELECT 列1,列2,... WHERE 条件1
   INSERT INTO TABLE 表名
     SELECT 列1,列2,... WHERE 条件2
   INSERT INTO TABLE 表名
     SELECT 列1,列2,... WHERE 条件3;
   ```

## hive表优化
### 分区表
分区是指按照数据表的某列或某些列分为多个区，区从形式上可以理解为文件夹，比如我们要收集某个大型网站的日志数据，一个网站每天的日志数据存在同一张表上，由于每天会生成大量的日志，导致数据表的内容巨大，在查询时进行全表扫描耗费的资源非常多。那其实这个情况下，我们可以按照日期对数据表进行分区，不同日期的数据存放在不同的分区，在查询时只要指定分区字段的值就可以直接从该分区查找
因为分区在特定的区域（子目录）下检索数据，它起到了减少扫描成本的作用，提高查询效率
分区分为单值分区和范围分区，单值分区又分为静态分区和动态分区
分区一般用partitioned by关键字指定，分区表创建可以使用直接创建和CREATE TABLE LIKE，分区表不能用CREATE TABLE AS SELECT创建
#### 单值分区
单值分区即分区键按是否等于某个值划分数据。单值分区根据插入时是否需要手动指定分区可以分为
1. 单值静态分区：导入数据时需要手动指定分区
2. 单值动态分区：导入数据时，系统可以动态判断目标分区

* 静态分区：导入数据时需要手动指定分区
  * 创建静态分区：注意分区键不能与列重名
    ```
    CREATE [EXTERNAL] TABLE <table_name>
    (<col_name> <data_type> [, <col_name> <data_type> ...])
    -- 指定分区键和数据类型
    PARTITIONED BY  (<partition_key> <data_type>, ...) 
    [CLUSTERED BY ...] 
    [ROW FORMAT <row_format>] 
    [STORED AS TEXTFILE|ORC|CSVFILE]
    [LOCATION '<file_path>']    
    [TBLPROPERTIES ('<property_name>'='<property_value>', ...)];
    ```
  * 向静态分区写入数据
    ```
    load data [local] inpath '<filepath>' overwrite into table <table_name> partition (<partition_name=partition_value>);
    ```
    ```
    insert overwrite/into table <tbl_name> partition (<partition_key=partition_value>,...) select <select_statement>;
    ```
* 动态分区：导入数据时，系统可以动态判断目标分区
  * 创建动态分区
    动态分区需要先开启功能，可以在配置文件开启，也可以set
    ```
    SET hive.exec.dynamic.partition=true;
    SET hive.exec.dynamic.partition.mode=nonstrict; # 可以全部分区都是动态，否则必须有静态
    SET hive.exec.max.dynamic.partitions.pernode = 1000;
    SET hive.exec.max.dynamic.partitions=1000;
    ```
    创建方式与静态分区表完全一样，一张表可同时被静态和动态分区键分区，只是动态分区键需要放在静态分区键的后面（因为HDFS上的动态分区目录下不能包含静态分区的子目录），如下 spk 即 static partition key， dpk 即 dynamic partition key
    ```
    CREATE TABLE <table_name>
    (<col_name> <data_type> [, <col_name> <data_type> ...])
    PARTITIONED BY ([<spk> <data_type>, ... ,] <dpk> <data_type>, [<dpk>
    <data_type>,...]);
    ```
  * 向动态分区表插入数据
    ```
    insert into/overwrite table <table_name> partition (<spk sp_value>,...,<dpk>,...)
    select <select statement>;
    ```
    插入数据时，select语句多出的字段就会自动被作为动态分区键，因此一定要注意select语句后面字段的顺序，动态分区键必须在最后且顺序保持一致
#### 范围分区
单值分区每个分区对应于分区键的一个取值，而每个范围分区则对应分区键的一个区间，只要落在指定区间内的记录都被存储在对应的分区下。分区范围需要手动指定，分区的范围为前闭后开区间[最小值, 最大值)。最后出现的分区可以使用 MAXVALUE 作为上限，MAXVALUE 代表该分区键的数据类型所允许的最大值

 ```
 CREATE [EXTERNAL] TABLE <table_name>
    (<col_name> <data_type>, <col_name> <data_type>, ...)
    PARTITIONED BY RANGE (<partition_key1> <data_type>,<partition_key2> <data_type> ...) 
        (PARTITION [<partition_name>] VALUES LESS THAN (<key1_cutoff,key2_cutoff,...>), 
        [PARTITION [<partition_name>] VALUES LESS THAN (<key1_cutoff,key2_cutoff,...>),
              ...]
        PARTITION [<partition_name>] VALUES LESS THAN (<key1_cutoff,key2_cutoff,...>|<MAXVALUE1,MAXVALUE2,...>) 
        )
    [ROW FORMAT <row_format>] [STORED AS TEXTFILE|ORC|CSVFILE]
    [LOCATION '<file_path>']    
    [TBLPROPERTIE ('<property_name>'='<property_value>', ...)];
 ```
#### 查看分区
 ```
 show partitions <table_name>;
 ```
### 分桶表
分桶是指定分桶表的某一列，让该列数据按照哈希取模的方式随机、均匀地分发到各个桶文件中。因为分桶操作需要根据某一列具体数据来进行哈希取模操作，故指定的分桶列必须基于表中的某一列（字段）。因为分桶改变了数据的存储方式，它会把哈希取模相同或者在某一区间的数据行放在同一个桶文件中。如此一来便可提高查询效率，如：我们要对两张在同一列上进行了分桶操作的表进行JOIN操作的时候，只需要对保存相同列值的桶进行JOIN操作即可。同时分桶也能让取样（Sampling）更高效
#### 创建分桶表
分桶表的建表有三种方式：直接建表，CREATE TABLE LIKE 和 CREATE TABLE AS SELECT，关键字为CLUSTERED BY
 ```
 CREATE [EXTERNAL] TABLE <table_name>
    (<col_name> <data_type> [, <col_name> <data_type> ...])
    [PARTITIONED BY ...] 
    CLUSTERED BY (<col_name>) 
        [SORTED BY (<col_name> [ASC|DESC] [, <col_name> [ASC|DESC]...])] 
        INTO <num_buckets> BUCKETS  
    [ROW FORMAT <row_format>] 
    [STORED AS TEXTFILE|ORC|CSVFILE]
    [LOCATION '<file_path>']    
    [TBLPROPERTIES ('<property_name>'='<property_value>', ...)];
 ```
分桶键只能有一个即<col_name>。表可以同时分区和分桶，当表分区时，每个分区下都会有<num_buckets> 个桶。我们也可以选择使用 SORTED BY … 在桶内排序，排序键和分桶键无需相同。ASC 为升序选项，DESC 为降序选项，默认排序方式是升序。<num_buckets> 指定分桶个数，也就是表目录下小文件的个数
* clustered by指定按哪一列进行分桶
* sorted by指定按那些列进行排序，默认升序
#### 向分桶表写入数据
分桶表在创建的时候只会定义Scheme，且写入数据的时候不会自动进行分桶、排序，需要人工先进行分桶、排序后再写入数据，确保目标表中的数据和它定义的分布一致
reduce的个数和分桶的个数要一致，主要有两种方法来保证
1. 打开强制分桶开关
   ```
   SET hive.enforce.bucketing=true;
   ```
   这个选项可以自动控制上一轮reducer的数量从而适配bucket的个数，并且识别分桶键，然后可插入数据
   ```
   INSERT INTO/OVERWRITE TABLE <table_name>
   SELECT <select_statement>
   [SORT BY <sort_key> [ASC|DESC], [<sort_key> [ASC|DESC], ...]];
   ```
2. 手动设置reducer的个数与分桶数一致，插入数据时手动指定分桶键
   ```
   set mapreduce.job.reduces=<num_buckets>;
   ```
   * cluster by：不能指定排序方式，只能默认升序，且排序键与分桶键一致
     ```
     INSERT INTO/OVERWRITE TABLE <bucketed_table>
     SELECT <select_statement>
     CLUSTER BY <bucket_sort_key>;
     ```
   * distribute by+sort by：distribute by指定分桶键，sort by指定排序键，这样可以分桶键与排序键不同，且可以指定排序方式
     ```
     INSERT INTO/OVERWRITE TABLE <bucketed_table>
     SELECT <select_statement>
     DISTRIBUTE BY <bucket_sort_key>
     [SORT BY <sort_key> [ASC|DESC],[<sort_key> [ASC|DESC],...];
     ```
   
#### 分桶总结
分桶公式：
bucket = hash_function(bucketing_column) mod num_buckets
* 分桶字段的选择
  1. int类型字段比较友好
  2. 取hash后各分区块数据量比较均匀的字段
  3. join的连接字段

  当join连接的字段值取hash不够均匀时,多取一个其它字段作为分桶字段
* BUCKETS数量的选择
  1. 当数据量够大时设置为约等于≈128M的倍数
  2. 当数据量不够大时考虑,计算的并行度（比如129MB设置2或者4）
  bucket个数会决定在该表或者该表的分区对应的hdfs目录下生成对应个数的文件,而mapreduce的个数是根据文件块的个数据确定的map个数