# Python连接hadoop
python与hadoop各种组件的连接方式
## 连接hive
Linux建议使用使用pyhive包，windows建议使用impyla包。以下介绍pyhive
### 安装
* 安装依赖包sasl，thrift，thrift-sasl
* 安装pyhive包

### 使用
* 连接
  ```
  from pyhive import hive
  conn = hive.Connection(host='IP', port=10000, username='hadoop', database='database名')
  cursor = conn.cursor()
  ```
* 运行hiveSQL语句
  ```
  cursor.execute("hiveSQL")
  ```
  注意不用在结尾加分号
* 查看结果
  ```
  cursor.fetchall()
  ```
* 关闭连接
  ```
  cursor.close()
  conn.close()
  ```
  ## 连接hbase
  