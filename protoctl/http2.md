# http2

- [http2](#http2)
  - [http1.1所遇到的问题](#http11所遇到的问题)
    - [1.高延迟带来页面加载速度的降低](#1高延迟带来页面加载速度的降低)
    - [2. 无状态特性带来的巨大的HTTP头部](#2-无状态特性带来的巨大的http头部)
    - [3. 无法被动感知服务器状态变更](#3-无法被动感知服务器状态变更)
  - [http2概述](#http2概述)
    - [http2特性](#http2特性)
    - [连接属性](#连接属性)
  - [协议](#协议)
    - [基于TCP建立会话](#基于tcp建立会话)
    - [基于TLS/SSL](#基于tlsssl)
      - [TLS通讯过程](#tls通讯过程)
    - [帧(Frame)](#帧frame)
      - [length](#length)
      - [Type](#type)
    - [消息(Message)](#消息message)
    - [流(Stream)](#流stream)

## http1.1所遇到的问题

### 1.高延迟带来页面加载速度的降低

网络带宽大幅提高之后，而延迟水平并没有大幅度的提高。主要有两个问题到导致：

* 物理距离的不可避免，例如一台在太平洋一端的服务器，另一端访问，物理延迟就高达200ms
* 最后一公里人问题，数据在到达客户端之前，经过太多交换机路由器设备，出现排队等待现象

另外http1.1本身的限制，加剧了这种现象。首先浏览器同时并发的连接数是有限的，(chrome)浏览器只支持6个，**同一个TCP连接同时只能允许一个HTTP事务完成**才能开始下一个事务。

### 2. 无状态特性带来的巨大的HTTP头部

应为HTPP遵循`REST`架构特性，每一个请求都要求是无状态的，每次请求/响应都必须带上巨大的头部。特别是`Cookie`这样头部。

### 3. 无法被动感知服务器状态变更

基于断连的http1.1，`请求/响应`模式下，只能通过轮询的方式，获取到服务端的变更。或者增加`websocket`作为补充协议的方式，实现服务器对浏览器的通知。

## http2概述

`http2`协议的前身是谷歌支持的`SPDY`协议，后续谷歌为支持`http2`协议，放弃了对`SPDY`的支持，定义在`RFC7540`中。

`http2`做出的修改：

* 在应用层上做出修改，并充分挖掘TCP协议性能
* `请求/响应`基本模型不变
* `scheme`不改变，还是使用`http://`，没有特定`http2://`
* `http1.x`协议的客户端和服务器可以无缝通过代理方式转接到http/2上
* 不能识别`http2`协议的代理服务器可以将请求降级到http1.x上
* `http2`支持多路复用，http事务不用在TCP连接上串行执行

### http2特性

* 传输数据量大幅减少
    - 以二进制方式传输(http1使用ASCII码传输)
    - 标头压缩
* 多路复用以及相关功能
    - 消息优先级
* 服务器消息推送
    - 并行推送

### 连接属性

* 连接Connection: 1个TCP连接，包含一个或者多个Stram
* 数据流Stream：一个双向通信的数据流，包含一个或者多条Message
* 消息Message：对应HTTP/1.1中的请求或者响应，包含一个或者多个Frame
* 数据帧Frame: 最小单位，以二进制压缩格式存放HTTP/1中的内容

## 协议

`http2`协议既可以基于`TCP`协议，也可以在`TLS`协议之上。然而在浏览器中，强制要求基于`TLS`协议，实现`http2`协议


### 基于TCP建立会话

在http请求头中, 使用`Connection`, `Upgrade`头升级为http2协议。 基于`TCP`协议的htt2通常被称为`h2c`。

```http
GET / HTTP/1.1
Connection: Upgrade, HTTP2-Settings
Upgrade: h2c
HTTP2-Settings: AAAABBDDCCDDFS__

HTTP/1.1 101 Switching Protocols
Connection: Upgrade
Upgrade: h2c
```

在这之后，客户端还需要向服务器发一个Magic帧，称之为`HTTP/2 Connection Preface`。

```
0x505249202a20485454502f322e300d0a0d0a534d0d0a0d0a
```

之后便是通用的发送设置帧

### 基于TLS/SSL

在TLS层ALPN（Application Layer Protocol Negotiation）拓展做协商，只认识http1.1的代理服务器不会干扰http2。基于TLS协议的http2，通常被称为`h2`。

#### TLS通讯过程

1. 验证身份：在第一步`client hello`与`sever hello`只见，如果服务器需要验证其身份，将发送证书确认
2. 达成安全套件共识：协商使用哪种加密算法，客户端发送支持的协议集合，服务器端选择使用（RFC7301）
3. 传递密钥：各自获取到相同的对称加密密钥
4. 加密通讯：传输的数据，都需要对称加密后发送

### 帧(Frame)

`Frame`有一下特性：

* 每一个Frame都有一个`Stream ID`标识它属于哪个`Stream`
* 有两种帧，`HEADERS`帧和`DATA`帧，两条共同组成一个Message
* 同一个`stream`传输中，帧必须有序，不同`stream`，帧在传输中无序，接收时组装

#### length

`Frame`的大小，由第一个3字节的长度表示。但一个帧的大小不宜过大，过大会导致接收端需要更大的缓存来处理组装。

* 所有的实现都可以接受16kB以下的帧
* 传递16KB到16MB的帧时，必须接收端首先公布自己可以处理的大小

#### Type

使用一个字节表示帧的类型，如下：

|帧类型|编码|用途|
|:--|:--|:--|
|DATA|0x0|传递HTTP包体|
|HEADERS|0x1|传递HTTP头部|
|REIORITY|0x2|指定Stream流的优先级|
|SETTINGS|0x4|修改连接或者Stream流的配置|
|PUSH_PROMISE|0x5|服务端推送资源时描述请求的帧|
|PING|0x6|心跳检测，兼具计算RTT往返时间的功能|
|GOAWAY|0x7|优雅的终止连接，或者通知错误|
|WINDOW_UPDATE|0x8|实现流量控制|
|CONTINUATION|0x9|传递较大HTTP头部时的持续帧|

设置帧：

* Setting帧不是协商，而是通知接收端其的特性和能力
* 一个设置帧可以同时设置多个对象`Identifier-Value`，类`Key-Vale`格式

支持的设置类型：

|设置类型|编码|用途|
|:--|:--|:--|
|SETTINGS_HEADER_TABLE_SIZE|0x01|通知对端索引表的最大尺寸，单位字节，最大4096|
|SETTINGS_ENABLE_PUSH|0x02|0禁用服务器端推送功能，1启用|
|SETTINGS_MAX_CONCURRENT_STREAM|0x03|高度接受端允许的最大并发数量|
|SETTINGS_INITIAL_WINDOW_SIZE|0x04|声明发送端的接口大小，用与Stream级别流控，初始值65535字节|
|SETTINGS_MAX_FRAME_SIZE|0x05|设置帧的最大大小|
|SETTINGS_MAX_HEADER_LIST_SIZE|0x06|知会对端索引表的最大尺寸，单位为字节，基于未压缩头部|

### 消息(Message)

`Message`是一个业务层面的概念，无法从帧的层面获知。 

### 流(Stream)

`Stream`通过`Stream ID`来标识，接收端可以根据此，并发的组装消息。

* 同一个`Stream`中的Frame必须有序的，无法并发
* 控制帧`SETTINGS_MAX_CONCURRENT_STREAMS`控制着并发的`Stream`数
* 由客户端建立的`Stream`，`Stream ID`必须是奇数
* 由服务器端建立的`Stream`，`Stream ID`必须是偶数
* 心建立的`Stream`, ID必须大于曾经建立的状态为`Opened`或者`Reserved`的流的ID
* 在新建立的流上发送帧时，更小的ID的流处于`idle`状态时，置为close状态
* `Stream ID`不能复用，长连接耗尽ID，应该创建新的连接
* `Stream ID`为0的流，仅仅用来传输控制帧