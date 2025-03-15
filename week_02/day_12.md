## 一. 线程核心知识

### 1. 线程的本质
线程是操作系统能够进行运算调度的最小单位，被包含在进程之中，是进程的实际执行单元。一个进程可以包含多个线程，所有线程共享进程的资源（如内存、文件句柄等），但每个线程拥有独立的执行栈和程序计数器。

### 2. 线程 `vs` 进程
| 特性       | 进程                                      | 线程                                      |
|------------|------------------------------------------|------------------------------------------|
| 资源开销   | 高（独立内存空间、上下文切换复杂）         | 低（共享内存，切换速度快）                |
| 独立性     | 完全独立，崩溃不影响其他进程               | 共享内存，一个线程崩溃可能导致整个进程终止 |
| 通信方式   | 复杂（管道、信号、共享内存等）              | 简单（直接读写共享变量）                   |
| 应用场景   | 需要高安全性和独立性的任务                 | 需要高并发和资源共享的任务                 |

### 3. 线程的生命周期
- **创建 `Created`** 线程被创建，但尚未被调度执行。
- **就绪 `Ready`** 线程已准备好运行，等待CPU时间片。
- **运行 `Running`** 线程正在CPU上执行。
- **阻塞 `Blocked`** 线程等待某事件（如I/O操作完成、锁释放）。
- **终止 `Terminated`** 线程执行完毕或被强制终止。

### 4. 线程的同步与通信
- 多个线程访问共享资源时，可能引发竞态条件（`Race Condition`），导致数据不一致。
- 互斥锁（`Mutex`）：确保同一时间只有一个线程访问共享资源。
- 条件变量（`Condition Variable`）：允许线程在特定条件下等待或唤醒其他线程。
- 信号量（`Semaphore`）：控制对共享资源的并发访问数量。

### 5. 线程池
- 预先创建一组线程，任务到来时直接分配，避免频繁创建/销毁线程的开销。
- 减少线程创建的开销。
- 控制并发线程数量，防止资源耗尽。
- 统一管理任务队列和线程状态。

---
## 二. 线程创建函数 `pthread_create`

- 创建一个新线程并执行指定的函数。

```c
#include <pthread.h>
int pthread_create(pthread_t *thread, const pthread_attr_t *attr, void *(*start_routine)(void *), void *arg);

void* thread_func(void *arg)
{
    printf("Thread running\n");
    return NULL;
}

int main()
{
    pthread_t tid;
    if (pthread_create(&tid, NULL, thread_func, NULL) != 0) {
        perror("pthread_create failed");
    }
    // ...
}

/*
参数
    thread：指向线程标识符的指针（输出参数）。
    attr：线程属性（可设为 NULL 使用默认属性）。
    start_routine：线程入口函数（函数指针，返回 void*，参数为 void*）。
    arg：传递给入口函数的参数（可设为 NULL）。
返回值
    成功返回 0，失败返回错误码（非 errno，需用 strerror 转换）。
*/
```
---

## 三. 线程等待函数 `pthread_join`
- 阻塞当前线程，直到目标线程结束，并获取其返回值。

```c
int pthread_join(pthread_t thread, void **retval);

void* thread_func(void *arg)
{
    return (void*)42;
}

int main()
{
    pthread_t tid;
    pthread_create(&tid, NULL, thread_func, NULL);
    void *retval;
    pthread_join(tid, &retval);
    printf("Thread returned: %ld\n", (long)retval); // 输出 42
}

/*
参数
    thread：目标线程标识符。
    retval：存储线程返回值的指针（可设为 NULL）。
*/
```
---

## 四. 线程分离函数 `pthread_detach`
- 将线程标记为分离状态，线程结束后自动释放资源（无需 `pthread_join`）

```c
int pthread_detach(pthread_t thread);

void* thread_func(void *arg)
{
    printf("Detached thread running\n");
    return NULL;
}

int main()
{
    pthread_t tid;
    pthread_create(&tid, NULL, thread_func, NULL);
    pthread_detach(tid); // 线程结束后自动回收资源
    // 无需调用 pthread_join
}
```
---

## 五. 线程属性函数

- 线程属性对象（`pthread_attr_t`）用于定制线程的初始状态（如分离状态、栈大小、调度策略等）
```c
#include <pthread.h>
int pthread_attr_init(pthread_attr_t *attr);    // 初始化属性对象
int pthread_attr_destroy(pthread_attr_t *attr); // 销毁属性对象
int pthread_attr_setdetachstate(pthread_attr_t *attr, int detachstate);
int pthread_attr_setstacksize(pthread_attr_t *attr, size_t stacksize);
int pthread_attr_setschedpolicy(pthread_attr_t *attr, int policy); // 策略：SCHED_FIFO, SCHED_RR
int pthread_attr_setschedparam(pthread_attr_t *attr, const struct sched_param *param);

// 创建分离线程
pthread_attr_t attr;
pthread_attr_init(&attr);
pthread_attr_setdetachstate(&attr, PTHREAD_CREATE_DETACHED);

pthread_t tid;
pthread_create(&tid, &attr, thread_func, NULL);
pthread_attr_destroy(&attr);
```

---
## 六. `pthread_exit` 显式终止当前线程

- 终止调用它的线程，并返回一个值（`retval`）给调用 `pthread_join` 的线程。
- 如果线程未分离（`detached`），必须由其他线程调用 `pthread_join` 回收其资源。
- 在线程函数中主动退出。
- 主线程调用 `pthread_exit` 后，进程不会终止，直到所有线程完成。

```c
void pthread_exit(void *retval);

void* thread_func(void* arg)
{
    printf("chid thread running ...\n");
    pthread_exit((void*)42); // 显式退出并返回值 42
}

int main()
{
    pthread_t tid;
    pthread_create(&tid, NULL, thread_func, NULL);
    
    void* retval;
    pthread_join(tid, &retval); // 等待线程结束并获取返回值
    printf("%ld\n", (long)retval); // 输出 42
    
    return 0;
}
```

---

## 七. `pthread_cancel` 请求终止目标线程

- 向目标线程发送取消请求，目标线程需在取消点（`cancellation point`）响应请求。
- 线程可设置取消状态（`enabled/disabled`）和取消类型（`deferred/asynchronous`）
- 强制终止长时间运行或阻塞的线程（如死循环、等待`I/O`）。
- 需配合线程的取消处理清理函数（`cleanup handlers`）使用，避免资源泄漏。

```c
int pthread_cancel(pthread_t thread);

#include <pthread.h>
#include <stdio.h>
#include <unistd.h>

void cleanup(void* arg)
{
    printf("Cleaning up resources: %s\n", (char*)arg);
}

void* thread_func(void* arg)
{
    pthread_cleanup_push(cleanup, "Releasing lock"); // Register cleanup function
    while (1) {
        pthread_testcancel(); // Manually insert cancellation point
        printf("Thread is running...\n");
        sleep(1);
    }
    pthread_cleanup_pop(0); // 0 means do not execute cleanup function (only on cancellation)
    return NULL;
}

int main()
{
    pthread_t tid;
    pthread_create(&tid, NULL, thread_func, NULL);
    sleep(3); // Wait for 3 seconds before canceling the thread
    pthread_cancel(tid); // Send cancellation request
    pthread_join(tid, NULL); // Wait for the thread to terminate
    printf("Thread has been terminated\n");
    return 0;
}
```

---

## 八. 总结

### 1. `pthread_exit vs pthread_cancel`

| 特性               | `pthread_exit`                          | `pthread_cancel`                     |
|--------------------|----------------------------------------|---------------------------------------|
| **调用者**         | 线程自身                               | 其他线程                              |
| **控制权**         | 完全由线程自身控制                     | 依赖目标线程的取消点和状态设置        |
| **资源清理**       | 需在线程函数中手动处理                 | 需通过清理函数（`pthread_cleanup_*`） |
| **适用场景**       | 主动退出                               | 强制终止失控线程                      |
| **风险**           | 返回值需正确处理                       | 可能导致资源泄漏或数据不一致          |

### 2. 线程状态管理函数
| 函数                     | 作用                                                         |
|--------------------------|-------------------------------------------------------------|
| `pthread_join`           | 等待线程终止并回收资源。                                     |
| `pthread_detach`         | 分离线程，使其终止后自动回收资源。                           |
| `pthread_setcancelstate` | 启用/禁用线程取消功能。                                      |
| `pthread_setcanceltype`  | 设置取消类型（延迟或异步）。                                 |

```c
pthread_t pthread_self(void);                   // 返回当前线程的线程 ID
int pthread_equal(pthread_t t1, pthread_t t2);  // 比较两个线程 ID 是否相等
```
---
