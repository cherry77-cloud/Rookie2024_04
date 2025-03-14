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
### 1. `socket()` -> 创建 `socket`. 创建一个通信端点（`Socket`），指定协议族、类型和协议
```c
int socket(int domain, int type, int protocol);
// domain   => 协议族，如 AF_INET（IPv4）、AF_INET6（IPv6）、AF_UNIX（本地通信）
// type     => Socket 类型，如 SOCK_STREAM（TCP）、SOCK_DGRAM（UDP）
// protocol => 具体协议，通常设为 0。
// 返回值：成功返回 Socket 文件描述符，失败返回 -1。

int sockfd = socket(AF_INET, SOCK_STREAM, 0); // 创建 TCP Socket
```
