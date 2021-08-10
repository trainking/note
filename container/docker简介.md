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

## 什么是容器化？

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