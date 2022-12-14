## 一、环境准备
### 1.1 主机环境
| 主机      | IP             | 服务          |
| --------- | -------------- | ------------- |
| 主机test1 | 172.24.211.113 | Kafka_kraft_1 |
| 主机test2 | 172.24.211.114 | Kafka_kraft_2 |
开放端口：
```bash
iptables -I INPUT -p tcp --dport 9092 -j ACCEPT
iptables -I INPUT -p tcp --dport 9093 -j ACCEPT
```
### 1.2 Docker环境
安装docker：
```bash
yum install docker
```
安装docker-compose：
```bash
yum install docker-compose
```
启动docker：
```bash
systemctl start docker
```

## 二、容器编排
### 2.1 主机一
docker-compose.yaml（172.24.211.113）
```yaml
version: "3"
services:
   kafka:
     image: 'bitnami/kafka:latest'
     container_name: kafka_kraft
     restart: always
     user: root
     ports:
       - '9092:9092'
     environment:
       - KAFKA_ENABLE_KRAFT=yes
       - KAFKA_CFG_PROCESS_ROLES=broker,controller
       - KAFKA_CFG_CONTROLLER_LISTENER_NAMES=CONTROLLER
       - KAFKA_CFG_LISTENERS=PLAINTEXT://:9092,CONTROLLER://:9093
       - KAFKA_CFG_LISTENER_SECURITY_PROTOCOL_MAP=CONTROLLER:PLAINTEXT,PLAINTEXT:PLAINTEXT
       - KAFKA_CFG_ADVERTISED_LISTENERS=PLAINTEXT://172.24.211.113:9092
       - KAFKA_BROKER_ID=1
       - KAFKA_KRAFT_CLUSTER_ID=LelM2dIFQkiUFvXCEcqRWA
       - KAFKA_CFG_CONTROLLER_QUORUM_VOTERS=1@172.24.211.113:9093,2@172.24.211.114:9093
       - ALLOW_PLAINTEXT_LISTENER=yes
       - KAFKA_CFG_AUTO_CREATE_TOPICS_ENABLE=false
       - TZ=Asia/Shanghai
     volumes:
       - /data/deploy/kafkaCluster/kraft:/bitnami/kafka:rw
     network_mode: host
```
启动：
```bash
root@test1:/# docker-compose up -d
```
查看容器状态：
```bash
root@test1:/# docker ps
CONTAINER ID        IMAGE                  COMMAND                  CREATED             STATUS              PORTS               NAMES
ea96a941db3c        bitnami/kafka:latest   "/opt/bitnami/scri..."   18 minutes ago      Up 18 minutes                           kafka_kraft
```
### 2.2 主机二
docker-compose.yaml（172.24.211.114）
```yaml
version: "3"
services:
   kafka:
     image: 'bitnami/kafka:latest'
     container_name: kafka_kraft
     restart: always
     user: root
     ports:
       - '9092:9092'
     environment:
       - KAFKA_ENABLE_KRAFT=yes
       - KAFKA_CFG_PROCESS_ROLES=broker,controller
       - KAFKA_CFG_CONTROLLER_LISTENER_NAMES=CONTROLLER
       - KAFKA_CFG_LISTENERS=PLAINTEXT://:9092,CONTROLLER://:9093
       - KAFKA_CFG_LISTENER_SECURITY_PROTOCOL_MAP=CONTROLLER:PLAINTEXT,PLAINTEXT:PLAINTEXT
       - KAFKA_CFG_ADVERTISED_LISTENERS=PLAINTEXT://172.24.211.114:9092
       - KAFKA_BROKER_ID=2
       - KAFKA_KRAFT_CLUSTER_ID=LelM2dIFQkiUFvXCEcqRWA
       - KAFKA_CFG_CONTROLLER_QUORUM_VOTERS=1@172.24.211.113:9093,2@172.24.211.114:9093
       - ALLOW_PLAINTEXT_LISTENER=yes
       - KAFKA_CFG_AUTO_CREATE_TOPICS_ENABLE=false
       - TZ=Asia/Shanghai
     volumes:
       - /data/deploy/kafkaCluster/kraft:/bitnami/kafka:rw
     network_mode: host
```
启动：
```bash
root@test2:/# docker-compose up -d
```
查看容器状态：
```bash
root@test2:/# docker ps
CONTAINER ID        IMAGE                  COMMAND                  CREATED             STATUS              PORTS               NAMES
ea96a941db3c        bitnami/kafka:latest   "/opt/bitnami/scri..."   21 minutes ago      Up 21 minutes                           kafka_kraft
```

## 三、测试
进入容器：
```bash
root@test2:/# docker exec -it ea96a941db3c /bin/bash
```
进入测试脚本目录：
```bash
root@ea96a941db3c:/# cd /opt/bitnami/kafka/bin
```
### 3.1 创建topic
创建topic（主机一）：
```bash
root@test1:/# kafka-topics.sh --create --topic foo --partitions 5 --replication-factor 2 --bootstrap-server 172.24.211.113:9092,172.24.211.114:9092
Created topic foo.
```
查看topic（主机一）：
```bash
root@test1:/# kafka-topics.sh --list --bootstrap-server 172.24.211.113:9092,172.24.211.114:9092
foo
```
查看topic详情（主机二）：
```bash
root@test2:/# kafka-topics.sh --describe --topic foo --bootstrap-server 172.24.211.113:9092,172.24.211.114:9092
Topic: foo      TopicId: MRj-PkasTnurXnhm6l_yOg PartitionCount: 5       ReplicationFactor: 2    Configs:
        Topic: foo      Partition: 0    Leader: 2       Replicas: 2,1   Isr: 2,1
        Topic: foo      Partition: 1    Leader: 1       Replicas: 1,2   Isr: 1,2
        Topic: foo      Partition: 2    Leader: 2       Replicas: 2,1   Isr: 2,1
        Topic: foo      Partition: 3    Leader: 1       Replicas: 1,2   Isr: 1,2
        Topic: foo      Partition: 4    Leader: 2       Replicas: 2,1   Isr: 2,1
```
### 3.2 生产者与消费者
开启生产者（主机二）：
```bash
root@test2:/# kafka-console-producer.sh --broker-list 172.24.211.113:9092,172.24.211.114:9092 --topic foo
>hello
```
开启消费者（主机一）：
```bash
root@test1:/# kafka-console-consumer.sh --bootstrap-server 172.24.211.113:9092,172.24.211.114:9092 --topic foo --from-beginning
hello
```
### 3.3 删除Topic
```bash
root@test1:/# kafka-topics.sh --delete --topic foo --bootstrap-server 172.24.211.113:9092,172.24.211.114:9092
```
> 删除Topic前务必先停止生产者与消费者，否则删除失败！
