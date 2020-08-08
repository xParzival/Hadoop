搭建版本Hadoop3.1.3+hive3.1.2+hbase2.0.0-alpha4+spark2.4.5+zookeeper3.6.1

四台主机功能表：

组件|hadoop101|hadoop102|hadoop103|hadoop104
-|-|-|-|-
HDFS|namenode<br>datanode|datanode|secondarynamenode<br>datanode|datanode
YARN|nodemanager|resourcemanager<br>nodemanager|nodemanager|nodemanager
MapReduce|-|-|-|historyserver
metastore|-|-|-|mysql
zookeeper|-|server1|server2|server3
hbase|HMaster<br>HRegionServer|HRegionServer|HRegionServer|HRegionServer

启动顺序：
1. hadoop
2. zookeeper
3. hbase
4. hive

关闭顺序与启动顺序相反