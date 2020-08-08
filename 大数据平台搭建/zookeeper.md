# zookeeper
## 搭建zookeeper
### 安装zookeeper
* 安装包解压到/opt/module，根目录/opt/module/zookeeper
* 添加系统环境变量

### 配置zookeeper
* 添加zookeeper数据文件夹/opt/module/zookeeper/data
* 添加配置文件zoo.cfg
  ```
  # cp /opt/module/zookeeper/conf/zoo_sample.cfg /opt/module/zookeeper/conf/zoo.cfg
  # vim /opt/module/zookeeper/conf/zoo.cfg

  tickTime=2000
  dataDir=/opt/module/zookeeper/data
  clientPort=2181
  initLimit=5
  syncLimit=2
  server.1=192.168.229.160:2888:3888
  server.2=192.168.229.161:2888:3888
  server.3=192.168.229.162:2888:3888
  4lw.commands.whitelist=*
  ```
* 在/opt/module/zookeeper/data添加myid文件。文件内容为一个数字，对应zoo.cfg中server.x的x
  例如对于server.1对应的机器上添加
  ```# echo "1" >> /opt/module/zookeeper/data```

### 启动zookeeper
* 在指定了zookeeper的每台机器上执行
  ```zkServer.sh start```