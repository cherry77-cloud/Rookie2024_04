## 一. 添加自定义命令及用自定义命令生成文件

### 1. 子模块 `MathFunctions` 的 `CMakeLists.txt`
```cmake
add_executable(MakeTable MakeTable.cxx)

# 定义自定义命令，描述如何生成 Table.h
add_custom_command(
    OUTPUT ${CMAKE_CURRENT_BINARY_DIR}/Table.h   # 指定输出文件路径
    COMMAND MakeTable ${CMAKE_CURRENT_BINARY_DIR}/Table.h  # 运行 MakeTable 生成文件
    DEPENDS MakeTable  # 声明依赖 MakeTable 可执行文件，确保其先被编译
)

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

## 三. 控制库的构建类型
### 1. 顶层 `CMakeLists.txt`
```cmake
# 在顶层 CMakeLists.txt 文件中，使用 option() 命令添加 BUILD_SHARED_LIBS 选项。该选项允许用户选择是否构建共享库（动态库）。
option(BUILD_SHARED_LIBS "Build using shared libraries" ON)

# 为了将生成的库文件和可执行文件输出到指定目录
set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY "${PROJECT_BINARY_DIR}")
set(CMAKE_LIBRARY_OUTPUT_DIRECTORY "${PROJECT_BINARY_DIR}")
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY "${PROJECT_BINARY_DIR}")
```

### 2. 更新头文件以支持动态库导出
```cpp
// 在 Windows 平台上，动态库需要显式导出符号。为此，更新 MathFunctions/MathFunctions.h 文件，
// 使用 __declspec(dllexport) 和 __declspec(dllimport) 定义导出符号
#if defined(_WIN32)
#  if defined(EXPORTING_MYMATH)
#    define DECLSPEC __declspec(dllexport)  // 导出符号
#  else
#    define DECLSPEC __declspec(dllimport)  // 导入符号
#  endif
#else // 非 Windows 平台
#  define DECLSPEC  // 非 Windows 平台不需要特殊处理
#endif

namespace mathfunctions {
    double DECLSPEC sqrt(double x);  // 声明导出函数
}
```

### 3. 子模块 `MathFunctions` 的 `CMakeLists.txt`
```cmake
# 定义 EXPORTING_MYMATH 符号
target_compile_definitions(MathFunctions PRIVATE "EXPORTING_MYMATH")

include(MakeTable.cmake)
add_library(SqrtLibrary STATIC mysqrt.cxx ${CMAKE_CURRENT_BINARY_DIR}/Table.h)
              
# 设置 SqrtLibrary 的位置无关代码属性
set_target_properties(SqrtLibrary PROPERTIES
    POSITION_INDEPENDENT_CODE ${BUILD_SHARED_LIBS}
)
```

- 通过 `BUILD_SHARED_LIBS` 变量，可以灵活控制库的构建类型（静态库或动态库）。
- 在 `Windows` 平台上，需要使用 `__declspec(dllexport)` 和 `__declspec(dllimport)` 导出和导入符号。
- 通过设置 `POSITION_INDEPENDENT_CODE` 属性，确保静态库与共享库兼容。

---

## 四. 添加导出配置
### 1. 顶层 `CMakeLists.txt`
```cmake
# 安装导出文件
install(EXPORT MathFunctionsTargets
  FILE MathFunctionsTargets.cmake
  DESTINATION lib/cmake/MathFunctions
)

include(CMakePackageConfigHelpers)

# 生成MathFunctionsConfig.cmake
configure_package_config_file(
  ${CMAKE_CURRENT_SOURCE_DIR}/Config.cmake.in
  "${CMAKE_CURRENT_BINARY_DIR}/MathFunctionsConfig.cmake"
  INSTALL_DESTINATION "lib/cmake/MathFunctions"
  NO_SET_AND_CHECK_MACRO
  NO_CHECK_REQUIRED_COMPONENTS_MACRO
)

# 生成版本配置文件
write_basic_package_version_file(
  "${CMAKE_CURRENT_BINARY_DIR}/MathFunctionsConfigVersion.cmake"
  VERSION "${Tutorial_VERSION_MAJOR}.${Tutorial_VERSION_MINOR}"
  COMPATIBILITY AnyNewerVersion
)

# 安装配置文件
install(FILES
  ${CMAKE_CURRENT_BINARY_DIR}/MathFunctionsConfig.cmake
  ${CMAKE_CURRENT_BINARY_DIR}/MathFunctionsConfigVersion.cmake
  DESTINATION lib/cmake/MathFunctions
)

# 支持从构建目录直接引用
export(EXPORT MathFunctionsTargets
  FILE "${CMAKE_CURRENT_BINARY_DIR}/MathFunctionsTargets.cmake"
)
```

### 2. 子模块 `MathFunctions` 的 `CMakeLists.txt`
```cmake
# 在 add_library(MathFunctions MathFunctions.cxx) 下面添加，处理路径兼容性
target_include_directories(MathFunctions INTERFACE
    $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}>
    $<INSTALL_INTERFACE:include>
)

# 更新安装命令以导出目标，在底部添加
set(installable_libs MathFunctions tutorial_compiler_flags)
if(TARGET SqrtLibrary)
  list(APPEND installable_libs SqrtLibrary)
endif()

install(TARGETS ${installable_libs}
        EXPORT MathFunctionsTargets
        DESTINATION lib
)

install(FILES MathFunctions.h DESTINATION include)
```

### 3. 在项目根目录创建 `Config.cmake.in`
```cmake
@PACKAGE_INIT@  # 将被替换为路径处理代码
include("${CMAKE_CURRENT_LIST_DIR}/MathFunctionsTargets.cmake")  # 包含导出的目标
```
---

## 五. 打包调试版和发行版
### 1. 文件目录层次
```txt
├── MathFunctions\
│   ├── CMakeLists.txt
│   ├── MakeTable.cmake
│   ├── MakeTable.cxx
│   ├── MathFunctions.cxx
│   ├── MathFunctions.h
│   ├── mysqrt.cxx
│   └── mysqrt.h
├── CMakeLists.txt
├── Config.cmake.in
├── CTestConfig.cmake
├── License.txt
├── MultiCPackConfig.cmake
├── tutorial.cxx
└── TutorialConfig.h.in
```

### 2. 顶层 `CMakeLists.txt`
```cmake
cmake_minimum_required(VERSION 3.15)

# set the project name and version
project(Tutorial VERSION 1.0)

set(CMAKE_DEBUG_POSTFIX d)

add_library(tutorial_compiler_flags INTERFACE)
target_compile_features(tutorial_compiler_flags INTERFACE cxx_std_11)

# add compiler warning flags just when building this project via
# the BUILD_INTERFACE genex
set(gcc_like_cxx "$<COMPILE_LANG_AND_ID:CXX,ARMClang,AppleClang,Clang,GNU,LCC>")
set(msvc_cxx "$<COMPILE_LANG_AND_ID:CXX,MSVC>")
target_compile_options(tutorial_compiler_flags INTERFACE
  "$<${gcc_like_cxx}:$<BUILD_INTERFACE:-Wall;-Wextra;-Wshadow;-Wformat=2;-Wunused>>"
  "$<${msvc_cxx}:$<BUILD_INTERFACE:-W3>>"
)

# control where the static and shared libraries are built so that on windows
# we don't need to tinker with the path to run the executable
set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY "${PROJECT_BINARY_DIR}")
set(CMAKE_LIBRARY_OUTPUT_DIRECTORY "${PROJECT_BINARY_DIR}")
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY "${PROJECT_BINARY_DIR}")

option(BUILD_SHARED_LIBS "Build using shared libraries" ON)

if(APPLE)
  set(CMAKE_INSTALL_RPATH "@executable_path/../lib")
elseif(UNIX)
  set(CMAKE_INSTALL_RPATH "$ORIGIN/../lib")
endif()

# configure a header file to pass the version number only
configure_file(TutorialConfig.h.in TutorialConfig.h)

# add the MathFunctions library
add_subdirectory(MathFunctions)

# add the executable
add_executable(Tutorial tutorial.cxx)
set_target_properties(Tutorial PROPERTIES DEBUG_POSTFIX ${CMAKE_DEBUG_POSTFIX})

target_link_libraries(Tutorial PUBLIC MathFunctions tutorial_compiler_flags)

# add the binary tree to the search path for include files
# so that we will find TutorialConfig.h
target_include_directories(Tutorial PUBLIC
                           "${PROJECT_BINARY_DIR}"
                           )

# add the install targets
install(TARGETS Tutorial DESTINATION bin)
install(FILES "${PROJECT_BINARY_DIR}/TutorialConfig.h"
  DESTINATION include
  )

# enable testing
enable_testing()

# does the application run
add_test(NAME Runs COMMAND Tutorial 25)

# does the usage message work?
add_test(NAME Usage COMMAND Tutorial)
set_tests_properties(Usage
  PROPERTIES PASS_REGULAR_EXPRESSION "Usage:.*number"
  )

# define a function to simplify adding tests
function(do_test target arg result)
  add_test(NAME Comp${arg} COMMAND ${target} ${arg})
  set_tests_properties(Comp${arg}
    PROPERTIES PASS_REGULAR_EXPRESSION ${result}
    )
endfunction()

# do a bunch of result based tests
do_test(Tutorial 4 "4 is 2")
do_test(Tutorial 9 "9 is 3")
do_test(Tutorial 5 "5 is 2.236")
do_test(Tutorial 7 "7 is 2.645")
do_test(Tutorial 25 "25 is 5")
do_test(Tutorial -25 "-25 is (-nan|nan|0)")
do_test(Tutorial 0.0001 "0.0001 is 0.01")

# setup installer
include(InstallRequiredSystemLibraries)
set(CPACK_RESOURCE_FILE_LICENSE "${CMAKE_CURRENT_SOURCE_DIR}/License.txt")
set(CPACK_PACKAGE_VERSION_MAJOR "${Tutorial_VERSION_MAJOR}")
set(CPACK_PACKAGE_VERSION_MINOR "${Tutorial_VERSION_MINOR}")
set(CPACK_GENERATOR "TGZ")
set(CPACK_SOURCE_GENERATOR "TGZ")
include(CPack)

# install the configuration targets
install(EXPORT MathFunctionsTargets
  FILE MathFunctionsTargets.cmake
  DESTINATION lib/cmake/MathFunctions
)

include(CMakePackageConfigHelpers)
# generate the config file that is includes the exports
configure_package_config_file(${CMAKE_CURRENT_SOURCE_DIR}/Config.cmake.in
  "${CMAKE_CURRENT_BINARY_DIR}/MathFunctionsConfig.cmake"
  INSTALL_DESTINATION "lib/cmake/example"
  NO_SET_AND_CHECK_MACRO
  NO_CHECK_REQUIRED_COMPONENTS_MACRO
  )
# generate the version file for the config file
write_basic_package_version_file(
  "${CMAKE_CURRENT_BINARY_DIR}/MathFunctionsConfigVersion.cmake"
  VERSION "${Tutorial_VERSION_MAJOR}.${Tutorial_VERSION_MINOR}"
  COMPATIBILITY AnyNewerVersion
)

# install the configuration file
install(FILES
  ${CMAKE_CURRENT_BINARY_DIR}/MathFunctionsConfig.cmake
  DESTINATION lib/cmake/MathFunctions
  )

# generate the export targets for the build tree
# needs to be after the install(TARGETS) command
export(EXPORT MathFunctionsTargets
  FILE "${CMAKE_CURRENT_BINARY_DIR}/MathFunctionsTargets.cmake"
)
```

### 3. 子模块 `MathFunctions` 的 `CMakeLists.txt`
```cmake
# add the library that runs
add_library(MathFunctions MathFunctions.cxx)

# state that anybody linking to us needs to include the current source dir
# to find MathFunctions.h, while we don't.
target_include_directories(MathFunctions
                           INTERFACE
                            $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}>
                            $<INSTALL_INTERFACE:include>
                           )

# should we use our own math functions
option(USE_MYMATH "Use tutorial provided math implementation" ON)
if(USE_MYMATH)

  target_compile_definitions(MathFunctions PRIVATE "USE_MYMATH")

  include(MakeTable.cmake) # generates Table.h

  # library that just does sqrt
  add_library(SqrtLibrary STATIC
              mysqrt.cxx
              ${CMAKE_CURRENT_BINARY_DIR}/Table.h
              )

  # state that we depend on our binary dir to find Table.h
  target_include_directories(SqrtLibrary PRIVATE
                             ${CMAKE_CURRENT_BINARY_DIR}
                             )

  # state that SqrtLibrary need PIC when the default is shared libraries
  set_target_properties(SqrtLibrary PROPERTIES
                        POSITION_INDEPENDENT_CODE ${BUILD_SHARED_LIBS}
                        )

  # link SqrtLibrary to tutorial_compiler_flags
  target_link_libraries(SqrtLibrary PUBLIC tutorial_compiler_flags)

  target_link_libraries(MathFunctions PRIVATE SqrtLibrary)
endif()

# link MathFunctions to tutorial_compiler_flags
target_link_libraries(MathFunctions PUBLIC tutorial_compiler_flags)

# define the symbol stating we are using the declspec(dllexport) when
# building on windows
target_compile_definitions(MathFunctions PRIVATE "EXPORTING_MYMATH")

# setup the version numbering
set_property(TARGET MathFunctions PROPERTY VERSION "1.0.0")
set_property(TARGET MathFunctions PROPERTY SOVERSION "1")

# install libs
set(installable_libs MathFunctions tutorial_compiler_flags)
if(TARGET SqrtLibrary)
  list(APPEND installable_libs SqrtLibrary)
endif()
install(TARGETS ${installable_libs}
        EXPORT MathFunctionsTargets
        DESTINATION lib)
# install include headers
install(FILES MathFunctions.h DESTINATION include)
```

### 4. `MultiCPackConfig.cmake`
```cmake
include("release/CPackConfig.cmake")

set(CPACK_INSTALL_CMAKE_PROJECTS
    "debug;Tutorial;ALL;/"
    "release;Tutorial;ALL;/"
    )
```

### 5. 打包
```bash
cd debug
cmake -DCMAKE_BUILD_TYPE=Debug ..
cmake --build .
cd ../release
cmake -DCMAKE_BUILD_TYPE=Release ..
cmake --build .
cpack --config MultiCPackConfig.cmake
```
---
## CMake 常用常量和变量

### 1. 常用常量
- **`CMAKE_SOURCE_DIR`**: 顶级源代码目录的路径（即包含顶层 `CMakeLists.txt` 的目录）。
- **`CMAKE_BINARY_DIR`**: 顶级构建目录的路径（即运行 `cmake` 命令的目录）。
- **`CMAKE_CURRENT_SOURCE_DIR`**: 当前处理的 `CMakeLists.txt` 所在的目录。
- **`CMAKE_CURRENT_BINARY_DIR`**: 当前构建目录的路径。
- **`CMAKE_MODULE_PATH`**: 指定查找 CMake 模块的路径。
- **`CMAKE_INSTALL_PREFIX`**: 安装目录的路径（默认为 `/usr/local` 或 `C:/Program Files`）。
- **`CMAKE_BUILD_TYPE`**: 构建类型（如 `Debug`、`Release`、`RelWithDebInfo`、`MinSizeRel`）。
- **`CMAKE_CXX_COMPILER`**: C++ 编译器的路径。
- **`CMAKE_C_COMPILER`**: C 编译器的路径。
- **`CMAKE_SYSTEM_NAME`**: 目标系统的名称（如 `Linux`、`Windows`、`Darwin`）。
- **`CMAKE_SYSTEM_VERSION`**: 目标系统的版本。
- **`CMAKE_SYSTEM_PROCESSOR`**: 目标系统的处理器架构（如 `x86_64`、`ARM`）。

### 2. 平台相关常量
- **`WIN32`**: 在 Windows 平台上为 `True`。
- **`APPLE`**: 在 macOS 或 iOS 平台上为 `True`。
- **`UNIX`**: 在 UNIX 平台（包括 Linux 和 macOS）上为 `True`。
- **`CMAKE_HOST_SYSTEM_NAME`**: 主机系统的名称。
- **`CMAKE_HOST_SYSTEM_PROCESSOR`**: 主机系统的处理器架构。

### 3. 编译器相关常量
- **`CMAKE_CXX_STANDARD`**: 指定 C++ 标准版本（如 `11`、`14`、`17`、`20`）。
- **`CMAKE_C_STANDARD`**: 指定 C 标准版本（如 `99`、`11`、`17`）。
- **`CMAKE_CXX_COMPILER_ID`**: 编译器 ID（如 `GNU`、`Clang`、`MSVC`）。
- **`CMAKE_CXX_FLAGS`**: C++ 编译器的全局标志。
- **`CMAKE_C_FLAGS`**: C 编译器的全局标志。

### 4. 构建目标相关常量
- **`CMAKE_LIBRARY_OUTPUT_DIRECTORY`**: 库文件的输出目录。
- **`CMAKE_RUNTIME_OUTPUT_DIRECTORY`**: 可执行文件的输出目录。
- **`CMAKE_ARCHIVE_OUTPUT_DIRECTORY`**: 静态库文件的输出目录。
- **`CMAKE_DEBUG_POSTFIX`**: 在 Debug 模式下为库或可执行文件添加后缀（如 `_d`）。
- **`CMAKE_POSITION_INDEPENDENT_CODE`**: 是否生成位置无关代码（PIC）。

### 5. 安装相关常量
- **`CMAKE_INSTALL_PREFIX`**: 安装路径的前缀。
- **`CMAKE_INSTALL_BINDIR`**: 可执行文件的安装目录（默认为 `bin`）。
- **`CMAKE_INSTALL_LIBDIR`**: 库文件的安装目录（默认为 `lib`）。
- **`CMAKE_INSTALL_INCLUDEDIR`**: 头文件的安装目录（默认为 `include`）。

### 6. 查找工具相关常量
- **`CMAKE_FIND_ROOT_PATH`**: 指定查找库和头文件的根路径。
- **`CMAKE_FIND_LIBRARY_PREFIXES`**: 库文件的前缀（如 `lib`）。
- **`CMAKE_FIND_LIBRARY_SUFFIXES`**: 库文件的后缀（如 `.so`、`.a`、`.dll`）。

### 7. 生成器相关常量
- **`CMAKE_GENERATOR`**: 当前使用的生成器（如 `Unix Makefiles`、`Ninja`、`Visual Studio`）。
- **`CMAKE_GENERATOR_PLATFORM`**: 生成器的目标平台（如 `x64`、`Win32`）。
- **`CMAKE_GENERATOR_TOOLSET`**: 生成器的工具集（如 `v142` 用于 Visual Studio）。

### 8. 其他常用变量
- **`PROJECT_NAME`**: 当前项目的名称。
- **`PROJECT_SOURCE_DIR`**: 当前项目的源代码目录。
- **`PROJECT_BINARY_DIR`**: 当前项目的构建目录。
- **`CMAKE_VERSION`**: CMake 的版本号。
- **`CMAKE_MAJOR_VERSION`**: CMake 的主版本号。
- **`CMAKE_MINOR_VERSION`**: CMake 的次版本号。
- **`CMAKE_PATCH_VERSION`**: CMake 的补丁版本号。

### 9. 环境变量
- **`ENV{PATH}`**: 访问系统的 `PATH` 环境变量。
- **`ENV{HOME}`**: 访问用户的主目录路径。

### 10. 控制流相关常量
- **`CMAKE_ALLOW_LOOSE_LOOP_CONSTRUCTS`**: 允许宽松的循环语法。
- **`CMAKE_MINIMUM_REQUIRED_VERSION`**: 指定 CMake 的最低版本要求。

### 11. 调试和日志
- **`CMAKE_MESSAGE_LOG_LEVEL`**: 控制日志输出级别（如 `DEBUG`、`VERBOSE`、`STATUS`）。
- **`CMAKE_VERBOSE_MAKEFILE`**: 启用详细的 Makefile 输出。

### 12. 交叉编译相关
- **`CMAKE_CROSSCOMPILING`**: 是否正在交叉编译。
- **`CMAKE_TOOLCHAIN_FILE`**: 指定工具链文件的路径。

### 13. 模块路径
- **`CMAKE_MODULE_PATH`**: 指定查找 CMake 模块的路径。

### 14. 测试相关
- **`CTEST_COMMAND`**: CTest 命令的路径。
- **`CMAKE_TESTING_ENABLED`**: 是否启用测试。

### 15. 包管理相关
- **`CMAKE_FIND_PACKAGE_NAME`**: 当前查找的包名称。
- **`CMAKE_FIND_PACKAGE_SORT_DIRECTION`**: 控制包查找的排序方向。
