### 1. 链接的本质与执行时机
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

### 2. 编译系统工作流程
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
