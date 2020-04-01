* Zookeeper作为Hadoop和Hbase的重要组件，可以为分布式应用程序协调服务，同时还能使用Java和C的接口
* Zookeeper特点：
  1. 简单：ZooKeeper核心是一个精简的文件系统
  2. 富有表现力：基本操作是一组丰富的构件(building block)，可用于实现多种协调数据结构和协议
  3. 高可用性：运行于一组机器之上，并且在设计上具有高可用性
  4. 松耦合交互方式：ZooKeeper支持的交互过程中参与者不需要彼此了解
  5. 资源库：ZooKeeper提供了一个通用协调模式实现方法的开源共享库
##### 1. 安装和运行ZooKeeper
* 单机模式
  * 安装包解压到指定文件夹，配置环境变量
  * 创建配置文件zoo.cfg放入conf/子目录，或者指定环境变量ZOOCFGDIR的目录
    * tickTime：ZooKeeper基本时间单位，以毫秒为单位
    * dataDir：ZooKeeper存储持久数据的本地文件系统位置，默认/tmp/zookeeper
    * clientPort：用于监听客户端连接的端口，默认2181
    * 4lw.commands.whitelist：ZooKeeper四字命令白名单，设置为*代表所有命令都为白名单
    * 启动ZooKeeper服务
    ```zkServer.sh start```
* 集群模式(至少三台服务器)
  * 安装包解压到每台服务器指定文件夹，配置环境变量
  * 在每台服务器上创建配置文件zoo.cfg放入conf/子目录，或者指定环境变量ZOOCFGDIR的目录
  * 在集群模式下，所有的zk进程可以使用相同的配置文件（各个zk进程部署在不同的机器上面），例如如下配置：
    ```
    tickTime=2000
    dataDir=/tmp/zookeeper
    clientPort=2181
    initLimit=5
    syncLimit=2
    server.1=192.168.229.160:2888:3888
    server.2=192.168.229.161:2888:3888
    server.3=192.168.229.162:2888:3888
    4lw.commands.whitelist=*
    ```
    * initLimit：ZooKeeper集群模式下包含多个zk进程，其中一个进程为leader，余下的进程为follower。当follower最初与leader建立连接时，它们之间会传输相当多的数据，尤其是follower的数据落后leader很多。这个配置项是用来配置 Zookeeper接受客户端初始化连接时最长能忍受多少个tickTime的时间长度
    * syncLimit：Leader 与 Follower之间发送消息，请求和应答时间长度，最长不能超过多少个tickTime的时间长度
    * tickTime：tickTime则是上述两个超时配置的基本单位，例如对于initLimit，其配置值为5，说明其超时时间为 2000ms * 5 = 10秒。一般称为心跳时间间隔
    * server.id=host:port1:port2：其中id为一个数字，表示zk进程的id，这个id也是dataDir目录下myid文件的内容。host是该zk进程所在的IP地址，port1表示follower和leader交换消息所使用的端口，port2表示选举leader所使用的端口
    * dataDir：其配置的含义跟单机模式下的含义类似，不同的是集群模式下还有一个myid文件，需要自行创建。myid文件的内容只有一行，且内容只能为1 - 255之间的数字，这个数字亦即上面介绍server.id中的id，表示zk进程的id
    * 4lw.commands.whitelist：ZooKeeper四字命令白名单，设置为*代表所有命令都为白名单
  * 启动ZooKeeper服务：在每台服务器上启动ZooKeeper服务
  ```zookeeper.sh start```
* 四字命令：ZooKeeper有很多用于管理的四字命令
  
  <table>
      <tr>
          <th>类别</th>
          <th>命令</th>
          <th>描述</th>
      </tr>
      <tr>
          <td rowspan=7>服务器状态</td>
          <td>ruok</td>
          <td>如果服务器正在运行并且未处于出错状态则输出imok</td>
      </tr>
      <tr>
          <td>conf</td>
          <td>输出服务器的配置信息zoo.cfg</td>
      </tr>
      <tr>
          <td>envi</td>
          <td>输出服务器环境信息，例如版本信息等</td>
      </tr>
      <tr>
          <td>srvr</td>
          <td>输出服务器的统计信息，包括延迟统计、znode数量、服务器运行模式等</td>
      </tr>
      <tr>
          <td>stat</td>
          <td>输出服务器的统计信息和已连接的客户端</td>
      </tr>
      <tr>
          <td>srst</td>
          <td>重置服务器的统计信息</td>
      </tr>
      <tr>
          <td>isro</td>
          <td>显示服务器是否处于只读(ro)模式或者读写(rw)模式</td>
      </tr>
      <tr>
          <td rowspan=3>客户端连接</td>
          <td>dump</td>
          <td>列出集合中所有会话和短暂znode，必须连接到leader才能使用此命令
      </tr>
      <tr>
          <td>cons</td>
          <td>列出所有服务器客户端的连接统计信息</td>
      </tr>
      <tr>
          <td>crst</td>
          <td>重置连接统计信息</td>
      </tr>
      <tr>
          <td rowspan=3>观察</td>
          <td>wchs</td>
          <td>列出服务器上所有观察的摘要信息</td>
      </tr>
      <tr>
          <td>wchc</td>
          <td>按连接列出服务器上所有的观察</td>
      </tr>
      <tr>
          <td>wchp</td>
          <td>按znode路径列出服务器上所有的观察</td>
      </tr>
      <tr>
          <td>监控</td>
          <td>mntr</td>
          <td>按Java属性格式列出服务器统计信息</td>
      </tr>
  </table>

  * 检查ZooKeeper是否在运行
    ```echo ruok | nc localhost 2181```
  * 也可以在 http://localhost:8080/commands 查看命令列表

##### 2. ZooKeeper服务
* ZooKeeper中组成员的关系
  * 理解ZooKeeper的一种方法是将其看作一个具有高可用性特征的文件系统。这个文件系统没有目录，而是统一使用节点的概念，称为znode。znode既可以保存数据(如同文件)，又可以保存其他znode(如同目录)。所有的znode构成了一个层次化的命名空间
* 数据模型：ZooKeeper维护着一个树状层次结构，节点称为znode，可以用来存储数据并有一个与之关联的ACL。ZooKeeper被设计用来实现协调服务而不是存储大容量数据，因此一个znode存储的数据被限制在1MB以内
  * znode数据访问具有原子性，要么读到数据要么读取失败，要么写入成功要么写入失败，没有部分读取或部分写入。不支持添加操作
  * znode通过路径被引用，但是只允许使用绝对路径，没有相对路径的用法
* znode类型：znode类型在创建时确定且之后不能再修改
  1. 短暂znode：创建短暂znode的客户端会话结束后短暂znode会被删除。不允许有子节点
  2. 持久znode：只有当客户端明确要删除时才会删除
* 顺序znode：名称中包含ZooKeeper指定顺序号的znode。如果在创建时设置了顺序标识，那么该znode名称之后会附加一个由单调递增的计数器(父节点维护)添加的值
* 观察：znode以某种方式发生变化时，观察(watch)机制可以让客户端得到通知。可以针对ZooKeeper服务的操作来设置观察，该服务的其他操作可以出发观察。观察只能被出发一次
* 操作：ZooKeeper 9种基本操作

  操作|说明
  -|-
  create|创建一个znode(必须要有父节点)
  delete|删除一个znode(该znode不能有任何子节点)
  exists|测试一个znode是否存在并查询它的元数据
  getACL,setACL|获取/设置一个znode的ACL
  getChildren|获取一个znode的子节点列表
  getData,setData|获取/设置一个znode所保存的数据
  sync|将客户端的znode视图与ZooKeeper同步

  1. 集合更新multi：用于将多个基本操作集合成一个操作单元，并确保这些基本操作同时被执行成功或者同时失败
  2. 关于API：主要有Java和C语言两种API，也可以使用Perl、Python和REST的contrib绑定API。对于每一种绑定语言在执行操作时都可以选择同步执行或异步执行
  3. 观察触发器：在exists、getChildren和getData这些读操作上可以设置观察，这些观察可以被写操作create、delete、setData触发。ACL相关操作不参与触发任何观察。当观察触发时会产生一个观察事件，这个观察和触发它的操作共同决定观察事件的类型
     * 当所观察的znode被创建、删除或其数据被更新时设置在exists操作上的观察被触发
     * 当所观察的znode被删除或其数据被更新时设置在getData操作上的观察将被触发
     * 所观察的znode一个字节点被创建或删除时，或所观察的znode自己被删除时，设置在getChildren操作上的观察会被触发

     操作|创建znode|创建子节点|删除znode|删除子节点|setData
     -|-|-|-|-|-
     exists|NodeCreated||NodeDeleted||NodeDataChanged
     getData|||NodeDeleted||NodeDataChanged
     getChildren||NodeChildrenChanged|NodeDeleted|NodeChildrenChanged|

  4. ACL列表
     * 每个znode创建时都会带有一个ACL列表，用于决定谁可以对它执行何种操作。ZooKeeper提供了以下几种身份验证方式
       1. digest：通过用户名和密码来识别客户端
       2. sasl：通过Kerberos来识别客户端
       3. ip：通过客户端的IP地址来识别客户端
     * exists操作不受ACL权限的限制，任何客户端都可以调用exists。ACL权限集合如下表

       ACL权限|允许的操作
       -|-
       CREATE|create(子节点)
       READ|getChildren,getData
       WRITE|setData
       DELETE|delete(子节点)
       ADMIN|setACL

* 运行方式
  * ZooKeeper服务有两种不同的运行模式
    1. 独立模式：只有一个ZooKeeper服务器，不保证高可用性和可恢复性，适用于测试环境
    2. 复制模式：运行于一个计算机集群上，被称为集合体。ZooKeeper通过复制实现高可用性，只要集群中超过半数的机器处于可用状态就能提供服务，因此一个集合体通常包含奇数台机器
  * 复制模式运行方式：Zab协议
    1. 阶段1：领导者选举。集合体中所有机器通过一个选择过程来选出一台被称为leader的机器，其他机器被称为follower。一旦半数以上的follower已经将其状态与leader同步，则这个阶段已完成
    2. 阶段2：原子广播。所有的写请求都会被转发给leader，再由leader将更新广播给follower。当半数以上的follower已经将修改持久化后，leader才会提交这个更新，然后客户端才会收到一个更新的响应。如果leader故障，其余机器会选出另外一个leader并和新leader一起继续提供服务。如果以前的leader故障恢复，会成为一个follower
* 一致性
  * 集合体中一个follower可能滞后于leader几个更新，理想情况是将客户端都连接到与leader一致的服务器上
  * 每一个对znode的更新都被赋予一个全局唯一的ID，称为zxid。ZooKeeper要求对所有更新进行编号并排序，决定了分布式系统的执行顺序
  * 以下几点保证数据一致性
    1. 顺序一致性：来自任意特定客户端的更新都会按其发送顺序被提交
    2. 原子性：每个更新要么成功要么失败
    3. 单一系统映像：一个客户端无论连接到哪一台服务器，它看到的都是同样的系统视图
    4. 持久性：一个更新一旦成功，其结果就会持久存在并且不会被撤销
    5. 及时性：任何客户端所看到的滞后系统视图都是有限的，不会超过几十秒
* 会话(session)
  * 每个ZooKeeper客户端配置中都有集合体中服务器的列表，启动时客户端会尝试连接到列表中一台服务器，若连接失败则会尝试连接另一台服务器，以此类推直到成功建立连接或因为所有ZooKeeper服务器都不可用而失败
  * 一旦客户端与一台ZooKeeper服务器建立连接，这台服务器就会为该客户端创建一个session。每个session都会有一个超时时间设置，由创建会话的应用设定。服务器在超时时间内没有收到任何请求则会话过期，过期后的会话无法重新打开，任何与该会话相关联的短暂znode会丢失
  * 只要一个会话空闲超过一定时间，都可以通过客户端发送ping请求（心跳请求）来保持会话不过期。这个请求由客户端自动发送
  * ZooKeeper客户端可以自动的进行故障切换，并且在另一台服务器接替故障服务器后所有的会话仍然有效
* 状态
  ![ZooKeeper状态转换](https://upload-images.jianshu.io/upload_images/4366140-ed4e22e6f51d6432.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
  * ZooKeeper对象在生命周期中会经历几种不同的状态(States)，States被定义成代表ZooKeeper对象不同状态的枚举值类型值。与ZooKeeper服务器建立连接的过程中为CONNECTING状态，一旦建立连接就会进入CONNECTED状态。如果close()方法被调用或出现会话超时，就会进入CLOSED状态
* 锁服务
  * 分布式锁能够在一组进程之间提供互斥机制，使得在任何时刻只有一个进程可以持有锁