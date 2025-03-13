## 一. 添加自定义命令及用自定义命令生成文件
```cmake
# 首先, 在 MathFunctions/CMakeLists.txt 的顶部, 添加 MakeTable 的可执行文件
add_executable(MakeTable MakeTable.cxx)

# 然后, 添加一个自定义命令, 该命令指定如何通过运行 MakeTable 来生成 Table.h
add_custom_command(
    OUTPUT ${CMAKE_CURRENT_BINARY_DIR}/Table.h
    COMMAND MakeTable ${CMAKE_CURRENT_BINARY_DIR}/Table.h
    DEPENDS MakeTable
)

# 必须让 CMake 知道 mysqrt.cxx 取决于生成的文件Table.h, 通过将生成的添加 Table.h 到库 MathFunctions 的源列表中来完成此操作
add_library(SqrtLibrary STATIC mysqrt.cxx ${CMAKE_CURRENT_BINARY_DIR}/Table.h)

# 必须将当前的二进制目录添加到包含目录列表中,以便mysqrt.cxx可以找到并包含Table.h。
target_include_directories(SqrtLibrary PRIVATE ${CMAKE_CURRENT_BINARY_DIR})
target_include_directories(MathFunctions INTERFACE ${CMAKE_CURRENT_SOURCE_DIR})
```
