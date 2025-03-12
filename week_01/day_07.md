## 一. `select` 系统调用详解

### 1. 功能与用途
`select` 是一种 **I/O 多路复用** 机制，允许程序同时监视多个文件描述符（如套接字、管道等）的状态变化，包括：
- **可读事件**（数据到达、连接关闭等）。
- **可写事件**（缓冲区可写入）。
- **异常事件**（带外数据或错误）。

适用于需要同时处理多个 `I/O` 操作的场景（如网络服务器、聊天程序）。

### 2. 函数原型
```c
#include <sys/select.h>

int select(
    int nfds,                // 最大文件描述符值 + 1
    fd_set *readfds,         // 监听可读事件的文件描述符集合
    fd_set *writefds,        // 监听可写事件的文件描述符集合
    fd_set *exceptfds,       // 监听异常事件的文件描述符集合
    struct timeval *timeout  // 超时时间（NULL 表示阻塞等待）
);
```

### 3. 核心参数说明
- `nfds`指定被监听文件描述符的最大值 + 1
- `readfds/writefds/exceptfds`: 使用 `fd_set` 结构表示文件描述符集合。
```c
FD_ZERO(fd_set *set);      // 清空集合
FD_SET(int fd, fd_set *set);   // 添加 fd 到集合
FD_CLR(int fd, fd_set *set);   // 从集合中移除 fd
FD_ISSET(int fd, fd_set *set); // 检查 fd 是否在集合中
```
- `timeout`: 指定超时时间，若 `timeout = NULL` -> 阻塞直到事件发生。若 `timeval = {0, 0}` -> 非阻塞模式，立即返回。
```c
struct timeval {
    long tv_sec;   // 秒
    long tv_usec;  // 微秒
};
```

### 4. 返回值
- 成功：返回就绪的文件描述符总数。
- 超时：返回 `0`。
- 错误：返回 `-1`，并设置 `errno`（如 `EINTR` 表示被信号中断）。

### 5. 文件描述符就绪条件
#### 5.1 可读 (`readfds`)
- 接收缓冲区数据量 ≥ 低水位标记 `SO_RCVLOWAT`。
- 对端关闭连接（`read` 返回 `0`）。
- 监听套接字有新连接请求。
- 套接字有未处理的错误。
#### 5.2 可写 (`writefds`)
- 发送缓冲区可用空间 ≥ 低水位标记 `SO_SNDLOWAT`。
- 写操作被关闭（触发 `SIGPIPE`）。
- 非阻塞 `connect` 完成（成功或失败）。
- 套接字有未处理的错误。
#### 5.3 异常 (`exceptfds`)
- 仅当接收到 带外数据（`OOB`） 时触发。

---
