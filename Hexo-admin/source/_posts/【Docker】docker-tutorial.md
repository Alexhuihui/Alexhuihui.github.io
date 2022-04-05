---
title: 【Docker】docker tutorial
date: 2022-04-04 11:53:36
categories: Docker
tags: Docker
---

### docker 是什么

1. Docker is a tool for running applications in an isolated Environment
2. Similar to VM
3. App run in same environment
4. Just works
5. Standard for software deployment

<!-- more --> 

#### containers

容器是一个应用层的抽象，可以打包代码和依赖。多个容器可以运行在同一台机器并共享一个OS内核，每个容器都是运行在单独的进程中的。

#### VM

VM是一个物理硬件的一个抽象，把一台服务变成了多个服务。每个VM都是对OS的完整复制，占用空间更大，启动更慢。



### pulling nginx image

```
# 拉取镜像
docker pull nginx
# 展示所有镜像
docekr images
# 运行容器
docker run --name website -d -p 3000:80 -p 8080:80 nginx:latest
```

### format docker ps

```
docker ps --format="ID\t{{.ID}}\nNAME\t{{.Names}}\nIMAGE\t{{.Image}}\nPORTS\t{{.Ports}}\nCOMMAND\t{{.Command}}\nCREATED\t{{.CreatedAt}}\nSTATUS\t{{.Status}}\n"

export Format="ID\t{{.ID}}\nNAME\t{{.Names}}\nIMAGE\t{{.Image}}\nPORTS\t{{.Ports}}\nCOMMAND\t{{.Command}}\nCREATED\t{{.CreatedAt}}\nSTATUS\t{{.Status}}\n"

docker ps --format=$Format
```



### Volumes

允许宿主机和容器、容器和容器之间相互共享数据包括文件或者文件夹。

```
docker run --name website -v $(pwd):/usr/share/nginx/html:ro -d -p 8080:80 nginx:latest
```

容器之间共享数据

```
docker run --name website_copy --volumes-from website -d -p 8081:80 nginx
```



### Dockerfile

Build own image

[docker官网文档](https://docs.docker.com/engine/reference/builder/)

在项目根目录创建 Dockerfile 文件

```dockerfile
FROM nginx:latest
ADD . /usr/share/nginx/html
```

```sh
docker build  --tag website:latest .
docker run --name website -d -p 8080:80 website:latest
```

构建 user-service-api

```dockerfile
FROM node:latest
WORKDIR /app
ADD . .
RUN npm install
CMD node index.js
```

编写 .dockerignore

```dockerfile
node_modules
Dockerfile
.git 
```

使用缓存，只改变源码的情况下不用重新下载依赖

```dockerfile
FROM node:latest
WORKDIR /app
ADD package*.json .
RUN npm install
ADD . .
CMD node index.js
```



### Alpine

选择镜像tag里包含alpine去构建，会大大的减少镜像体积

### Tags and Version

Usage:  docker tag SOURCE_IMAGE[:TAG] TARGET_IMAGE[:TAG]

Create a tag TARGET_IMAGE that refers to SOURCE_IMAGE



### Docker Registries

注册 hub.docker.com 账号，创建仓库，把本地的镜像通过tag命令指定成远程仓库的名称，然后推送上去。

### Docker Inspect

```sh
docker inspect name/id
docker logs name/id
docker exec -it name/id /bash
```

