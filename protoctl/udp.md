# UDP

## 概述

**UDP**是无连接，不可靠数据报协议，不同于TCP面向连接的字节流形式。UDP无需维护连接，也就没有了握手的RTT消耗，同时因为**UDP**没有要求数据包一定保持顺序，所以避免了**队头阻塞**问题。

> 队头阻塞（Head-of-line blocking, HOL blocking）是计算机网络中的一种性能受限现象。它的原因是因为一列的第一个数据包（队头）受阻，而导致整列受阻。它有可能在缓存式交换机，HTTP流水线等多个情况下出现。

一次双向的，**UDP**会话：

![UDP](../images/udp.png)

**UDP**的特性决定了，他天然有以下缺点：

* 无法提供可靠的字节流
* 客户端无法感知服务器的状态，仅能通过超时来判断

## 快速开始

### 简单例子

服务端：
```go
package main
import (
	"fmt"
	"net"
)

func main() {
	listener, err := net.ListenUDP("udp", &net.UDPAddr{IP: net.ParseIP("127.0.0.1"), Port: 9981})
	if err != nil {
		fmt.Println(err)
		return
	}
	fmt.Printf("Local: <%s> \n", listener.LocalAddr().String())
	data := make([]byte, 1024)
	for {
		n, remoteAddr, err := listener.ReadFromUDP(data)
		if err != nil {
			fmt.Printf("error during read: %s", err)
		}
		fmt.Printf("<%s> %s\n", remoteAddr, data[:n])
		_, err = listener.WriteToUDP([]byte("world"), remoteAddr)
		if err != nil {
			fmt.Printf(err.Error())
		}
	}
}
```

客户端：

```go
package main
import (
	"fmt"
	"net"
)

func main() {
	sip := net.ParseIP("127.0.0.1")
	srcAddr := &net.UDPAddr{IP: net.IPv4zero, Port: 0}
	dstAddr := &net.UDPAddr{IP: ip, Port: 9981}
	conn, err := net.DialUDP("udp", srcAddr, dstAddr)
	if err != nil {
		fmt.Println(err)
	}
	defer conn.Close()
	conn.Write([]byte("hello"))
	fmt.Printf("<%s>\n", conn.RemoteAddr())
}
```

## Write与Read系列犯法

在客户端中， 使用`Write`与`Read`方法读写数据，而在服务端中，使用`WriteToUDP`和`ReadFromUDP`。其实二者，本质并没有什么区别，虽然UDP是无连接的，但是为了实现上的一致，`*UDPConn`结构体中，还是继承了`Connected`这样的状态。

客户端知道了连接四元组中的，所有元素，所以我们认为他是连接的；而服务端只有收到消息才能知道客户端的存在，所以他是未连接的。

* 已连接`*UDPConn`使用`Write`和`Read`
* 未连接`*UDPConn`使用`WriteToUDP`和`ReadFromUDP`

## 传输方式

计算机网络中，有以下四种传输方式:

* 单播(unicast)：封包向单一的地址发送包，如一切基于TCP的协议都是此模式
* 组播(multicast)：也叫多播，多点广播或者群播。指消息同时传输给一组目的地址，消息在每条网络链路上只传递一次
* 广播(broadcast): 封包传递给一个广播域，所有设备也是限定在一个范围之中
* 任播(anycast)：是一种网络寻址和路由的策略，使得资料可以根据路由拓朴来决定送到**最近**或**最好**的目的地

`UDP`很好的支持了这四种模式。
