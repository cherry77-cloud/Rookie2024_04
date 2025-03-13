## 一. 添加自定义命令及用自定义命令生成文件
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
