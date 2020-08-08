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
  DELIMITED # 分隔符设置开始语句
  [FIELDS TERMINATED BY '分隔符' [ESCAPED BY char]] # 字段分隔符
  [COLLECTION ITEMS TERMINATED BY char] # 设置一个复杂类型(array,struct)字段的各个item之间的分隔符
  [MAP KEYS TERMINATED BY char] # 设置一个复杂类型(Map)字段的key value之间的分隔
  [LINES TERMINATED BY char] # 行与行之间的分隔符
  [NULL DEFINED AS char] # 空值设置
  [SERDE serde_name WITH SERDEPROPERTIE (
      property_name=property_value,
      property_name=property_value,
      ...)];
  ```
* 文件格式：默认TEXTFILE，其他支持的文件格式有RCFILE、ORC、PARQUET、AVRO等，需要用对应的serde属性来设置

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
