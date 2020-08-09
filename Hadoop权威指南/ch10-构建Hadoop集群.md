##### 1. 集群的构建和安装(完全分布式集群)
* 构建流程范例(以四台服务器为例)
  1. 准备三台安装好Java和Hadoop的服务器
  
  2. 集群配置
     1. 集群部署规划

        组件|hadoop101|hadoop102|hadoop103|hadoop104
        -|-|-|-|-
        HDFS|namenode<br>datanode|datanode|secondarynamenode<br>datanode|datanode
        YARN|nodemanager|resourcemanager<br>nodemanager|nodemanager|nodemanager
        MapReduce|-|-|-|historyserver

     2. 配置集群
        * 重要Hadoop配置文件(详细配置参数见本节后文)

          文件名称|格式|描述
          -|-|-
          hadoop-env.sh|Bash脚本|配置运行Hadoop需要的环境变量
          mapred-env.sh|Bash脚本|配置运行MapReduce需要的环境变量，会覆盖hadoop-env.sh的设置
          yarn-env.sh|Bash脚本|配置运行yarn需要的环境变量，会覆盖hadoop-env.sh的设置
          core-site.xml|Hadoop配置XML|Hadoop Core配置项，比如HDFS、MapReduce和YARN常用的I/O设置等
          hdfs-site.xml|Hadoop配置XML|Hadoop守护进程配置项，包括namenode、datanode和辅助namenode等
          mapred-site.xml|Hadoop配置XML|MapReduce守护进程配置项，包括作业历史服务器
          yarn-site.xml|Hadoop配置XML|YARN守护进程配置项，包括资源管理器、web应用代理服务器和节点管理器
          hadoop-policy.xml|Hadoop配置XML|安全模式下运行Hadoop时的访问控制列表配置项
          slaves|纯文本|运行datanode和节点管理器的机器列表，每行一个
          hadoop-metrics2.properties|Java属性|控制如何在Hadoop发布度量的属性
          log4j.properties|Java属性|系统日志文件、namenode审计日志、任务JVM进程的任务日志的属性
     
     3. 编写集群分发脚本xsync，并修改权限，以便服务器可同步修改配置
        * Hadoop没有将配置信息放在一个单独的全局位置中，而是集群的每个节点各自保存一套配置文件，由管理员完成配置文件的同步。有很多工具可以帮助同步配置文件，比如dsh或pdsh。Hadoop集群管理工具诸如Cloudera Manager和Apache Ambari等。这里我们自己写一个脚本进行同步
          ```
          mkdir ~/bin # 在hadoop用户家目录下创建bin文件夹
          vim ~/bin/xsync # 创建xsync同步脚本文件
          ```
        * 文件内容如下
          ```
          #!/bin/bash
          # 1.获取输入参数个数，如果没有参数，直接退出
          pcount=$#
          if [ $pcount -eq 0]
          then
          echo no args
          exit
          fi
          
          # 2.获取文件名称
          p1=$1
          fname=`basename $p1`
          echo fname=$fname
          
          # 3.获取上级目录的绝对路径
          pdir=`cd -P $(dirname $p1); pwd`
          echo pdir=$pdir
          
          # 4.获取当前用户名称
          user=`whoami`
          
          # 5.循环对每台服务器配置
          for host in {102..104}; do
              echo ------------------- hadoop$host -------------------
              rsync -rvl $pdir/$fname $user@hadoop$host:$pdir
          done
          ```

  3. 集群的单节点启动
     1. 集群第一次启动需要格式化HDFS文件系统：通过创建存储目录和初始版本的namenode持久数据结构，格式化进程创建一个空文件系统
        * 由于namenode管理所有的文件系统元数据，datanode可以动态的加入或离开集群，所以初始的格式化进程不涉及datanode
        ```hdfs namenode -format```
     2. 启动namenode和datanode
        * 在namenode所在机器启动namenode
        ```hadoop-daemon.sh start namenode```
        * 在secondary namenode所在机器启动secondarynamenode
        ```hadoop-daemon.sh start secondarynamenode```
        * 在其他机器上分别启动datanode
        ```hadoop-daemon.sh start datanode```
     3. 启动YARN
        * 在resource manager所在机器启动resourcemanager
        ```yarn-daemon.sh start resourcemanager```
        * 其他节点分别启动node manager
        ```yarn-daemon.sh start nodemanager```
     4. 在history server所在机器启动historyserver
        ```mr-jobhistory-daemon.sh start historyserver```
     * 以上所有的脚本都有对应的 stop 命令
     * 这样当然可以启动集群，但是当节点很多的时候，没有办法一个一个启动，因此实际生产需要配置ssh无密登录然后进行群体启动

  4. 集群的群体启动
     1. SSH无密登录配置
        * 配置namenode所在机器对其他机器无密登录
        * 配置resource manager所在机器对其他机器无密登录
     2. 格式化HDFS文件系统(第一次启动时)
        ```hdfs namenode -format```
     3. 在namenode所在机器启动HDFS守护进程
        ```start-dfs.sh```
        * 该脚本运行所做的事情如下：
          1. 向Hadoop配置文件询问namonode主机名```hdfs getconf -namenodes```(HDFS HA有可能有多个namenode)
          2. 调用hadoop-daemon.sh脚本，在namenode所在机器上启动namonode，在辅助namenode所在机器启动secondarynamenode启动
          3. 调用hadoop-daemon.sh脚本，在slaves文件列举的每台机器上启动datanode
     4. 在resource manager所在机器启动yarn守护进程
        ```start-yarn.sh```
        * 该脚本运行所做的事情如下：
          1. 调用yarn-daemon.sh脚本，在本地机器启动一个resource manager
          2. 调用yarn-daemon.sh脚本，在slaves文件列举的每台机器上启动node manager
     5. 在history server所在机器启动历史服务器
        ```sbin/mr-jobhistory-daemon.sh start historyserver```
     * 启动脚本都有对应的 stop 脚本：stop-dfs.sh、stop-yarn.sh

  5. 创建用户目录
    * 建立并运行了Hadoop集群后需要给用户访问手段。给每个用户创建home目录并设置访问许可
      ```
      hadoop fs -mkdir -p /user/username
      hdfs dfs -mkdir -p /user/username # 这两种方式都能创建目录，推荐后一种
      hdfs dfs -chown username:username /user/username # 设置用户访问权限
      hdfs dfsadmin -setSpaceQuota 1t /user/username # 给用户目录设置容量限制，按需设置，例为1TB
      ```

##### 2. Hadoop配置
* 有多个配置文件适用于Hadoop，都放在 Hadoop安装目录/etc/hadoop 目录中
* Hadoop环境配置文件 hadoop-env.sh、yarn-env.sh、mapred-env.sh
  1. Java：在hadoop-env.sh设置Java环境变量
     ```JAVA_HOME=Java安装目录```
  2. Hadoop配置文件目录：hadoop-env.sh的HADOOP_CONF_DIR参数控制，默认为 hadoop安装目录/etc/hadoop。安置在Hadoop安装路径外以便系统升级，也可在启动守护进程时使用--config选项指定
  3. 内存堆大小：hadoop-env.sh的HADOOP_HEAPSIZE参数控制，默认为各个守护进程1000MB(1GB)内存。也可以单独为某守护进程设置堆大小，比如yarn-env.sh中设置参数YARN_RESOURCEMANAGER_HEAPSIZE可覆盖resource manager的内存堆大小
  4. 系统日志文件存放目录：hadoop-env.sh的HADOOP_LOG_DIR参数控制，默认为$HADOOP_HOME/logs
  5. SSH设置：hadoop-env.sh的HADOOP_SSH_OPTS参数控制，可以向SSH传递很多有用参数，具体参考ssh的配置手册
* Hadoop守护进程关键属性
  * Hadoop守护进程的配置文件有core-site.xml，hdfs-site.xml，yarn-site.xml，mapred-site.xml，在此编号为1、2、3、4
  1. core-site.xml
     
     属性名称|类型|默认值|详细说明
     -|-|-|-
     fs.defaultFS|URI|file:///|默认文件系统。URI定义主机名称和namenode的RPC服务器工作的端口号，默认8020
     hadoop.tmp.dir|目录路径|/tmp/hadoop-${user.name}|Hadoop的临时缓存目录，每次重启会清除，因此一般会自行设置
     io.file.buffer.size|int|4096|辅助I/O操作的缓冲区大小，以字节为单位，默认4KB

  2. hdfs-site.xml
   
     属性名称|类型|默认值|详细说明
     -|-|-|-
     dfs.namenode.name.dir|逗号分隔的目录名|file://${hadoop.tmp.dir}/dfs/name|namenode存储永久性元数据的目录列表。namenode在列表上的各个目录中均存放相同的元数据文件
     dfs.replication|int|3|数据在HDFS中保存的副本数量
     dfs.datanode.data.dir|逗号分隔的目录名|file://${hadoop.tmp.dir}/dfs/data|datanode存放数据块的目录列表
     dfs.namenode.checkpoint.dir|逗号分隔的目录名|file://${hadoop.tmp.dir}/dfs/namesecondary|辅助namenode存放检查点的目录列表在所列目录中均存放一份检查点文件的副本
     dfs.namenode.rpc-bind-host|URI||namenode的RPC服务器将绑定的地址。默认不设置。设置为0.0.0.0可以让namenode可以监听所有接口
     dfs.hosts|文件名||存放允许作为datanode连接namenode的主机列表的文件路径。默认不设置，即所有主机都可以连接namenode
     dfs.hosts.exclude|文件名||存放不允许作为datanode连接namenode的主机列表的文件路径。默认不设置，即没有主机不允许连接namenode
     dfs.blocksize|int|134217728|设置HDFS数据块大小，以字节为单位。默认128MB
     dfs.datanode.du.reserved|int|0|datanode能够存储数据的空间大小，以字节为单位。默认为0即机器上所有闲置空间
  
  1. yarn-site.xml

     属性名称|类型|默认值|详细说明
     -|-|-|-
     yarn.resourcemanager.hostname|主机名|0.0.0.0|运行资源管理器的机器主机名(或IP)，以下简称为${y.rm.hostname}
     yarn.resourcemanager.address|主机名和端口号|${y.rm.hostname}:8032|运行资源管理器的RPC服务器的主机名和端口
     yarn.nodemanager.local-dirs|逗号分隔的目录名|${hadoop.tmp.dir}/nm-local-dir|节点管理器存放容器运行过程中产生的中间数据的目录列表，应用结束时被清除
     yarn.nodemanager.aux-services|逗号分隔的目录名||节点管理器运行的附加服务列表，每项服务由属性yarn.nodemanager.auxservices.servicename.class所定义的类实现。默认不添加，一般设置为mapreduce_shuffle
     yarn.log-aggregation-enable|bull|False|是否启用任务运行日志聚合功能，将日志存放在HDFS系统上
     yarn.log-aggregation.retain-seconds|int|-1|任务运行日志聚集存放的时间，以秒为单位
     yarn.log.server.url|url||任务运行日志聚集存放的服务器地址，与mapred-site.xml的mapreduce.jobhistory.address一致，前面要加http
     yarn.nodemanager.resource.memory-mb|int|8192|节点管理器运行的容器可分配的物理内存(MB)
     yarn.nodemanager.vmem-pmem-ratio|float|2.1|容器所占的虚拟内存与物理内存之比
     yarn.nodemanager.resource.cpuvcores|int|8|节点管理器运行的容器可分配的CPU核数
     yarn.resourcemanager.nodes.include-path|文件名||存放允许作为节点管理器加入集群的机器列表的文件路径
     yarn.resourcemanager.nodes.exclude-path|文件名||存放集群中待解除作为节点管理器的机器列表的文件路径

  2. mapred-site.xml

     属性名称|类型|默认值|详细说明
     -|-|-|-
     mapreduce.framework.name|String|local|运行MapReduce任务的框架，默认为local(本地)，可选classic、yarn
     mapreduce.jobhistory.address|主机名和端口号|0.0.0.0:10020|历史服务器的RPC地址
     mapreduce.jobhistory.webapp.address|主机名和端口号|0.0.0.0:19888|历史服务器的web UI地址
     mapreduce.map.memory.mb|int|1024|map容器所用的内存容量
     mapreduce.reduce.memory.mb|int|1024|reduce容器所用的内存容量
     mapred.child.java.opts|String|-Xmx200m|JVM选项，用于启动运行map和reduce任务的容器进程
     mapreduce.map.java.opts|String|-Xmx200m|JVM选项，针对运行map任务的子进程
     mapreduce.reduce.java.opts|String|-Xmx200m|JVM选项，针对运行reduce任务的子进程
     mapreduce.map.cpu.vcores|int|1|分配给map任务的CPU核数
     mapreduce.reduce.cpu.vcores|int|1|分配给reduce任务的CPU核数

##### 3.Hadoop守护进程的地址和端口
* Hadoop的守护进程一般同时运行RPC和HTTP两个服务器，RPC服务器支持守护进程间的通信，HTTP服务器提供与用户交互的Web页面。需要分别为各个服务配置网络地址和端口号。端口号为0表示服务器会选一个空闲的端口号，一般不推荐
* 这些地质属性有两个责任
  1. 决定服务器将绑定的网络接口
  2. 客户端或集群中其他机器使用他们连接服务器
* RPC服务器的属性

  属性名称|配置文件|默认值|详细说明
  -|-|-|-
  fs.defaultFS|core-site.xml|file:///|默认文件系统。URI定义主机名称和namenode的RPC服务器工作的端口号，默认8020
  dfs.namenode.rpc-bind-host|hdfs-site.xml||namenode的RPC服务器将绑定的地址。默认不设置。设置为0.0.0.0可以让namenode可以监听所有接口
  dfs.datanode.ipc.address|hdfs-site.xml|0.0.0.0:50020|datanode的RPC服务器地址和端口
  yarn.resourcemanager.hostname|yarn-site.xml|0.0.0.0|运行资源管理器的机器主机名(或IP)，以下简称为${y.rm.hostname}
  yarn.resourcemanager.bind-host|yarn-site.xml||资源管理器的RPC和HTTP服务器绑定的地址
  yarn.resourcemanager.address|yarn-site.xml|${y.rm.hostname}:8032|资源管理器RPC服务器地址和端口。客户端(一般在集群外部)通过它与资源管理器通信
  yarn.resourcemanager.admin.address|yarn-site.xml|${y.rm.hostname}:8033|资源管理器admin RPC服务器地址和端口。admin客户端(由yarnrmadmin调用，一般在集群外部)通过它与资源管理器通信
  yarn.resourcemanager.scheduler.address|yarn-site.xml|${y.rm.hostname}:8030|资源管理器的调度器RPC服务器地址和端口。Application Master(在集群内部)通过它与资源管理器通信
  yarn.resourcemanager.resourcetracker.address|yarn-site.xml|${y.rm.hostname}:8031|资源管理器resource tracker的RPC服务器地址和端口。节点管理器(在集群内部)通过它与资源管理器通信
  yarn.nodemanager.hostname|yarn-site.xml|0.0.0.0|节点管理器运行所在的主机名，以下缩写为${y.nm.hostname}
  yarn.nodemanager.bind-host|yarn-site.xml||节点管理器RPC和HTTP服务器绑定的地址
  yarn.nodemanager.address|yarn-site.xml|${y.nm.hostname}:0|节点管理器RPC服务器地址和端口，Application Master(在集群内部)通过它与节点管理器通信
  yarn.nodemanager.localizer.address|yarn-site.xml|${y.nm.hostname}:8040|节点管理器的localizer的RPC服务器地址和端口
  mapreduce.jobhistory.address|mapred-site.xml|0.0.0.0:10020|MapReduce作业历史服务器的RPC地址和端口，客户端(一般在集群外部)用于查询作业历史
  mapreduce.jobhistory.bind-host|mapred-site.xml||MapReduce作业历史服务器的RPCheHTTP绑定的地址

* HTTP服务器的属性

  属性名称|配置文件|默认值|详细说明
  -|-|-|-
  dfs.namenode.http-address|hdfs-site.xml|0.0.0.0:50070|namenode的HTTP服务器地址和端口
  dfs.namenode.http-bind-host|hdfs-site.xml||namenode的HTTP服务器绑定的地址
  dfs.datanode.http.address|hdfs-site.xml|0.0.0.0:50075|datanode的HTTP服务器地址和端口
  yarn.resourcemanager.webapp.address|yarn-site.xml|${y.rm.hostname}:8088|资源管理器的HTTP服务器地址和端口
  yarn.nodemanager.webapp.address|yarn-site.xml|${y.nm.hostname}:8042|节点管理器的HTTP服务器地址和端口
  yarn.web-proxy.address|yarn-site.xml||web应用代理服务器的HTTP服务器地址和端口默认情况将在资源管理器进程中运行
  mapreduce.jobhistory.webapp.address|mapred-site.xml|0.0.0.0:19888|MapReduce作业历史服务器和端口
  mapreduce.shuffle.port|mapred-site.xml|13562|shuffle句柄的HTTP端口号。为map输出结果服务，但不是用户可访问的web UI

* TCP/IP服务器属性
  
  属性名称|配置文件|默认值|详细说明
  -|-|-|-
  dfs.datanode.address|hdfs-site.xml|0.0.0.0:50010|datanode的TCP/IP服务器地址和端口，用来支持datanode的块传输

##### 4. Hadoop安全性
* 用Kerberos实现用户认证，Hadoop不直接管理用户隐私，而Kerberos也不关心用户的授权细节
* Kerberos认证的三个步骤
  1. 认证。客户端向认证服务器发送一条报文，并获取一个含时间戳的票据授予票据(Ticket-Granting Ticket,TGT)
  2. 授权。he护短使用TGT向票据授予服务器(Ticket-Granting Server,TGS)，请求一个服务票据
  3. 服务请求。客户端向服务器初始服务票据，以证实自己的合法性。该服务器提供客户端所需服务，在Hadoop应用中服务器可以是namenode或资源管理器
  * 认证服务器和票据授予服务器构成了密钥分配中心(Key Distribution Center,KDC)
* 使用方法
  1. 在系统中安装、配置和运行一个KDC 
  2. 启用Kerberos认证：core-site.xml文件hadoop.security.authentication设置为kerberos
  3. 启用服务级别的授权：core-site.xml文件hadoop.security.authorization设置为true。
  4. 配置ACL：配置hadoop-policy.xml文件中的访问控制列表(ACL)以决定哪些用户和组能够访问那些Hadoop服务，默认情况下为*，即所有用户能访问所有服务。
     ```
     user1[,user2,user3...] group1[,group2,group3...]
     # ACL格式：前一段为逗号隔开的用户名列表，后一段为逗号隔开的组名列表，用空格来间隔
     ```
* 委托令牌：在Hadoop中客户端与服务器会频繁交互，如果每次都使用三步骤Kerberos认证会给KDC很大压力。因此使用委托令牌来支持后续的认证
  1. 委托令牌由服务器创建(Hadoop中为namenode)，客户端首次RPC访问namenode时没有委托令牌，需要进行Kerberos认证。
  2. 第一次认证后，客户端从namenode获取委托令牌，后续的RPC调用中只需要出示委托令牌即可认证客户端身份进行操作
  3. 客户端执行HDFS块操作时namenode会授予特殊的委托令牌称为“快访问令牌”(block access token),同时会将令牌发送给datanode使其也能验证
  4. 在MapReduce中各组件也使用委托令牌访问HDFS,作业结束后委托令牌失效