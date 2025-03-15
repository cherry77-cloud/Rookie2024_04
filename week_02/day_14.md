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

### 2. 共享内存 `Shared Memory`

- 内存共享：多个进程映射同一物理内存区域，直接读写内存实现数据交换。
- 同步需求：需配合信号量或互斥锁避免数据竞争。

- 实现步骤 `System V`
  - 创建共享内存段：`shmget()` 分配共享内存。
  - 附加到进程地址空间：`shmat()` 将共享内存映射到进程。
  - 读写数据：直接操作共享内存指针。
  - 释放资源：`shmdt()` 解除映射，`shmctl()` 删除内存段。

```c++
#include <sys/shm.h>
#include <stdio.h>

int main() {
    int shm_id = shmget(IPC_PRIVATE, 1024, IPC_CREAT | 0666);
    char *shm_ptr = (char *)shmat(shm_id, NULL, 0);

    pid_t pid = fork();
    if (pid == 0) {          // 子进程
        sprintf(shm_ptr, "Data from child");
        shmdt(shm_ptr);
    } else {                 // 父进程
        wait(NULL);          // 等待子进程完成
        printf("Parent received: %s\n", shm_ptr);
        shmdt(shm_ptr);
        shmctl(shm_id, IPC_RMID, NULL); // 删除共享内存
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

## 4. 本地套接字 `Unix Domain Socket`

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
