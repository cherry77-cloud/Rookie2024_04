### 一. 进程的创建与装载

#### 进程创建步骤
- **创建虚拟地址空间**：分配独立的虚拟内存区域。
- **建立文件映射**：读取可执行文件头（如`ELF`或`PE`），映射代码段、数据段到内存。
- **设置执行入口**：将CPU指令寄存器指向程序入口地址（如ELF的`e_entry`），启动执行。

#### 可执行文件类型
| 文件类型 | 系统       | 特点                         |
|----------|------------|------------------------------|
| `ELF`      | `Linux/Unix` | 支持动态链接，段结构清晰      |
| `PE`       | `Windows`    | 包含`DOS`头，依赖`DLL`动态库      |

---

### 二. `ELF`文件装载过程（`Linux`）
#### 用户层流程
- 创建子进程：`bash`调用`fork()`生成新进程。
- 执行程序：子进程调用`execve()`加载`ELF`文件，原进程等待。

```c++
int execve(const char *filename, char *const argv[], char *const envp[]);
```
#### 内核层流程
- 系统调用链：`execve()` → `sys_execve()` → `do_execve()` → `search_binary_handle()` → `load_elf_binary()`。
- 关键函数：`load_elf_binary()`（定义于`fs/Binfmt_elf.c`）：
    - 验证`ELF`格式：检查魔数（`0x7F 0x45 0x4C 0x46`）、程序头表合法性。
    - 定位动态链接器：解析`.interp`段，获取动态链接器路径（如`/lib/ld-linux.so`）。
    - 映射内存段：按程序头表（`Program Header`）将代码段、数据段映射到内存。
    - 初始化寄存器：设置`EDX为DT_FINI`（程序终止函数地址）。
    - 跳转入口点：静态链接：直接跳转至`e_entry`地址; 动态链接：跳转至动态链接器入口，完成符号解析后再执行程序。
