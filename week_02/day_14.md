## 亲缘进程间通信方式
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

### 4. 本地套接字 `Unix Domain Socket`

- 基于文件的套接字：通过文件路径标识，支持流式（`TCP-like`）或数据报（`UDP-like`）通信。
- 高效通信：数据直接在进程间传递，无需网络协议栈。
- 实现步骤（流式套接字）
  - 服务端：创建套接字 → 绑定地址 → 监听 → 接受连接。
  - 客户端：创建套接字 → 连接服务端 → 发送/接收数据。
```c++
#include <sys/socket.h>
#include <sys/un.h>
#include <stdio.h>

#define SOCK_PATH "/tmp/example.sock"

int main() {
    int sockfd = socket(AF_UNIX, SOCK_STREAM, 0);
    struct sockaddr_un addr = {.sun_family = AF_UNIX};
    strcpy(addr.sun_path, SOCK_PATH);

    bind(sockfd, (struct sockaddr *)&addr, sizeof(addr));
    listen(sockfd, 5);

    pid_t pid = fork();
    if (pid == 0) {          // 子进程（客户端）
        int connfd = socket(AF_UNIX, SOCK_STREAM, 0);
        connect(connfd, (struct sockaddr *)&addr, sizeof(addr));
        write(connfd, "Hello from child", 16);
        close(connfd);
    } else {                 // 父进程（服务端）
        int connfd = accept(sockfd, NULL, NULL);
        char buf[100];
        read(connfd, buf, sizeof(buf));
        printf("Parent received: %s\n", buf);
        close(connfd);
        unlink(SOCK_PATH);   // 删除套接字文件
    }
    return 0;
}
```
---
