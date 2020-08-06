搭建版本Hadoop3.1.3+hive3.1.2+hbase2.2.5

四台主机功能表：

组件|hadoop101|hadoop102|hadoop103|hadoop104
-|-|-|-|-
HDFS|namenode<br>datanode|datanode|secondarynamenode<br>datanode|datanode
YARN|nodemanager|resourcemanager<br>nodemanager|nodemanager|nodemanager
MapReduce|-|-|-|historyserver
metastore|-|-|-|mysql

