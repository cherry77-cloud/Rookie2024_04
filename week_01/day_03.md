### 1. 引用的基本概念  
- **引用不是对象**：  
  - 不占用额外内存空间，本质是对象的别名。  
  - 不支持引用算术运算（如 `ref++`），不可修改绑定关系。  

- **引用必须初始化**：  
  - 必须绑定到已存在的对象，且绑定后不可更改。  
  - 未初始化的引用会导致编译错误。  

- **通过引用访问对象**：  
  - 对引用的操作等价于对原对象的操作（如 `int& r = a; r = 5` 等价于 `a = 5`）。  

---

### 2. 左值（lvalue）  
- **定义**：具有明确存储位置（内存地址）的对象。  
- **特性**：  
  - 可出现在赋值表达式左侧（如 `a = 10`）。  
  - 支持取地址操作（`&a`）。  
  - 生命周期持久（超出作用域才销毁）。  
- **典型例子**：  
  - 变量名（`int x;`）  
  - 返回左值引用的函数（`int& func();`）  
  - 前置自增表达式（`++i`）  

---

### 3. 右值（rvalue）  
- **定义**：临时对象，无持久存储位置。  
- **特性**：  
  - 不能出现在赋值左侧（如 `10 = a` 非法）。  
  - 不可取地址（`&(a + b)` 非法）。  
  - 生命周期短暂（表达式结束时销毁）。  
- **典型例子**：  
  - 字面值（`42`, `"hello"`）  
  - 算术表达式结果（`a + b`）  
  - 返回非引用的函数（`int func();`）  

---

### 4. 将亡值（xvalue）  
- **定义**：右值的子集，表示资源可被转移的临时对象。  
- **特性**：  
  - 支持移动语义（资源所有权转移）。  
  - 通过 `std::move` 或返回右值引用的函数生成。  
- **示例**：  
```cpp
std::vector<int> v1 = std::vector<int>(); // 右值 -> 将亡值
std::vector<int> v2 = std::move(v1);      // 转移资源
```
---

### 5. 左值引用  
- **语法**：`T&`   
- 非 `const` 左值引用只能绑定到**左值**（如变量、可修改的对象）。  
```cpp
int a = 10;
int& ref = a;    // ✅ 正确
int& ref2 = 20;  // ❌ 错误：右值无法绑定到非 const 左值引用
```  
- `const` 左值引用可绑定到**左值或右值**，并延长右值的生命周期。  
```cpp
const int& cref1 = a;     // ✅ 绑定左值
const int& cref2 = 30;    // ✅ 绑定右值，生命周期延长至引用作用域结束
```  
 
- **避免拷贝**：函数参数传递时直接操作原对象。  
```cpp
void process(const std::string& str) { /* 避免字符串拷贝 */ }
```  
- **链式调用**：返回对象引用以支持连续操作。  
```cpp
class Widget {
public:
    Widget& setName(const std::string& name) {
        /* ... */ 
        return *this; 
    }
};
widget.setName("A").setName("B"); // 链式调用
```  

---

### 6. 右值引用  
- **语法**：`T&&`    
- 仅能绑定到**右值**或**将亡值（xvalue）**。  
```cpp
int&& rref1 = 42;            // ✅ 绑定右值
int&& rref2 = std::move(a);  // ✅ 绑定将亡值（通过 std::move 转换左值）
int&& rref3 = a;             // ❌ 错误：不能直接绑定左值
```  
- 使用 `std::move` 将左值显式转换为右值引用。  
```cpp
std::vector<int> v1;
std::vector<int> v2 = std::move(v1); // 转移 v1 的资源到 v2
```     
--- 


### 7. 完美转发（Perfect Forwarding）   
- **万能引用**：模板中的 `T&&` 可绑定左值或右值。  
```cpp
template<typename T>
void wrapper(T&& arg) {
    target(std::forward<T>(arg)); // 无损转发
}
```  
- **引用折叠规则**：  
    - `T& &` → `T&`  
    - `T&& &` → `T&`  
    - `T& &&` → `T&`  
    - `T&& &&` → `T&&`  
- **`std::forward`**：根据模板参数 `T` 的推导结果，将参数转发为左值或右值。  


```cpp
void process(int& x) { std::cout << "Lvalue: " << x << std::endl; }
void process(int&& x) { std::cout << "Rvalue: " << x << std::endl; }

template<typename T>
void relay(T&& arg) {
    process(std::forward<T>(arg)); // 完美转发
}

int main() {
    int a = 10;
    relay(a);       // 转发左值
    relay(20);      // 转发右值
    return 0;
}
```

| **特性**             | **左值引用 (`T&`)**       | **右值引用 (`T&&`)**      |
|-----------------------|---------------------------|---------------------------|
| **绑定对象**         | 左值                     | 右值/将亡值               |
| **修改绑定**         | 不可重绑定               | 不可重绑定               |
| **延长右值生命周期** | 支持（仅 `const T&`）    | 不适用                   |
| **典型应用场景**     | 避免拷贝、链式调用       | 移动语义、完美转发       |

---

### 8. 三五法则（Rule of Three/Five）  

#### 三法则: 若自定义以下任一函数，需定义全部三个：  
1. **析构函数**  
2. **拷贝构造函数**  
3. **拷贝赋值运算符**  
 
当类管理动态资源（如堆内存、文件句柄等）时，需遵循三法则，确保资源正确释放和拷贝。  
  
```cpp
class Resource {
public:
    // 构造函数
    Resource(size_t size) : data_(new int[size]), size_(size) {}

    // 析构函数
    ~Resource() { delete[] data_; }

    // 拷贝构造函数
    Resource(const Resource& other) : data_(new int[other.size_]), size_(other.size_) {
        std::copy(other.data_, other.data_ + size_, data_);
    }

    // 拷贝赋值运算符
    Resource& operator=(const Resource& other) {
        if (this != &other) {
            delete[] data_;
            data_ = new int[other.size_];
            size_ = other.size_;
            std::copy(other.data_, other.data_ + size_, data_);
        }
        return *this;
    }

    private:
        int* data_;
        size_t size_;
  };
```
#### 五法则: 若涉及移动语义，还需定义以下两个函数：
1. **移动构造函数**
2. **移动赋值运算符**

当类支持移动语义时，需遵循五法则，确保资源高效转移。
```c++
class Resource {
public:
    // 构造函数
    Resource(size_t size) : data_(new int[size]), size_(size) {}

    // 析构函数
    ~Resource() { delete[] data_; }

    // 拷贝构造函数
    Resource(const Resource& other) : data_(new int[other.size_]), size_(other.size_) {
        std::copy(other.data_, other.data_ + size_, data_);
    }

    // 拷贝赋值运算符
    Resource& operator=(const Resource& other) {
        if (this != &other) {
            delete[] data_;
            data_ = new int[other.size_];
            size_ = other.size_;
            std::copy(other.data_, other.data_ + size_, data_);
        }
        return *this;
    }

    // 移动构造函数
    Resource(Resource&& other) noexcept : data_(other.data_), size_(other.size_) {
        other.data_ = nullptr; // 置空源对象指针
        other.size_ = 0;
    }

    // 移动赋值运算符
    Resource& operator=(Resource&& other) noexcept {
        if (this != &other) {
            delete[] data_;
            data_ = other.data_;
            size_ = other.size_;
            other.data_ = nullptr; // 置空源对象指针
            other.size_ = 0;
        }
        return *this;
    }

private:
    int* data_;
    size_t size_;
};
```
---

```c++
#include <iostream>
#include <string>
#include <utility>

class Processor {
public:
    void operator()(const std::string& s) const {
        std::cout << "Processing lvalue: " << s << std::endl;
    }

    void operator()(std::string&& s) const {
        std::cout << "Processing rvalue: " << s << std::endl;
    }
};

template <typename Func, typename T>
void execute(Func&& func, T&& arg) 
{
    func(std::forward<T>(arg));
}

int main() 
{
    Processor proc;
    std::string str = "Hello";

    std::cout << "Executing with lvalue:" << std::endl;
    execute(proc, str);

    std::cout << "Executing with rvalue (std::move):" << std::endl;
    execute(proc, std::move(str));

    std::cout << "Executing with rvalue (temporary object):" << std::endl;
    execute(proc, std::string("World"));

    return 0;
}
```
