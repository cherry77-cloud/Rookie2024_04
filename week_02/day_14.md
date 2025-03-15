## 一. 亲缘进程间通信方式

### 1. 匿名管道 `Anonymous Pipe`

- 单向通信：通过 `pipe()` 创建一对文件描述符（`fd[0]` 读端，`fd[1]` 写端），数据从写端流向读端。
- 进程关系：仅适用于父子进程或有共同祖先的进程。
- 数据特性：数据按顺序读写，容量有限（默认 `64KB`），读空或写满时会阻塞。

```c++
#include <unistd.h>
#include <stdio.h>

int main() {
    int fd[2];
    char buffer[20];
    pipe(fd);  // 创建管道

    pid_t pid = fork();
    if (pid == 0) {          // 子进程
        close(fd[1]);        // 关闭写端
        read(fd[0], buffer, sizeof(buffer));
        printf("Child received: %s\n", buffer);
        close(fd[0]);
    } else {                 // 父进程
        close(fd[0]);        // 关闭读端
        write(fd[1], "Hello from parent", 17);
        close(fd[1]);
    }
    return 0;
}
```

---

### 2. 匿名内存

- 匿名性：内存区域不与任何文件关联，仅通过文件描述符或指针访问。
- 亲缘进程：父进程创建匿名内存后，子进程通过 `fork()` 继承内存映射。

#### 使用 `memfd_create` 系统调用

```c++
#include <sys/mman.h>
#include <sys/syscall.h>
#include <unistd.h>
#include <stdio.h>
#include <sys/wait.h>

#define SIZE 4096

int main() {
    // 创建匿名内存文件描述符
    int fd = syscall(SYS_memfd_create, "anon_mem", MFD_CLOEXEC);
    ftruncate(fd, SIZE); // 设置内存大小

    // 映射到进程地址空间
    char *ptr = mmap(NULL, SIZE, PROT_READ | PROT_WRITE, MAP_SHARED, fd, 0);

    pid_t pid = fork();
    if (pid == 0) {          // 子进程
        sprintf(ptr, "Hello from child process!");
        munmap(ptr, SIZE);   // 解除映射
        close(fd);
        _exit(0);
    } else {                 // 父进程
        wait(NULL);          // 等待子进程结束
        printf("Parent received: %s\n", ptr);
        munmap(ptr, SIZE);
        close(fd);
    }
    return 0;
}
```

#### 使用 `MAP_ANONYMOUS` 标志的 `mmap`
```c++
#include <sys/mman.h>
#include <stdio.h>
#include <unistd.h>
#include <sys/wait.h>

#define SIZE 4096

int main() {
    // 分配匿名共享内存
    char *ptr = mmap(NULL, SIZE, PROT_READ | PROT_WRITE, MAP_SHARED | MAP_ANONYMOUS, -1, 0);

    pid_t pid = fork();
    if (pid == 0) {          // 子进程
        sprintf(ptr, "Data written by child");
        _exit(0);
    } else {                 // 父进程
        wait(NULL);          // 等待子进程结束
        printf("Parent received: %s\n", ptr);
        munmap(ptr, SIZE);
    }
    return 0;
}
```

---

### 3. 信号 `Signal`

- 异步通知：通过发送预定义信号（如 `SIGUSR1`）触发目标进程的注册函数。
- 信号处理：进程需注册信号处理函数（`signal()` 或 `sigaction()`）。

- 实现步骤
  - 注册信号处理函数：子进程调用 `signal(SIGUSR1, handler)`。
  - 发送信号：父进程调用 `kill(pid, SIGUSR1)`。
  - 处理信号：子进程在 `handler` 函数中响应。

```c++
#include <signal.h>
#include <unistd.h>

void handler(int sig) {
    printf("Received signal %d\n", sig);
}

int main() {
    signal(SIGUSR1, handler); // 注册信号处理函数
    pid_t pid = fork();

    if (pid == 0) {          // 子进程
        pause();             // 等待信号
    } else {                 // 父进程
        sleep(1);            // 确保子进程注册完成
        kill(pid, SIGUSR1);  // 发送信号
        wait(NULL);
    }
    return 0;
}
```
---

## 二. 进程间通信

### 1. 命名管道 `Named Pipe / FIFO`

- 文件系统标识：通过一个特殊的文件路径（如 `/tmp/myfifo`）标识管道，允许任意进程读写。
- 半双工通信：数据单向流动，若需双向通信需创建两个 `FIFO`。
- 阻塞与非阻塞模式：默认阻塞（读端无数据时读操作阻塞，写端无读端时写操作阻塞）。

```c++
// 创建 FIFO（仅需一次）
mkfifo("/tmp/myfifo", 0666);

// 进程 A（写入）
int fd = open("/tmp/myfifo", O_WRONLY);
write(fd, "Hello", 6);
close(fd);

// 进程 B（读取）
int fd = open("/tmp/myfifo", O_RDONLY);
char buf[100];
read(fd, buf, sizeof(buf));
close(fd);
```

---

### 2. 共享内存 `Shared Memory`

- 内存映射：多个进程通过映射同一块物理内存实现数据共享。
- 同步需求：需配合信号量（`Semaphore`）或互斥锁避免数据竞争。
- 实现步骤（`System V` 共享内存）
  - 创建共享内存段：`shmget()`。
  - 附加到进程地址空间：`shmat()`。
  - 读写数据：直接操作内存指针。
  - 释放资源：`shmdt()` 解除映射，`shmctl()` 删除内存段。

```c++
// 进程 A（创建并写入）
int shm_id = shmget(IPC_PRIVATE, 1024, IPC_CREAT | 0666);
char *ptr = (char*)shmat(shm_id, NULL, 0);
sprintf(ptr, "Shared data");

// 进程 B（通过已知 shm_id 访问）
int shm_id = ...; // 需通过其他方式传递 shm_id（如文件或环境变量）
char *ptr = (char*)shmat(shm_id, NULL, 0);
printf("Received: %s\n", ptr);
shmdt(ptr);
```

---

### 3. 消息队列 `Message Queue`

- 消息传递：进程通过发送和接收结构化消息通信。
- 消息类型：每条消息可指定类型，支持按优先级或类型读取。

```c++
// 定义消息结构体
struct msg_buffer {
    long msg_type;
    char msg_text[100];
};

// 进程 A（发送）
int msg_id = msgget(1234, IPC_CREAT | 0666);
struct msg_buffer msg = {.msg_type = 1};
strcpy(msg.msg_text, "Hello");
msgsnd(msg_id, &msg, sizeof(msg), 0);

// 进程 B（接收）
int msg_id = msgget(1234, 0666);
struct msg_buffer msg;
msgrcv(msg_id, &msg, sizeof(msg), 1, 0);
printf("Received: %s\n", msg.msg_text);
```

---

### 4. 套接字 `Socket`

- 本地套接字（`Unix Domain Socket`）：基于文件系统路径的套接字，支持流式（`SOCK_STREAM`）或数据报（`SOCK_DGRAM`）通信。
- 网络套接字：支持跨机器通信（如`TCP/UDP`）。

```c++
// 服务端
int sockfd = socket(AF_UNIX, SOCK_STREAM, 0);
struct sockaddr_un addr = {.sun_family = AF_UNIX};
strcpy(addr.sun_path, "/tmp/mysocket");
bind(sockfd, (struct sockaddr*)&addr, sizeof(addr));
listen(sockfd, 5);
int connfd = accept(sockfd, NULL, NULL);
write(connfd, "Hello", 6);

// 客户端
int sockfd = socket(AF_UNIX, SOCK_STREAM, 0);
connect(sockfd, (struct sockaddr*)&addr, sizeof(addr));
char buf[100];
read(sockfd, buf, sizeof(buf));
```

---
