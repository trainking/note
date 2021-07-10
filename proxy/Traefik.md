# Traefik

`Traefik` 是一个开源的，边缘路由（即外网路由）的反向代理工具。由`golang`开发，简单易用，实践中可作为`Api Gateway`使用。

- [Traefik](#traefik)
  - [0x00 概念](#0x00-概念)
    - [Entrypoints](#entrypoints)
    - [Providers](#providers)
      - [file](#file)
    - [Routers](#routers)
    - [Middlewares](#middlewares)
  - [0x01 配置](#0x01-配置)
    - [serversTransport](#serverstransport)
  - [0x02 实践](#0x02-实践)
    - [将http请求重定向到https](#将http请求重定向到https)

## 0x00 概念

一个请求到达`Traefik`经过`Entrypoints`监听的端口，根据`Router`的`rules`，经过`middlewares`处理后，准发到由`Providers`发现的`Services`上处理。

**职责**:

* ***Providers***: 发现`Services`, 获取他们的ip,端口，活跃情况
* ***Entrypoints***: 监听端口
* ***Routers***：剖析请求，根据`rules`转发到指定`Services`
* ***Services***: 发送请求到业务处理方
* ***Middlewares***: 过滤请求，更新请求，或拒绝特定的请求

### Entrypoints

静态配置项完整参考

```yaml
## Static configuration
entryPoints:
  name:
    # [host]:prot[/tcp | /udp] 默认 tcp
    address: ":8888" # same as ":8888/tcp"
    transport:
      lifeCycle:
        requestAcceptGraceTimeout: 42
        graceTimeOut: 42
    # 响应超时时间
      respondingTimeouts:
        readTimeout: 42    # 读响应超时 秒
        writeTimeout: 42   # 写响应超时 秒
        idleTimeout: 42    # 空闲链接超时时间 秒
    # 转发名单
    proxyProtocol:
      insecure: true
      trustedIPs:
        - "127.0.0.1"
        - "192.168.0.1"
    # 转发头`X-Forwarded-*`
    forwardedHeaders:
      insecure: true
      trustedIPs:
        - "127.0.0.1"
        - "192.168.0.1"
```

### Providers

服务提供者，可以通过多种方式来发现服务。

#### file

通过配置文件的形式，支持YAML和TOML文件格式。

```yaml
providers:
  file:
    directory: "/path/to/dynamic/conf"
    # filename: /etc/traefik/dynamic/dynamic_conf.yml
    # watch: true
```

* 可以指定目录也可以指定文件，指定目录会扫描该目录下所有文件
* `watch`设置为true，Traefik会自动监视配置文件的变更，路径和目录都适用

**dynamic_conf.yml**：

```yaml
# http routing section
http:
  routers:
    # Define a connection between requests and services
    to-whoami:
      # rule: "Host(`example.com`) && PathPrefix(`/whoami/`)"
      # 注意结尾的 /
      rule: "PathPrefix(`/whoami/`)"
       # If the rule matches, applies the middleware
      middlewares:
      - test-user
      # If the rule matches, forward to the whoami service (declared below)
      service: whoami

  middlewares:
    # Define an authentication mechanism
    test-user:
      basicAuth:
        users:
        - test:$apr1$H6uskkkW$IgXLP6ewTrSuBkTrqE8wj/  #htpasswd

  services:
    # Define how to reach an existing service on our infrastructure
    whoami:
      loadBalancer:
        servers:
        - url: http://192.168.112.1:1323/
```

### Routers

### Middlewares

## 0x01 配置

### serversTransport

----

> 语法：serversTransport:
> 
> 默认值：无
> 
> 作用域：Entrypoints | Router

`serversTransport` 是客户端到`Tacefik`连接处理配置。

---

> insecureSkipVerify true|false
> 
> 默认值：false
> 
> 作用域：serversTransport

`serversTransport.insecureSkipVerify` 是否禁用`SSL`证书，默认是`false`，不禁用；开发阶段可以开启，便于调试。

```yaml
## Static configuration
serversTransport:
  insecureSkipVerify: true
```

---

> rootCAs list
> 
> 默认值：无
> 
> 作用域：serversTransport

`serversTransport.rootCAs` 证书列表。

```
## Static configuration
serversTransport:
  rootCAs:
    - foo.crt
    - bar.crt
```

---

> maxIdleConnsPerHost int
> 
> 默认值：2
> 
> 作用域：serversTransport

每个Host最大的空闲连接数。

```yaml
serversTransport:
  maxIdleConnsPerHost: 7
```

---

## 0x02 实践

### 将http请求重定向到https

```yaml
entryPoints:
  web:
    address: :80
    http:
      redirections:
        entryPoint:
          to: websecure
          scheme: https

  websecure:
    address: :443
```