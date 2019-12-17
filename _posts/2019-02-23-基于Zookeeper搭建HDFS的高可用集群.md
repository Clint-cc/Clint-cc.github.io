---
layout: post
title: "基于Zookeeper搭建HDFS的高可用集群"
date: 2019-02-23
description: ""
tag: 大数据
---  
## 文章目录
<nav>
    <a href="#一高可用简介">一、高可用简介</a><br/>
    <a href="#二HA工作原理">二、HA工作原理</a><br/>
    <a href="#三高可用整体架构">三、高可用整体架构</a><br/>
    <a href="#四环境准备">四、环境准备</a><br/>
    <a href="#五HA集群规划">五、HA集群规划</a><br/>
    <a href="#六配置Zookeeper集群">六、配置Zookeeper集群</a><br/>
    <a href="#七配置HDFS-HA集群和YARN-HA集群">七、配置HDFS-HA集群和YARN-HA集群</a><br/>
    <a href="#八启动ha集群">八、启动ha集群</a><br/>
    <a href="#九查看集群">九、查看集群</a><br/>
</nav>

### 一、高可用简介
+ HA（High Available），即高可用,其实就是7*24小时不中断服务；
+ 实现高可用最关键的策略是消除单点故障。HA严格来说应该分成各个组件的HA机制：HDFS高可用和YARN高可用；
+ Hadoop2.0之前，在HDFS集群中NameNode存在单点故障（SPOF）；
+ NameNode主要在以下两个方面影响 HDFS 集群:
    + NameNode 机器发生意外，如宕机，集群将无法使用，直到管理员重启；
    + NameNode 机器需要升级，包括软件、硬件升级，此时集群也将无法使用；

### 二、HA工作原理
##### 2-1 HDFS-HA
简单来说，HA主要是通过NameNode消除单点故障，HDFS-HA功能通过配置Active/Standby两个NameNodes实现在集群中对NameNode的热备来解决上述问题。如果出现故障，如机器崩溃或机器需要升级维护，这时可通过此种方式将NameNode很快的切换到另外一台机器。

##### 2-2 基于QJM的共享存储系统的数据同步机制
目前Hadoop支持使用Quorum Journal Manager(QJM)或Network File System(NFS)作为共享的存储系统，这里以QJM集群为例进行说明：Active NameNode首先把EditLog提交到JournalNode集群，然后Standby NameNode再从JournalNode集群定时同步EditLog，当Active NameNode宕机后，Standby NameNode在确认元数据完全同步之后就可以对外提供服务。

需要说明的是向JournalNode集群写入EditLog是遵循 “过半写入则成功” 的策略，所以你至少要有3个JournalNode节点，当然你也可以继续增加节点数量，但是应该保证节点总数是奇数。同时如果有 2N+1 台 JournalNode，那么根据过半写的原则，最多可以容忍有N台JournalNode节点挂掉。

##### 2-3 YARN-HA
YARN ResourceManager的高可用与HDFS NameNode的高可用类似，但是ResourceManager不像NameNode ，没有那么多的元数据信息需要维护，所以它的状态信息可以直接写到Zookeeper上，并依赖Zookeeper来进行主备选举。

### 三、高可用整体架构
##### 3-1、HDFS高可用架构如下：
![clint](/images/posts/hadoop/ha.jpg)

图片引用自：[https://www.edureka.co/blog/how-to-set-up-hadoop-cluster-with-hdfs-high-availability/](https://www.edureka.co/blog/how-to-set-up-hadoop-cluster-with-hdfs-high-availability/)

##### 3-2、HDFS高可用架构主要由以下组件所构成：
+ **ActiveNameNode和StandbyNameNode**：两台 NameNode 形成互备，一台处于 Active 状态，为主 NameNode，另外一台处于 Standby 状态，为备 NameNode，只有主 NameNode 才能对外提供读写服务。

+ **主备切换控制器 ZKFailoverController**：ZKFailoverController 作为独立的进程运行，对 NameNode 的主备切换进行总体控制。ZKFailoverController 能及时检测到 NameNode 的健康状况，在主 NameNode 故障时借助 Zookeeper 实现自动的主备选举和切换，当然 NameNode 目前也支持不依赖于 Zookeeper 的手动主备切换。

+ **Zookeeper 集群**：为主备切换控制器提供主备选举支持。

+ **共享存储系统**：共享存储系统是实现 NameNode 的高可用最为关键的部分，共享存储系统保存了 NameNode 在运行过程中所产生的 HDFS 的元数据。主 NameNode 和 NameNode 通过共享存储系统实现元数据同步。在进行主备切换的时候，新的主 NameNode 在确认元数据完全同步之后才能继续对外提供服务。

+ **DataNode 节点**：除了通过共享存储系统共享 HDFS 的元数据信息之外，主 NameNode 和备 NameNode 还需要共享 HDFS 的数据块和 DataNode 之间的映射关系。DataNode 会同时向主 NameNode 和备 NameNode 上报数据块的位置信息。

### 四、环境准备
##### 参考我之前的博客
> **[Hadoop完全分布式运行模式](https://clint-cc.github.io/2019/02/Hadoop%E9%9B%86%E7%BE%A4%E5%AE%8C%E5%85%A8%E8%BF%90%E8%A1%8C%E6%A8%A1%E5%BC%8F/)**

### 五、HA集群规划
hadoop101|hadoop102|hadoop103
:-:|:-:|:-:
主NameNode|备NameNode|
JournalNode|JournalNode|JournalNode|
DataNode|DataNode|DataNode|
备ResouceManager|主ResouceManager|
NodeManager|NodeManager|NodeManager|
Zookeeper|Zookeeper|Zookeeper|

### 六、配置Zookeeper集群
##### 6-1、在hadoop101、hadoop102、hadoop103上部署Zookeeper
+ 下载并解压Zookeepr，官网下载地址：[https://archive.apache.org/dist/zookeeper/zookeeper-3.4.14/](https://archive.apache.org/dist/zookeeper/zookeeper-3.4.14/)
```shell
[root@hadoop101 software]# tar -zxvf zookeeper-3.4.14.tar.gz -C /opt/module/
```

##### 6-2、在安装目录/opt/module/zookeeper-3.4.14中创建zkData文件夹
+ 在zkData下创建myid文件，在文件添加与server对应的编号：如 1
+ 用xsync脚本分发到每一个服务器上，分别修改hadoop102、hadoop103上的myid为2,3
```shell
[root@hadoop101 zookeeper-3.4.14]# mkdir zkData
```

##### 6-3、配置zoo.cfg文件
```shell
[root@hadoop101 conf]# mv zoo_sample.cfg zoo.cfg
# 增加如下配置：
#######################cluster##########################
server.1=hadoop101:2888:3888
server.2=hadoop102:2888:3888
server.3=hadoop103:2888:3888
```
+ 配置参数 Server.A=B:C:D 解读
    + A是一个数字，表示这个是第几号服务器；
    + B是这个服务器的IP地址；
    + C是这个服务器与集群中的Leader服务器交换信息的端口；
    + D是万一集群中的Leader服务器挂了，需要一个端口来重新进行选举，选出一个新的Leader，而这个端口就是用来执行选举时服务器相互通信的端口。
+ 集群模式下配置一个文件myid，这个文件在dataDir目录下，这个文件里面有一个数据 就是A的值，Zookeeper启动时读取此文件，拿到里面的数据与zoo.cfg里面的配置信息比较从而判断到底是哪个server。

##### 6-4、分别启动zookeeper,查看状态
分别在三台机器上执行如下命令启动zk集群：
```shell
[root@hadoop101 zookeeper-3.4.14]# bin/zkServer.sh start
```
在三台机器上执行如下命令查看zk集群的状态：
```shell
[root@hadoop101 zookeeper-3.4.14]# bin/zkServer.sh status
```

### 七、配置HDFS-HA集群和YARN-HA集群
##### 7-1、在opt目录下创建ha文件夹，将hadoop-2.7.7全部拷贝到此文件夹（配置完一起分发）
```shell
[root@hadoop102 opt]# cp -r hadoop-2.7.7/ /opt/ha/
```

##### 7-2、配置core-site.xml
```xml
<configuration>
    <!-- 把两个 NameNode）的地址组装成一个集群 mycluster -->
    <property>
        <name>fs.defaultFS</name>
        <value>hdfs://mycluster</value>
    </property>

    <!-- 指定 hadoop 运行时产生文件的存储目录 -->
    <property>
        <name>hadoop.tmp.dir</name>
        <value>/opt/ha/hadoop-2.7.7/data/tmp</value>
        </property>

    <!-- ZooKeeper 集群的地址 -->
    <property>
        <name>ha.zookeeper.quorum</name>
        <value>hadoop101:2181,hadoop102:2181,hadoop103:2181</value>
    </property>

</configuration>
```

##### 7-3、配置hdfs-site.xml
```xml
<configuration>
    <!-- 完全分布式集群名称 -->
    <property>
        <name>dfs.nameservices</name>
        <value>mycluster</value>
    </property>

    <!-- 集群中 NameNode 节点都有哪些 -->
    <property>
        <name>dfs.ha.namenodes.mycluster</name>
        <value>nn1,nn2</value>
    </property>

    <!-- nn1 的 RPC 通信地址 -->
    <property>
        <name>dfs.namenode.rpc-address.mycluster.nn1</name>
        <value>hadoop101:9000</value>
    </property>

    <!-- nn2 的 RPC 通信地址 -->
    <property>
        <name>dfs.namenode.rpc-address.mycluster.nn2</name>
        <value>hadoop102:9000</value>
    </property>

    <!-- nn1 的 http 通信地址 -->
    <property>
        <name>dfs.namenode.http-address.mycluster.nn1</name>
        <value>hadoop101:50070</value>
    </property>

    <!-- nn2 的 http 通信地址 -->
    <property>
        <name>dfs.namenode.http-address.mycluster.nn2</name>
        <value>hadoop102:50070</value>
    </property>

    <!-- 指定 NameNode 元数据在 JournalNode 上的存放位置 -->
    <property>
        <name>dfs.namenode.shared.edits.dir</name>
        <value>qjournal://hadoop101:8485;hadoop102:8485;hadoop103:8485/mycluster</value>
    </property>

    <!-- 配置隔离机制，即同一时刻只能有一台服务器对外响应 -->
    <property>
        <name>dfs.ha.fencing.methods</name>
        <value>sshfence</value>
    </property>

    <!-- 使用隔离机制时需要 ssh 无秘钥登录 -->
    <property>
        <name>dfs.ha.fencing.ssh.private-key-files</name>
        <value>/home/root/.ssh/id_rsa</value>
    </property>

    <!-- 声明 journalnode 服务器存储目录 -->
    <property>
        <name>dfs.journalnode.edits.dir</name>
        <value>/opt/ha/hadoop-2.7.7/data/jn</value>
    </property>

    <!-- 关闭权限检查 -->
    <property>
        <name>dfs.permissions.enable</name>
        <value>false</value>
    </property>

    <!-- 访问代理类：client，mycluster，active 配置失败自动切换实现方式 -->
    <property>
        <name>dfs.client.failover.proxy.provider.mycluster</name>
        <value>org.apache.hadoop.hdfs.server.namenode.ha.ConfiguredFailoverProxyProvider</value>
    </property>

    <!-- 开启故障自动转移 -->
    <property>
        <name>dfs.ha.automatic-failover.enabled</name>
        <value>true</value>
    </property>

</configuration>
```
##### 7-4、配置yarn-site.xml
```xml
<configuration>
    <!--配置NodeManager上运行的附属服务。需要配置成mapreduce_shuffle后才可以在Yarn上运行MapReduce程序 -->
    <property>
        <name>yarn.nodemanager.aux-services</name>
        <value>mapreduce_shuffle</value>
    </property>

    <!--启用 RM ha -->
    <property>
        <name>yarn.resourcemanager.ha.enabled</name>
        <value>true</value>
    </property>

    <!--声明两台 resourcemanager 的地址 -->
    <property>
        <name>yarn.resourcemanager.cluster-id</name>
        <value>cluster-yarn1</value>
    </property>

    <!-- RM 的逻辑 ID 列表 -->
    <property>
        <name>yarn.resourcemanager.ha.rm-ids</name>
        <value>rm1,rm2</value>
    </property>

    <!-- RM1 的服务地址 -->
    <property>
        <name>yarn.resourcemanager.hostname.rm1</name>
        <value>hadoop101</value>
    </property>

    <!-- RM2 的服务地址 -->
    <property>
        <name>yarn.resourcemanager.hostname.rm2</name>
        <value>hadoop102</value>
    </property>

    <!-- 指定 zookeeper 集群的地址 -->
    <property>
        <name>yarn.resourcemanager.zk-address</name>
        <value>hadoop101:2181,hadoop102:2181,hadoop103:2181</value>
    </property>

    <!-- 启用自动恢复 -->
    <property>
        <name>yarn.resourcemanager.recovery.enabled</name>
        <value>true</value>
    </property>

    <!-- 指定 resourcemanager 的状态信息存储在 zookeeper 集群 -->
    <property>
        <name>yarn.resourcemanager.store.class</name>
        <value>org.apache.hadoop.yarn.server.resourcemanager.recovery.ZKRMStateStore</value>
    </property>
</configuration>
```

##### 7-5、将配置好的配置文件分发到其他机器上
+ 进入/opt/ha/hadoop-2.7.7/etc目录下，将其hadoop文件目录下的配置文件全部分发到其他机器。
```shell
[root@hadoop102 etc]# xsync hadoop
```

### 八、启动ha集群
##### 8-1、每个节点上启动zookeeper（如果之前启动没关就不再重新启动）
```shell
[root@hadoop101 zookeeper-3.4.14]# bin/zkServer.sh start
[root@hadoop102 zookeeper-3.4.14]# bin/zkServer.sh start
[root@hadoop103 zookeeper-3.4.14]# bin/zkServer.sh start
```
+ 查看状态：

```shell
[root@hadoop101 zookeeper-3.4.14]# bin/zkServer.sh status
[root@hadoop102 zookeeper-3.4.14]# bin/zkServer.sh status
[root@hadoop103 zookeeper-3.4.14]# bin/zkServer.sh status
```
![clint](/images/posts/hadoop/zkjq.jpg)

##### 8-2、在每台机器上启动journalnode节点（一定是先启动这一步，因为元数据不在本地存的）
+ 分别到三台服务器的/opt/ha/hadoop-2.7.7目录下启动journalnode进程；

```shell
[root@hadoop101 hadoop-2.7.7]# sbin/hadoop-daemon.sh start journalnode
[root@hadoop102 hadoop-2.7.7]# sbin/hadoop-daemon.sh start journalnode
[root@hadoop103 hadoop-2.7.7]# sbin/hadoop-daemon.sh start journalnode
```

##### 8-3、在hadoop101上初始化NameNode
```shell
[root@hadoop101 hadoop-2.7.7]# bin/hdfs namenode -format
```

##### 8-4、在hadoop102上同步nn1的元数据
```shell
[root@hadoop102 hadoop-2.7.7]# bin/hdfs namenode -bootstrapStandby
```

##### 8-5、在任意一台NameNode上初始化ZooKeeper中的HA状态
```shell
[root@hadoop101 hadoop-2.7.7]# bin/hdfs zkfc -formatZK
```

##### 8-6、在hadoop101启动HDFS
```shell
[root@hadoop101 hadoop-2.7.7]# sbin/start-dfs.sh
```

##### 8-7、在hadoop102上启动YARN
```shell
[root@hadoop102 hadoop-2.7.7]# sbin/start-yarn.sh
```

### 九、查看集群
##### 9-1、查看进程
+ 成功启动后，每台服务器上的进程如下：
![clint](/images/posts/hadoop/h1jps.jpg)
![clint](/images/posts/hadoop/h2jps.jpg)
![clint](/images/posts/hadoop/h3jps.jpg)

##### 9-2、查看Web UI
+ HDFS 和 YARN 的端口号分别为 50070 和 8088，界面如下：
    + hadoop101上的NameNode处于可用状态：  
![clint](/images/posts/hadoop/ha1.jpg)
    + hadoop102上的NameNode处于备用状态：
![clint](/images/posts/hadoop/ha2.jpg)
    + hadoop102上的ResourceManager处于可用状态：
![clint](/images/posts/hadoop/yarn-ha1.jpg)
    + 当我把hadoop102的RM kill掉的时候hadoop101上的ResourceManager处于可用状态：
![clint](/images/posts/hadoop/yarn-ha2.png)
