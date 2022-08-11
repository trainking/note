# Socket网络编程

`Socket`是套接字的统称，是网络编程的基础。在`Windows`上，一个socket对象类型是`SOCKET`，是一个句柄对象。而在Linux上，一个scoket对象的类型是int，与文件描述符fd兼容，体现Linux的**一切皆是文件**的思想。

很多网络库中，会定义一个`SOCK_TYPE`类型来屏蔽二者的差别。

## Socket通信实例

服务器:

```cpp
/**
 * @file server.cpp
 * @author your name (you@domain.com)
 * @brief 
 * @version 0.1
 * @date 2022-08-06
 * TCP服务器通信流程
 * @copyright Copyright (c) 2022
 * 
 */
#include <sys/types.h>
#include <sys/socket.h>
#include <arpa/inet.h>
#include <unistd.h>
#include <iostream>
#include <string.h>

int main(int argc, char* argv[])
{
    // 1. 创建一个Socket
    int listenfd = socket(AF_INET, SOCK_STREAM, 0);
    if (listenfd == -1)
    {
        std::cout << "create listen sockt error."<< std::endl;
        return -1;
    }

    // 2. 初始化服务器地址
    struct sockaddr_in bindaddr;
    bindaddr.sin_family = AF_INET;
    bindaddr.sin_addr.s_addr = htonl(INADDR_ANY);
    bindaddr.sin_port = htons(3000);
    if (bind(listenfd, (struct sockaddr *)&bindaddr, sizeof(bindaddr)) == -1)
    {
        std::cout << "bind listen socket error." << std::endl;
        return -1;
    }

    // 3. 启动监听
    if (listen(listenfd, SOMAXCONN) == -1)
    {
        std::cout << "listen error." << std::endl;
        return -1;
    }

    std::cout << "server is start" << std::endl;

    while(true)
    {
        struct sockaddr_in clientaddr;
        socklen_t clientaddrlen = sizeof(clientaddr);
        // 4. 接受客户端连接
        int clientfd = accept(listenfd, (struct sockaddr *)&clientaddr, &clientaddrlen);
        if (clientfd != -1)
        {
            char recvBuf[32] = {0};
            int ret = recv(clientfd, recvBuf, 32, 0);
            if (ret > 0)
            {
                std::cout << "recv data from client, data: " << recvBuf << std::endl;
                // 6. 将收到的数据Echo回给客户端
                ret = send(clientfd, recvBuf, strlen(recvBuf), 0);
                if (ret != strlen(recvBuf))
                    std::cout << "send data error." << std::endl;
                else
                    std::cout << "send data to client successfully, data: " << recvBuf << std::endl;
            } else {
                std::cout << "recv data error" << std::endl;
            }

            close(clientfd);
        }
    }
    close(listenfd);
    return 0;
}
```

客户端:

```cpp
/**
 * @file client.cpp
 * @author your name (you@domain.com)
 * @brief 
 * @version 0.1
 * @date 2022-08-12
 * 
 * @copyright Copyright (c) 2022
 * 
 */
#include <sys/types.h>
#include <sys/socket.h>
#include <arpa/inet.h>
#include <unistd.h>
#include <iostream>
#include <string.h>

#define SERVER_ADDRESS "127.0.0.1"
#define SERVER_PORT 3000
#define SEND_DATA "HELLOWORLD"

int main(int argc, char* argv[])
{
    // 1. 创建一个socket
    int clientfd = socket(AF_INET, SOCK_STREAM, 0);
    if (clientfd == -1)
    {
        std::cout << "create client socket error." << std::endl;
        return -1;
    }

    // 2. 连接服务器
    struct sockaddr_in serveraddr;
    serveraddr.sin_family = AF_INET;
    serveraddr.sin_addr.s_addr = inet_addr(SERVER_ADDRESS);
    serveraddr.sin_port = htons(SERVER_PORT);
    if (connect(clientfd, (struct sockaddr *)&serveraddr, sizeof(serveraddr)) == -1)
    {
        std::cout << "connect data error." << std::endl;
        return -1;
    }

    int ref = send(clientfd, SEND_DATA, strlen(SEND_DATA), 0);
    if (ref != strlen(SEND_DATA))
    {
        std::cout << "send data error." << std::endl;
        return -1;
    }
     
    std::cout << "send data successfully, data: " << SEND_DATA << std::endl;

    // 4. 从服务器收取数据
    char recvBuf[32] = {0};
    int ret = recv(clientfd, recvBuf, 32, 0);
    if (ret > 0)
    {
        std::cout << "recv data successfully, data: " << recvBuf << std::endl;
    } else {
        std::cout << "recv data error, data: "<< recvBuf <<std::endl;
    }

    close(clientfd);
    
    return 0;
}
```