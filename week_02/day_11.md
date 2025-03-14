## 一. `open()` 打开或创建文件
- 打开现有文件或创建新文件，返回一个文件描述符（`File Descriptor`, `int` 类型）。
- 文件描述符是进程访问文件的句柄，是后续读写操作的入口。

```c
#include <fcntl.h>
int open(const char *pathname, int flags, mode_t mode); // mode 参数仅在创建文件时生效

int fd = open("data.txt", O_WRONLY | O_CREAT | O_EXCL, 0644);
if (fd == -1) {
    perror("open failed");
    exit(1);
}

/*
pathname
    文件路径（绝对或相对路径）。
flags
    文件打开模式的标志位（必选一个基础模式 + 可选组合标志）：
    基础模式（必选其一）：
        O_RDONLY: 只读模式。
        O_WRONLY: 只写模式。
        O_RDWR: 读写模式。
    可选标志（通过 | 组合）：
        O_CREAT: 文件不存在时创建新文件。
        O_APPEND: 追加模式（写入时自动定位到文件末尾）。
        O_TRUNC: 若文件已存在且为普通文件，清空文件内容（长度截断为0）。
        O_EXCL: 与 O_CREAT 联用，若文件已存在则报错（防止覆盖）。
        O_NONBLOCK: 非阻塞模式（对设备文件或管道有效）。
        O_SYNC: 同步写入（数据及元数据写入磁盘后才返回）。
mode
    新文件的权限（仅在 O_CREAT 时有效），通常用八进制表示（如 0644）。
    实际权限 = mode & ~umask（umask 是进程的默认权限掩码）。
*/
```

---

## 二. `close()` 关闭文件
- 关闭文件描述符，释放内核资源。
- 未关闭文件可能导致资源泄漏（文件描述符数量有上限）。

```c
#include <unistd.h>
int close(int fd);

if (close(fd) == -1) {
    perror("close failed");
}
```

---

## 三. `read()` 从文件读取数据
- 从文件描述符对应的文件中读取数据到缓冲区。
- 默认是阻塞的（若文件无数据，进程会等待）。


```c
#include <unistd.h>
ssize_t read(int fd, void *buf, size_t count);

char buffer[1024];
ssize_t bytes_read = read(fd, buffer, sizeof(buffer));
if (bytes_read == -1) {
    perror("read failed");
} else if (bytes_read == 0) {
    printf("Reached end of file.\n");
} else {
    printf("Read %zd bytes: %.*s\n", bytes_read, (int)bytes_read, buffer);
}

/*
参数
    fd：文件描述符。
    buf：存储数据的缓冲区地址。
    count：请求读取的字节数。

返回值
    成功：返回实际读取的字节数（可能小于 count，如文件末尾或信号中断）。
    返回 0：表示到达文件末尾（EOF）。
    失败：返回 -1，并设置 errno。

注意
    需处理部分读取的情况（如网络或管道场景）。
    非阻塞模式下可能返回 -1，且 errno 为 EAGAIN 或 EWOULDBLOCK。
*/
```
---

## 四. `write()` 向文件写入数据
- 将缓冲区数据写入文件描述符对应的文件。
- 默认是阻塞的（如磁盘满时会等待）。

```c
#include <unistd.h>
ssize_t write(int fd, const void *buf, size_t count);

char data[] = "Hello, Unix I/O!";
ssize_t bytes_written = write(fd, data, strlen(data));
if (bytes_written == -1) {
    perror("write failed");
} else if (bytes_written < strlen(data)) {
    printf("Partial write: %zd/%zu bytes written\n", bytes_written, strlen(data));
}

/*
参数
    同 read()，但 buf 是写入数据的来源。

返回值
    成功：返回实际写入的字节数（可能小于 count，如磁盘满或信号中断）。
    失败：返回 -1，并设置 errno。

注意
    需处理部分写入的情况（循环写入直到所有数据写完）。
    使用 O_APPEND 标志可避免并发写入时的覆盖问题。
*/
```
---
