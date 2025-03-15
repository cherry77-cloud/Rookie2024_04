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

## 二. 读写锁 `Read-Write Lock`
- 读写锁`Read-Write Lock`是一种 细粒度锁，允许多个线程同时读取共享资源，但写操作必须独占访问。这种锁特别适用于读多写少的场景（例如缓存系统、配置文件读取），能显著提高并发性能。
- 读共享 `Shared Read` 多个线程可同时持有读锁，但此时禁止写操作。
- 写独占 `Exclusive Write` 写锁被持有时，禁止其他线程读或写。
- 某些实现中，写锁请求优先于读锁，防止“写者饥饿”问题。

### 核心函数
- `pthread_rwlock_init()` 初始化读写锁，可设置属性（如是否支持优先级继承）
- `pthread_rwlock_rdlock()` 获取读锁（共享锁），允许其他读线程继续获取读锁。
- `pthread_rwlock_wrlock()` 获取写锁（独占锁），禁止其他线程读或写。
- `pthread_rwlock_tryrdlock() 与 pthread_rwlock_trywrlock()` 非阻塞尝试获取读/写锁，失败立即返回 `EBUSY`。
- `pthread_rwlock_unlock()` 释放读写锁（无论是读锁还是写锁）。
- `pthread_rwlock_destroy()` 销毁读写锁，释放资源。需确保无线程持有或等待锁。

### 通过 `pthread_rwlockattr_t` 可定制读写锁行为，例如设置写者优先级
```c
pthread_rwlockattr_t attr;
pthread_rwlockattr_init(&attr);
pthread_rwlockattr_setkind_np(&attr, PTHREAD_RWLOCK_PREFER_WRITER_NONRECURSIVE_NP);  // 写者优先
pthread_rwlock_init(&rwlock, &attr);
```

```c
#include <pthread.h>
#include <stdio.h>
#include <unistd.h>

int shared_data = 0;      // 共享数据
pthread_rwlock_t rwlock;  // 读写锁

// 读线程：读取共享数据
void* reader(void* arg)
{
    int id = *(int*)arg;
    for (int i = 0; i < 5; i++) {
        pthread_rwlock_rdlock(&rwlock);  // 获取读锁（共享）
        printf("Reader %d reads data: %d\n", id, shared_data);
        pthread_rwlock_unlock(&rwlock);  // 释放读锁
        sleep(1);  // 模拟耗时操作
    }
    return NULL;
}

// 写线程：修改共享数据
void* writer(void* arg)
{
    int id = *(int*)arg;
    for (int i = 0; i < 3; i++) {
        pthread_rwlock_wrlock(&rwlock);  // 获取写锁（独占）
        shared_data++;
        printf("Writer %d writes data: %d\n", id, shared_data);
        pthread_rwlock_unlock(&rwlock);  // 释放写锁
        sleep(2);  // 模拟耗时操作
    }
    return NULL;
}

int main()
{
    pthread_t readers[3], writers[2];
    int reader_ids[] = {1, 2, 3};
    int writer_ids[] = {1, 2};

    pthread_rwlock_init(&rwlock, NULL);

    // 创建 3 个读线程和 2 个写线程
    for (int i = 0; i < 3; i++) {
        pthread_create(&readers[i], NULL, reader, &reader_ids[i]);
    }
    for (int i = 0; i < 2; i++) {
        pthread_create(&writers[i], NULL, writer, &writer_ids[i]);
    }

    // 等待所有线程结束
    for (int i = 0; i < 3; i++) {
        pthread_join(readers[i], NULL);
    }
    for (int i = 0; i < 2; i++) {
        pthread_join(writers[i], NULL);
    }

    pthread_rwlock_destroy(&rwlock);
    return 0;
}
```

---
