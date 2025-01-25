---
title: 【ELK】springboot结合elk搭建日志平台
date: 2023-12-06 19:54:06
categories: ELK
tags: ELK

---

本文介绍如果通过搭建ELK收集springboot的日志。

<!-- more --> 

### 部署ELK

通过docker compose安装

1. 安装docker compose

   ```
   yum install -y yum-utils device-mapper-persistent-data lvm2
   yum install docker-compose
   ```

2. 创建以下目录以及对应的配置文件

   ```
   .
   ├── docker-compose.yml
   ├── elasticsearch
   │   ├── config
   │   │   └── elasticsearch.yml
   │   ├── data
   │   └── logs
   ├── kibana
   │   └── config
   │       └── kibana.yml
   └── logstash
       ├── config
       │   ├── logstash.yml
       │   └── small-tools
       │       └── dev-enlightrn-hub.config
       └── data
   ```

   ```
   # elasticsearch.yml
   cluster.name: "docker-cluster"
   network.host: 0.0.0.0
   http.port: 9200
   # 开启es跨域
   http.cors.enabled: true
   http.cors.allow-origin: "*"
   http.cors.allow-headers: Authorization
   # # 开启安全控制
   xpack.security.enabled: true
   xpack.security.transport.ssl.enabled: true
   ```

   ```
   # kibana.yml
   server.name: kibana
   server.host: "0.0.0.0"
   server.publicBaseUrl: "http://kibana:5601"
   elasticsearch.hosts: [ "http://elasticsearch:9200" ] 
   xpack.monitoring.ui.container.elasticsearch.enabled: true
   elasticsearch.username: "elastic"
   elasticsearch.password: "123456"
   i18n.locale: zh-CN
   ```

   ```
   # logstash.yml
   http.host: "0.0.0.0"
   xpack.monitoring.enabled: true
   xpack.monitoring.elasticsearch.hosts: [ "http://elasticsearch:9200" ]
   xpack.monitoring.elasticsearch.username: "elastic"
   xpack.monitoring.elasticsearch.password: "123456"
   ```

   ```
   # dev-enlightrn-hub.config
   input { #输入
    
       tcp {
           mode => "server"
           host => "0.0.0.0"   # 允许任意主机发送日志
           type => "dev-enlighten-hub"      # 设定type以区分每个输入源
           port => 9999
           codec => json_lines # 数据格式
       }
    
   }
    
    
   filter {
       mutate {
           # 导入之过滤字段
           remove_field => ["LOG_MAX_HISTORY_DAY", "LOG_HOME", "APP_NAME"]
           remove_field => ["@version", "_score", "port", "level_value", "tags", "_type", "host"]
       }
   }
    
    
   output { #输出-控制台
       stdout{
           codec => rubydebug
       }
   }
    
    
   output { #输出-es
    
       if [type] == "dev-enlighten-hub" {
           elasticsearch {
               action => "index"                       # 输出时创建映射
               hosts  => "http://elasticsearch:9200"   # ES地址和端口
               user => "elastic"                       # ES用户名
               password => "123456"                    # ES密码
               index  => "dev-enlighten-hub-%{+YYYY.MM.dd}"         # 指定索引名-按天
               codec  => "json"
           }
       }
    
   }
   ```

3. 编写docker-compose.yml

   ```
   version: '3.3'
   networks:
     elk:
       driver: bridge
   services:
     elasticsearch:
       image: registry.cn-hangzhou.aliyuncs.com/zhengqing/elasticsearch:7.14.1
       container_name: elk_elasticsearch
       restart: unless-stopped
       volumes:
         - "./elasticsearch/data:/usr/share/elasticsearch/data"
         - "./elasticsearch/logs:/usr/share/elasticsearch/logs"
         - "./elasticsearch/config/elasticsearch.yml:/usr/share/elasticsearch/config/elasticsearch.yml"
       environment:
         TZ: Asia/Shanghai
         LANG: en_US.UTF-8
         TAKE_FILE_OWNERSHIP: "true"  # 权限
         discovery.type: single-node
         ES_JAVA_OPTS: "-Xmx1g -Xms1g"
         ELASTIC_PASSWORD: "111111" # elastic账号密码
       ports:
         - "9200:9200"
         - "9300:9300"
       networks:
         - elk
    
     kibana:
       image: registry.cn-hangzhou.aliyuncs.com/zhengqing/kibana:7.14.1
       container_name: elk_kibana
       restart: unless-stopped
       volumes:
         - "./kibana/config/kibana.yml:/usr/share/kibana/config/kibana.yml"
       ports:
         - "5601:5601"
       depends_on:
         - elasticsearch
       links:
         - elasticsearch
       networks:
         - elk
    
     logstash:
       image: registry.cn-hangzhou.aliyuncs.com/zhengqing/logstash:7.14.1
       container_name: elk_logstash
       restart: unless-stopped
       environment:
         LS_JAVA_OPTS: "-Xmx1g -Xms1g"
       volumes:
         - "./logstash/data:/usr/share/logstash/data"
         - "./logstash/config/logstash.yml:/usr/share/logstash/config/logstash.yml"
         - "./logstash/config/small-tools:/usr/share/logstash/config/small-tools"
       command: logstash -f /usr/share/logstash/config/small-tools
       ports:
         - "9600:9600"
         - "9999:9999"
       depends_on:
         - elasticsearch
       networks:
         - elk
   ```

4. 执行启动命令

   ```
   docker compose up -d
   ```

5. 如果启动失败可以通过下述命令进行检查配置项

   ```
   docker-compose run --rm logstash logstash -t -f /usr/share/logstash/config/small-tools
   ```

### 修改spring boot项目

1. 添加maven依赖

   ```
   <dependency>
         <groupId>net.logstash.logback</groupId>
         <artifactId>logstash-logback-encoder</artifactId>
         <version>7.3</version>
   </dependency>
   ```

2. 添加logback-spring.xml

   ```
   <?xml version="1.0" encoding="UTF-8"?>
   <configuration scan="true" scanPeriod="60 seconds" debug="false">
     <!-- 日志存放路径 -->
     <property name="log.path" value="${user.dir}/logs"/>
     <!-- 日志输出格式 -->
     <property name="log.pattern" value="%d{HH:mm:ss.SSS} [%thread] %-5level %logger{20} - [%method,%line] - %msg%n"/>
   
     <!-- 控制台输出 -->
     <appender name="console" class="ch.qos.logback.core.ConsoleAppender">
       <encoder>
         <pattern>${log.pattern}</pattern>
       </encoder>
     </appender>
   
     <!-- 系统日志输出 -->
     <appender name="file_info" class="ch.qos.logback.core.rolling.RollingFileAppender">
       <file>${log.path}/info.log</file>
       <!-- 循环政策：基于时间创建日志文件 -->
       <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
         <!-- 日志文件名格式 -->
         <fileNamePattern>${log.path}/info.%d{yyyy-MM-dd}.log</fileNamePattern>
         <!-- 日志最大的历史 7天 -->
         <maxHistory>7</maxHistory>
       </rollingPolicy>
       <encoder>
         <pattern>${log.pattern}</pattern>
       </encoder>
       <filter class="ch.qos.logback.classic.filter.LevelFilter">
         <!-- 过滤的级别 -->
         <level>INFO</level>
         <!-- 匹配时的操作：接收（记录） -->
         <onMatch>ACCEPT</onMatch>
         <!-- 不匹配时的操作：拒绝（不记录） -->
         <onMismatch>DENY</onMismatch>
       </filter>
     </appender>
   
     <appender name="file_error" class="ch.qos.logback.core.rolling.RollingFileAppender">
       <file>${log.path}/error.log</file>
       <!-- 循环政策：基于时间创建日志文件 -->
       <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
         <!-- 日志文件名格式 -->
         <fileNamePattern>${log.path}/error.%d{yyyy-MM-dd}.log</fileNamePattern>
         <!-- 日志最大的历史 60天 -->
         <maxHistory>60</maxHistory>
       </rollingPolicy>
       <encoder>
         <pattern>${log.pattern}</pattern>
       </encoder>
       <filter class="ch.qos.logback.classic.filter.LevelFilter">
         <!-- 过滤的级别 -->
         <level>ERROR</level>
         <!-- 匹配时的操作：接收（记录） -->
         <onMatch>ACCEPT</onMatch>
         <!-- 不匹配时的操作：拒绝（不记录） -->
         <onMismatch>DENY</onMismatch>
       </filter>
     </appender>
   
     <!-- 将日志文件输出到Logstash -->
     <appender name="logstash" class="net.logstash.logback.appender.LogstashTcpSocketAppender">
       <!-- 获取logstash地址作为输出的目的地 -->
       <destination>172.22.0.4:9999</destination>
       <encoder chatset="UTF-8" class="net.logstash.logback.encoder.LogstashEncoder"/>
     </appender>
   
     <!-- 系统模块日志级别控制  -->
     <logger name="com.gyhappylife" level="info"/>
     <!-- Spring日志级别控制  -->
     <logger name="org.springframework" level="warn"/>
   
     <root level="info">
       <appender-ref ref="console"/>
     </root>
   
     <!--系统操作日志-->
     <root level="info">
       <appender-ref ref="file_info"/>
       <appender-ref ref="file_error"/>
       <appender-ref ref="logstash"/>
     </root>
   </configuration>
   ```

### 注意事项

如果spring应用也是跑在docker中的话 那么它的网段可能会和ELK不一样 。这样就必须在iptables中添加路由规则例如:

```
$ iptables -I DOCKER-USER -i br-3bed419583c5 -o br-787e52a81bb4 -j ACCEPT
$ iptables -I DOCKER-USER -i br-787e52a81bb4 -o br-3bed419583c5 -j ACCEPT
```