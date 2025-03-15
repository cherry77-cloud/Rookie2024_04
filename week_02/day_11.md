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
    失败：返回 -1，并设置 errno。
    返回0：表示到达文件末尾（EOF）。
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
## 五. `umask()` 设置文件创建时的权限掩码

- 设置进程的默认权限掩码，影响新创建文件的最终权限。
- `实际权限 = mode & ~umask`（`umask` 的二进制位为`1`时，屏蔽对应权限）。
- 常见 `umask` 值
  - `022`: 屏蔽组和其他用户的写权限（默认常见值）。
  - `002`: 屏蔽其他用户的写权限。
  - `077`: 屏蔽所有组和其他用户的权限。
---

## 六. `stat()` 和 `lstat()` 获取文件元数据

- 获取文件的元数据（如类型、权限、大小、时间戳等）。

```c
#include <sys/stat.h>
#include <unistd.h>

int stat(const char *pathname, struct stat *statbuf);    // 跟随符号链接
int lstat(const char *pathname, struct stat *statbuf);   // 不跟随符号链接

struct stat {
    dev_t     st_dev;         // 文件所在设备的 ID
    ino_t     st_ino;         // 文件的 inode 号
    mode_t    st_mode;        // 文件类型和权限
    nlink_t   st_nlink;       // 硬链接数
    uid_t     st_uid;         // 文件所有者的用户 ID
    gid_t     st_gid;         // 文件所有者的组 ID
    dev_t     st_rdev;        // 设备文件的设备 ID（仅对设备文件有效）
    off_t     st_size;        // 文件大小（字节）
    blksize_t st_blksize;     // 文件系统的 I/O 块大小
    blkcnt_t  st_blocks;      // 文件占用的磁盘块数

    // 时间戳（精度到纳秒）
    struct timespec st_atim;  // 最后访问时间
    struct timespec st_mtim;  // 最后修改时间（文件内容）
    struct timespec st_ctim;  // 最后状态变更时间（元数据，如权限）
};
```

文件类型判断宏和权限位检查宏
```c
S_ISREG(st_mode)   // 普通文件
S_ISDIR(st_mode)   // 目录
S_ISLNK(st_mode)   // 符号链接
S_ISCHR(st_mode)   // 字符设备文件
S_ISBLK(st_mode)   // 块设备文件
S_ISFIFO(st_mode)  // 管道文件（FIFO）
S_ISSOCK(st_mode)  // 套接字文件

S_IRUSR  // 用户读权限（0400）
S_IWUSR  // 用户写权限（0200）
S_IXUSR  // 用户执行权限（0100）
S_IRGRP  // 组读权限（0040）
S_IWGRP  // 组写权限（0020）
S_IXGRP  // 组执行权限（0010）
S_IROTH  // 其他用户读权限（0004）
S_IWOTH  // 其他用户写权限（0002）
S_IXOTH  // 其他用户执行权限（0001）
```
---

## 七. 文件属性操作函数

### `chmod()` 修改文件权限
- 修改文件的权限位（`mode`），如 `rwxr-xr--`。

```c
#include <sys/stat.h>
int chmod(const char *pathname, mode_t mode);
/*
参数
    pathname：文件路径。
    mode：新权限（八进制数，如 0644）。
返回值
    成功返回 0，失败返回 -1。
*/
```

### `chown()`修改文件所有者和组

- 修改文件的所有者用户 `ID（uid）`和组 `ID（gid）`。

```c
#include <unistd.h>
int chown(const char *pathname, uid_t owner, gid_t group);
/*
参数
    pathname：文件路径。
    owner：新所有者的用户 ID（设为 -1 表示不修改）。
    group：新组的组 ID（设为 -1 表示不修改）。
返回值
    成功返回 0，失败返回 -1。
*/
```

### `access()` 检查文件权限
- 检查当前进程对文件的访问权限。
```c
#include <unistd.h>
int access(const char *pathname, int mode);

/*
mode：
    F_OK：文件是否存在。
    R_OK：是否可读。
    W_OK：是否可写。
    X_OK：是否可执行。
*/
```

### `truncate()` 和 `ftruncate()` 修改文件大小
```c
#include <unistd.h>
int truncate(const char *pathname, off_t length);   // 通过路径
int ftruncate(int fd, off_t length);               // 通过文件描述符
```

- `stat()` 和 `lstat()` 获取文件元数据，区别在于是否跟随符号链接。
- `struct stat` 存储文件类型、权限、大小、时间戳等关键信息。
- `chmod()` 修改文件权限（八进制模式）。
- `chown()` 修改文件所有者和组。
- 其他工具函数 `access()、utime()、truncate() ` 等用于更精细的文件属性操作。

```c
#include <stdio.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <unistd.h>
#include <pwd.h>
#include <grp.h>
#include <time.h>
#include <string.h>

int main(int argc, char * argv[]) 
{
    if (argc < 2) {
        printf("%s filename\n", argv[0]);
        return -1;
    }
    struct stat st;
    int ret = stat(argv[1], &st);
    if (ret == -1) {
        perror("stat");
        return -1;
    }
    char perms[11] = {0};
    switch(st.st_mode & S_IFMT) {
        case S_IFLNK:
            perms[0] = 'l';
            break;
        case S_IFDIR:
            perms[0] = 'd';
            break;
        case S_IFREG:
            perms[0] = '-';
            break; 
        case S_IFBLK:
            perms[0] = 'b';
            break; 
        case S_IFCHR:
            perms[0] = 'c';
            break; 
        case S_IFSOCK:
            perms[0] = 's';
            break;
        case S_IFIFO:
            perms[0] = 'p';
            break;
        default:
            perms[0] = '?';
            break;
    }

    // 文件所有者
    perms[1] = (st.st_mode & S_IRUSR) ? 'r' : '-';
    perms[2] = (st.st_mode & S_IWUSR) ? 'w' : '-';
    perms[3] = (st.st_mode & S_IXUSR) ? 'x' : '-';

    // 文件所在组
    perms[4] = (st.st_mode & S_IRGRP) ? 'r' : '-';
    perms[5] = (st.st_mode & S_IWGRP) ? 'w' : '-';
    perms[6] = (st.st_mode & S_IXGRP) ? 'x' : '-';

    // 其他人
    perms[7] = (st.st_mode & S_IROTH) ? 'r' : '-';
    perms[8] = (st.st_mode & S_IWOTH) ? 'w' : '-';
    perms[9] = (st.st_mode & S_IXOTH) ? 'x' : '-';

    // 硬连接数
    int linkNum = st.st_nlink;
    // 文件所有者
    char * fileUser = getpwuid(st.st_uid)->pw_name;
    // 文件所在组
    char * fileGrp = getgrgid(st.st_gid)->gr_name;
    // 文件大小
    long int fileSize = st.st_size;

    // 获取修改的时间
    char * time = ctime(&st.st_mtime);
    char mtime[512] = {0};
    strncpy(mtime, time, strlen(time) - 1);
    
    char buf[1024];
    sprintf(buf, "%s %d %s %s %ld %s %s", perms, linkNum, fileUser, fileGrp, fileSize, mtime, argv[1]);
    printf("%s\n", buf);
    return 0;
}
```

---

## 八. 目录操作核心函数

### 1. `opendir()` 打开目录
- 打开一个目录，返回指向目录流的指针（`DIR` 类型）
```c
#include <dirent.h>
DIR *opendir(const char *name);

/*
参数
    name 为目录路径（如 "." 表示当前目录）。
返回值
    成功：返回 DIR 指针（目录流句柄）。
    失败：返回 NULL，并设置 errno。
*/
```

### 2. `readdir()` 读取目录项
- 从目录流中读取一个目录项（即目录中的一个文件或子目录）。
```c
#include <dirent.h>
struct dirent *readdir(DIR *dirp);

/*
参数
    dirp 为 opendir() 返回的目录流指针。
返回值
    成功：返回指向 struct dirent 的指针（存储目录项信息）。
    失败或到达目录末尾：返回 NULL（需用 errno 区分错误）。
*/
```

### 3. `closedir()` 关闭目录流
- 关闭目录流，释放资源。
```c
#include <dirent.h>
int closedir(DIR *dirp);

/*
参数
    dirp 为目录流指针。
返回值
    成功：返回 0。
    失败：返回 -1，并设置 errno。
*/
```

### 4. `rewinddir()` 重置目录流位置
- 将目录流的读取位置重置到开头。

```c
#include <dirent.h>
void rewinddir(DIR *dirp);
```

### 5. 目录相关结构体

#### `struct dirent` 目录项信息
存储单个目录项的信息（不同系统可能扩展字段，但以下字段是通用的）
```c
#include <dirent.h>
struct dirent {
    ino_t          d_ino;       // 文件的 inode 号
    off_t          d_off;       // 目录项在目录流中的偏移（非 POSIX 标准）
    unsigned short d_reclen;    // 目录项记录长度
    unsigned char  d_type;      // 文件类型（如 DT_REG、DT_DIR）
    char           d_name[256]; // 文件名（以空字符结尾）
};

/*
d_type：文件类型标志（需检查系统是否支持
    DT_REG：普通文件。
    DT_DIR：目录。
    DT_LNK：符号链接。
    DT_FIFO：管道文件。
    DT_SOCK：套接字文件。
    DT_CHR：字符设备文件。
    DT_BLK：块设备文件。
    DT_UNKNOWN：未知类型。
*/
```

#### `DIR` 结构体
- 定义：表示目录流的句柄（内部实现依赖操作系统，用户无需关心其具体字段）。
- 用途：通过 `opendir()` 获取，用于后续的 `readdir()` 和 `closedir()` 操作。

```c
// Linux 中 DIR 结构体的典型定义: 来自 glibc 的实现
struct __dirstream {
    int fd;                       // 目录文件的文件描述符
    off_t tell;                   // 当前读取位置
    size_t buf_size;              // 缓冲区大小
    size_t allocation;            // 分配的空间大小
    size_t size;                  // 有效数据大小
    off_t filepos;                // 文件位置
    char *data;                   // 数据缓冲区
    unsigned short int offset;    // 当前读取偏移
    unsigned short int code;      // 错误码
};

typedef struct __dirstream DIR;
```

```c
#include <sys/types.h>
#include <dirent.h>
#include <stdio.h>
#include <string.h>
#include <stdlib.h>

int getFileNum(const char * path);

int main(int argc, char * argv[]) 
{
    if (argc < 2) {
        printf("%s path\n", argv[0]);
        return -1;
    }
    int num = getFileNum(argv[1]);
    printf("%d\n", num);
    return 0;
}

int getFileNum(const char * path) 
{
    DIR * dir = opendir(path);
    if (dir == NULL) {
        perror("opendir");
        exit(0);
    }
    struct dirent *ptr;
    int total = 0;
    while ((ptr = readdir(dir)) != NULL) {
        char * dname = ptr->d_name;
        if (strcmp(dname, ".") == 0 || strcmp(dname, "..") == 0) {
            continue;
        }
        if (ptr->d_type == DT_DIR) {
            char newpath[256];
            sprintf(newpath, "%s/%s", path, dname);
            total += getFileNum(newpath);
        }
        if (ptr->d_type == DT_REG) {
            total++;
        }
    }
    closedir(dir);
    return total;
}
```

- `opendir()`、`readdir()`、`closedir()` 目录遍历的基本工具。
- `rewinddir()` 重置目录流位置。
- `DIR` 目录流的句柄。
- `struct dirent` 存储目录项的名称、类型等元数据。
- 实现类似 `ls` 的目录列表功能。递归遍历文件树（如统计文件数量、搜索文件）。

---

## 九. 文件重定向
### 1. `dup()` 复制文件描述符
- 复制一个已有的文件描述符，生成一个新的描述符指向同一个文件/资源。
- 新描述符是当前可用的最小未使用值。

```c
#include <unistd.h>
int dup(int oldfd);

int fd = open("file.txt", O_WRONLY | O_CREAT, 0644);
int new_fd = dup(fd); // new_fd 是系统分配的最小可用 FD

write(new_fd, "Hello", 5); // 写入 file.txt
close(fd);
close(new_fd);

/*
参数
    oldfd：需要复制的原文件描述符。
返回值
   成功：返回新的文件描述符。
   失败：返回 -1，并设置 errno。
*/
```

### 2. `dup2()` 指定新描述符的复制

- 将 `oldfd` 复制到指定的 `newfd`，若 `newfd` 已打开，则先关闭它。
```c
#include <unistd.h>
int dup2(int oldfd, int newfd);

int fd = open("output.txt", O_WRONLY | O_CREAT, 0644);
dup2(fd, STDOUT_FILENO); // STDOUT_FILENO 是 1
close(fd); // 关闭原 fd，因为 STDOUT_FILENO 已指向文件

/*
参数
    oldfd：原文件描述符。
    newfd：目标文件描述符（需为有效值，通常为 0 <= newfd < OPEN_MAX）。
返回值
    成功：返回 newfd。
    失败：返回 -1，并设置 errno。
*/
```
标准文件描述符

| 文件描述符 | 名称           | 默认设备       | 宏定义常量         |
|------------|----------------|----------------|--------------------|
| 0          | 标准输入       | 键盘（终端）   | `STDIN_FILENO`      |
| 1          | 标准输出       | 屏幕（终端）   | `STDOUT_FILENO`      |
| 2          | 标准错误输出   | 屏幕（终端）   | `STDERR_FILENO`      |

核心区别
| 特性                | `dup()`                         | `dup2()`                         |
|---------------------|----------------------------------|----------------------------------|
| **参数**            | 仅需原文件描述符 `oldfd`         | 需原文件描述符 `oldfd` 和目标描述符 `newfd` |
| **新 `FD` 选择**      | 自动分配最小的可用 `FD`            | 强制使用指定的 `newfd`，若 `newfd` 已打开则先关闭它 |
| **返回值**          | 返回新分配的 `FD`                  | 返回指定的 `newfd`（若成功）      |
| **原子性**          | 仅复制 `FD`                        | 关闭原 `newfd` + 复制 `FD` 是原子操作 |
| **典型用途**        | 简单复制 `FD`                      | 精确控制 `FD`（如重定向标准输入/输出） |

- `dup` 和 `dup2`：复制文件描述符，使得两个描述符指向同一个文件或资源。
- 共享文件偏移量和状态标志：复制后的描述符共享文件偏移量和状态标志（如 `O_APPEND`）。
- `dup` 自动分配最小可用文件描述符。
- `dup2` 可以指定目标文件描述符，若目标已打开则先关闭它。

---
