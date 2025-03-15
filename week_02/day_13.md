## 一. 互斥量 `Mutex`
- 互斥量是线程同步中最基础的锁机制，用于确保多个线程对共享资源的 独占访问。当多个线程需要操作同一份数据（如全局变量、内存缓冲区等）时，互斥量可以防止数据竞争，保证操作的原子性和一致性。
- 临界区 (`Critical Section`)：需要被互斥量保护的代码段，同一时刻只能有一个线程执行。
- 原子性 (`Atomicity`)：临界区内的操作要么完整执行，要么完全不执行，不会被其他线程打断。
- 阻塞 `vs` 非阻塞：`lock()` 会阻塞线程直到获取锁，`trylock()` 立即返回是否成功。

### 核心函数
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
## 三. 信号量 `Semaphore`
- 信号量 `Semaphore` 是一种更通用的同步机制，用于控制对有限数量资源的并发访问。与互斥量（仅保护单个资源）不同，信号量可以管理多个相同类型的资源（如线程池、数据库连接池）。它基于经典的 `PV`操作，是解决多线程/进程同步问题的核心工具之一。
- 计数器机制：信号量维护一个整数值，表示可用资源数量。
  - 正数：表示当前可用资源数。
  - 零或负数：绝对值表示等待资源的线程/进程数。
- 原子操作：
  - `P` 操作 (`sem_wait`)：请求资源，若计数器 `> 0` 则减`1`，否则阻塞。
  - `V` 操作 (`sem_post`)：释放资源，计数器加`1`，并唤醒等待线程。
- 类型：
  - 二进制信号量：计数器值只能是 0 或 1，等同于互斥锁。
  - 计数信号量：计数器值可大于 1，用于管理多个资源。

### 核心函数
- `sem_init()` 初始化未命名的信号量。
- `sem_wait()`
  - 功能：执行 `P` 操作，请求资源。若资源不足（计数器 `≤ 0`），线程阻塞。
  - 非阻塞版本：`sem_trywait()` 立即返回错误，`sem_timedwait()` 支持超时等待。
- `sem_post()` 执行 `V` 操作，释放资源，计数器加`1`，并唤醒一个等待线程。
- `sem_destroy()` 销毁信号量，释放内核资源。需确保无线程等待信号量。

### 生产者-消费者模型（有限缓冲区）
```c
#include <semaphore.h>
#include <pthread.h>
#include <stdio.h>

#define BUFFER_SIZE 5

int buffer[BUFFER_SIZE];
sem_t empty;   // 空槽位信号量（初始为缓冲区大小）
sem_t full;    // 已填充槽位信号量（初始为0）
pthread_mutex_t mutex;  // 互斥量保护缓冲区操作

void* producer(void* arg)
{
    int item = 0;
    while (1) {
        sem_wait(&empty);          // 等待空槽位（P操作）
        pthread_mutex_lock(&mutex); 

        // 生产数据到缓冲区
        buffer[item % BUFFER_SIZE] = item;
        printf("Producer: item %d\n", item);
        item++;

        pthread_mutex_unlock(&mutex);
        sem_post(&full);           // 增加已用槽位（V操作）
    }
    return NULL;
}

void* consumer(void* arg)
{
    int item;
    while (1) {
        sem_wait(&full);           // 等待已填充槽位（P操作）
        pthread_mutex_lock(&mutex);

        // 消费缓冲区数据
        item = buffer[item % BUFFER_SIZE];
        printf("Consumer: item %d\n", item);

        pthread_mutex_unlock(&mutex);
        sem_post(&empty);          // 释放空槽位（V操作）
    }
    return NULL;
}

int main()
{
    pthread_t prod, cons;
    sem_init(&empty, 0, BUFFER_SIZE);  // 初始空槽位=5
    sem_init(&full, 0, 0);            // 初始已用槽位=0
    pthread_mutex_init(&mutex, NULL);

    pthread_create(&prod, NULL, producer, NULL);
    pthread_create(&cons, NULL, consumer, NULL);

    pthread_join(prod, NULL);
    pthread_join(cons, NULL);

    sem_destroy(&empty);
    sem_destroy(&full);
    pthread_mutex_destroy(&mutex);
    return 0;
}
```

---

## 四. 条件变量 `Condition Variable`
- 条件变量`Condition Variable`是用于线程间状态通知的同步机制，通常与互斥量配合使用。它的核心功能是让线程在某个条件不满足时主动阻塞，并在条件可能满足时被唤醒，从而避免忙等待`Busy Waiting`，提升效率。典型应用场景包括生产者-消费者模型、任务队列调度等。
- 线程检查某个共享状态（如队列是否为空），若不满足条件则阻塞。
- 当其他线程修改了共享状态（如向队列添加数据），通过信号唤醒等待线程。
- **虚假唤醒**`Spurious Wakeup`线程可能在没有收到明确信号时被唤醒，因此必须在循环中检查条件。

### 核心函数
- `pthread_cond_init()` 初始化条件变量，可设置属性（如进程间共享）。
- `pthread_cond_wait()` 释放互斥锁并阻塞线程，等待条件变量被唤醒。
- `pthread_cond_signal() 与 pthread_cond_broadcast()`
  - `signal()` 唤醒至少一个等待线程。
  - `broadcast()` 唤醒所有等待线程。
- `pthread_cond_timedwait()` 带超时机制的等待，避免永久阻塞。
- `pthread_cond_destroy()` 销毁条件变量，需确保无线程在等待。

```c
pthread_cond_t cond;
pthread_cond_init(&cond, NULL);

pthread_mutex_lock(&mutex);
while (condition_not_met) {  // 必须用 while，不能用 if
    pthread_cond_wait(&cond, &mutex);
}
// 条件满足后的操作
pthread_mutex_unlock(&mutex);

pthread_mutex_lock(&mutex);
// 修改共享状态，使条件可能满足
pthread_cond_signal(&cond);  // 或 broadcast()
pthread_mutex_unlock(&mutex);
```

### 生产者-消费者模型
```c
#include <pthread.h>
#include <stdio.h>
#include <stdlib.h>

#define QUEUE_SIZE 5

int queue[QUEUE_SIZE];
int count = 0;  // 队列中元素数量
pthread_mutex_t mutex = PTHREAD_MUTEX_INITIALIZER;
pthread_cond_t cond_producer = PTHREAD_COND_INITIALIZER;  // 生产者条件变量（队列未满）
pthread_cond_t cond_consumer = PTHREAD_COND_INITIALIZER;  // 消费者条件变量（队列非空）

// 生产者线程
void* producer(void* arg)
{
    int item = 0;
    while (1) {
        pthread_mutex_lock(&mutex);
        // 队列已满，等待消费者消费
        while (count == QUEUE_SIZE) {
            pthread_cond_wait(&cond_producer, &mutex);
        }
        // 生产数据
        queue[count] = item;
        printf("Producer: item %d added, queue size: %d\n", item, count + 1);
        count++;
        item++;
        pthread_cond_signal(&cond_consumer);  // 通知消费者队列非空
        pthread_mutex_unlock(&mutex);
    }
    return NULL;
}

// 消费者线程
void* consumer(void* arg)
{
    while (1) {
        pthread_mutex_lock(&mutex);
        // 队列为空，等待生产者生产
        while (count == 0) {
            pthread_cond_wait(&cond_consumer, &mutex);
        }
        // 消费数据
        int item = queue[count - 1];
        count--;
        printf("Consumer: item %d removed, queue size: %d\n", item, count);
        pthread_cond_signal(&cond_producer);  // 通知生产者队列未满
        pthread_mutex_unlock(&mutex);
    }
    return NULL;
}

int main()
{
    pthread_t prod, cons;
    pthread_create(&prod, NULL, producer, NULL);
    pthread_create(&cons, NULL, consumer, NULL);

    pthread_join(prod, NULL);
    pthread_join(cons, NULL);

    pthread_cond_destroy(&cond_producer);
    pthread_cond_destroy(&cond_consumer);
    pthread_mutex_destroy(&mutex);
    return 0;
}
```

---

## 五. 自旋锁 `Spinlock`
- 自旋锁`Spinlock`是一种忙等待锁，当线程尝试获取锁时，若锁已被占用，线程会持续循环检查锁的状态（“自旋”），直到锁被释放。与互斥量不同，自旋锁不会让线程进入睡眠状态，因此 避免了上下文切换的开销，但会占用 `CPU` 资源。它适用于极短临界区且多核`CPU`的场景（如内核中断处理、高频无阻塞操作）。
- 忙等待 (`Busy Waiting`)：线程在等待锁时持续占用 `CPU`，不释放执行权。
- 无调度开销：不涉及线程阻塞和唤醒，适合极短临界区。
- 多核优化：在单核 `CPU` 上无意义（自旋时其他线程无法运行）。
- 不可递归：同一线程重复加锁会导致死锁（除非实现支持递归自旋锁）。

### `POSIX` 自旋锁函数
- `pthread_spin_init()`	初始化自旋锁，可设置进程共享属性。
- `pthread_spin_destroy()`	销毁自旋锁。
- `pthread_spin_lock()`	阻塞式加锁，若锁被占用则自旋等待。
- `pthread_spin_trylock()`	非阻塞尝试加锁，失败立即返回错误。
- `pthread_spin_unlock()`	释放锁。

```c
// 自旋锁是高性能场景下的利器，但需严格把控使用条件：短临界区、多核环境、无阻塞操作
#include <pthread.h>
#include <stdio.h>

int counter = 0;
pthread_spinlock_t spinlock;

void* increment(void* arg)
{
    for (int i = 0; i < 100000; ++i) {
        pthread_spin_lock(&spinlock);  // 自旋等待
        counter++;
        pthread_spin_unlock(&spinlock);
    }
    return NULL;
}

int main()
{
    pthread_t t1, t2;
    pthread_spin_init(&spinlock, PTHREAD_PROCESS_PRIVATE);  // 初始化自旋锁

    pthread_create(&t1, NULL, increment, NULL);
    pthread_create(&t2, NULL, increment, NULL);

    pthread_join(t1, NULL);
    pthread_join(t2, NULL);

    printf("Final counter: %d\n", counter);  // 正确输出 200000
    pthread_spin_destroy(&spinlock);
    return 0;
}
```

---
## 六. 总结
### 信号量 `Semaphore`
- 核心机制：`计数器 + PV操作（wait/signal）`。
- 特点：
  - 管理多个资源，支持线程/进程间同步。
  - 分为二进制信号量和计数信号量。
- 适用场景：
  - 资源池管理（如数据库连接池）。
  - 生产者-消费者模型（控制缓冲区空位和已用资源）。
  - 跨进程同步（通过共享内存）。
### 2. 互斥量 `Mutex`
- 核心机制：排他锁，保证临界区独占访问。
- 特点：
  - 线程获取锁失败时会阻塞（释放`CPU`）。
  - 不可递归（除非设置递归属性）。
- 适用场景：
  - 保护共享变量（如全局计数器）。
  - 确保数据结构操作的原子性（如链表插入）。
