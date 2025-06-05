





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

## 四. 信号屏蔽与阻塞函数
通过设置信号屏蔽字（`Signal Mask`），指定哪些信号在进程执行期间会被暂时阻塞（`Blocked`），即不会递送给进程处理。

### 1. 信号集（`Signal Set`）
信号集用于表示一组信号，以下函数用于操作信号集：

- `int sigemptyset(sigset_t *set)`：清空信号集。
- `int sigfillset(sigset_t *set)`：填充所有信号到信号集。
- `int sigaddset(sigset_t *set, int signum)`：添加指定信号到信号集。
- `int sigdelset(sigset_t *set, int signum)`：从信号集中移除指定信号。
- `int sigismember(const sigset_t *set, int signum)`：检查信号是否在集合中。


### 2. `sigprocmask()` 用于设置或修改进程的信号屏蔽字

```c
int sigprocmask(int how, const sigset_t *set, sigset_t *oldset);
/*
参数：
    how：操作类型，可选值：
    SIG_BLOCK：将 set 中的信号添加到当前屏蔽字。
    SIG_UNBLOCK：从当前屏蔽字中移除 set 中的信号。
    SIG_SETMASK：直接将当前屏蔽字设置为 set。
    set：新的信号集。
    oldset：保存旧的信号屏蔽字（可为 NULL）。
*/

#include <stdio.h>
#include <signal.h>
#include <unistd.h>

int main() {
    sigset_t new_mask, old_mask;

    // 初始化信号集
    sigemptyset(&new_mask);
    sigaddset(&new_mask, SIGINT);  // 添加 SIGINT 到屏蔽集

    // 屏蔽 SIGINT 信号
    sigprocmask(SIG_BLOCK, &new_mask, &old_mask);

    printf("SIGINT 已被屏蔽，按下 Ctrl+C 不会终止进程...\n");
    sleep(5);  // 模拟关键代码执行

    // 恢复原来的信号屏蔽字
    sigprocmask(SIG_SETMASK, &old_mask, NULL);
    printf("SIGINT 已解除屏蔽，按下 Ctrl+C 可终止进程。\n");

    while(1) pause();  // 等待信号
    return 0;
}
```

### 3. `sigpending()` 用于获取当前被阻塞且未处理的信号
```c
#include <signal.h>
int sigpending(sigset_t *set);
// set： 一个指向 sigset_t 类型的指针，用于存储当前被阻塞且未处理的信号集。
// 如果成功调用，set 会被填充为当前未处理的信号集。

#include <stdio.h>
#include <signal.h>
#include <unistd.h>

int main() {
    sigset_t new_mask, old_mask, pending_mask;
    sigemptyset(&new_mask);
    sigaddset(&new_mask, SIGINT);  // Add SIGINT to the signal set

    sigprocmask(SIG_BLOCK, &new_mask, &old_mask);

    printf("SIGINT is blocked. Pressing Ctrl+C will not terminate the process...\n");
    sleep(5);  // Simulate critical code execution

    if (sigpending(&pending_mask) == -1) {
        perror("sigpending");
        return 1;
    }

    if (sigismember(&pending_mask, SIGINT)) {
        printf("SIGINT is blocked and pending.\n");
    } else {
        printf("No pending SIGINT signals.\n");
    }

    sigprocmask(SIG_SETMASK, &old_mask, NULL);
    printf("SIGINT is unblocked. Pressing Ctrl+C will terminate the process.\n");

    while(1) pause();  // Wait for signals
    return 0;
}
```
---
