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
