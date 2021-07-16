# Traefik

`Traefik` 是一个开源的，边缘路由（即外网路由）的反向代理工具。由`golang`开发，简单易用，实践中可作为`Api Gateway`使用。

- [Traefik](#traefik)
  - [0x00 概念](#0x00-概念)
    - [Entrypoints](#entrypoints)
      - [HTTP Options](#http-options)
    - [Providers](#providers)
      - [file](#file)
    - [Routers](#routers)
      - [指定EntryPoints](#指定entrypoints)
      - [Rule](#rule)
      - [优先级](#优先级)
    - [Middlewares](#middlewares)
  - [0x01 安装与启动](#0x01-安装与启动)
    - [Docker](#docker)
    - [二进制文件](#二进制文件)
    - [基础配置（静态配置）](#基础配置静态配置)
  - [0x02 配置](#0x02-配置)
    - [serversTransport](#serverstransport)
    - [EntryPoints](#entrypoints-1)
  - [0x03 实践](#0x03-实践)
    - [将http请求重定向到https](#将http请求重定向到https)
    - [使用第三方验证器验证](#使用第三方验证器验证)

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

#### HTTP Options

Entrypoints 可以在入口点直接切入转发规则，但是只能运用与http。例如：

[将http请求重定向到https](#将http请求重定向到https)

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

路由根据规则将请求转发到能处理它的服务上去，在这个过程中，会有中间件修改这个请求。

```yaml
http:
  routers:
    my-router:
      rule: "Path(`/foo`)"
      service: service-foo
```

#### 指定EntryPoints

默认情况下，`HTTP Router`会接受所有EntryPoints的流量，然后通过规则来去区分，如指定`Host`:

```yaml
http:
  routers:
    Router-1:
      # By default, routers listen to every entry points
      rule: "Host(`example.com`)"
      service: "service-1"
```

也可以指定从`EntryPoints`获取流量:

```yaml
http:
  routers:
    Router-1:
      # won't listen to entry point web
      entryPoints:
        - "websecure"
        - "other"
      rule: "Host(`example.com`)"
      service: "service-1"
```

#### Rule

路由匹配规则，使用类函数的声明。`||`和`&&`来表示多个条件的逻辑关系。

|Rule|说明|
|:--|--|
|Headers(`key`, `value`)|检查请求头中是否有存在`key:value`的头|
|HeadersRegexp(`key`, `regexp`)|检查请求头中是否有存在key，且值满足`regexp`正则匹配|
|Host(`example.com`, ...)|检查host请求头是否满足|
|HostRegexp(`example.com`, `{subdomain:[a-z]+}.example.com`, ...)|检查host请求头是否能被正则匹配|
|Method(`GET`, ...)|检查请求的方法是否匹配|
|Path(`/path`, `/articles/{cat:[a-z]+}/{id:[0-9]+}`, ...)|检查请求路径是否匹配，可以用正则|
|PathPrefix(`/products/`, `/articles/{cat:[a-z]+}/{id:[0-9]+}`)|匹配路径前缀，可以用正则|
|Query(`foo=bar`, `bar=baz`)|检查query params|

**注意事项**：

* `Host` 和 `HostRegexp` 不支持除了ASCII以外的字符，所以不支持中文域名
* regexp 支持golang支持的所有格式
* 使用 AND (&&) 和 OR (||) 运算符组合多个匹配器。可以使用括号进行组合  
* 转发规则和中间件都是在到达服务之前生效
* Path只转发到确定路径上，`/who/`与`/who` 不等价

#### 优先级

`traefik` 默认情况下，按规则长度降序排序匹配，所以最长长度具有最高优先级。

如以下：

```yaml
## Dynamic configuration
http:
  routers:
    Router-1:
      rule: "HostRegexp(`.*\.traefik\.com`)"
      # ...
    Router-2:
      rule: "Host(`foobar.traefik.com`)"
      # ...
```

`Router-1` 是30个字符，而 `Router-2` 是26个字符，所以所有`foobar.traefik.com`请求都会被第一个`Router-1`所捕获。

可以通过设置`priority`值，来指定匹配优先级（降序），设置为0表示按默认匹配：

```yaml
http:
  routers:
    Router-1:
      rule: "HostRegexp(`.*\.traefik\.com`)"
      entryPoints:
      - "web"
      service: service-1
      priority: 1
    Router-2:
      rule: "Host(`foobar.traefik.com`)"
      entryPoints:
      - "web"
      priority: 2
      service: service-2
```

### Middlewares

中间件只在路由规则匹配，且请求到达服务处理之前生效。中间件的执行**顺序**与其在`Router`中声明顺序相同。

示例：

```yaml
http:
  routers:
    router1:
      service: myService
      middlewares:
        - "foo-add-prefix"
      rule: "Host(`example.com`)"

  middlewares:
    foo-add-prefix:
      addPrefix:
        prefix: "/foo"

  services:
    service1:
      loadBalancer:
        servers:
          - url: "http://127.0.0.1:80"
```

**可用中间件列表**:

|中间件|目的|作用域|
|:--|:--|:--|
|AddPrefix|添加路径前缀|Path路径|
|BasicAuth|Basic Auth机制|Authentication|
|Buffering|缓存请求/相应|请求生命周期|
|Chain|组合多个中间件|中间件工具|
|CircuitBreaker|停止调用不健康服务|请求生命周期|
|Compress|压缩响应|Content阶段|
|DigestAuth|摘要身份验证（md5）|Authentication|
|Errors|自定义错误页面|请求生命周期|
|ForwardAuth|委托认证（调用第三方提供认证）|Authentication|
|Headers|添加或者修改头部|Security|
|IPWhiteList|Ip白名单|请求生命周期|
|InFlightReq|限制同时链接数|请求生命周期|
|PassTLSClientCert|在Header中添加客户端证书|Security|
|RateLimit|限制会话频率|Security|
|RedirectScheme|重定向|请求生命周期|
|RedirectRegex|重定向（正则匹配）|请求生命周期|
|ReplacePath|修改请求路径|Path路径|
|ReplacePathRegex|修改请求路径（正则匹配）|Path路径|
|Retry|出现错误时自动重试请求|请求生命周期|
|StripPrefix|更改请求路径|Path路径|
|StripPrefixRegex|更改请求路径|Path路径|


## 0x01 安装与启动

### Docker

```
docker run -d -p 8080:8080 -p 80:80 -v $pwd/traefik.yaml:/etc/traefik/traefik.yml traefik:v2.4
```

### 二进制文件

linux
```
# Compare this value to the one found in traefik-${traefik_version}_checksums.txt
sha256sum ./traefik_${traefik_version}_linux_${arch}.tar.gz
```

mac
```
# Compare this value to the one found in traefik-${traefik_version}_checksums.txt
shasum -a256 ./traefik_${traefik_version}_darwin_amd64.tar.gz
```

windows(powershell)
```
Get-FileHash ./traefik_${traefik_version}_windows_${arch}.zip -Algorithm SHA256
```

### 基础配置（静态配置）

`Traefik` 分为静态配置和动态配置，静态配置是启动所必须之配置项；动态配置是代理服务转发的配置项。

静态配置可以通过以下途径配置:

* 启动命令行参数, `traefik --help`查看
* 配置文件
* 环境变量

配置文件默认检索`/etc/traefik/traefik.yml`或`yaml` `toml`后缀的文件。

可以通过命令行指定：

```
traefik --configFile=foo/bar/myconfigfile.yml
```

## 0x02 配置

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

----

> 语法：forwardingTimeouts:
> 
> 默认值：无
> 
> 作用域：serversTransport

将请求转发到服务端的一些超时配置。

----

> 语法：dialTimeout 1s
> 
> 默认值：30s
> 
> 作用域：serversTransport

建立到后端的链接最大超时时间，0表示不超时控制。

```yaml
## Static configuration
serversTransport:
  forwardingTimeouts:
    dialTimeout: 1s
```

----

> 语法：responseHeaderTimeout 1s
> 
> 默认值：0s
> 
> 作用域：serversTransport

请求完全写入成功之后，读取到响应头的时间，0表示不超时。

```yaml
## Static configuration
serversTransport:
  forwardingTimeouts:
    responseHeaderTimeout: 1s
```

----

> 语法：idleConnTimeout 1s
> 
> 默认值：90s
> 
> 作用域：serversTransport

空闲（保持活动）连接在关闭之前保持空闲的最长时间，0表示不限制。

```yaml
## Static configuration
serversTransport:
  forwardingTimeouts:
    idleConnTimeout: 1s
```

----


### EntryPoints

----

> 语法：address [host]:prot[/tcp | /udp]
> 
> 默认值：无
> 
> 作用域：EntryPoints

指定监听的端口和协议。

```yaml
entryPoints:
  web:
    address: ":80"

  websecure:
    address: ":443"
```
----

> 语法：forwardedHeaders
> 
> 默认值：无
> 
> 作用域：EntryPoints

信任转发的`X-Forwarded-*`头信息。

```
## Static configuration
entryPoints:
  web:
    address: ":80"
    forwardedHeaders:
      insecure: true   # 所有的拓展请求头都会被转发
```
----

> 语法：trustedIPs
> 
> 默认值：无
> 
> 作用域：EntryPoints

信任ip白名单

```
entryPoints:
  web:
    address: ":80"
    forwardedHeaders:
      trustedIPs:
        - "127.0.0.1/32"
        - "192.168.1.7"
```
----


## 0x03 实践

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

### 使用第三方验证器验证

```yaml
http:
  routers:
    # Define a connection between requests and services
    to-whoami:
      # rule: "Host(`example.com`) && PathPrefix(`/whoami/`)"
      rule: "PathPrefix(`/whoami/`)"
       # If the rule matches, applies the middleware
      middlewares:
      - test-auth

      # If the rule matches, forward to the whoami service (declared below)
      service: whoami

  middlewares:
    # Define an authentication mechanism
    # test-user:
    #   basicAuth:
    #     users:
    #     - test:$apr1$H6uskkkW$IgXLP6ewTrSuBkTrqE8wj/
    test-auth:
      forwardAuth:
        address: http://192.168.1.30:1323/auth
        authResponseHeaders:
          - "X-Auth-User"

  services:
    # Define how to reach an existing service on our infrastructure
    whoami:
      loadBalancer:
        servers:
        - url: http://192.168.1.30:1323
```

`authResponseHeaders` 将验证请求返回的头，加入到原请求的请求头中。用于添加验证后的状态信息，如会员信息。
