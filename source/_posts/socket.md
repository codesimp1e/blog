---
title: 【理论篇】从0到1实现基于协程的WebServer(2) —— socket与IO多路复用(select和poll)
tags:
  - socket
  - IO多路复用
  - select
  - poll
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
  while(true){
    int conn = accept(fd, &client_addr, &len);

    char buf[1024];
    int rd_len = read(conn, buf, sizeof(buf));
    std::cout << rd_len << ':' << buf << std::endl;
    write(conn, buf, rd_len);
  }
  
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
# 如何处理多个连接
之前的例子中的 io 操作都是同步阻塞的，包括 `accept`、`read`、`write` 等。这就会带来一个问题，如果在循环中处理客户端的 io 读写事件的话，会非常耗时，如果此时其他客户端再请求连接的话，往往会被阻塞等待。

为了解决上述问题，有些人可能会想到使用多线程技术，将客户端的事件处理另起一个线程处理，而在主线程中不断等待客户端的连接。这确实能够解决同步 io 阻塞带来的问题。然而现实中服务器的网络连接通常都在数千以上，这就意味着服务器要同时进行数千个线程的调度，这对于 CPU 的开销是非常大的。

幸好，linux 内核提供了 IO 多路复用的能力，即内核一旦发现进程指定的一个或多个 IO 条件已就绪就通知进程，可以在一个线程内处理多个请求：`select`、`poll`、`epoll`。
### select
`select` 函数允许程序监视多个文件描述符，等待一个或多个文件描述符变为准备就绪状态，以进行相应的 I/O 操作。`select` 函数定义如下：
```c
int select(int nfds, fd_set *readfds, fd_set *writefds, fd_set *exceptfds, struct timeval *timeout);
```
参数：
- `nfds`：要监视的最大文件描述符加一。
- `readfds`：待监视的读文件描述符集合。
- `writefds`：待监视的写文件描述符集合。
- `exceptfds`：待监视的异常文件描述符集合。
- `timeout`：超时时间。NULL表示阻塞。

其中 `fd_set` 类型是位图形式，通过以下宏定义来对其进行操作：
```cpp
// 将 fd 从集合 set 中删除
void FD_CLR(int fd, fd_set *set);
// 检查 fd 是否在集合 set 中
int  FD_ISSET(int fd, fd_set *set);
// 将 fd 添加到集合 set 中
void FD_SET(int fd, fd_set *set);
// 将集合 set 清空
void FD_ZERO(fd_set *set);
```
对于上面的简易 echoserver 代码可以使用 `select` 进行改进，使得对应的读文件描述符数据在内核中准备好时再进行读取操作，避免等待一个 IO 操作完成再进行下一次操作。对于已经进入 listen 状态的socket：fd，我们可以：
1. 使用 `FD_SET(fd, &readfds_all)` 其添加到读集合中
2. 使用 `select` 函数来监听集合的读事件
3. 使用 `FD_ISSET(fd, &readfds)` 判断 fd 是否准备好进行 IO 操作，若准备好则可以进行 `accept` 操作获取对端连接的文件描述符。

如下所示：
```c++
fd_set readfds_all;
FD_ZERO(&readfds_all);
int max_fd = fd;
FD_SET(fd, &readfds_all);
while (true) {
  fd_set readfds = readfds_all;
  int ret = select(max_fd + 1, &readfds, NULL, NULL, NULL);
  if (ret < 0) {
    perror("select");
    return 0;
  }
  if (FD_ISSET(fd, &readfds)) {
    int conn = accept(fd, (sockaddr *)&client_addr, &len);
    if (conn < 0) {
      perror("accept");
      return 0;
    }
    if (ret == 1) {
      continue;
    }
  }
}
```
其中 `readfds_all` 用于保存所有需要监听的套接字。由于 `select` 会改变传入的 readfds 参数，用于表示已就绪的文件描述符，所以需要一个拷贝一份`readfds_all`用于表示已就绪的文件描述符。

在进行完 accept 之后，同样需要对新的文件描述符 `conn` 监听可读事件，所以需要将 `conn` 添加到 `readfds_all` 中, 同时为了方便管理，将 `conn` 添加到  `connected_fds` 中，在断开时可以直接删除。完整代码：
```c++
#include <algorithm>
#include <arpa/inet.h>
#include <cerrno>

#include <cstdio>
#include <netinet/in.h>
#include <set>
#include <sys/select.h>
#include <sys/socket.h>
#include <unistd.h>

int main() {
  int fd = socket(AF_INET, SOCK_STREAM, 0);
  if (fd < 0) {
    return 0;
  }
  int opt_val = 1;
  setsockopt(fd, SOL_SOCKET, SO_REUSEADDR, &opt_val, sizeof(opt_val));
  struct sockaddr_in addr {
    .sin_family = AF_INET
  };
  addr.sin_port = htons(8848);
  inet_pton(AF_INET, "127.0.0.1", &addr.sin_addr.s_addr);
  if (bind(fd, (sockaddr *)&addr, sizeof(addr)) < 0) {
    return 0;
  };
  struct sockaddr_in client_addr;
  socklen_t len = sizeof(client_addr);
  listen(fd, 64);
  char buf[1024];
  fd_set readfds_all;
  std::set<int> connected_fds;
  FD_ZERO(&readfds_all);
  int max_fd = fd;
  FD_SET(fd, &readfds_all);
  while (true) {
    fd_set readfds = readfds_all;
    int ret = select(max_fd + 1, &readfds, NULL, NULL, NULL);
    if (ret < 0) {
      perror("select");
      return 0;
    }
    if (FD_ISSET(fd, &readfds)) {
      int conn = accept(fd, (sockaddr *)&client_addr, &len);
      if (conn < 0) {
        perror("accept");
        return 0;
      }
      max_fd = std::max(conn, max_fd);
      connected_fds.insert(conn);
      FD_SET(conn, &readfds_all);
      if (ret == 1) {
        continue;
      }
    }
    for (auto it = connected_fds.begin(); it != connected_fds.end();) {
      int conn = *it;
      int ret_len = 0;
      if (FD_ISSET(conn, &readfds)) {
        ret_len = read(conn, buf, sizeof(buf));
        if (ret_len < 0) {
          perror("read");
          return 0;
        }
        if (ret_len == 0) {
          close(conn);
          FD_CLR(conn, &readfds_all);
          it = connected_fds.erase(it);
          continue;
        }
      }
      it++;
      write(conn, buf, ret_len);
    }
  }
}
```
## poll
