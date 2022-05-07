---
title: 【Kubernetes】使用minikube
date: 2022-05-07 22:05:44
categories: Kubernetes
tags: [Kubernetes, minikube]
---

# Kubernetes

## 使用minikube

### 安装

[官网地址](https://minikube.sigs.k8s.io/docs/start/)

在linux上安装

using **binary download**

```shell
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube
```

Start a cluster using the docker driver

```shell
minikube start --driver=docker
```

### 启动集群

不能用root启动，新增一个test用户加入到sudo组里，再把它加入到docker的用户组中

```shell
su test
minikube start
```

### 与集群交互

```shell
minikube kubectl -- get po -A
alias kubectl="minikube kubectl --"
minikube dashboard
```

### 部署应用

```shell
kubectl create deployment hello-minikube --image=k8s.gcr.io/echoserver:1.4
kubectl expose deployment hello-minikube --type=NodePort --port=8080
```

## 简述NodePort

### 前言

最近在学习Kubernetes的过程中，由于都是在K8s集群内部进行Docker通信的，就遇到了如何暴露服务给外界访问的问题，生产环境比较好的方案就是借助云服务商使用LoadBalancer的方式，但由于是测试环境就使用了比较简单的NodePort来暴露服务，在实践过程中，也加深了对K8s概念的理解。

### Service

把一组Pods抽象为网络服务，通过K8s你不需要通过修改程序的服务发现机制来管理通信。K8s给每个Pods独立的IP，以及给一组Pods一个DNS并通过[负载均衡](https://so.csdn.net/so/search?q=负载均衡&spm=1001.2101.3001.7020)的方式进行访问。

### 动机

K8s的Pods是有生命周期的，通常是可以被创建和销毁的，然后销毁之后就不会再启动了。如果采用的是Deployment，则可以动态的创建和销毁Pod。

每个Pod都有自己的IP，但是在Deployment中创建的Pod，先创建的可能会和后面创建Pod的IP地址不同。

那么就会出现一个问题，如果一组Pod为服务调用方，一组Pod为服务提供方，服务调用者怎么找到服务提供者的地址？



### Service 资源

Kubernetes的Service定义了一种抽象：逻辑上的一组Pod，一种可以访问它们的方式。这一组Pod能通过Service被访问到，通过是通过Selector来实现的。

举个例子，如果后台有三个节点提供图片访问服务，调用者可以通过Service进行访问，它不需要知道具体访问的是哪一个节点，具体的策略由Service来配置，并实现负载均衡。某种意义上也是服务发现和解耦。



### nodePort

外部流量访问K8s的一种方式，即`nodeIP:nodePort`，是提供给外部流量访问K8s集群资源的一种方式。

例如需要暴露服务的端口给外界访问的话可以通过命令：

```shell
kubectl expose deployment nginx --type=NodePort
```

可以随机暴露出一个端口外部访问的端口（默认值：30000-32767）出来。由于暴露的端口往往都比较大，这时候可以采用nginx反向代理的方式，为外界提供访问服务(HTTP:80，HTTPS:443)。

除了使用命令之外也可以使用yaml配置文件的方式进行服务的配置，如下所示：

```yaml
apiVersion: v1
kind: Service
metadata:
  name: webapp-service
spec:
  type: NodePort
  selector:
    app: webapp
  ports:
    - protocol: TCP
      port: 3000
      targetPort: 3000
      nodePort: 30100
```

然后如果是一些内部的服务，比如数据库服务，或者除了网关以外的微服务，这些服务是不需要外部访问的。因此没有必要设置`nodePort`属性。

### port

K8s集群内部服务访问service的入口。是service暴露在Cluster上的端口，`ClusterIP:Port`。

### targetPort

容器的端口，也是最终底层的服务所提供的端口，所以说targetPod也就是Pod的端口。从port或者是nodePort进入的流量，经过路由转发之后，最终都会都通过targetPort进入到Pod中。



### 总结

总体来说，除了targetPort是容器本身的端口之外，port和nodePod都是Service的端口。不同的是port是暴露给K8s访问的，nodePort是暴露给外部访问的。

## 将 minikube 的服务暴露到宿主机外

[minikube](https://minikube.sigs.k8s.io/) 是一款基于 Kubernetes 的定位于快速验证功能的小型容器编排环境。

由于它的定位特性，我们在使用中会发现 minikube 虚拟出了一个 IP 作为自身的节点 IP，该 IP 和宿主机不同。对于 `NodePort` 类型的 Service 也没有办法通过 `127.0.0.1` 访问。

### Host 内访问

可以通过 `<minikube-ip>:<service-port>` 来访问 service

### Host 外访问

通过 minikube 可以很方便地在本机访问，同时避免了对宿主机端口的占用。但是也带来了另一个问题：无法直接通过访问宿主机的端口来访问 services 进行调试。

比如我的实验机是一台云服务虚拟机，我在自己的电脑上可以访问这台虚拟机，但是不能访问虚拟机上 minikube 暴露的服务。

虽然没有办法让 minikube 直接通过宿主机端口对外暴露 services，但是如果我们把问题换个角度思考，就很容易找到解决办法：如何将一个 service（不特定类型）暴露到本机。

这时候最简单的办法就是通过 `kubectl port-forward` 转发端口。

我有一台web服务，service监听3000端口（port），我希望暴露在宿主机上的8888端口，就可以使用如下命令：

```shell
kubectl port-forward --address 0.0.0.0 service/webapp-service 8888:3000
```

### kubectl操作命令

伸缩pod

```shell
kubectl scale deployment my-nginx --replicas=0
```

清除 Deployment、ReplicaSet 和 Pod

```shell
kubectl delete services frontend backend
kubectl delete deployment frontend backend
```

