## 一. 互斥量 `Mutex`
- 互斥量是线程同步中最基础的锁机制，用于确保多个线程对共享资源的 独占访问。当多个线程需要操作同一份数据（如全局变量、内存缓冲区等）时，互斥量可以防止数据竞争，保证操作的原子性和一致性。
- 临界区 (`Critical Section`)：需要被互斥量保护的代码段，同一时刻只能有一个线程执行。
-原子性 (`Atomicity`)：临界区内的操作要么完整执行，要么完全不执行，不会被其他线程打断。
- 阻塞 `vs` 非阻塞：`lock()` 会阻塞线程直到获取锁，`trylock()` 立即返回是否成功。

### 核心函数详解
- `pthread_mutex_init()` 功能：初始化互斥量，可设置属性（如递归锁、错误检查锁等）。
- `pthread_mutex_destroy()` 销毁互斥量，释放资源。需确保互斥量未被锁定且无线程等待。
- `pthread_mutex_lock()` 阻塞式加锁。若锁已被其他线程持有，当前线程进入等待队列。
- `pthread_mutex_trylock()` 非阻塞式尝试加锁。若锁不可用，立即返回 `EBUSY`。
- `pthread_mutex_unlock()` 释放互斥量，唤醒等待线程。
 
```c
#include <pthread.h>
#include <stdio.h>

int counter = 0;
pthread_mutex_t mutex;

void* increment(void* arg)
{
    for (int i = 0; i < 100000; ++i) {
        pthread_mutex_lock(&mutex);  // 加锁
        counter++;                   // 临界区操作
        pthread_mutex_unlock(&mutex); // 解锁
    }
    return NULL;
}

int main()
{
    pthread_t t1, t2;
    // 初始化互斥量
    if (pthread_mutex_init(&mutex, NULL) != 0) {
        perror("Mutex init failed");
        return 1;
    }

    // 创建两个线程
    pthread_create(&t1, NULL, increment, NULL);
    pthread_create(&t2, NULL, increment, NULL);

    // 等待线程结束
    pthread_join(t1, NULL);
    pthread_join(t2, NULL);

    printf("Final counter: %d\n", counter);  // 正确输出 200000
    pthread_mutex_destroy(&mutex);  // 销毁互斥量
    return 0;
}
```

### 递归锁 `Recursive Mutex`
```c
pthread_mutexattr_t attr;
pthread_mutexattr_init(&attr);
pthread_mutexattr_settype(&attr, PTHREAD_MUTEX_RECURSIVE);  // 设置为递归锁

pthread_mutex_t mutex;
pthread_mutex_init(&mutex, &attr);

// 同一线程可重复加锁
pthread_mutex_lock(&mutex);
pthread_mutex_lock(&mutex);  // 不会阻塞
// ...
pthread_mutex_unlock(&mutex);
pthread_mutex_unlock(&mutex);
```

---
