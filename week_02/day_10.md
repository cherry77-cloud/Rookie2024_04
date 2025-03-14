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
字节序是数据在内存中存储的字节顺序，分为两种：
- 大端序（`Big-Endian`）：高位字节存储在低地址。`0x12345678` 存储为 `12 34 56 78`（从左到右地址递增）。
- 小端序（`Little-Endian`）：低位字节存储在低地址。`0x12345678` 存储为 `78 56 34 12`（从左到右地址递增）。
- 网络字节序：统一使用大端序，确保跨平台数据传输的一致性。
