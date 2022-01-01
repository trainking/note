#arpc

## 概述

`arpc`是一个简单，轻量的`rpc`框架。

## 快速开始

开启一个服务

```go
package main

import "github.com/lesismal/arpc"

func main() {
	server := arpc.NewServer()

	// register router
	server.Handler.Handle("/echo", func(ctx *arpc.Context) {
		str := ""
		if err := ctx.Bind(&str); err == nil {
			ctx.Write(str)
		}
	})

	server.Run("localhost:8888")
}
```