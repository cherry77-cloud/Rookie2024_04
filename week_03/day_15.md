## 一. 信号的基本概念
- 信号（`Signal`）是操作系统中用于通知进程异步事件发生的一种机制，广泛应用于进程控制、异常处理和进程间通信。
- 信号是操作系统内核或进程发送给目标进程的简短通知，用于指示特定事件（如用户中断、程序错误等）的发生。
- 异步性：信号可能在进程执行的任何时刻到达。
- 预定义类型：每种信号有唯一的编号和预定义行为（如终止进程、暂停进程等）。


| **信号名称**   | **信号编号** | **触发场景**                                                                 | **默认行为**              |
|----------------|--------------|-----------------------------------------------------------------------------|--------------------------|
| `SIGHUP`       | 1            | 终端断开或控制进程终止                                                     | 终止进程                 |
| `SIGINT`       | 2            | 用户按下 `Ctrl+C`                                                         | 终止进程                 |
| `SIGQUIT`      | 3            | 用户按下 `Ctrl+\`                                                         | 终止并生成核心转储       |
| `SIGKILL`      | 9            | 强制终止进程（**不可捕获或忽略**）                                         | 终止进程                 |
| `SIGSEGV`      | 11           | 进程访问非法内存地址（段错误）                                             | 终止并生成核心转储       |
| `SIGPIPE`      | 13           | 向无读端的管道或套接字写入数据                                             | 终止进程                 |
| `SIGALRM`      | 14           | 定时器超时（由 `alarm()` 或 `setitimer()` 设置）                           | 终止进程                 |
| `SIGTERM`      | 15           | 请求进程终止（可捕获或忽略）                                               | 终止进程                 |
| `SIGCHLD`      | 17           | 子进程状态改变（终止或暂停）                                               | 忽略                     |
| `SIGSTOP`      | 19           | 强制暂停进程（**不可捕获或忽略**，如 `kill -STOP <PID>`）                  | 暂停进程                 |
| `SIGCONT`      | 18           | 恢复已暂停的进程（如通过 `kill -CONT <PID>`）                              | 继续进程                 |
|`SIGURG`        | 23	        |紧急数据到达套接字（如TCP带外数据）	                                          |忽略 | 

- 常见默认行为：
  - 终止进程（`Terminate`）：如 `SIGINT（Ctrl+C）`、`SIGTERM`。
  - 终止并生成核心转储（`Terminate with Core Dump`）：如 `SIGSEGV`（段错误）、`SIGABRT`。
  - 忽略（`Ignore`）：如 `SIGCHLD`（子进程状态改变）。
  - 暂停进程（`Stop`）：如 `SIGSTOP、SIGTSTP（Ctrl+Z）`。
  - 继续进程（`Continue`）：如 `SIGCONT`。

---

## 二. 信号处理函数

| **函数**       | **适用场景**                     | **特点**                          |
|----------------|---------------------------------|-----------------------------------|
| `kill`         | 向其他进程发送任意信号           | 灵活，支持进程组和广播           |
| `raise`        | 向当前进程发送信号               | 简化版 `kill(getpid(), sig)`     |
| `sigqueue`     | 发送实时信号并携带数据           | 支持排队和附加数据               |
| `alarm`        | 简单定时任务（秒级）             | 仅触发一次 `SIGALRM`             |
| `setitimer`    | 高精度或周期性定时任务           | 微秒级精度，支持多种定时器类型   |


### 1. `signal()` 注册信号处理函数
- 为指定信号绑定自定义处理函数
```c
void (*signal(int signum, void (*handler)(int)))(int);

void handler(int sig) {
    printf("Received signal: %d\n", sig);
}

int main() {
    signal(SIGINT, handler);  // 捕获 Ctrl+C
    while(1) pause();          // 等待信号
    return 0;
}

/*
signum：信号编号（如 SIGINT）。
handler：处理函数指针，或 SIG_IGN（忽略信号）、SIG_DFL（恢复默认行为）。
*/
```

---

### 2. `sigaction()` 高级信号处理
- 提供更精细的信号控制（如屏蔽信号、设置标志）

```c
int sigaction(int signum, const struct sigaction *act, struct sigaction *oldact);

struct sigaction {
    void     (*sa_handler)(int);                          // 简单处理函数
    void     (*sa_sigaction)(int, siginfo_t *, void *);   // 携带附加数据的处理函数
    sigset_t sa_mask;                                     // 阻塞的信号集
    int      sa_flags;                                    // 标志（如 SA_SIGINFO、SA_RESTART）
};

void handler(int sig, siginfo_t *info, void *context) {
    printf("Received signal %d from PID %d\n", sig, info->si_pid);
}

int main() {
    struct sigaction sa;
    sa.sa_sigaction = handler;
    sa.sa_flags = SA_SIGINFO;
    sigemptyset(&sa.sa_mask);
    sigaction(SIGINT, &sa, NULL);
    while(1) pause();
    return 0;
}
```
---

## 三. 信号的发送
### 1. `kill()` 向进程发送信号
- 向指定进程或进程组发送信号。

```c
int kill(pid_t pid, int sig);
kill(1234, SIGTERM);  // 向 PID 1234 发送终止信号
// pid > 0：发送给指定进程。
// pid = 0：发送给同进程组的所有进程。
// pid = -1：发送给所有有权发送的进程。
// pid < -1：发送给进程组ID为 |pid| 的所有进程
```

### 2. `raise()` 向当前进程发送信号
- 向当前进程发送信号（等价于 `kill(getpid(), sig)`）。
```c
#include <signal.h>
int raise(int sig);

/*
参数：
    sig：信号编号（如 SIGINT）。
返回值：
    成功返回 0，失败返回非零值。
*/
```

### 3. `sigqueue()` 发送实时信号并携带数据
- 发送实时信号（`SIGRTMIN` 至 `SIGRTMAX`），可附加用户数据。
```c
#include <signal.h>
int sigqueue(pid_t pid, int sig, const union sigval value);

union sigval {
    int   sival_int;    // 传递整型数据
    void *sival_ptr;    // 传递指针
};

参数：
    - pid：目标进程ID。
    - sig：实时信号编号（如 SIGRTMIN+1）。
    - value：附加数据（联合体 union sigval）。
```

### 4. `alarm()` 设置定时发送`SIGALRM`信号
- 功能：设置定时器，超时后向当前进程发送 `SIGALRM` 信号。
```c
#include <unistd.h>
unsigned int alarm(unsigned int seconds);
/*
参数：
    seconds：定时时长（秒）。
返回值：
    返回剩余未触发的定时时间（若之前有定时器未触发）。
*/
```

### 5. `setitimer()` 高精度定时发送 `SIGALRM`
- 提供更精细的定时器控制（支持微秒级精度），可周期触发 `SIGALRM`。
```c
#include <sys/time.h>
int setitimer(int which, const struct itimerval *new_val, struct itimerval *old_val);

struct itimerval {
    struct timeval it_interval;  // 周期时间
    struct timeval it_value;     // 初始触发时间
};
struct timeval {
    time_t      tv_sec;         // 秒
    suseconds_t tv_usec;        // 微秒
};

/*
参数：
    which：定时器类型：
    ITIMER_REAL：真实时间（触发 SIGALRM）。
    ITIMER_VIRTUAL：进程用户态CPU时间（触发 SIGVTALRM）。
    ITIMER_PROF：进程用户态+内核态CPU时间（触发 SIGPROF）。
*/
```

---
