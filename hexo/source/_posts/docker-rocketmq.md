---
title: docker部署RocketMQ
date: 2019-03-07 11:50:01
tags: docker rocketmq
---

通过docker快速部署RocketMQ



## 1. Quick Start

### 部署Server

```bash
$ docker run -d -p 9876:9876 --name rocketmq-server  foxiswho/rocketmq:server
```



### 部署Broker

```bash
$ docker run -d -p 10911:10911 -p 10909:10909 --name rocketmq-broker --link rocketmq-server:namesrv -e "NAMESRV_ADDR=namesrv:9876" -e "JAVA_OPTS=-Duser.home=/opt"  -e "JAVA_OPT_EXT=-server -Xms128m -Xmx128m -Xmn128m" foxiswho/rocketmq:broker
```



### 部署Console控制台

```bash
$ docker run -d -p 10911:10911 -p 10909:10909 --name rocketmq-broker --link rocketmq-server:namesrv -e "NAMESRV_ADDR=namesrv:9876" -e "JAVA_OPTS=-Duser.home=/opt"  -e "JAVA_OPT_EXT=-server -Xms128m -Xmx128m -Xmn128m" foxiswho/rocketmq:broker
```

配置文件在容器内/etc/rocketmq/broker.conf，可以通过volume将次配置文件映射出来，比如：

```bash
$ docker run -d -p 10911:10911 -p 10909:10909 --name rocketmq-broker --link rocketmq-server:namesrv \
-e "NAMESRV_ADDR=namesrv:9876" \
-e "JAVA_OPTS=-Duser.home=/opt"  \
-e "JAVA_OPT_EXT=-server -Xms128m -Xmx128m -Xmn128m" \
-v /data/rocketmq/conf/broker.conf:/etc/rocketmq/broker.conf \
foxiswho/rocketmq:broker
```

浏览器访问:  localhost:8180

More info:  [Deployment](https://hub.docker.com/r/foxiswho/rocketmq) 



## 2. 部署方案

RocketMQ集群由NameServer和Broker两种角色组成，NameServer是无状态的可以横向部署多台达到消除单点的目的；Broker分多master、多master多slave同步、多master多slave异步这三种部署方案，一般生产环境都使用的是多master多slave异步这种方案，关于这三种方案的优缺点对比如下：

### 2.1 多Master模式（2m-noslave）

一个集群无Slave，全是Master，例如2个Master或者3个Master

优点：配置简单，单个Master宕机或重启维护对应用无影响，在磁盘配置为RAID10时，即使机器宕机不可恢复情况下，由于RAID10磁盘非常可靠，消息也不会丢（异步刷盘丢失少量消息，同步刷盘一条不丢）。性能最高。

缺点：单台机器宕机期间，这台机器上未被消费的消息在机器恢复之前不可订阅，消息实时性会受到受到影响。

### 2.2 多Master多Slave模式，异步复制（2m-2s-async）

每个Master配置一个Slave，有多对Master-Slave，HA采用异步复制方式，主备有短暂消息延迟，毫秒级。

优点：即使磁盘损坏，消息丢失的非常少，且消息实时性不会受影响，因为Master宕机后，消费者仍然可以从Slave消费，此过程对应用透明。不需要人工干预。性能同多Master模式几乎一样。

缺点：Master宕机，磁盘损坏情况，会丢失少量消息。

### 2.3 多Master多Slave模式，同步双写（2m-2s-sync）

每个Master配置一个Slave，有多对Master-Slave，HA采用同步双写方式，主备都写成功，向应用返回成功。

优点：数据与服务都无单点，Master宕机情况下，消息无延迟，服务可用性与数据可用性都非常高

缺点：性能比异步复制模式略低，大约低10%左右，发送单个消息的RT会略高。目前主宕机后，备机不能自动切换为主机，后续会支持自动切换功能。

| 部署方式                      | 优点                                                         | 缺点                                                         | 备注                                                         |
| ----------------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| 单个Master模式                | 一旦Broker重启或者宕机时，会导致整个服务不可用，不建议线上环境使用； |                                                              |                                                              |
| 多个Master模式                | 配置简单，单个Master宕机或重启维护对应用无影响，在磁盘配置为RAID10时，即使机器宕机不可恢复情况下，由于RAID10磁盘非常可靠，消息也不会丢（异步刷盘丢失少量消息，同步刷盘一条不丢），性能最高。 | 单台机器宕机期间，这台机器上未被消费的消息在机器恢复之前不可订阅，消息实时性会收到影响。 | 当使用多master无slave的集群搭建方式时，master的brokerRole配置必须为ASYNC_MASTER。如果配置为SYNC_MASTER，则producer发送消息时，返回值的SendStatus会一直是SLAVE_NOT_AVAILABLE。 |
| 多Master多Slave模式——异步复制 | 即使磁盘损坏，消息丢失的非常少，但消息实时性不会受影响，因为Master宕机后，消费者仍然可以从Slave消费，此过程对应用透明，不需要人工干预，性能同多Master模式几乎一样。 | Master宕机，磁盘损坏情况，会丢失少量信息。                   |                                                              |
| 多Master多Slave模式——同步双写 | 数据与服务都无单点，Master宕机情况下，消息无延迟，服务可用性与数据可用性都非常高； | 性能比异步复制模式稍低，大约低10%左右，发送单个消息的RT会稍高，目前主宕机后，备机不能自动切换为主机，后续会支持自动切换功能。 |                                                              |

## 3. 常用命令

### 3.1 shmqadmin

RocketMQ提供有控制台及一系列控制台命令，用于管理员对主题，集群，broker等信息的管理；

首先进入RocketMQ工程，进入/RocketMQ/bin，在该目录下有个mqadmin脚本。

查看帮助，在mqadmin下可以查看有哪些命令：

```
shmqadmin
```



查看具体命令的使用:

```
sh mqadmin help 命令名称
```



```
## 例如，查看updateTopic的使用
sh mqadmin helpupdateTopic
```

### 3.2 创建Topic

详见：<https://blog.csdn.net/zhu_tianwei/article/details/40951301>



## 5. FAQ

### 5.1 broker启动报错`/runbroker.sh: line 62: 126674 Killed $JAVA ${JAVA_OPT} $@`

runbroker.sh 默认参数：Xms=8G，机器可用内存无法支配时会产生该问题，全部改小即可解决。

### 5.2 `com.alibaba.rocketmq.client.exception.MQClientException: No route info of this topic, TopicTest1`

启动broker时，指定autoCreateTopicEnable=true，建议线上关闭

```
nohup sh mqbroker -n localhost:9876 autoCreateTopicEnable=true > /dev/null 2>&1 &
```



或在配置文件中设置

```
nohup sh mqbroker -c $ROCKETMQ_HOME/conf/broker.conf > /dev/null 2>&1 &
```



```
brokerClusterName = DefaultCluster
brokerName = broker-a
brokerId = 0
brokerIP1 = 10.29.181.16
deleteWhen = 04
fileReservedTime = 48
brokerRole = ASYNC_MASTER
flushDiskType = ASYNC_FLUSH

# NameServer地址列表，多个nameServer地址用分号隔开
namesrvAddr = 10.29.181.16:9876

autoCreateTopicEnable = true
```

### 5.3 `org.apache.rocketmq.remoting.exception.RemotingConnectException: connect to <172.19.0.1:10909> failed`

10.29.181.15服务器上有多个网卡，172.19.0.1 为docker0的虚拟网卡的IP，但是启动RocketMQ时并没有设置这个IP地址。

```
 Docker 服务启动后默认会创建一个 docker0 网桥（其上有一个 docker0 内部接口），它在内核层连通了其他的物理或虚拟网卡，这就将所有容器和本地主机都放到同一个物理网络。

　　Docker 默认指定了 docker0 接口 的 IP 地址和子网掩码，让主机和容器之间可以通过网桥相互通信，它还给出了 MTU（接口允许接收的最大传输单元），通常是 1500 Bytes，或宿主主机网络路由上支持的默认值。这些值都可以在服务启动的时候进行配置。

可以用编辑/etc/docker/daemon.json文件，添加内容 "bip": "ip/netmask" [ 切勿与宿主机同网段 ]
```

[RocketMQ 官方broker配置](http://rocketmq.apache.org/docs/rmq-deployment/)：

| 参数名    | 默认值 | 说明                                                         |
| --------- | ------ | ------------------------------------------------------------ |
| brokerIP1 | 本机IP | 本机ip地址，默认系统自动识别，但是某些多网卡机器会存在识别错误的情况，这种情况下可以人工配置 |

解决办法：

1. 把不需要的网卡全部禁用，只留下有效的网卡，这样Broker不会生成奇怪的代理地址。
2. 在`broker.properties`中设置`brokerIP1=10.29.181.16`

### 5.4 `No route info of this topic`

产生`No route info of this topic`异常原因可能有如下几种：

- 1> Broker禁止自动创建Topic，且用户没有通过手工方式创建Topic。

启动borker时指定`autoCreateTopicEnable=true`

- 2> Broker未连接到Name Server

查看broker的启动日志：

```
2018-09-13 16:21:35 INFO BrokerControllerScheduledThread1 - register broker to name server 10.29.181.16:9876 OK
2018-09-13 16:22:05 INFO BrokerControllerScheduledThread1 - register broker to name server 10.29.181.16:9876 OK
```



说明broker已经成功连接到Name Server.

或执行`sh mqadmin clusterList -n localhost:9876`

```
#Cluster Name     #Broker Name            #BID  #Addr                  #Version                #InTPS(LOAD)       #OutTPS(LOAD) #PCWait(ms) #Hour #SPACE
DefaultCluster    DEFAULT_BROKER          0     10.29.181.16:10911  V4_2_0_SNAPSHOT          0.00(0,0ms)         0.00(0,0ms)          0 422168.55 -1.0000
```

- 3> Producer未连接到Name Server

关闭防火墙：`systemctl stop firewalld.service`，再做重试。

### 5.5 `The producer service state not OK, CREATE_JUST`

检查是否调用：`producer.start();`



## 6 术语

### 6.1 Disk Flush（磁盘刷新/同步操作）

将内存的数据落地，存储在磁盘中，RocketMQ提供了以下两种模式：

- SYNC_FLUSH（同步刷盘）：生产者发送的每一条消息都在保存到磁盘成功后才返回告诉生产者成功。这种方式不会存在消息丢失的问题，但是有很大的磁盘IO开销，性能有一定影响。
- ASYNC_FLUSH（异步刷盘）：生产者发送的每一条消息并不是立即保存到磁盘，而是暂时缓存起来，然后就返回生产者成功。随后再异步的将缓存数据保存到磁盘，有两种情况：1是定期将缓存中更新的数据进行刷盘，2是当缓存中更新的数据条数达到某一设定值后进行刷盘。这种方式会存在消息丢失（在还未来得及同步到磁盘的时候宕机），但是性能很好。默认是这种模式。

### 6.2 Broker Replication（Broker间数据同步/复制）

集群环境下需要部署多个Broker，Broker分为两种角色：一种是master，即可以写也可以读，其brokerId=0，只能有一个；另外一种是slave，只允许读，其brokerId为非0。一个master与多个slave通过指定相同的brokerName被归为一个broker set（broker集）。通常生产环境中，我们至少需要2个broker set。Broker Replication只的就是slave获取或者是复制master的数据。

- Sync Broker：生产者发送的每一条消息都至少同步复制到一个slave后才返回告诉生产者成功，即“同步双写”。
- Async Broker：生产者发送的每一条消息只要写入master就返回告诉生产者成功。然后再“异步复制”到slave。

推荐的几种Broker集群方式：（官网提供了下面几种集群方式的配置文件供参考，在$ROCKETMQ_HOME/target/apache-rocketmq-all/conf目录下）

- 2m-2s-sync：两主两从同步双写（两个master，两个slave，数据同步双写到master和slave）
- 2m-2s-async：两主两从异步复制（两个master，两个slave，master数据通过异步复制到slave）
- 2m-noslave：两主（只有两个master，没有slave）

> 注意：
> 1、上述“2”只是说作为一个集群的最低配置数量，可以根据实际情况扩展。
> 2、所有的刷盘（Dish Flush）操作全部默认为：ASYNC_FLUSH（异步刷盘）。

Name Server集群：Name Server集群比较简单，只要部署多个实例就行了，多个实例间不需要进行数据共享，只要保证一个实例存活就可以正常运转。

### 6.3 核心概念

- 生产者（Producer）：消息发送方，将业务系统中产生的消息发送到brokers（brokers可以理解为消息代理，生产者和消费者之间是通过brokers进行消息的通信），rocketmq提供了以下消息发送方式：同步、异步、单向。
- 生产者组（Producer Group）：相同角色的生产者被归为同一组，比如通常情况下一个服务会部署多个实例，这多个实例就是一个组，生产者分组的作用只体现在消息回查的时候，即如果一个生产者组中的一个生产者实例发送一个事务消息到broker后挂掉了，那么broker会回查此实例所在组的其他实例，从而进行消息的提交或回滚操作。
- 消费者（Consumer）：消息消费方，从brokers拉取消息。站在用户的角度，有以下两种消费者。
  - 主动消费者（PullConsumer）：从brokers拉取消息并消费。
  - 被动消费者（PushConsumer）：内部也是通过pull方式获取消息，只是进行了扩展和封装，并给用户预留了一个回调接口去实现，当消息到底的时候会执行用户自定义的回调接口。
- 消费者组（Consumer Group）：和生产者组类似。其作用体现在实现消费者的负载均衡和容错，有了消费者组变得异常容易。需要注意的是：同一个消费者组的每个消费者实例订阅的主题必须相同。
- 主题（Topic）：主题就是消息传递的类型。一个生产者实例可以发送消息到多个主题，多个生产者实例也可以发送消息到同一个主题。同样的，对于消费者端来说，一个消费者组可以订阅多个主题的消息，一个主题的消息也可以被多个消费者组订阅。
- 消息（Message）：消息就像是你传递信息的信封。每个消息必须指定一个主题，就好比每个信封上都必须写明收件人。
- 消息队列（Message Queues）：在主题内部，逻辑划分了多个子主题，每个子主题被称为消息队列。这个概念在实现最大并发数、故障切换等功能上有巨大的作用。
- 标签（Tag）：标签，可以被认为是子主题。通常用于区分同一个主题下的不同作用或者说不同业务的消息。同时也是避免主题定义过多引起性能问题，通常情况下一个生产者组只向一个主题发送消息，其中不同业务的消息通过标签或者说子主题来区分。
- 消息代理（Broker）：消息代理是RockerMQ中很重要的角色。它接收生产者发送的消息，进行消息存储，为消费者拉取消息服务。它还存储消息消耗相关的元数据，包括消费群体，消费进度偏移和主题/队列信息。
- 命名服务（Name Server）：命名服务作为路由信息提供程序。生产者/消费者进行主题查找、消息代理查找、读取/写入消息都需要通过命名服务获取路由信息。
- 消息顺序（Message Order）：当我们使用DefaultMQPushConsumer时，我们可以选择使用“orderly”还是“concurrently”。
  - orderly：消费消息的有序化意味着消息被生产者按照每个消息队列发送的顺序消费。如果您正在处理全局顺序为强制的场景，请确保您使用的主题只有一个消息队列。注意：如果指定了消费顺序，则消息消费的最大并发性是消费组订阅的消息队列数。
  - concurrently：当同时消费时，消息消费的最大并发仅限于为每个消费客户端指定的线程池。注意：此模式不再保证消息顺序。

