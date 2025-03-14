## 一. `Socket` 编程基础

### 1. 什么是 `Socket`
- `Socket`（套接字）是网络通信的抽象端点，提供进程间跨网络的通信能力
- 类比: 类似于电话插孔（`Socket`）和电话线（网络），双方通过插孔建立连接后即可通信

### 2. `Socket` 的类型
- 流式 `Socket`（`SOCK_STREAM`）：基于 `TCP` 协议，提供可靠、面向连接的通信。数据按顺序传输，无丢失或重复
- 数据报 `Socket`（`SOCK_DGRAM`）：基于 `UDP` 协议，提供无连接的通信。传输速度快，但不保证可靠性
- 原始 `Socket`（`SOCK_RAW`）：允许直接访问底层协议（如 `ICMP`）

### 3. `Socket` 地址结构
```c
// IPv4 地址：struct sockaddr_in
struct sockaddr_in {
    sa_family_t    sin_family;   // 地址族（AF_INET）
    in_port_t      sin_port;     // 端口号（16位）
    struct in_addr sin_addr;     // IPv4 地址（32位）
    char           sin_zero[8];  // 填充字段
};


// IPv6 地址：struct sockaddr_in6
struct sockaddr_in6 {
    sa_family_t     sin6_family;    // 地址族：AF_INET6（固定值）
    in_port_t       sin6_port;      // 端口号（16位，需用 `htons` 转换字节序）
    uint32_t        sin6_flowinfo;  // IPv6 流信息（通常设为 0）
    struct in6_addr sin6_addr;      // IPv6 地址（128位）
    uint32_t        sin6_scope_id;  // 作用域 ID（用于链路本地地址）
};

struct in6_addr {
    unsigned char s6_addr[16]; // 16字节的 IPv6 地址（如 "2001:db8::1"）
};


// 通用地址结构：struct sockaddr（用于函数参数类型统一）
struct sockaddr {
    sa_family_t sa_family;    // 地址族标识（如 AF_INET、AF_INET6）
    char        sa_data[14];  // 地址数据（具体内容由地址族决定）
};
```
- 类型统一：`Socket API` 函数（如 `bind()`, `connect()`）需要接受不同地址族的结构体参数，通过 `sockaddr` 统一接口
- 强制类型转换：实际使用时，需将具体结构体（如 `sockaddr_in6`）指针强制转换为 `sockaddr*`

| 结构体         | 地址族（`sa_family`） | 地址长度 | 关键字段                  |
|----------------|---------------------|----------|---------------------------|
| `sockaddr_in`  | `AF_INET`           | 32位     | `sin_port`, `sin_addr`    |
| `sockaddr_in6` | `AF_INET6`          | 128位    | `sin6_port`, `sin6_addr`  |
| `sockaddr_un`  | `AF_UNIX`           | 可变     | `sun_path`（文件路径）    |

---

## 二. 字节序

### 1. 什么是字节序
字节序是数据在内存中存储的字节顺序，分为两种：
- 大端序（`Big-Endian`）：高位字节存储在低地址。`0x12345678` 存储为 `12 34 56 78`（从左到右地址递增）。
- 小端序（`Little-Endian`）：低位字节存储在低地址。`0x12345678` 存储为 `78 56 34 12`（从左到右地址递增）。
- 网络字节序：统一使用大端序，确保跨平台数据传输的一致性。

### 2. 字节序转换函数
```c
// 发送数据前：将主机序的端口和 IP 地址转换为网络序
// 接收数据后：将网络序的端口和 IP 地址转换回主机序

// 1. htons 将 16 位无符号整数从主机序转为网络序
uint16_t htons(uint16_t host_short);
struct sockaddr_in addr;
addr.sin_port = htons(8080); // 主机序 → 网络序

// 2. htonl 将 32 位无符号整数从主机序转为网络序
uint32_t htonl(uint32_t host_long);
addr.sin_addr.s_addr = htonl(INADDR_ANY); // 绑定所有接口

// 3. ntohs 将 16 位无符号整数从网络序转为主机序
uint16_t ntohs(uint16_t network_short);

// 4. ntohl 将 32 位无符号整数从网络序转为主机序
uint32_t ntohl(uint32_t network_long);
```

## 三. IP 地址转换函数
- **`字符串 IP`**: `IPv4`点分十进制表示法，例如 `"192.168.1.1"`; `IPv6`冒号分隔的十六进制表示法，例如 `"2001:db8::1"`
- **`二进制 IP`** 是计算机可处理的 `IP` 地址表示形式，通常用于网络协议和底层编程。`IPv4 32` 位无符号整数，例如 `0xC0A80101` 对应 `192.168.1.1`。`IPv6 128` 位无符号整数，例如 `0x20010db8000000000000000000000001` 对应 `2001:db8::1`
```c
// inet_pton 将字符串 IP 转换为二进制格式
int inet_pton(int af, const char *src, void *dst);

struct in_addr addr4;
inet_pton(AF_INET, "192.168.1.1", &addr4);

struct in6_addr addr6;
inet_pton(AF_INET6, "2001:db8::1", &addr6);


// inet_ntop 将二进制 IP 转换为字符串格式
const char *inet_ntop(int af, const void *src, char *dst, socklen_t size);

struct in_addr addr4;
char str[INET_ADDRSTRLEN];
inet_ntop(AF_INET, &addr4, str, INET_ADDRSTRLEN);

struct in6_addr addr6;
char str[INET6_ADDRSTRLEN];
inet_ntop(AF_INET6, &addr6, str, INET6_ADDRSTRLEN);
```

| 函数/概念         | 用途                                | 示例场景                     |
|-------------------|-------------------------------------|------------------------------|
| `htons()` / `htonl()` | 主机序 → 网络序（端口/IP）         | 设置 `sockaddr_in` 结构体    |
| `ntohs()` / `ntohl()` | 网络序 → 主机序（端口/IP）         | 解析接收到的数据             |
| `inet_pton()`      | 字符串 `IP` → 二进制 `IP`（支持 `IPv4/IPv6`） | 配置服务器地址               |
| `inet_ntop()`      | 二进制 `IP` → 字符串 `IP`（支持 `IPv4/IPv6`） | 日志输出或显示 `IP`            |
---

## 四. 核心 `Socket` 操作函数
### 1. `socket()` 创建 `Socket`
```c
int socket(int domain, int type, int protocol);
// 创建一个通信端点 Socket，指定协议族、类型和协议。
// domain   => 协议族，如 AF_INET（IPv4）、AF_INET6（IPv6）、AF_UNIX（本地通信）
// type     => Socket 类型，如 SOCK_STREAM（TCP）、SOCK_DGRAM（UDP）
// protocol => 具体协议，通常设为 0。
// 返回值：成功返回 Socket 文件描述符，失败返回 -1。

int sockfd = socket(AF_INET, SOCK_STREAM, 0); // 创建 TCP Socket
```

### 2. `bind()` 绑定地址和端口
```c
// 将 Socket 绑定到本地地址和端口（服务器端必用）。
int bind(int sockfd, const struct sockaddr *addr, socklen_t addrlen);
// addr：指向地址结构体的指针（如 struct sockaddr_in）。
// addrlen：地址结构体的长度。

struct sockaddr_in addr;
addr.sin_family = AF_INET;
addr.sin_port = htons(8080);           // 端口
addr.sin_addr.s_addr = htonl(INADDR_ANY); // 绑定所有接口
bind(sockfd, (struct sockaddr*)&addr, sizeof(addr));
```

### 3. `listen()` 监听连接请求 `TCP`
```c
// 将 Socket 设置为被动监听模式，等待客户端连接。
int listen(int sockfd, int backlog);
// backlog：等待连接队列的最大长度（建议至少设为 5）

listen(sockfd, 10); // 允许最多 10 个连接排队
```

### 4. `accept()` 接受客户端连接 `TCP`
```c
// 从监听队列中接受一个客户端连接，返回新的 Socket 文件描述符。
int accept(int sockfd, struct sockaddr *addr, socklen_t *addrlen);
// addr 保存客户端地址信息（可选，设为 NULL 可忽略）。

struct sockaddr_in client_addr;
socklen_t addrlen = sizeof(client_addr);
int client_fd = accept(sockfd, (struct sockaddr*)&client_addr, &addrlen);
```

### 5. `connect()` 连接到服务器（`TCP/UDP`）
```c
// 客户端主动连接服务器（TCP）或指定默认目标地址（UDP）。
int connect(int sockfd, const struct sockaddr *addr, socklen_t addrlen);

struct sockaddr_in server_addr;
server_addr.sin_family = AF_INET;
server_addr.sin_port = htons(8080);
inet_pton(AF_INET, "192.168.1.1", &server_addr.sin_addr);
connect(sockfd, (struct sockaddr*)&server_addr, sizeof(server_addr));
```

### 6. `TCP`数据传输函数 `send() / recv()`
```c
// 发送和接收数据（面向连接）。
ssize_t send(int sockfd, const void *buf, size_t len, int flags);
ssize_t recv(int sockfd, void *buf, size_t len, int flags);
// flags：控制选项（如 MSG_WAITALL 等待完整数据）。


char buffer[1024];
send(sockfd, "Hello", 5, 0);       // 发送数据
recv(sockfd, buffer, sizeof(buffer), 0); // 接收数据
```

### 7. `UDP` 数据传输函数 `sendto() / recvfrom()`
```c
// 发送和接收数据报（无连接）。
ssize_t sendto(int sockfd, const void *buf, size_t len, int flags, const struct sockaddr *dest_addr, socklen_t addrlen);
ssize_t recvfrom(int sockfd, void *buf, size_t len, int flags, struct sockaddr *src_addr, socklen_t *addrlen);

struct sockaddr_in server_addr;
sendto(sockfd, "Hello", 5, 0, (struct sockaddr*)&server_addr, sizeof(server_addr));
recvfrom(sockfd, buffer, sizeof(buffer), 0, NULL, NULL);
```

### 8. `close()`
```c
// 释放 Socket 资源。
int close(int sockfd);
close(sockfd);
```
---

## 五. 高级 `Socket` 操作函数
### 1. `setsockopt() / getsockopt()` 设置/获取 `Socket` 选项
```c
// 配置 Socket 参数（如地址复用、超时、缓冲区大小）。
int setsockopt(int sockfd, int level, int optname, const void *optval, socklen_t optlen);
int getsockopt(int sockfd, int level, int optname, void *optval, socklen_t *optlen);

// SO_REUSEADDR：允许地址复用（解决端口占用问题）。
// SO_RCVTIMEO / SO_SNDTIMEO：设置接收/发送超时。
int opt = 1;
setsockopt(sockfd, SOL_SOCKET, SO_REUSEADDR, &opt, sizeof(opt));
```

### 2. `shutdown()` 半关闭连接
```c
// 关闭 Socket 的读或写通道。
int shutdown(int sockfd, int how);
// how：SHUT_RD（关闭读）、SHUT_WR（关闭写）、SHUT_RDWR（全关闭）

shutdown(sockfd, SHUT_WR); // 关闭写端，发送 FIN 报文
```

## 六. `Unix Domain Socket` 流程
### 服务器端 `TCP`
- **创建 `Socket`** `socket(AF_UNIX, SOCK_STREAM, 0)`
- **绑定地址** `bind(sockfd, (struct sockaddr*)&addr, sizeof(addr))`
- **监听连接** `listen(sockfd, backlog)`
- **接受连接** `accept(sockfd, NULL, NULL)`
- **收发数据** `read()/write()` 或 `recv()/send()`
- **关闭 `Socket`** `close(sockfd)`

### 客户端 `TCP`
- **创建 `Socket`** `socket(AF_UNIX, SOCK_STREAM, 0)`
- **连接服务器** `connect(sockfd, (struct sockaddr*)&addr, sizeof(addr))`
- **收发数据** `read()/write()` 或 `recv()/send()`
- **关闭 `Socket`** `close(sockfd)`
---

### 服务器端 `UDP`
- **创建 `Socket`** `socket(AF_UNIX, SOCK_DGRAM, 0)`
- **绑定地址** `bind(sockfd, (struct sockaddr*)&addr, sizeof(addr))`
- **收发数据** `recvfrom()` 和 `sendto()`
- **关闭 `Socket`** `close(sockfd)`

### 客户端 `UDP`
- **创建 `Socket`** `socket(AF_UNIX, SOCK_DGRAM, 0)`
- **收发数据** `sendto()` 和 `recvfrom()`
- **关闭 `Socket`** `close(sockfd)`
---
