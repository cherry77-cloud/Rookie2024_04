## 一. 进程的创建与装载

#### 进程创建步骤
- **创建虚拟地址空间**：分配独立的虚拟内存区域
- **建立文件映射**：读取可执行文件头（如`ELF`或`PE`），映射代码段、数据段到内存
- **设置执行入口**：将CPU指令寄存器指向程序入口地址（如ELF的`e_entry`），启动执行

#### 可执行文件类型
| 文件类型 | 系统       | 特点                         |
|----------|------------|------------------------------|
| `ELF`      | `Linux/Unix` | 支持动态链接，段结构清晰      |
| `PE`       | `Windows`    | 包含`DOS`头，依赖`DLL`动态库      |

---

## 二. `ELF`文件装载过程（`Linux`）
#### 用户层流程
- 创建子进程：`bash`调用`fork()`生成新进程
- 执行程序：子进程调用`execve()`加载`ELF`文件，原进程等待

```c++
int execl(const char *path, const char *arg, ..., (char *) NULL);
int execv(const char *path, char *const argv[]);
int execle(const char *path, const char *arg, ..., (char *) NULL, char *const envp[]);
int execve(const char *path, char *const argv[], char *const envp[]);
int execlp(const char *file, const char *arg, ..., (char *) NULL);
int execvp(const char *file, char *const argv[]);
int execvpe(const char *file, char *const argv[], char *const envp[]);
```
#### 内核层流程
- 系统调用链：`execve()` → `sys_execve()` → `do_execve()` → `search_binary_handle()` → `load_elf_binary()`
- 关键函数：`load_elf_binary()`（定义于`fs/Binfmt_elf.c`）
    - 验证`ELF`格式：检查魔数（`0x7F 0x45 0x4C 0x46`）、程序头表合法性
    - 定位动态链接器：解析`.interp`段，获取动态链接器路径（如`/lib/ld-linux.so`）
    - 映射内存段：按程序头表（`Program Header`）将代码段、数据段映射到内存
    - 初始化寄存器：设置`EDX为DT_FINI`（程序终止函数地址）
    - 跳转入口点：静态链接 -> 直接跳转至`e_entry`地址; 动态链接 -> 跳转至动态链接器入口，完成符号解析后再执行程序

```c++
#include <stdio.h>
#include <sys/types.h>
#include <unistd.h>

int main()
{
    char buf[1024] = {0};
    pid_t pid;
    while(1) {
        printf("minibash$");
        scanf("%s", buf);
        pid = fork();
        if(pid == 0) {
            if(execlp(buf, 0 ) < 0) {
                printf("exec error\n");
            }
        } else if(pid > 0){
            int status;
            waitpid(pid,&status,0);
        } else {
            printf("fork error %d\n",pid);
        }
    }
    return 0;
}
```

---


### 三. `PE`文件装载（`Windows`）
- 解析文件头：读取`DOS`头、`PE`头、段表
- 地址空间检查：若目标地址被占用，重新选择基址（`Rebasing`）
- 段映射：将PE文件的代码段（`.text`）、数据段（`.data`）映射到内存
- 依赖解析：装载所需的DLL文件，解析导入表（`Import Table`）
- 初始化环境：依据`PE`头初始化栈、堆空间
- 启动进程：创建主线程，从入口点（`AddressOfEntryPoint`）开始执行

| 段名          | 权限              | 内容描述                                 |
|---------------|-------------------|----------------------------------------|
| `.text`       | 只读、可执行       | 存储程序的可执行代码（机器指令）          |
| `.data`       | 读写              | 已初始化的全局变量和静态变量              |
| `.bss`        | 读写              | 未初始化的全局变量和静态变量（实际不占磁盘空间）|
| `.rodata`     | 只读              | 常量数据（如字符串常量、const全局变量）    |
| `.init`       | 只读、可执行       | 程序初始化代码（如`_start`前的构造函数调用）|
| `.fini`       | 只读、可执行       | 程序终止代码（如析构函数调用）             |
| `.plt`        | 只读、可执行       | 过程链接表（用于动态链接的函数跳转）        |
| `.got`        | 读写              | 全局偏移表（存储动态链接的全局变量地址）     |
| `.ctors`      | 读写              | C++全局构造函数指针列表                  |
| `.dtors`      | 读写              | C++全局析构函数指针列表                   |
| `.eh_frame`   | 只读              | 异常处理框架信息（支持C++异常机制）         |
| `.dynamic`    | 读写              | 动态链接信息（依赖库、符号表地址等）        |
| `.interp`     | 只读              | 动态链接器路径（如`/lib/ld-linux.so`）      |
| `.symtab`     | 只读              | 符号表（函数和全局变量的符号信息）          |
| `.strtab`     | 只读              | 字符串表（存储符号名称等字符串）            |
| `.shstrtab`   | 只读              | 段名称字符串表（存储段名的字符串）           |
| `.debug`      | 只读              | 调试信息（如DWARF格式的调试符号）           |
| `.comment`    | 只读              | 编译器/链接器生成的注释信息（如GCC版本）     |
| `.tdata`      | 读写              | 线程局部存储的初始化数据                  |
| `.tbss`       | 读写              | 线程局部存储的未初始化数据（不占磁盘空间）   |
| `.rela.text`  | 只读              | `.text`段的重定位信息（用于静态链接）        |
| `.rela.data`  | 只读              | `.data`段的重定位信息                      |

---

## 四. `fork()` 函数
#### 功能
- **创建子进程**：通过复制父进程的地址空间生成一个几乎完全相同的子进程。
- **执行流程**：
  - 父进程继续执行原代码。
  - 子进程从`fork()`返回处开始执行。

#### 返回值
- **父进程**：返回子进程的`PID（>0）`。
- **子进程**：返回`0`。
- **失败**：返回`-1`（如系统进程数达到上限）。


```c
#include <unistd.h>
#include <stdio.h>

int main() {
    pid_t pid = fork();
    if (pid == 0) {
        printf("child process PID: %d\n", getpid());
    } else if (pid > 0) {
        printf("Father Process PID: %d，Child Process PID: %d\n", getpid(), pid);
    } else {
        perror("fork fail");
    }
    return 0;
}
```

---

## `wait()` 函数详解

#### 功能
- **回收子进程**：父进程调用`wait()`阻塞自身，直到任一子进程终止。
- **获取退出状态**：通过参数`status`获取子进程的退出状态。

#### status：指向整数的指针，用于存储子进程的退出状态。
- `WIFEXITED(status)`：如果子进程通过调用`exit`或`return`正常终止，返回真。
- `WEXITSTATUS(status)`：返回子进程的退出状态。仅在`WIFEXITED()`为真时定义。
- `WIFSIGNALED(status)`：如果子进程因未捕获的信号终止，返回真。
- `WTERMSIG(status)`：返回导致子进程终止的信号编号。仅在`WIFSIGNALED()`为真时定义。
- `WIFSTOPPED(status)`：如果子进程当前处于停止状态，返回真。
- `WSTOPSIG(status)`：返回导致子进程停止的信号编号。仅在`WIFSTOPPED()`为真时定义。
- `WIFCONTINUED(status)`：如果子进程因收到`SIGCONT`信号而恢复运行，返回真。

#### 返回值
成功：返回终止子进程的`PID`。失败：返回`-1`（如没有子进程）

```c++
#include <sys/wait.h>
#include <stdio.h>
#include <unistd.h>
#include <stdlib.h>

int main() {
    pid_t pid = fork();
    if (pid == 0) {
        // Child process
        printf("Child process running, PID: %d\n", getpid());
        sleep(2);  // Simulate work
        exit(42);  // Child exit code
    } else if (pid > 0) {
        // Parent process
        int status;
        printf("Parent waiting for child to finish...\n");
        pid_t child_pid = wait(&status);  // Block until child terminates
        if (WIFEXITED(status)) {
            printf("Child PID: %d exited with status: %d\n", child_pid, WEXITSTATUS(status));
        }
    } else {
        perror("fork failed");
        exit(1);
    }
    return 0;
}
```
- 阻塞行为：`wait()`会阻塞父进程，直到子进程终止。
- 多子进程处理：若有多个子进程，`wait()`会回收任意一个终止的子进程。
- 僵尸进程避免：未调用`wait()`会导致子进程成为僵尸进程。

---
