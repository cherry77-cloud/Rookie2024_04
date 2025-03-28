## 一. 链接的本质与执行时机
链接是将各种代码和数据片段收集并组合为单一可执行文件的过程，该文件可加载到内存中执行。链接过程的核心价值体现在：
- **分离编译**：允许模块化开发
- **符号解析**：解决跨模块的符号引用
- **空间优化**：实现代码共享和复用

| 链接类型     | 执行阶段         | 特点                          | 应用场景              |
|--------------|------------------|-----------------------------|---------------------|
| 编译时链接   | 源代码翻译阶段   | 生成完整可执行文件            | 静态库集成          |
| 加载时链接   | 程序加载阶段     | 需要加载器参与                | 动态库基础链接      |
| 运行时链接   | 程序运行过程     | 通过`API`动态加载              | 插件系统/热更新机制 |

---

## 二. 编译系统工作流程
- 大多数编译系统提供编译器驱动程序, 它代表用户在需要时调用预处理器, 编译器, 汇编器和链接器.
- 驱动程序首先运行`C`预处理器`cpp`, 它将`C`的源程序`main.c`翻译成一个`ASCII`码的中间文件 `main.i`
```bash
# 处理宏定义、头文件展开, 生成预处理后文本文件
cpp main.c > main.i
```

- 接下来, 驱动程序运行`C`编译器`cc1`, 它将 `main.i` 翻译成一个`ASCII`汇编语言文件 `main.s`
```bash
# 语法/语义分析, 生成平台相关的汇编代码
cc1 -Og main.i -o main.s
```

- 然后, 驱动程序运行汇编器`as`, 它将`main.s`翻译成一个可重定位目标文件 `main.o`
```bash
# 生成可重定位目标文件, 包含机器指令和符号表
as main.s -o main.o
```

- 驱动程序经过相同的过程生成`sum.o`. 最后, 它运行链接器程序`ld`, 将 `main.o` 和 `sum.o` 以及一些必要的系统目标文件组合起来, 创建一个可执行目标文件`prog`.
```bash
# 符号解析与重定位, 生成可执行目标文件
ld -o prog [system object files and args] /tmp/main.o /tmp/sum.o
ld -static \
    /usr/lib/x86_64-linux-gnu/crt1.o \
    /usr/lib/x86_64-linux-gnu/crti.o \
    /usr/lib/gcc/x86_64-linux-gnu/11/crtbeginT.o \
    hello.o \
    -L/usr/lib/gcc/x86_64-linux-gnu/11 \
    -L/usr/lib/x86_64-linux-gnu \
    -start-group \
    -lgcc \
    -lgcc_eh \
    -lc \
    -end-group \
    /usr/lib/gcc/x86_64-linux-gnu/11/crtend.o \
    /usr/lib/x86_64-linux-gnu/crtn.o
```

- 要运行可执行文件`prog`, 在`Linux Shell`的命令行上输入它的名字, `shell`调用操作系统中一个叫做加载器的函数, 它将可执行文件`prog`中的代码和数据复制到内存, 然后将控制转移到这个程序的开头.

---
## 三. 静态链接

- 静态链接是将多个可重定位目标文件（`.o` 文件）合并成一个可执行目标文件的过程。静态链接器（如 `Linux` 的 `ld` 程序）以一组可重定位目标文件和命令行参数作为输入，生成一个完全链接的、可以加载和运行的可执行目标文件作为输出。

- 为了构造可执行目标文件，链接器必须完成两个主要任务：

#### 符号解析 (`Symbol Resolution`)
- **符号定义**：每个符号对应一个函数、全局变量或静态变量。
- **符号引用**：目标文件中可能引用其他目标文件中定义的符号。
- **目的**：将每个符号引用与一个符号定义关联起来，确保所有符号引用都能正确解析。

#### 符号类型
| 符号类型       | 描述                                                                 |
|----------------|----------------------------------------------------------------------|
| **强符号**     | 已初始化的全局变量或函数定义。                                       |
| **弱符号**     | 未初始化的全局变量或通过 `weak` 属性声明的函数。                     |
| **局部符号**   | 使用 `static` 关键字定义的变量或函数，仅在当前文件可见。             |

#### 符号解析规则
1. 不允许存在多个同名的强符号。
2. 如果存在一个强符号和多个弱符号，选择强符号。
3. 如果存在多个弱符号，随机选择一个。

#### 重定位 (`Relocation`)
- **问题**：编译器和汇编器生成的代码和数据节从地址 `0` 开始，但最终可执行文件需要将代码和数据加载到具体的内存地址。
- **任务**：
  1. **合并节**：将不同目标文件中相同类型的节（如 `.text`、`.data`）合并。
  2. **分配地址**：为每个符号分配具体的内存地址。
  3. **修正引用**：根据符号的最终地址，修改所有对符号的引用。

---

## 四. 动态链接
#### 动态链接基本思想
- **延迟链接**：将链接过程推迟到运行时，模块以独立文件（`.so`/`.dll`）形式存在
- **资源共享**：多个进程共享同一物理内存中的代码段，减少内存占用
- **灵活更新**：模块独立更新，无需重新编译主程序


| 优势                | 原理说明                                                                 |
|---------------------|--------------------------------------------------------------------------|
| **节省内存**        | 共享代码段在内存中仅保留一份副本                                        |
| **提升性能**        | 减少物理页面换入换出，提高`CPU`缓存命中率                                |
| **插件支持**        | 运行时动态加载模块（如`dlopen()`）实现功能扩展                          |
| **跨平台兼容**      | 通过中间层（动态链接库）消除平台差异                                    |


#### 地址无关代码（`PIC`）
- **模块内部访问**：使用相对地址（指令跳转/数据访问）
- **模块外部访问**：通过`GOT`（全局偏移表）间接寻址
- **数据段分离**：指令部分保持只读，数据段每个进程独立副本

#### 全局偏移表（`GOT`）与过程链接表（`PLT`）
- **`GOT`**：存储外部变量和函数的绝对地址，`.got`（变量）和`.got.plt`（函数）
- **`PLT`**：延迟绑定关键组件，首次调用函数时触发动态链接器解析地址

#### 延迟绑定（`Lazy Binding`）
- **机制**：函数第一次调用时才解析符号并重定位，减少启动开销
  1. 调用`PLT`表项
  2. 触发动态链接器解析真实地址
  3. 更新`GOT`，后续调用直接跳转

#### 动态链接器（`ld.so`）
- **自举过程**：不依赖其他共享对象，自行完成初始化
- **核心任务**：
  - 装载共享对象并构建依赖关系树
  - 合并全局符号表，处理符号冲突（全局符号介入）
  - 重定位`GOT/PLT`，执行模块初始化代码（`.init`段）

动态链接通过运行时加载和地址无关技术，实现了内存节省、部署灵活和跨平台兼容。尽管存在启动延迟和间接访问开销，但通过延迟绑定和`PIC`技术有效缓解了性能问题，成为现代操作系统的核心基础架构。显式运行时链接进一步扩展了动态性，为模块化开发和插件系统提供了坚实基础。

```c++
#include <stdio.h>
#include <dlfcn.h>
#include <stdlib.h>

int main(int argc, char* argv[])
{
    void *handle;
    double (*func)(double);
    char *error;

    handle = dlopen(argv[1], RTLD_NOW);
    if (handle == NULL) {
        printf("Open library %s error: %s\n", argv[1], dlerror());
        return -1;
    }

    func = dlsym(handle, "sin");
    if ((error = dlerror()) != NULL) {
        printf("Symbol sin not found: %s\n", error);
        exit(-1);
    }

    printf("%f\n", func(3.1415926/2));
    dlclose(handle);
    return 0;
}
```
---
## 五. 链接器需要静态链接？

- **1. 避免依赖性问题**
  - **自包含**：静态链接将所有依赖库打包到链接器中，无需外部共享库支持。
  - **环境独立**：可在任何支持其二进制格式的系统上运行，无需额外安装共享库。

- **2. 解决自举问题**
  - **工具链构建**：静态链接打破“先有鸡还是先有蛋”的循环依赖，实现工具链自举。
  - **跨平台构建**：静态链接的链接器无需目标系统的共享库支持，简化工具链移植。

- **3. 确保稳定性和兼容性**
  - **版本固化**：静态链接将特定版本的库固化到链接器中，避免因系统库升级导致行为不一致。
  - **行为可预测**：在不同环境中表现一致，适用于对构建结果有严格要求的场景。

- **4. 性能优化**
  - **启动速度**：静态链接的可执行文件无需加载动态库，启动更快。
  - **内存占用**：按需加载代码，减少内存碎片。

- **5. 特定场景需求**
  - **最小化系统**：在极简系统（如`initramfs`）中，静态链接确保关键功能不依赖外部库。
  - **安全隔离**：减少对外部库的依赖，降低因共享库漏洞引发的安全风险。

---

## 六. 目标文件与链接工具总结

#### 目标文件的三种形式
- **可重定位目标文件**：包含二进制代码和数据，可以在编译时与其他可重定位目标文件合并，生成可执行目标文件。
- **可执行目标文件**：包含可直接复制到内存并执行的二进制代码和数据。
- **共享目标文件**：一种特殊类型的可重定位目标文件，可以在加载或运行时动态加载到内存并链接。

#### 链接器的任务
- **符号解析**：将目标文件中的每个全局符号绑定到一个唯一的定义。
- **重定位**：确定每个符号的最终内存地址，并修改对这些符号的引用。

#### 静态链接与动态链接
- **静态链接**：由静态链接器在编译时完成，将多个可重定位目标文件合并成一个可执行目标文件。
- **动态链接**：由动态链接器在加载时或运行时完成，处理共享目标文件（共享库）。动态链接可以在程序启动时隐式完成，或通过调用`dlopen`函数显式完成。

#### `Linux`系统中的工具
- **`AR`**：创建静态库，插入、删除、列出和提取成员。
- **`STRINGS`**：列出一个目标文件中所有可打印的字符串。
- **`STRIP`**：从目标文件中删除符号表信息。
- **`NM`**：列出一个目标文件的符号表中定义的符号。
- **`SIZE`**：列出目标文件中节的名字和大小。
- **`READELF`**：显示一个目标文件的完整结构，包括 ELF 头中编码的所有信息。
- **`OBJDUMP`**：显示目标文件中的所有信息，特别是反汇编`.text`节中的二进制指令。
- **`LDD`**：列出一个可执行文件在运行时所需要的共享库。


#### 共享库的优势
- **位置无关代码**：共享库可以加载到内存的任何位置，并在多个进程间共享。
- **运行时加载**：应用程序可以在运行时使用动态链接器加载和访问共享库的函数和数据。
---

```c++
// gcc -nostdlib -o hello hello.c
char* str = "Hello world!\n";

void print()
{
    asm volatile (
        "movq $13, %%rdx \n\t"  // 字符串长度
        "movq %0, %%rsi \n\t"   // 字符串地址
        "movq $1, %%rdi \n\t"   // 文件描述符(1=stdout)
        "movq $1, %%rax \n\t"   // 系统调用号(1=write)
        "syscall \n\t"          // 64位系统调用指令
        : 
        : "r" (str)
        : "rdx", "rsi", "rdi", "rax"
    );
}

void my_exit()
{
    asm volatile (
        "movq $42, %%rdi \n\t"  // 退出状态码
        "movq $60, %%rax \n\t"  // 系统调用号(60=exit)
        "syscall \n\t"
        :
        :
        : "rdi", "rax"
    );
}

void _start()
{
    print();
    my_exit();
}
```
