## 一. `open()` 打开或创建文件
- 打开现有文件或创建新文件，返回一个文件描述符（`File Descriptor`, `int` 类型）。
- 文件描述符是进程访问文件的句柄，是后续读写操作的入口。

```c
#include <fcntl.h>
int open(const char *pathname, int flags, mode_t mode); // mode 参数仅在创建文件时生效

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
