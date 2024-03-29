# websocket

- [websocket](#websocket)
  - [websocket解决什么问题？](#websocket解决什么问题)
  - [websocket协议的约束](#websocket协议的约束)
  - [协议格式](#协议格式)
    - [帧](#帧)
      - [帧类型](#帧类型)
      - [帧长度](#帧长度)
    - [1.建立会话](#1建立会话)
      - [格式](#格式)
      - [请求头部参数](#请求头部参数)
      - [返回头部](#返回头部)
        - [Accept生成规则](#accept生成规则)
    - [2. 传输消息](#2-传输消息)
    - [3. 掩码处理](#3-掩码处理)
    - [4. 会话维持](#4-会话维持)
    - [5. 关闭会话](#5-关闭会话)

## websocket解决什么问题？

解决http短链接的，`请求-响应`模式下，无法获取服务器及时更新的问题。`websocket`是`订阅-发布`模式，由第一次的http请求，向服务器订阅一类消息。`websocket`协议是兼容`http`协议的，`ws`默认端口是**80**，`wss`默认端口是`443`。

几点注意：

* websocket标准: rfc6455
* 支持拓展，如：permessage-deflate

## websocket协议的约束

* `websocket`牺牲了简单性，为了维护`websocket`链接，需要付出许多成本，例如当服务器想要做负载均衡时，要把消息的连接和处理解决，形成了`负载均衡-连接服务-消息分发-逻辑服务`这样的分层结构。
* `websokcet`长连接需要基于`ping/pong`心跳机制维持
* 兼容`HTTP`协议
* 本质上是，在web约束下，暴露TCP给应用层
* 元素据不做任何编码，需要开发者自定义`序列化/反序列化`数据格式，常用`protobuf`等
* 基于帧而不是基于流，数据帧只能承载两种数据，**字符**或者**二进制**数据
* 同样有同源策略问题（非浏览器无效）
* 支持定义子协议

## 协议格式

### 帧

帧中使用头两个字节保存头部，内容有：

* FIN 表示一条消息的结尾，等于1表示这是消息的最后一帧
* RSV 保留值(RSV1/RSV2/RSV3)，默认都是0，只有使用拓展时，由拓展决定其值
* opcode 帧的类型，`0` 持续帧，`1~7` 非控制帧，`8-F`控制帧，参考下面
* frame-masked: 掩码位，1 使用掩码
* payload-len, 负载数据的长度，只能使用7位，1位被`frame-masked`占用

#### 帧类型

opcode取值

|opcode|类型|
|:--|--|
|0|继续前一帧|
|1|文本帧，（UTF-8）|
|2|二进制帧|
|3-7|为非控制帧保留|
|8|关闭帧|
|9|心跳帧 ping|
|A|心跳帧 pong|
|B-F|为控制帧保留|

#### 帧长度

消息内容长度=应用消息长度+拓展数据长度

```
if payloadLen <= 125:
    仅仅使用payloadLen
elseif payloadLen == 126:
    再拓展16位（2个字节）作为拓展长度
elseif payloadLen == 127:
    再拓展64位（8个字节）作为拓展长度
```

### 1.建立会话

#### 格式

* ws-URI = "ws://host[:port]path[?query]" 默认端口80
* wss-URI = "wss://host[:port]path[?query]" 默认端口443

#### 请求头部参数

GET 请求

* Sec-WebSocket-Key: 握手随机数
* Sec-WbSocket-Version: 协议版本，目前13
* Connection: 必须有`Upgrade`
* Upgrade: websocket， 升级协议
* Sec-WebSocket-Protocol: 选择子协议，非必需
* Sec-WebSocket-Extensions: 拓展协议，非必须
* Origin: 跨域，非必须


#### 返回头部

返回码必须是`101`

* Connection: 必须有`Upgrade`
* Upgrade: websocket， 升级协议
* Sec-WebSocket-Accept: 根据请求随机数生成的base64编码

##### Accept生成规则

```
Base64(SHA1(随机数+GUID))
```
GUID: 258EAFA5-E914-47DA-95CA-C5AB0DC85B11

### 2. 传输消息

Message 消息：

* 1条消息由1或者多条数据帧组成，这些帧属统一类型
* 代理服务器可以合并拆分数据帧
* 发送消息时，会话必须处于`Open`状态
* **客户端**发送的帧，必须基于掩码编码
* 一旦发送或者接收到关闭帧，则处于`Close`状态
* 当TCP链接关闭后，WebSocket链接才处于完全关闭状态

### 3. 掩码处理

解决代理服务器无法识别websocket，而导致的缓存污染攻击。

`frame-masked`为1时，生成随机的32喂`fram-masking-key`, 放在`playload Data`之前。

数据按4字节（32位）分片，与 `fram-masking-key`求异或

### 4. 会话维持

通过维护心跳帧，维持会话：

ping 帧：

* opcode=9
* 可能含有数据

Pong 帧：

* opcode=A
* 必须与ping帧相同

### 5. 关闭会话

* 关闭时，必须双向关闭
* 正常情况，在TCP连接断开之前，双向关闭
* 关闭帧可能含有数据，但仅限说明关闭原因，遵循mask掩码规则

关闭帧错误码

|错误码|含义|
|:--|--|
|1000|正常关闭|
|1001|表示浏览器页面跳转或者服务器将要关机|
|1002|发现协议错误|
|1003|接收到不能处理的数据帧|
|1004|预留|
|1005|预留，（不能用在关闭帧里）期望但没有接收到错误码|
|1006|预留，（不能用在关闭帧里）期望给出非正常关闭的错误码|
|1007|消息格式不符合opecode（例如文本帧内容没有用UTF-8编码）|
|1008|接收到的消息，不遵守某些策略|
|1009|消息超出最大长度|
|1010|客户端明确需要使用拓展，但服务器没有给出拓展协商信息|
|1011|服务器遇到未知条件不能完成请求|
|1015|预留，（不能用在关闭帧里）表示TLS握手失败|

## 面试题

### 1. WebSocket是否存在tcp的粘包问题？

标准实现的`WebSocket`协议是`Message-Based`协议，本身会完成包的分片和组合，又因为一个websocket的Message最大长度可以高达9,223,372,036,854,775,807 字节。所以，不会又粘包的问题。
