# websocket

## websocket解决什么问题？

解决http短链接的，`请求-响应`模式下，无法获取服务器及时更新的问题。`websocket`是`订阅-发布`模式，由第一次的http请求，向服务器订阅一类消息。`websocket`协议是兼容`http`协议的，`ws`默认端口是**80**，`wss`默认端口是`443`。
