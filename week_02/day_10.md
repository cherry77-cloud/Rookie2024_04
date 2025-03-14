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
// 通用地址结构：struct sockaddr（用于函数参数类型统一）
```
