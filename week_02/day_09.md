## 一. 添加自定义命令及用自定义命令生成文件

### 1. 子模块 `MathFunctions` 的 `CMakeLists.txt`
```cmake
# 创建名为 MakeTable 的可执行文件，用于生成 Table.h 文件，MakeTable.cxx 是生成器的源代码，编译后将用于生成头文件
add_executable(MakeTable MakeTable.cxx)

# 定义自定义命令，描述如何生成 Table.h  此命令声明了文件生成规则，确保在需要时自动执行生成操作
add_custom_command(
    OUTPUT ${CMAKE_CURRENT_BINARY_DIR}/Table.h   # 指定输出文件路径
    COMMAND MakeTable ${CMAKE_CURRENT_BINARY_DIR}/Table.h  # 运行 MakeTable 生成文件
    DEPENDS MakeTable  # 声明依赖 MakeTable 可执行文件，确保其先被编译
)

# 创建静态库 SqrtLibrary，其源文件为 mysqrt.cxx 和生成的 Table.h
# 将 Table.h 添加到源文件列表是为了声明依赖关系：
# - 当 Table.h 更新时，触发 SqrtLibrary 的重新编译
# - 尽管 Table.h 是生成的头文件，但需要显式声明依赖
add_library(SqrtLibrary STATIC
    mysqrt.cxx
    ${CMAKE_CURRENT_BINARY_DIR}/Table.h
)

# 为 SqrtLibrary 添加私有包含目录（当前二进制目录）
# PRIVATE 表示仅 SqrtLibrary 自身需要此路径，用于包含生成的 Table.h
target_include_directories(SqrtLibrary
    PRIVATE ${CMAKE_CURRENT_BINARY_DIR}
)

# 为 MathFunctions 添加接口包含目录（当前源码目录）
# INTERFACE 表示依赖 MathFunctions 的其他目标（如 Tutorial）需要此路径
# 确保其他目标可以找到 MathFunctions.h 等头文件
target_include_directories(MathFunctions
    INTERFACE ${CMAKE_CURRENT_SOURCE_DIR}
)
```

### 2. `mysqrt.cxx` 函数修改
```cpp
double mysqrt(double x)
{
    if (x <= 0) { return 0; }
    double result = x;
    if (x >= 1 && x < 10) {
          std::cout << "Use the table to help find an initial value " << std::endl;
          result = sqrtTable[static_cast<int>(x)];
    }
    for (int i = 0; i < 10; ++i) {
        if (result <= 0) { result = 0.1; }
        double delta = x - (result * result);
        result = result + 0.5 * delta / result;
        std::cout << "Computing sqrt of " << x << " to be " << result << std::endl;
    }
    return result;
}
```
---

## 二. 打包安装程序
### 1. 在顶层 `CMakelists.txt` 的底部中添加以下代码
```cmake
# 包含安装系统运行时库的模块（确保程序依赖的库被打包）
include(InstallRequiredSystemLibraries)

# 设置许可证文件路径
set(CPACK_RESOURCE_FILE_LICENSE "${CMAKE_CURRENT_SOURCE_DIR}/License.txt")

# 设置版本号（从项目版本继承）
set(CPACK_PACKAGE_VERSION_MAJOR "${PROJECT_VERSION_MAJOR}")  # 例如 Tutorial_VERSION_MAJOR
set(CPACK_PACKAGE_VERSION_MINOR "${PROJECT_VERSION_MINOR}")  # 例如 Tutorial_VERSION_MINOR

# 设置源代码打包格式（可选，生成源代码包时使用）
set(CPACK_SOURCE_GENERATOR "TGZ")  # 支持 TGZ、ZIP、7Z 等

# 包含 CPack 模块（必须放在所有 CPACK_* 变量设置之后）
include(CPack)
```

### 2. `cpack`命令
```bash
cpack -G ZIP                             # 打包为 ZIP 格式
cpack -G NSIS                            # 打包为 NSIS 安装包（Windows 默认）
cpack -G TGZ                             # 打包为 TGZ 格式（Linux/macOS）
cpack -C Debug                           # 打包 Debug 版本
cpack -C Release                         # 打包 Release 版本
cpack --config CPackSourceConfig.cmake   # 打包源代码
```

---
