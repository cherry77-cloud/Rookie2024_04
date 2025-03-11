## `C++` 智能指针总结

### 1. 引入目的
- **自动化内存管理**：通过 `RAII`（资源获取即初始化）机制自动管理内存，减少手动 `new/delete` 的复杂性。
- **核心目标**：防止内存泄漏、悬挂指针、双重释放等问题。

---

### 2. 原生指针的缺陷
| 问题类型          | 描述                                                                 |
|-------------------|----------------------------------------------------------------------|
| **内存泄漏**       | 分配内存后忘记释放（`delete`）                                       |
| **悬挂指针**       | 指针指向的内存被释放后，仍被访问                                     |
| **双重释放**       | 多个指针指向同一内存，重复调用 `delete` 导致未定义行为               |

---

### 3. 智能指针特性
| 特性              | 说明                                                                 |
|-------------------|----------------------------------------------------------------------|
| **自动销毁**       | 生命周期结束时自动释放资源                                           |
| **引用计数**       | `shared_ptr` 跟踪引用数量，计数归零时释放资源                         |
| **避免内存泄漏**   | 通过 `RAII` 确保资源自动释放                                            |
| **类型安全**       | 提供更严格的资源所有权管理（注：类型检查与原生指针一致）             |
| **独占所有权**     | `unique_ptr` 确保资源仅由一个所有者管理                               |
| **共享所有权**     | `shared_ptr` 允许多个指针共享同一资源                                 |
| **观察者模式**     | `weak_ptr` 用于观察资源而不影响其生命周期                             |

---

### 4. `std::unique_ptr`
#### 核心特性
- **独占所有权**：同一时刻仅一个 `unique_ptr` 拥有资源。
- **轻量级**：无引用计数，性能开销小。
- **不可拷贝**：只能通过移动语义转移所有权。

#### 构造函数与操作
| 操作类型          | 行为                                                                 |
|-------------------|----------------------------------------------------------------------|
| 默认构造          | 创建空的 `unique_ptr`                                                |
| 指针构造          | 从裸指针接管所有权                                                   |
| 移动构造/赋值     | 转移资源所有权，原指针置空                                           |
| 自定义删除器      | 支持自定义删除器，用于管理非 `new` 分配的资源（如文件句柄）          |

---

### 5. `std::shared_ptr`
#### 核心特性
- **共享所有权**：多个 `shared_ptr` 可指向同一资源。
- **引用计数**：通过控制块管理强引用（`use_count`）和弱引用（`weak_count`）。

| 操作类型          | 行为                                                                 |
|-------------------|----------------------------------------------------------------------|
| **默认构造**       | 创建空的 `shared_ptr`                                                |
| **指针构造**       | 创建新控制块并接管资源（注：避免用同一裸指针初始化多个 `shared_ptr`）|
| **拷贝构造/赋值**  | 增加引用计数                                                         |
| **移动构造/赋值**  | 转移所有权，原指针置空                                               |
| **自定义删除器**   | 支持自定义删除器，用于管理非 `new` 分配的资源                        |


```cpp
std::shared_ptr<int> ptr1(new int(10)); // 从裸指针构造
std::shared_ptr<int> ptr2 = ptr1; // 拷贝构造，引用计数增加
std::shared_ptr<int> ptr3 = std::move(ptr1); // 移动构造，ptr1 置空
```
---

### 6. `std::weak_ptr`
#### 核心特性 
- 非拥有观察者：不增加引用计数，不管理资源生命周期。
- 解决循环引用：打破 `shared_ptr` 双向依赖导致的资源泄漏。
- `lock()`返回一个 `shared_ptr`，若资源存在则有效，否则为空
- `expired()`检查资源是否已被释放

```c++
std::shared_ptr<int> sharedPtr = std::make_shared<int>(10);
std::weak_ptr<int> weakPtr = sharedPtr;

if (auto lockedPtr = weakPtr.lock()) {
    // 资源仍存在，lockedPtr 是一个有效的 shared_ptr
} else {
    // 资源已被释放
}
```
---

### 7. 补充
- 优先使用：`make_shared<T>()` 和 `make_unique<T>()` 创建智能指针，避免直接操作裸指针。
- `C++` 智能指针通过 `RAII` 机制和引用计数，提供了高效、安全的内存管理方式
- `unique_ptr`：独占所有权，适合单一所有者场景。`shared_ptr`：共享所有权，适合多所有者场景。`weak_ptr`：观察者模式，解决循环引用问题。
---

```c++
#include <iostream>
#include <memory>

class Test {
public:
    Test(int val) : value(val) { std::cout << "Test Constructor: " << value << std::endl; }
    ~Test() { std::cout << "Test Destructor: " << value << std::endl; }
    void show() const { std::cout << "Value: " << value << std::endl; }
private:
    int value;
};

std::unique_ptr<Test> createTest(int val) 
{
    auto ptr = std::make_unique<Test>(val);
    return ptr;
}

int main() 
{
    std::unique_ptr<Test> ptr1(new Test(100));  
    ptr1->show();

    auto ptr2 = std::make_unique<Test>(200);
    ptr2->show();

    std::unique_ptr<Test> ptr3 = std::move(ptr1);
    if (!ptr1) {
        std::cout << "ptr1 is now null after std::move." << std::endl;
    }
    ptr3->show();

    ptr2.reset(new Test(300));   // Test Destructor: 200
    ptr2->show();

    Test* rawPtr = ptr3.release();
    if (!ptr3) {
        std::cout << "ptr3 is now null after release." << std::endl;
    }
    if (rawPtr) {
        std::cout << "rawPtr (from ptr3): ";
        rawPtr->show();
        delete rawPtr;  // Test Destructor: 100
        rawPtr = nullptr;
    }

    auto ptr4 = createTest(400);   // Test Destructor: 400  Test Destructor: 300
    return 0;
}
```

---

```c++
#include <iostream>
#include <memory>

class Test {
public:
    Test(int val) : value(val) {
        std::cout << "Test Constructor: " << value << std::endl;
    }
    ~Test() {
        std::cout << "Test Destructor: " << value << std::endl;
    }
    void show() const {
        std::cout << "Value: " << value << std::endl;
    }

private:
    int value;
};

int main() 
{
    // 1. 创建一个 shared_ptr
    std::shared_ptr<Test> sp1(new Test(100));
    std::cout << "sp1 use_count: " << sp1.use_count() << std::endl;
    sp1->show();

    // 2. 通过拷贝构造共享所有权
    std::shared_ptr<Test> sp2 = sp1;
    std::cout << "After sp2 = sp1:" << std::endl;
    std::cout << "sp1 use_count: " << sp1.use_count() << std::endl;
    std::cout << "sp2 use_count: " << sp2.use_count() << std::endl;

    // 3. 通过拷贝赋值共享所有权
    std::shared_ptr<Test> sp3;
    sp3 = sp2;
    std::cout << "After sp3 = sp2:" << std::endl;
    std::cout << "sp1 use_count: " << sp1.use_count() << std::endl;
    std::cout << "sp2 use_count: " << sp2.use_count() << std::endl;
    std::cout << "sp3 use_count: " << sp3.use_count() << std::endl;

    // 4. 重置 shared_ptr
    sp2.reset(new Test(200));
    std::cout << "After sp2.reset(new Test(200)):" << std::endl;
    std::cout << "sp1 use_count: " << sp1.use_count() << std::endl;
    std::cout << "sp2 use_count: " << sp2.use_count() << std::endl;
    std::cout << "sp3 use_count: " << sp3.use_count() << std::endl;
    sp2->show();

    std::cout << "Exiting main..." << std::endl;
    return 0;
}
```

---

```c++
// 双向关联导致循环引用
#include <iostream>
#include <memory>

class B; // 前向声明

class A {
public:
    std::shared_ptr<B> ptrB;

    A() { std::cout << "A Constructor" << std::endl; }
    ~A() { std::cout << "A Destructor" << std::endl; }
};

class B {
public:
    std::shared_ptr<A> ptrA;

    B() { std::cout << "B Constructor" << std::endl; }
    ~B() { std::cout << "B Destructor" << std::endl; }
};

int main() 
{
    {
        std::shared_ptr<A> a = std::make_shared<A>();
        std::shared_ptr<B> b = std::make_shared<B>();
        a->ptrB = b;
        b->ptrA = a;
    }
    std::cout << "Exiting main..." << std::endl;
    return 0;
}
```
