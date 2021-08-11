# docker简介

- [docker简介](#docker简介)
  - [什么是容器化？](#什么是容器化)
  - [镜像](#镜像)
    - [拉取镜像](#拉取镜像)
    - [列出镜像](#列出镜像)
    - [删除镜像](#删除镜像)
  - [容器](#容器)
    - [run](#run)
    - [容器列表](#容器列表)
    - [后台运行](#后台运行)
    - [终止容器](#终止容器)
    - [重启容器](#重启容器)
    - [删除容器](#删除容器)
  - [Dockerfile](#dockerfile)
  - [数据卷](#数据卷)
    - [创建数据卷](#创建数据卷)
    - [查看数据卷](#查看数据卷)
    - [绑定数据卷](#绑定数据卷)
  - [网络](#网络)
    - [创建](#创建)
    - [绑定到网络](#绑定到网络)
  - [docker-compose](#docker-compose)
    - [安装](#安装)
  - [ck-server](#ck-server)

## 什么是容器化？

容器为应用程序提供了隔离的运行空间：每个容器内都包含一个独享的完整用户环境空间，并且一个容器内的变动不会影响其他容器的运行环境。容器技术使用了namespaces来进行空间隔离，通过文件系统的挂载点来决定容器可以访问哪些文件，通过cgroups来确定每个容器可以利用多少资源。此外容器之间共享同一个系统内核，这样当同一个库被多个容器使用时，内存的使用效率会得到提升。

## 镜像

### 拉取镜像

```
docker pull ubuntu:16.04
```

### 列出镜像

```
docker image list
```

### 删除镜像

```
docker image rm [name or ID]
```

or 

```
docker rmi [name or ID]
```

## 容器

### run

```
docker run -it --rm ubuntu:16.04 /bin/bash
```

### 容器列表

```
docker container list
```

查看所有

```
docker container list -a
```

### 后台运行

```
docker run -d ubuntu:16.04 /bin/sh -c "while true; do echo hello world; sleep 1; done"
```

### 终止容器

```
docker container stop [name or ID]
```

### 重启容器

```
docker container restart [name or ID]
```

### 删除容器

```
docker container rm [name or ID]
```

## Dockerfile

```
FROM ubuntu:16.04

RUN /bin/sh -c "while true; do echo hello world; sleep 1; done" &
```

## 数据卷

### 创建数据卷

```
docker volume create my-vol
```

### 查看数据卷

```
docker volume inspect my-vol
```

### 绑定数据卷

```
docker run -it --name helloworld --mount source=my-vol,target=/webapp hello:v1.0
```

## 网络

### 创建

```
docker network create -d bridge my-net
```

### 绑定到网络

```
docker run -it --name helloworld --network my-net --mount source=my-vol,target=/webapp hello:v1.0
```

## docker-compose

### 安装

```
pip install -U docker-compose
```

## ck-server

```
docker run -d --name ck-server --ulimit nofile=262144:262144 -p 8123:8123 -p 9000:9000 -p 9009:9009 yandex/clickhouse-server
```