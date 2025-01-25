---
title: 【Kafka】Linux安装Kafka以及可视化界面
date: 2022-11-29 19:54:06
categories: Kafka
tags: Kafka

---

Kafka是当下非常流行的消息中间件，据官网透露，已有成千上万的公司在使用它。本文介绍如何在`Linux`上安装`kafka`以及`Yahoo`开源的一款可视化管理页面。

<!-- more --> 

### `Kafka`简介

Kafka是由`LinkedIn`公司开发的一款开源分布式消息流平台，由`Scala`和`Java`编写。主要作用是为处理实时数据提供一个统一、高吞吐、低延迟的平台，其本质是基于`发布订阅模式`的消息引擎系统。

Kafka具有以下特性：

- 高吞吐、低延迟：`Kafka`收发消息非常快，使用集群处理消息延迟可低至`2ms`。
- 高扩展性：`Kafka`可以弹性地扩展和收缩，可以扩展到上千个broker，数十万个partition，每天处理数万亿条消息。
- 永久存储：`Kafka`可以将数据安全地存储在分布式的，持久的，容错的群集中。
- 高可用性：`Kafka`在可用区上可以有效地扩展群集，某个节点宕机，集群照样能够正常工作。

### `Kafka`安装

- 首先我们需要下载`Kafka`的安装包，[下载地址](https://downloads.apache.org/kafka/3.3.1/kafka_2.13-3.3.1.tgz)

- 下载完成后将`Kafka`解压到指定目录

  ```
  cd /opt/module
  tar -zxvf kafka_2.13-3.3.1.tgz
  cd kafka_2.13-3.3.1
  ```

- `Kafka`可以使用`ZooKeeper`或`KRaft`启动，使用任意一个即可，这里使用`ZooKeeper`。

[![img](https://raw.githubusercontent.com/Alexhuihui/photo/main/20221129092557.png)](https://raw.githubusercontent.com/Alexhuihui/photo/main/20221129092557.png)

- 启动`Zookeeper`服务，服务将运行在`2181`端口

  ```
  # 后台运行服务，并把日志输出到当前文件夹下的zookeeper-out.file文件中
  nohup bin/zookeeper-server-start.sh config/zookeeper.properties > zookeeper-out.file 2>&1 &
  ```

- 由于目前Kafka是部署在Linux服务器上的，外网如果想要访问，需要修改Kafka的配置文件`config/server.properties`，修改下Kafka的监听地址，否则会无法连接；

  ```
  # The address the socket server listens on. If not configured, the host name will be equal to the value of
  # java.net.InetAddress.getCanonicalHostName(), with PLAINTEXT listener name, and port 9092.
  #   FORMAT:
  #     listeners = listener_name://host_name:port
  #   EXAMPLE:
  #     listeners = PLAINTEXT://your.host.name:9092
  listeners=PLAINTEXT://:9092
  ```

- 最后启动Kafka服务，服务将运行在`9092`端口。

  ```
  # 后台运行服务，并把日志输出到当前文件夹下的kafka-out.file文件中
  nohup bin/kafka-server-start.sh config/server.properties > kafka-out.file 2>&1 &
  ```

### `Kafka`命令行操作

> 接下来我们使用命令行来操作下Kafka，熟悉下Kafka的使用。

- 首先创建一个叫`consoleTopic`的Topic；

  ```
  bin/kafka-topics.sh --create --topic consoleTopic --bootstrap-server 10.0.20.3:9092
  ```

- 接下来查看Topic；

  ```
  bin/kafka-topics.sh --describe --topic consoleTopic --bootstrap-server 10.0.20.3:9092
  ```

- 会显示如下Topic信息；

  [![img](https://raw.githubusercontent.com/Alexhuihui/photo/main/20221129100810.png)](https://raw.githubusercontent.com/Alexhuihui/photo/main/20221129100810.png)

- 向Topic中发送消息：

  ```
  bin/kafka-console-producer.sh --topic consoleTopic --bootstrap-server 10.0.20.3:9092
  ```

- 直接在命令行中输入信息即可发送；

  [![img](https://raw.githubusercontent.com/Alexhuihui/photo/main/20221129100935.png)](https://raw.githubusercontent.com/Alexhuihui/photo/main/20221129100935.png)

- 重新打开一个窗口，通过如下命令可以从Topic中获取消息：

  ```
  bin/kafka-console-consumer.sh --topic consoleTopic --from-beginning --bootstrap-server 10.0.20.3:9092
  ```

  [![img](https://raw.githubusercontent.com/Alexhuihui/photo/main/20221129101257.png)](https://raw.githubusercontent.com/Alexhuihui/photo/main/20221129101257.png)

### `Kafka`可视化

- 下载`cmak`的安装包，[下载地址](https://github.com/yahoo/CMAK/releases/download/3.0.0.6/cmak-3.0.0.6.zip)

- 必须通过`jdk11`来启动，[参考文档](https://lenjor.github.io/2020/12/Linux-Java-JDK-install/)

- 修改`conf/application.conf`

  ```
  kafka-manager.zkhosts="127.0.0.1:2181"
  kafka-manager.zkhosts=${?ZK_HOSTS}
  cmak.zkhosts="127.0.0.1:2181"
  cmak.zkhosts=${?ZK_HOSTS}
  ```

- 启动

  ```
  nohup bin/cmak -java-home /opt/soft/jdk-11.0.17 > cmak-out.file 2>&1 &
  ```

- 配置`nginx`通过域名访问

  ```
  upstream kafka{
         server localhost:9000;
      }
  server{
        listen 80;
        server_name kafka.alexmmd.top;
  
        access_log /var/log/nginx/kafka-access.log  main;
  
        if ( $host ~* "\d+\.\d+\.\d+\.\d+" ) {
          return 400;
         }
        location / {
           proxy_set_header Host $http_host;
           proxy_set_header X-Real-IP $remote_addr;
           proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
           proxy_pass http://kafka;
        }
  
         error_page 500 502 503 504 /50x.html;
         location = /50x.html{
           root html;
         }
  }
  ```

### 参考文档

[kafka的listeners的配置](https://juejin.cn/post/6893410969611927566)

[kafka的安装](https://juejin.cn/post/6971224791793532941)