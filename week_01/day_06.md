

## 六. `waitpid()` 函数详解

- `waitpid()` 是一个用于等待特定子进程终止或状态变化的系统调用。与 `wait()` 相比，`waitpid()` 提供了更多的灵活性，允许父进程指定等待的子进程以及控制等待的行为（如非阻塞模式）。


```c
#include <sys/wait.h>
pid_t wait(int *status);
pid_t waitpid(pid_t pid, int *status, int options);
```

- `options`：控制 `waitpid()` 行为的选项，常用选项包括：
  - `WNOHANG`：非阻塞模式。如果没有子进程终止，立即返回0。
  - `WUNTRACED`：回收已停止的子进程。
  - `WCONTINUED`：回收因 `SIGCONT` 信号恢复运行的子进程。
 
```c
#include <sys/wait.h>
#include <stdio.h>
#include <unistd.h>
#include <stdlib.h>

int main() {
    pid_t pid = fork();
    if (pid == 0) {
        // Child process
        printf("Child process running, PID: %d\n", getpid());
        sleep(2);  // Simulate work
        exit(42);  // Child exit code
    } else if (pid > 0) {
        // Parent process
        int status;
        printf("Parent waiting for child to finish...\n");
        while (1) {
            pid_t child_pid = waitpid(pid, &status, WNOHANG);  // Non-blocking wait
            if (child_pid == 0) {
                printf("Child still running, parent doing other work...\n");
                sleep(1);  // Simulate parent doing other work
            } else if (child_pid > 0) {
                if (WIFEXITED(status)) {
                    printf("Child PID: %d exited with status: %d\n", child_pid, WEXITSTATUS(status));
                }
                break;
            } else {
                perror("waitpid failed");
                exit(1);
            }
        }
    } else {
        perror("fork failed");
        exit(1);
    }
    return 0;
}
```

### 返回值
- 成功：返回终止或状态变化的子进程的`PID`。
- 非阻塞模式：如果没有子进程终止或状态变化，返回`0`。
- 失败：返回`-1`（如没有子进程或参数错误）。

| 特性          | 描述                                                                 |
|---------------|----------------------------------------------------------------------|
| 功能          | 回收指定子进程资源，支持非阻塞模式。                                       |
| 阻塞行为      | 默认阻塞，可通过 `WNOHANG` 实现非阻塞。                                      |
| 返回值        | 成功返回子进程PID，非阻塞模式下无子进程终止时返回0，失败返回-1。                |
| 避免僵尸进程  | 必须调用 `waitpid()` 或 `wait()` 回收子进程。                                  |
| 扩展功能      | 支持处理子进程的停止和恢复状态。                                           |

---

## 七. 僵尸进程、守护进程与孤儿进程

### 僵尸进程（`Zombie Process`）

- **僵尸进程**：子进程已经终止，但其退出状态尚未被父进程回收（通过 `wait()` 或 `waitpid()`），导致子进程的进程描述符仍然保留在系统中。
- **资源占用**：僵尸进程不占用内存，但仍占用进程表项（PID）。
- **危害**：大量僵尸进程会耗尽可用PID，导致新进程无法创建。
- 父进程未调用 `wait()` 或 `waitpid()` 回收子进程的退出状态。

### 避免方法
- **显式回收**：父进程必须调用 `wait()` 或 `waitpid()` 回收子进程。
- **信号处理**：父进程可以捕获 `SIGCHLD` 信号，在信号处理函数中调用 `wait()`。
---

### 孤儿进程（`Orphan Process`）
- 孤儿进程：父进程在子进程终止前已退出，子进程被 `init` 进程（`PID 1`）接管。
- 无害性：孤儿进程会被 init 进程回收，不会成为僵尸进程。
- 生命周期：孤儿进程继续运行，直到完成其任务或终止。
- `init` 进程接管：`init` 进程定期调用 `wait()` 回收孤儿进程，避免其成为僵尸进程。

---

### 守护进程（Daemon Process）
- 守护进程：在后台运行的进程，通常用于提供系统服务（如 `sshd`、`httpd`），脱离终端控制。
- 生命周期长：守护进程通常在系统启动时启动，直到系统关闭才终止。
- 脱离终端：守护进程不与任何终端关联，避免因终端关闭而终止。
- 创建步骤
  - 第一次 `fork()`：创建子进程，父进程退出，子进程成为孤儿进程。
  - 创建新会话：调用 `setsid()` 脱离终端。
  - 第二次 `fork()`（可选）：确保进程不是会话组长。
  - 设置工作目录：`chdir("/")`。
  - 关闭文件描述符：关闭所有打开的文件（如 `close(STDIN_FILENO)`）。
  - 重定向标准`I/O`：通常指向 `/dev/null`。
 
```c
#include <unistd.h>
#include <stdlib.h>
#include <sys/stat.h>
#include <fcntl.h>

int main() {
    pid_t pid = fork();
    if (pid > 0) exit(0);  // 父进程退出

    setsid();               // 创建新会话
    chdir("/");             // 设置根目录为工作目录
    umask(0);               // 重置文件权限掩码

    // 关闭所有文件描述符
    for (int i = 0; i < sysconf(_SC_OPEN_MAX); i++) close(i);

    // 重定向标准I/O到/dev/null
    open("/dev/null", O_RDWR);  // stdin
    dup(0);                     // stdout
    dup(0);                     // stderr

    // 守护进程主逻辑
    while (1) {
        // 执行后台任务（如日志轮转、监控服务）
        sleep(3600);
    }
    return 0;
}
```
---
```c
bool daemonize(void)
{
    pid_t pid = fork();
    if (pid < 0) {
        return false;
    }
    else if (pid > 0) {
        exit(0);
    }
    umask(0);
    pid_t sid = setsid();
    if (sid < 0) {
        return false;
    }
    if ((chdir("/")) < 0) {
        return false;
    }

    close(STDIN_FILENO);
    close(STDOUT_FILENO);
    close(STDERR_FILENO);

    open("/dev/null", O_RDONLY);
    open("/dev/null", O_RDWR);
    open("/dev/null", O_RDWR);
    return true;
}
```
| 进程类型   | 定义                                                                 | 特点                                                                 | 处理方式                                                                 |
|------------|----------------------------------------------------------------------|----------------------------------------------------------------------|--------------------------------------------------------------------------|
| 僵尸进程   | 子进程已终止，但父进程未回收其退出状态。                                   | 占用`PID`，不占用内存；大量僵尸进程会导致PID耗尽。                        | 父进程调用 `wait()` 或 `waitpid()` 回收子进程。                           |
| 守护进程   | 后台运行的进程，脱离终端控制，通常用于提供系统服务。                         | 生命周期长，不与终端关联；通常在系统启动时启动，系统关闭时终止。           | 通过 `fork()`、`setsid()`、关闭文件描述符等步骤创建。                     |
| 孤儿进程   | 父进程在子进程终止前已退出，子进程被 `init` 进程接管。                       | 无害，`init` 进程会回收孤儿进程；继续运行直到完成任务或终止。              | 由 `init` 进程自动回收，无需额外处理。                                    |
