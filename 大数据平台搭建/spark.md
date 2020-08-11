# spark
## 搭建spark
### 安装scala
* 解压安装包到/opt/module/scala
* 添加系统环境变量

### 安装并配置spark
#### 安装spark
* 解压安装包到/opt/module/spark
* 添加系统环境变量

#### 配置spark
* /opt/module/spark/conf/spark-env.sh
  ```
  # cp /opt/module/spark/conf/spark-env.sh.template /opt/module/spark/conf/spark-env.sh
  # vim /opt/module/spark/conf/spark-env.sh
  ```
  在文件中添加如下配置
  ```
  export JAVA_HOME=/opt/module/jdk1.8.0_241
  export SCALA_HOME=/opt/module/scala-2.11.12
  export HADOOP_HOME=/opt/module/hadoop-3.1.3
  export HADOOP_CONF_DIR=/opt/module/hadoop-3.1.3/etc/hadoop
  export SPARK_MASTER_IP=192.168.184.103
  export SPARK_MASTER_HOST=hadoop103
  export SPARK_LOCAL_IP=192.168.184.104
  export SPARK_WORKER_MEMORY=1g
  export SPARK_WORKER_CORES=2
  export SPARK_HOME=/opt/module/spark-2.4.6-bin-hadoop2.7
  export LD_LIBRARY_PATH=/opt/module/hadoop-3.1.3/lib/native
  export SPARK_DIST_CLASSPATH=$(/opt/module/hadoop-3.1.3/bin/hadoop classpath)
  ```
  在主服务器和各从服务器上，要将SPARK_LOCAL_IP改为本地的IP地址
* /opt/module/spark/conf/slaves
  文件中只需添加从服务器的地址，一行一个
  ```
  hadoop101
  hadoop102
  hadoop104
  ```

### 启动spark
#### 启动spark集群
```# /opt/mudule/spark/sbin/start-all.sh```
注意，这里的脚本与hadoop的脚本同名，因此一般不把/opt/mudule/spark/sbin加入环境变量，启动时要用绝对路径
#### 使用spark
* scala启动
  ```# spark-shell```
  :quit命令退出
* python启动
  ```# pyspark```
  exit()命令退出

### spark连接hive
* 将hive-site.xml拷贝到spark配置文件夹/opt/module/spark/conf，hive-site.xml必须包含以下内容
  ```
  <property>
  <name>hive.metastore.uris</name>
  <value>thrift://localhost:9083</value>
  </property>
  ```
  即hive元数据存储的地址
* 启动hive的metastore服务，启动spark即可使用hiveSQL