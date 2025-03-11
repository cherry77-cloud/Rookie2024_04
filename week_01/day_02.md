## 一. `C++` 面向对象编程的三大特性  
#### 封装（`Encapsulation`）  
- **定义**：将数据与操作数据的方法绑定，并对外隐藏实现细节。  
- **核心机制**：  
  - **访问控制**：  
    - `public`：允许类外直接访问。  
    - `protected`：仅允许类内和派生类访问。  
    - `private`：仅允许类内访问（默认）。  
  - **数据隐藏**：通过接口（公有成员函数）操作数据，避免直接暴露内部状态。  

#### 继承（`Inheritance`）  
- **定义**：派生类继承基类的属性和行为，并可扩展或重写功能。  
- **关键点**：  
  - **代码重用**：减少冗余代码，提升开发效率。  
  - **层次化设计**：构建类之间的逻辑关系（如 `Animal` → `Dog`）。  
  - **继承类型**：  
    - 单继承：一个派生类继承一个基类。  
    - 多重继承：一个派生类继承多个基类（需注意菱形继承问题，可通过虚继承解决）。  

#### 多态（`Polymorphism`）  
- **定义**：同一接口在不同上下文中表现出不同行为。  
- **实现方式**：  
  - **编译时多态**（静态）：函数重载、运算符重载、模板。  
  - **运行时多态**（动态）：虚函数、抽象类、接口类。  

---

## 二. 多态机制详解  
#### 静态多态  
- **特点**：在编译期确定具体调用，无运行时开销。  
- **实现方式**：  
  - **函数重载**：同名函数通过参数列表区分（返回值类型不参与重载）。  
  - **运算符重载**：为自定义类型定义运算符行为（如 `+`、`<<`）。  
  - **模板**：泛型编程（如 `template<typename T> T add(T a, T b)`）。  

#### 动态多态  
- **特点**：运行期根据对象实际类型决定函数调用。  
- **核心机制**：  
  - **虚函数（`virtual`）**：  
    - 基类声明虚函数，派生类可重写（`override`）。  
    - 必须通过基类指针或引用调用才能触发多态。  
  - **纯虚函数与抽象类**：  
    - 纯虚函数：`virtual void func() = 0;`  
    - 抽象类：包含至少一个纯虚函数，不能实例化。  
  - **虚函数表（`vtable`）**：  
    - 每个含虚函数的类拥有一个 `vtable`，存储虚函数地址。  
    - 派生类 `vtable` 继承基类条目，重写的函数替换为派生类实现。  
  - **虚指针（`vptr`）**：  
    - 每个对象隐含的指针，指向所属类的 `vtable`。  
    - 对象构造时，`vptr` 被初始化为当前类的 `vtable` 地址。  

---

## 三. 虚函数表（`vtable`）与虚指针（`vptr`）的底层机制  
#### 虚函数表（`vtable`）  
- **创建时机**：  
  - **编译阶段**：编译器为每个含虚函数的类生成 `vtable`。  
  - **链接阶段**：链接器合并 `vtable` 并分配内存地址。  
- **存储位置**：  
  - 通常位于只读数据段（`.rodata`），防止程序修改。  
- **内存布局**：  
  - 按声明顺序存储虚函数地址。  
  - 多重继承时，派生类可能包含多个 `vtable`（每个基类对应一个）。  

#### 虚指针（`vptr`）  
- **创建时机**：  
  - **对象构造时**：构造函数隐式初始化 `vptr`。  
  - **对象析构时**：析构函数将 `vptr` 重置为基类 `vtable`（防止析构过程中的多态错误）。  
- **存储位置**：  
  - 位于对象内存布局的首部（某些编译器可能调整位置）。  

#### 动态多态的工作流程  
1. **对象构造**：`vptr` 指向当前类的 `vtable`。  
2. **虚函数调用**：  
   - 通过对象 `vptr` 找到 `vtable`。  
   - 从 `vtable` 中取出函数地址并调用。  
3. **析构过程**：  
   - 先调用派生类析构函数，再调用基类析构函数。  
   - `vptr` 在析构过程中逐步指向基类 `vtable`。  


1. **虚析构函数**：  
   - 若基类指针指向派生类对象，基类析构函数必须为虚函数，否则可能导致资源泄漏。  
2. **性能开销**：  
   - 虚函数调用需通过 `vptr` 间接寻址，比普通函数调用稍慢。
3. **对象切片**：  
   - 派生类对象赋值给基类对象时，`vptr` 会被重置为基类 `vtable`，多态特性丢失。  

---

## 四. 总结
1. **`vtable` 生成规则**：  
- 只有包含虚函数（包括继承的）的类才会生成 `vtable`。  
- 即使派生类未重写虚函数，仍会生成自己的 `vtable`（继承基类函数地址）。  
2. **多重继承的 `vtable`**：  
- 派生类会为每个基类维护单独的 `vtable`（若基类有虚函数）。  
- 派生类新增的虚函数可能附加到第一个基类的 `vtable` 末尾。  
3. **`RTTI`（运行时类型信息）**：  
- `vtable` 中通常包含类型信息（如 `type_info`），支持 `dynamic_cast` 和 `typeid`。  
---

```c++
#include <iostream>

int func(int x) { return x; }
float func(float x) { return x; }

class C {
public:
    int func(int x) { return x; }
    class C2 {
    public:
        int func(int x) { return x; }
    };
};

namespace N {
    int func(int x) { return x; }
    class C {
    public:
        int func(int x) { return x; }
    };
}

int main() 
{
    func(10);
    func(10.0f);
    C c;
    c.func(10);
    C::C2 c2;
    c2.func(20);
    N::func(30);
    N::C n_c;
    n_c.func(40);
    return 0;
}

// _Z4funcf               c++filt _Z4funcf  -> func(float)
// _Z4funci               c++filt _Z4funci  -> func(int)
// _ZN1C2C24funcEi        c++filt _ZN1C2C24funcEi -> C::C2::func(int)
// _ZN1C4funcEi           c++filt _ZN1C4funcE  ->  C::func(int)
// _ZN1N1C4funcEi         c++filt _ZN1N1C4funcEi  ->  N::C::func(int)
// _ZN1N4funcEi           c++filt _ZN1N4funcEi  ->  N::func(int)
```

---

```c++
#include <iostream>
#include <memory>

class Base {
public:
    virtual void func() {
        std::cout << "Base::func" << std::endl;
    }
    virtual ~Base() = default;
};

class Derive : public Base {
public:
    void func() override {
        std::cout << "Derive::func" << std::endl;
    }
};

int main()
{
    auto pbase1 = std::make_unique<Base>();
    auto pbase2 = std::make_unique<Derive>();
    pbase1->func(); // Base::func
    pbase2->func(); // Derive::func
    return 0;
}
```
---

```c++
#include <iostream>
#include <memory>
#include <cstdint>

class Base {
public:
    virtual void func() {
        std::cout << "Base::func" << std::endl;
    }
    virtual ~Base() = default;
};

class Derive : public Base {
public:
    void func() override {
        std::cout << "Derive::func" << std::endl;
    }
};

typedef void (*FuncPtr)(void);

int main()
{
    std::unique_ptr<Base> pbase1 = std::make_unique<Base>();
    std::unique_ptr<Base> pbase2 = std::make_unique<Derive>();

    uintptr_t* vptr1 = *(uintptr_t**)pbase1.get();
    uintptr_t* vptr2 = *(uintptr_t**)pbase2.get();

    FuncPtr f1 = reinterpret_cast<FuncPtr>(vptr1[0]);
    FuncPtr f2 = reinterpret_cast<FuncPtr>(vptr2[0]);
    
    f1();
    f2();
    return 0;
}
```
