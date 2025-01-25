---
title: 【ElasticSearch】docker安装es8和kibana8
date: 2022-12-10 19:54:06
categories: ElasticSearch
tags: ElasticSearch

---

本文介绍如何通过`Docker`安装`elasticsearch` 和 `kibana`

<!-- more --> 

### 下载镜像

```
docker pull docker.elastic.co/elasticsearch/elasticsearch:8.5.2

docker pull kibana:8.5.2
```

### 创建网络

```
docker network create elastic
```

### 启动es

```
docker run -it \
--name elasticsearch \
--net elastic \
--restart=always \
-p 9200:9200 \
-p 9300:9300 \
-e "discovery.type=single-node" \
-e "ES_JAVA_OPTS=-Xms512m -Xmx512m" \
docker.elastic.co/elasticsearch/elasticsearch:8.5.2
```

容器启动之后会打印密码和token，等会用于登录`kibana`

### 启动`kibana`

```
docker run \
--name kibana \
--net elastic \
-p 5601:5601 \
-d \
kibana:8.5.2
```

### 访问

等`kibana`容器启动成功之后，点击进入日志中输出的地址，然后按照提示依次输入token和密码就可以访问啦。

效果图：

[![img](https://raw.githubusercontent.com/Alexhuihui/photo/main/20221211104146.png)](https://raw.githubusercontent.com/Alexhuihui/photo/main/20221211104146.png)