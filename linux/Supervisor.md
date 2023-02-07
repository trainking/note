# Supervisor

## 0x01 概述

**Supervisor**是一个在类Unix系统上，监控和操作进程一个控制系统。与**launchd**、**daemontools**和**runit**等程序目标相同，但不同的是，Supervisor不是作为1号进程的替代品来运行，而是作为一个普通进程来启动的。

包含一个后台服务器**supervisord**和一个命令行工具**supervisorctl**。

### 特性

- 简单：通过一个ini文件配置

- 进程组：将管理进程当作子进程来管理，可以保证启动顺序性

- 准确性：可以准确获取到进程的状态，提供ctl工具查看和操作

### 系统要求

**Supervisor**不会为任何Windows系统提供支持，大多数的类Unix系统都可以支持。需要安装`Python 3.4`或`Python 2.7`以上版本。

## 0x02 安装

直接使用apt或yum安装即可：

```Shell
sudo apt install supervisor
```

安装完成之后，使用`echo_supervisord_conf`命令，可以输出一份ini配置文件

## 0x03 配置

默认配置文件读取顺序：

1. `../etc/supervisord.conf`

2. `../supervisord.conf`

3. `$CWD/supervisord.conf`

4. `$CWD/etc/supervisord.conf`

5. `/etc/supervisord.conf`

6. `/etc/supervisor/supervisord.conf`

### Supervisord

```Shell
[unix_http_server]
file=/var/run/supervisor.sock   
chmod=0700                       

[supervisord]
logfile=/var/log/supervisor/supervisord.log 
pidfile=/var/run/supervisord.pid 
childlogdir=/var/log/supervisor        
```

开启一个`unix_http_server`给到ctl使用。

重启Supservisord加载新配置：`pgrep supervisord | xargs sudo kill -HUP`

### Supervisctl

```Plain Text
[supervisorctl]
serverurl=unix:///var/run/supervisor.sock ; use a unix:// URL  for a unix socket
```

### Program

```Plain Text
[program:task.service]
command=/home/ubuntu/sk_peripheral/task.service -addr 10.13.102.158:20001 -config /home/ubuntu/sk_peripheral/configs/task.service.yml
```

## 0x04 命令行

- status xxx：查看状态

- start xxxx: 启动程序

- stop xxxx: 停止程序

- start all: 启动所有程序

- help: 查看帮助

- tail xxx: 查看输出日志tail

