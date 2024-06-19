---
title: 从0到1实现基于协程的WebServer(2) —— socket与IO多路复用
tags:
  - socket
  - IO多路复用
categories:
  - WebServer
date: 2024-06-19 19:38:24
---
# 如何建立网络通信
在 linux 系统中，如果想要在客户端和服务器之间建立网络连接比如 `TCP` 或者 `UDP` 连接，需要借助 socket 相关系统调用来设置地址和端口并建立连接。
## 建立服务端程序
一个简易的 echoserver 程序如下所示，该服务端程序可以接受一个客户端发来的消息并返回回去。
```cpp
#include <arpa/inet.h>
#include <cerrno>
#include <iostream>
#include <netinet/in.h>
#include <sys/socket.h>
#include <unistd.h>

int main() {
  int fd = socket(AF_INET, SOCK_STREAM, 0);
  if (fd < 0) {
    return 0;
  }
  int opt_val = 1;
  setsockopt(fd, SOL_SOCKET, SO_REUSEADDR, &opt_val, sizeof(opt_val));
  struct sockaddr_in addr {.sin_family = AF_INET};
  addr.sin_port = htons(8848);
  inet_pton(AF_INET, "127.0.0.1", &addr.sin_addr.s_addr);
  if (bind(fd, (sockaddr *)&addr, sizeof(addr)) < 0) {
    return 0;
  };
  listen(fd, 64);
  struct sockaddr client_addr;
  socklen_t len = 0;
  int conn = accept(fd, &client_addr, &len);

  char buf[1024];
  int rd_len = read(conn, buf, sizeof(buf));
  std::cout << rd_len << ':' << buf << std::endl;
  write(conn, buf, rd_len);
}
```
从创建一个 socket 到给客户端发送数据的流程如下：
1. 使用 `int fd = socket(AF_INET, SOCK_STREAM, 0);` 创建socket，并设置其协议为 ipv4 和 tcp 。
2. `setsockopt(fd, SOL_SOCKET, SO_REUSEADDR, &opt_val, sizeof(opt_val));` 设置该 socket 地址可重用，允许多个 socket 绑定到同一个地址。可以避免连接关闭后立即重新绑定失败。
3. `bind(fd, (sockaddr *)&addr, sizeof(addr))` 将 socket 绑定到指定的 ip地址和端口上。
4. `listen(fd, 64);` socket 进入监听。
5. `accept(fd, &client_addr, &len)` 监听客户端连接。
6. `read(conn, buf, sizeof(buf));` 读取客户端数据。
7. `write(conn, buf, rd_len);` 给客户端发送数据。
## 如何处理多个连接
之前的例子中的 io 操作都是同步阻塞的，包括 `accept`、`read`、`write` 等。这就会带来一个问题，如果在循环中处理客户端的 io 读写事件的话，会非常耗时，如果此时其他客户端再请求连接的话，往往会被阻塞等待。

为了解决上述问题，有些人可能会想到使用多线程技术，将客户端的事件处理另起一个线程处理，而在主线程中不断等待客户端的连接。这确实能够解决同步 io 阻塞带来的问题。然而现实中服务器的网络连接通常都在数千以上，这就意味着服务器要同时进行数千个线程的调度，这对于 CPU 的开销是非常大的。

幸好，linux 内核提供了 IO 多路复用，可以在一个线程内处理多个请求：`select`、`poll`、`epoll`。
### select
### poll
### epoll