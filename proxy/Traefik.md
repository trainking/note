# Traefik

`Traefik` 是一个开源的，边缘路由（即外网路由）的反向代理工具。由`golang`开发，简单易用，实践中可作为`Api Gateway`使用。

- [Traefik](#traefik)
  - [0x00 概念](#0x00-概念)
    - [entryPoints](#entrypoints)
    - [providers](#providers)
      - [file](#file)
    - [routing](#routing)
  - [0x01 实践](#0x01-实践)
    - [将http请求重定向到https](#将http请求重定向到https)

## 0x00 概念

### entryPoints

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
        readTimeout: 42
        writeTimeout: 42
        idleTimeout: 42
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

### providers

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

### routing

## 0x01 实践

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