---
title: 【gRPC】Java和Python使用gRPC
date: 2022-02-10 18:35:10
categories: gRPC
tags: gRPC
---

### 使用Java搭建gRPC服务端

#### 1、如何配置Maven

方式一：

> <dependency>
>    <groupId>io.grpc</groupId>
>    <artifactId>grpc-all</artifactId>
>    <version>1.26.0</version>
> </dependency>

<!-- more --> 

方式二：

> ```
> <dependency>
>     <groupId>io.grpc</groupId>
>     <artifactId>grpc-netty-shaded</artifactId>
>     <version>1.44.0</version>
>     <scope>runtime</scope>
> </dependency>
> <dependency>
>     <groupId>io.grpc</groupId>
>     <artifactId>grpc-protobuf</artifactId>
>     <version>1.44.0</version>
> </dependency>
> <dependency>
>     <groupId>io.grpc</groupId>
>     <artifactId>grpc-stub</artifactId>
>     <version>1.44.0</version>
> </dependency>
> <dependency> <!-- necessary for Java 9+ -->
>     <groupId>org.apache.tomcat</groupId>
>     <artifactId>annotations-api</artifactId>
>     <version>6.0.53</version>
>     <scope>provided</scope>
> </dependency>
> ```

方式三：

> <dependency>
>   <groupId>io.grpc</groupId>
>   <artifactId>grpc-netty</artifactId>
>   <version>1.26.0</version>
> </dependency>
> <dependency>
>   <groupId>io.grpc</groupId>
>   <artifactId>grpc-protobuf</artifactId>
>   <version>1.26.0</version>
> </dependency>
> <dependency>
>   <groupId>io.grpc</groupId>
>   <artifactId>grpc-stub</artifactId>
>   <version>1.26.0</version>
> </dependency>

三种方式的区别：
a.
方式一会把与gRPC相关的所有jar都引入，不管实际项目中是否会用到。引入相关jar较多，但其配置简单。
方式三只会引入与gRPC和netty相关的jar，如果要使用OkHttp，则要再引入。配置相对较多，但引入jar相对较少。
方式二引入的jar的数量介于方式一与方式三之间。

b.
方式一中netty是以原始jar引用的。这样方便在使用的过程中查看netty源代码，方便开发。
方式二中netty被放入grpc-netty-shaded-1.26.0.jar的io.grpc.netty.shaded包中。这样的好处是当项目中有多个模块使用了不同版本的netty时，gRPC功能不会受到影响。但坏处就是不方便查看netty源代码了。
方式三与方式一相同，但不会引入像OkHttp这样与netty与gRPC不相关的jar。

总之，方式一和方式三都适合用在只需要使用单一netty版本的环境，方式二适合用在多个netty版本共存的环境 。

#### 2、编写protobuf文件

在idea的插件市场下载安装protobuf的插件

创建src/main/proto/rpc_date.proto

```protobuf
syntax = "proto3"; // 协议版本

// 选项配置
option java_package = "top.alexmmd.grpc.api";
option java_outer_classname = "RPCDateServiceApi";
option java_multiple_files = true;

// 定义包名
package top.alexmmd.grpc.api;

// 服务接口.定义请求参数和相应结果
service RPCDateService {
  rpc getDate (RPCDateRequest) returns (RPCDateResponse) {
  }
}

// 定义请求体
message RPCDateRequest {
  string userName = 1;
}

// 定义相应内容
message RPCDateResponse {
  string serverDate = 1;
}
```

使用maven插件进行编译

![](https://raw.githubusercontent.com/Alexhuihui/photo/main/uTools_1644490459871.png)

把target目录中生成的文件复制到java目录中

#### 3、编写接口实现类

```java
// RPCDateServiceGrpc.RPCDateServiceImplBase 这个就是接口.
// RPCDateServiceImpl 我们需要继承他的,实现方法回调
public class RPCDateServiceImpl extends RPCDateServiceGrpc.RPCDateServiceImplBase {
    @Override
    public void getDate(RPCDateRequest request, StreamObserver<RPCDateResponse> responseObserver) {
        System.out.println("request = " + request);
        // 请求结果,我们定义的
        RPCDateResponse rpcDateResponse = null;
        String userName = request.getUserName();
        String response = String.format("你好: %s, 今天是%s.", userName, LocalDate.now().format(DateTimeFormatter.ofPattern("yyyy-MM-dd")));
        try {
            // 定义响应,是一个builder构造器.
            rpcDateResponse = RPCDateResponse
                    .newBuilder()
                    .setServerDate(response)
                    .build();
        } catch (Exception e) {
            responseObserver.onError(e);
        } finally {
            // 这种写法是observer,异步写法,老外喜欢用这个框架.
            responseObserver.onNext(rpcDateResponse);
        }
        responseObserver.onCompleted();
    }
}
```

#### 4、定义服务端

```java
public class GRPCServer {
    private static final int port = 9999;

    public static void main(String[] args) throws Exception {
        // 设置service接口.
        Server server = ServerBuilder.
                forPort(port)
                .addService(new RPCDateServiceImpl())
                .build().start();
        System.out.println(String.format("GRpc服务端启动成功, 端口号: %d.", port));
        server.awaitTermination();
    }
}
```

#### 5、定义客户端

```java
public class GRPCClient {
    private static final String host = "localhost";
    private static final int serverPort = 9999;

    public static void main(String[] args) throws Exception {
        // 1. 拿到一个通信的channel
        ManagedChannel managedChannel = ManagedChannelBuilder.forAddress(host, serverPort).usePlaintext().build();
        try {
            // 2.拿到道理对象
            RPCDateServiceGrpc.RPCDateServiceBlockingStub rpcDateService = RPCDateServiceGrpc.newBlockingStub(managedChannel);
            RPCDateRequest rpcDateRequest = RPCDateRequest
                    .newBuilder()
                    .setUserName("anthony")
                    .build();
            // 3. 请求
            RPCDateResponse rpcDateResponse = rpcDateService.getDate(rpcDateRequest);
            // 4. 输出结果
            System.out.println(rpcDateResponse.getServerDate());
        } finally {
            // 5.关闭channel, 释放资源.
            managedChannel.shutdown();
        }
    }
}
```

---

### 使用Python搭建gRPC客户端

#### 1、环境配置

- protobuf 运行时(runtime)

  ```python
  pip install grpcio
  ```

- 安装 python 下的 protoc 编译器

  ```python
  pip install grpcio-tools
  ```

#### 2、编译proto文件

```python
python -m grpc_tools.protoc --python_out=. --grpc_python_out=. -I. rpc_date.proto
```



#### 3、编写客户端

```python
import grpc
import rpc_date_pb2
import rpc_date_pb2_grpc


def run():
    # 连接 rpc 服务器
    channel = grpc.insecure_channel('127.0.0.1:9999')
    # 调用 rpc 服务
    stub = rpc_date_pb2_grpc.RPCDateServiceStub(channel)
    response = stub.getDate(rpc_date_pb2.RPCDateRequest(userName='czl'))
    print(response)


if __name__ == '__main__':
    run()
```

### 最后

proto文件中的包名也要保持一致，最好不要改动