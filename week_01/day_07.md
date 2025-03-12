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
#### 可读 (`readfds`)
- 接收缓冲区数据量 ≥ 低水位标记 `SO_RCVLOWAT`。
- 对端关闭连接（`read` 返回 `0`）。
- 监听套接字有新连接请求。
- 套接字有未处理的错误。
#### 可写 (`writefds`)
- 发送缓冲区可用空间 ≥ 低水位标记 `SO_SNDLOWAT`。
- 写操作被关闭（触发 `SIGPIPE`）。
- 非阻塞 `connect` 完成（成功或失败）。
- 套接字有未处理的错误。
#### 异常 (`exceptfds`)
- 仅当接收到 带外数据（`OOB`） 时触发。

```c
#include <stdio.h>
#include <sys/select.h>

int main()
{
    fd_set read_fds;
    FD_ZERO(&read_fds);
    FD_SET(STDIN_FILENO, &read_fds);

    struct timeval timeout = {5, 0}; // 超时5秒

    int ret = select(STDIN_FILENO + 1, &read_fds, NULL, NULL, &timeout);
    if (ret == -1) {
        perror("select error");
    } else if (ret == 0) {
        printf("Timeout\n");
    } else {
        if (FD_ISSET(STDIN_FILENO, &read_fds)) {
            printf("Input is readable\n");
        }
    }
    return 0;
}
```

- 适用场景：低并发、跨平台程序。
- 核心步骤：
  - 初始化 `fd_set` 并设置关注的文件描述符。
  - 调用 `select` 等待事件。
  - 使用 `FD_ISSET` 检查就绪的文件描述符。
---

## 二. `poll` 系统调用详解

### 1. 功能与用途
`poll` 是一种 **I/O 多路复用** 机制，允许程序同时监视多个文件描述符（如套接字、管道等）的状态变化，包括：
- **可读事件**（数据到达、连接关闭等）。
- **可写事件**（缓冲区可写入）。
- **异常事件**（带外数据或错误）。

相较于 `select`，`poll` 解决了文件描述符数量限制问题，但性能仍为线性复杂度（`O(n)`），适用于中等并发场景。

### 2. 函数原型
```c
#include <poll.h>

int poll(
    struct pollfd *fds,  // pollfd 结构体数组
    nfds_t nfds,         // 数组长度
    int timeout          // 超时时间（毫秒）
);
```

### 3. 核心数据结构
```c
struct pollfd {
    int   fd;       // 文件描述符
    short events;   // 关注的事件（输入）
    short revents;  // 实际发生的事件（输出）
};
// POLLIN	 数据可读（包括普通和优先数据）
// POLLOUT	 数据可写
// POLLERR	 发生错误
// POLLHUP	 连接挂起（对端关闭）
// POLLNVAL	 文件描述符未打开
```

### 4. 参数说明
- `fds`指向 `pollfd` 结构体数组，每个元素描述一个文件描述符及其关注的事件。

```c
struct pollfd fds[2];
fds[0].fd = sockfd;      // 监视套接字
fds[0].events = POLLIN;  // 关注可读事件
fds[1].fd = STDIN_FILENO;
fds[1].events = POLLIN;
```
- `nfds`指定 `fds` 数组的长度（需遍历的文件描述符数量）。

- `timeout`
  - `-1`：阻塞直到事件发生。
  - `0`：非阻塞模式，立即返回。
  - `>0`：最多等待指定毫秒数。

### 5. 返回值
- 成功：返回就绪的文件描述符总数。
- 超时：返回 `0`。
- 错误：返回 `-1`，并设置 `errno`（如 `EINTR` 表示被信号中断）。

```c
#include <stdio.h>
#include <poll.h>
#include <unistd.h>

int main()
{
    struct pollfd fds[2];
    fds[0].fd = STDIN_FILENO;
    fds[0].events = POLLIN;
    fds[1].fd = sockfd;  // 假设 sockfd 是已建立的套接字
    fds[1].events = POLLIN;

    int timeout = 5000;  // 超时5秒

    while (1) {
        int ret = poll(fds, 2, timeout);
        if (ret == -1) {
            perror("poll error");
            break;
        } else if (ret == 0) {
            printf("Timeout\n");
            continue;
        }

        // 检查标准输入是否可读
        if (fds[0].revents & POLLIN) {
            char buf[1024];
            ssize_t len = read(STDIN_FILENO, buf, sizeof(buf));
            if (len > 0) {
                printf("Input: %.*s", (int)len, buf);
            }
        }

        // 检查套接字是否可读
        if (fds[1].revents & POLLIN) {
            // 处理套接字数据
        }
    }
    return 0;
}
```

- 适用场景：中等并发、需要处理较多文件描述符且跨平台兼容性要求不高的场景。
- 核心步骤：
  - 初始化 `pollfd` 数组，设置关注的事件。
  - 调用 `poll` 等待事件。
  - 遍历 `pollfd` 数组，检查 `revents` 处理就绪事件。

---

## 三. `epoll` 系统调用详解

### 1. 功能与用途
`epoll` 是 `Linux` 特有的 **高性能 I/O 多路复用** 机制，用于高效管理大量文件描述符（如套接字）。相较于 `select` 和 `poll`，`epoll` 显著提升了高并发场景下的性能，适合构建 `Web` 服务器、实时通信系统等。

### 2. `epoll_create` 创建 `epoll` 实例
```c
#include <sys/epoll.h>
int epoll_create(int size);
// 功能：创建一个内核事件表，用于存储用户关注的文件描述符和事件。
// 参数：size 仅作历史兼容，可忽略（内核自动动态调整大小）。
// 返回值：成功返回 epoll 文件描述符（epfd），失败返回 -1。
```

### 3. `epoll_ctl` 管理事件表
```c
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event);
// 功能：向事件表添加/修改/删除事件。
// epfd：epoll_create 返回的文件描述符。
// op：操作类型：
//     EPOLL_CTL_ADD：注册新事件。
//     EPOLL_CTL_MOD：修改已注册事件。
//     EPOLL_CTL_DEL：删除事件。
// fd：要操作的目标文件描述符。
// event：事件配置（结构体 epoll_event）。
```
```c
struct epoll_event {
    uint32_t     events;   // 事件类型（EPOLLIN/EPOLLOUT/EPOLLERR 等）
    epoll_data_t data;     // 用户数据（如关联的 fd）
};

typedef union epoll_data {
    void    *ptr;         // 自定义指针
    int      fd;          // 关联的文件描述符
    uint32_t u32;
    uint64_t u64;
} epoll_data_t;
```

### 4. `epoll_wait` 等待事件
```c
int epoll_wait(int epfd, struct epoll_event *events, int maxevents, int timeout);
// 功能：阻塞等待注册的事件就绪。
// epfd：epoll 实例的文件描述符。
// events：输出参数，存储就绪事件的数组。
// maxevents：events 数组的最大容量。
// timeout：超时时间（毫秒），-1 表示阻塞，0 表示非阻塞。
// 返回值：成功返回就绪事件数量，失败返回 -1。
```

### 5. 工作模式
#### `LT`（电平触发，默认模式）
- 只要文件描述符处于就绪状态，每次 `epoll_wait` 都会通知应用程序。
- 类似 `select/poll`，适合简单场景，但可能重复触发事件。

#### `ET`（边沿触发）
- 仅在文件描述符状态 变化时 通知一次（如从不可读变为可读）。
- 必须一次性处理所有数据，否则可能丢失后续事件。
- 文件描述符必须设为非阻塞模式，避免阻塞在未完成的 I/O 操作。
- 在 events 字段中设置 EPOLLET 标志。

#### `EPOLLONESHOT` 事件
- 确保一个事件只被触发一次，需手动调用 `epoll_ctl` 重新注册。
- 避免多线程环境下同一文件描述符被多个线程处理。

#### 性能优势
- 回调机制：内核通过回调直接通知就绪事件，时间复杂度 `O(1)`。
- 零拷贝：无需每次传递完整文件描述符集合，减少内存拷贝开销。
- 无文件描述符数量限制（仅受系统资源约束）。
- 支持 `ET` 模式、`EPOLLONESHOT`，优化事件处理逻辑。

```c++
#include <stdio.h>
#include <unistd.h>
#include <sys/epoll.h>
#include <fcntl.h>

int main() {
    int epfd = epoll_create1(0);
    if (epfd == -1) {
        perror("epoll_create1");
        return 1;
    }

    struct epoll_event event;
    event.events = EPOLLIN | EPOLLET;  // 监听可读事件 + ET 模式
    event.data.fd = STDIN_FILENO;

    if (epoll_ctl(epfd, EPOLL_CTL_ADD, STDIN_FILENO, &event) == -1) {
        perror("epoll_ctl");
        return 1;
    }

    struct epoll_event events[10];
    while (1) {
        int n = epoll_wait(epfd, events, 10, -1); // 阻塞等待
        if (n == -1) {
            perror("epoll_wait");
            break;
        }

        for (int i = 0; i < n; i++) {
            if (events[i].data.fd == STDIN_FILENO) {
                char buf[1024];
                ssize_t len = read(STDIN_FILENO, buf, sizeof(buf));
                if (len > 0) {
                    printf("Input: %.*s", (int)len, buf);
                }
            }
        }
    }

    close(epfd);
    return 0;
}
```
