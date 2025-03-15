## 一. 线程创建函数 `pthread_create`

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

## 二. 线程等待函数 `pthread_join`
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

## 三. 线程分离函数 `pthread_detach`
- 将线程标记为分离状态，线程结束后自动释放资源（无需 `pthread_join`）

```c
int pthread_detach(pthread_t thread);

void* thread_func(void *arg)
{
    printf("Detached thread running\n");
    return NULL;
}

int main() {
    pthread_t tid;
    pthread_create(&tid, NULL, thread_func, NULL);
    pthread_detach(tid); // 线程结束后自动回收资源
    // 无需调用 pthread_join
}
```
---

## 四. 线程属性函数

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
