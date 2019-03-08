---
title: docker部署Kafka
date: 2019-03-07 11:50:01
tags: docker kafka
---


## 1. docker命令部署

### 安装Zookeeper集群  

启动zookeeper容器(三台机器均执行) 

``` bash
$ docker pull zookeeper:3.4.9

\# 创建zookeerper
docker run -itd \
 --name=zookeeper \
 --restart=always \
 --net=host \
 --privileged=true \
 -v /data/kafka_cluster/zookeeper/data:/data \
 -v /data/kafka_cluster/zookeeper/datalog:/datalog \
 -e ZOO_MY_ID=1 \
 -e ZOO_SERVERS="server.1=10.29.132.21:2888:3888 server.2=10.29.132.22:2888:3888 server.3=10.29.132.23:2888:3888" \
 zookeeper:3.4.9

docker run -itd \
 --name=zookeeper \
 --restart=always \
 --net=host \
 --privileged=true \
 -v /data/kafka_cluster/zookeeper/data:/data \
 -v /data/kafka_cluster/zookeeper/datalog:/datalog \
 -e ZOO_MY_ID=2 \
 -e ZOO_SERVERS="server.1=10.29.132.21:2888:3888 server.2=10.29.132.22:2888:3888 server.3=10.29.132.23:2888:3888" \
 zookeeper:3.4.9

docker run -itd \
 --name=zookeeper \
 --restart=always \
 --net=host \
 --privileged=true \
 -v /data/kafka_cluster/zookeeper/data:/data \
 -v /data/kafka_cluster/zookeeper/datalog:/datalog \
 -e ZOO_MY_ID=3 \
 -e ZOO_SERVERS="server.1=10.29.132.21:2888:3888 server.2=10.29.132.22:2888:3888 server.3=10.29.132.23:2888:3888" \
 zookeeper:3.4.9
```



### 安装kafka集群

``` bash
$ #创建kafka
docker run -itd \
--name=kafka \
--restart=always \
--net=host \
--privileged=true \
-v /etc/hosts:/etc/hosts \
-v /data/kafka_cluster/kafka/data:/kafka \
-v /data/kafka_cluster/kafka/logs:/opt/kafka/logs \
-e KAFKA_ADVERTISED_HOST_NAME=10.29.132.21 \
-e KAFKA_ADVERTISED_PORT=9092 \
-e KAFKA_ZOOKEEPER_CONNECT=10.29.132.21:2181,10.29.132.22:2181,10.29.132.23:2181 \
-e KAFKA_BROKER_ID=0 \
-e KAFKA_HEAP_OPTS="-Xmx6G -Xms6G" \
wurstmeister/kafka

 

#创建kafka集群
docker run -itd \
--name=kafka \
--restart=always \
--net=host \
--privileged=true \
-v /etc/hosts:/etc/hosts \
-v /data/kafka_cluster/kafka/data:/kafka \
-v /data/kafka_cluster/kafka/logs:/opt/kafka/logs \
-e KAFKA_ADVERTISED_HOST_NAME=10.29.132.22 \
-e KAFKA_ADVERTISED_PORT=9092 \
-e KAFKA_ZOOKEEPER_CONNECT=10.29.132.21:2181,10.29.132.22:2181,10.29.132.23:2181 \
-e KAFKA_BROKER_ID=1 \
-e KAFKA_HEAP_OPTS="-Xmx6G -Xms6G" \
wurstmeister/kafka

docker run -itd \
--name=kafka2 \
--restart=always \
--net=host \
--privileged=true \
-v /etc/hosts:/etc/hosts \
-v /data/kafka_cluster/kafka/data:/kafka \
-v /data/kafka_cluster/kafka/logs:/opt/kafka/logs \
-v /data/kafka_cluster/kafka/config:/opt/kafka/config \
-e KAFKA_ADVERTISED_HOST_NAME=kafka2 \
-e KAFKA_ZOOKEEPER_CONNECT=kafka1:2181,kafka2:2181,kafka3:2181 \
-e KAFKA_BROKER_ID=1 \
-e KAFKA_HEAP_OPTS="-Xmx6G -Xms6G" \
wurstmeister/kafka

docker run -itd \
--name=kafka3 \
--restart=always \
--net=host \
--privileged=true \
-v /etc/hosts:/etc/hosts \
-v /data/kafka_cluster/kafka/data:/kafka \
-v /data/kafka_cluster/kafka/logs:/opt/kafka/logs \
-v /data/kafka_cluster/kafka/config:/opt/kafka/config \
-e KAFKA_ADVERTISED_HOST_NAME=10.29.132.23 \
-e KAFKA_ADVERTISED_PORT=9094 \
-e KAFKA_PORT=9094 \
-e KAFKA_ZOOKEEPER_CONNECT=10.29.132.21:2181,10.29.132.22:2181,10.29.132.23:2181 \
-e KAFKA_BROKER_ID=2 \
-e KAFKA_HEAP_OPTS="-Xmx6G -Xms6G" \
wurstmeister/kafka

docker run -itd \
--name=kafka3 \
--restart=always \
--net=host \
--privileged=true \
-v /etc/hosts:/etc/hosts \
-v /data/kafka_cluster/kafka/data:/kafka \
-v /data/kafka_cluster/kafka/logs:/opt/kafka/logs \
-v /data/kafka_cluster/kafka/config:/opt/kafka/config \
-e KAFKA_ADVERTISED_HOST_NAME=kafka3 \
-e KAFKA_ZOOKEEPER_CONNECT=kafka1:2181,kafka2:2181,kafka3:2181 \
-e KAFKA_BROKER_ID=2 \
-e KAFKA_HEAP_OPTS="-Xmx6G -Xms6G" \
wurstmeister/kafka
```

### Kafka集群验证

查看topic

``` bash
$ ./kafka-topics.sh --list --zookeeper localhost:2181
```

进入docker内部：

``` bash
$ docker exec -it kafka bash
cd /opt/kafka/
bin/kafka-topics.sh --zookeeper 10.29.182.149:2181 --list
bin/kafka-topics.sh --zookeeper 10.29.132.21:2181,10.29.132.22:2181,10.29.132.23:2181 --list

# 创建topic
./kafka-topics.sh --create --topic user_group_status --replication-factor 1 --partitions 1 --zookeeper 10.29.182.149:2181

# 重启kafka
bin/kafka-server-start.sh config/server.properties

# 接收consumer数据
bin/kafka-console-consumer.sh --bootstrap-server 10.29.132.22:9093 --topic Alarmsys_Signal --from-beginning

# 发送producer数据
bin/kafka-console-producer.sh --broker-list 10.29.182.149:9092 --topic user_group_status
```

More info: [Deployment](https://hexo.io/docs/deployment.html)
