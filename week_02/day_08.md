## 一. 什么是 `CMake`？
- `CMake` 是一个跨平台的构建系统生成器（`Build System Generator`），用于管理软件编译过程。
- 核心目标：通过编写配置文件（`CMakeLists.txt`）生成不同平台和编译器的构建脚本（如 `Makefile`、`Visual Studio` 项目、`Xcode` 项目等）。
- 跨平台支持：支持 `Windows`、`Linux`、`macOS` 等操作系统，兼容 `GCC`、`Clang`、`MSVC` 等编译器。
- 开源项目：由 `Kitware` 公司维护，广泛应用于 `C/C++` 项目（如 `LLVM`、`KDE`、`OpenCV` 等）。

### `CMake` 的核心功能
- **生成构建系统**: 根据 `CMakeLists.txt` 自动生成目标平台的构建文件（如 `Unix` 的 `Makefile` 或 `Windows` 的 `.sln`）。
- **依赖管理**: 支持查找第三方库（如 `OpenSSL`、`Boost`）并自动配置头文件路径和链接库。
- **模块化配置**: 支持通过 `include()` 或 `add_subdirectory()` 管理多目录项目。
- **跨平台编译**: 同一份配置可在不同操作系统和编译器下使用，无需手动修改构建脚本。
- **测试与打包**: 集成 `CTest` 用于单元测试，`CPack` 用于生成安装包（如 `RPM`、`DEB`、`ZIP`）。

### `CMake` 的工作流程
- **配置（`Configure`）**: 读取 `CMakeLists.txt`，检查编译器和依赖项，生成缓存文件（`CMakeCache.txt`）。
- **生成（`Generate`）**: 根据配置生成目标平台的构建系统文件（如 `Makefile` 或 `.vcxproj`）。
- **构建（`Build`）**: 调用底层构建工具（如 `make` 或 `ninja`）编译代码。

---

## 二. 添加版本号和配置头文件

- **`cmake -G "MinGW Makefiles" ..`**: 调用`CMake`程序, 指定生成器类型。告诉`CMake`使用`MinGW`的`make`工具来构建项目, 指向`CMakeLists.txt`文件所在的源码目录（当前目录的上一级）
- **`cmake --build .`**: 使用`CMake`的构建命令，自动选择合适的构建工具。`.`表示当前目录
- **`mingw32-make`**: 直接调用`MinGW`提供的`make`工具
- **`_cplusplus`** 是 `C++` 标准定义的宏，用于检测当前编译器使用的 `C++` 标准版本

### 1. 文件目录结构
```txt
├── CMakeLists.txt
├── TutorialConfig.h.in
└── tutorial.cxx
```

### 2. 主项目的`CMakeLists.txt`
```cmake
cmake_minimum_required(VERSION 3.10)
project(Tutorial VERSION 1.0)
set(CMAKE_CXX_STANDARD 11)
set(CMAKE_CXX_STANDARD_REQUIRED True)
configure_file(TutorialConfig.h.in TutorialConfig.h)
add_executable(Tutorial tutorial.cxx)

# 为可执行目标 "Tutorial" 添加包含目录，使得编译器能够找到生成的头文件 TutorialConfig.h。
target_include_directories(Tutorial PUBLIC ${PROJECT_BINARY_DIR})
```

### 3. `TutorialConfig.h.in` 文件
```c++
#define Tutorial_VERSION_MAJOR @Tutorial_VERSION_MAJOR@
#define Tutorial_VERSION_MINOR @Tutorial_VERSION_MINOR@
```

---

## 三. 创建库文件和库文件可选编译
### 1. 文件结构目录
```txt
project/
├── CMakeLists.txt                # 主项目文件
├── tutorial.cxx                  # 主程序源文件
├── TutorialConfig.h.in           # 配置文件模板
├── MathFunctions/
│   ├── CMakeLists.txt            # 子模块文件
│   ├── MathFunctions.cxx         # 主库源文件
│   ├── mysqrt.cxx                # 自定义平方根实现
│   └── mysqrt.h                  # 自定义平方根头文件
└── build/                        # 构建目录
```

### 2. 主项目的`CMakeLists.txt`
```cmake
cmake_minimum_required(VERSION 3.10)
project(Tutorial VERSION 1.0)
set(CMAKE_CXX_STANDARD 11)
set(CMAKE_CXX_STANDARD_REQUIRED True)
option(USE_MYMATH "Use tutorial provided math implementation" ON)
configure_file(TutorialConfig.h.in TutorialConfig.h)

if (USE_MYMATH)
    # 添加子目录 MathFunctions，该目录应包含另一个 CMakeLists.txt
    add_subdirectory(MathFunctions)
    # 将 MathFunctions 库添加到额外链接库列表
    list(APPEND EXTRA_LIBS MathFunctions)
    # 将自定义数学库的头文件目录添加到额外包含目录列表
    list(APPEND EXTRA_INCLUDES "${PROJECT_SOURCE_DIR}/MathFunctions")
endif ()

add_executable(Tutorial tutorial.cxx)
target_link_libraries(Tutorial PUBLIC ${EXTRA_LIBS})

target_include_directories(Tutorial PUBLIC
    "${PROJECT_BINARY_DIR}"                # 包含生成的 TutorialConfig.h
    "${PROJECT_SOURCE_DIR}/MathFunctions"  # 包含自定义数学库头文件
)
```

### 3. 子模块 `MathFunctions` 的 `CMakeLists.txt`
```cmake
add_library(MathFunctions MathFunctions.cxx)

if (USE_MYMATH)
    # 为 MathFunctions 库添加私有编译定义 USE_MYMATH, PRIVATE 表示此定义仅作用于该库本身，不会传递给链接它的其他目标
    target_compile_definitions(MathFunctions PRIVATE USE_MYMATH)
    # 创建名为 SqrtLibrary 的静态库，包含 mysqrt.cxx 源文件, STATIC 表示生成静态库（.a/.lib）
    add_library(SqrtLibrary STATIC mysqrt.cxx)
    # 将 SqrtLibrary 作为私有依赖链接到 MathFunctions, PRIVATE 表示依赖关系仅作用于 MathFunctions，不会向上层传递
    target_link_libraries(MathFunctions PRIVATE SqrtLibrary)
endif ()
```

### 4. 代码中的条件编译

```c++
// MathFunctions.cxx
#ifdef USE_MYMATH
#include "mysqrt.h"
#endif

#ifdef USE_MYMATH
    return detail::mysqrt(x);
#else
    return std::sqrt(x);
#endif

// tutorial.cxx
#ifdef USE_MYMATH
#include "MathFunctions.h"
#endif

#ifdef USE_MYMATH
  const double outputValue = mathfunctions::sqrt(inputValue);
#else
  const double outputValue = std::sqrt(inputValue);
#endif
```
---

## 四. 为库添加使用依赖

### 1. 主项目的`CMakeLists.txt`
```cmake
cmake_minimum_required(VERSION 3.10)
project(Tutorial VERSION 1.0)

# 创建一个接口库目标 tutorial_compiler_flags，用于统一管理编译选项，接口库不生成实际代码，仅用于传递编译特性（如 C++ 标准）
add_library(tutorial_compiler_flags INTERFACE)

# 为接口库添加 C++11 标准要求，通过 INTERFACE 关键字传递给依赖它的目标， 所有链接此库的目标会自动启用 C++11
target_compile_features(tutorial_compiler_flags INTERFACE cxx_std_11)

configure_file(TutorialConfig.h.in TutorialConfig.h)
add_subdirectory(MathFunctions)
add_executable(Tutorial tutorial.cxx)
target_link_libraries(Tutorial PRIVATE tutorial_compiler_flags)
target_link_libraries(Tutorial PUBLIC MathFunctions)
target_include_directories(Tutorial PUBLIC "${PROJECT_BINARY_DIR}")
```

### 2. 子模块 `MathFunctions` 的 `CMakeLists.txt`
```cmake
add_library(MathFunctions MathFunctions.cxx)
target_include_directories(MathFunctions INTERFACE ${CMAKE_CURRENT_SOURCE_DIR})

option(USE_MYMATH "Use tutorial provided math implementation" ON)

if (USE_MYMATH)
    target_compile_definitions(MathFunctions PRIVATE "USE_MYMATH")
    add_library(SqrtLibrary STATIC mysqrt.cxx)
    target_link_libraries(SqrtLibrary PRIVATE tutorial_compiler_flags)
    target_link_libraries(MathFunctions PRIVATE SqrtLibrary)
endif()

target_link_libraries(MathFunctions PRIVATE tutorial_compiler_flags)
```

---

## 五. 生成器表达式
### 1. 主项目的`CMakeLists.txt`
```cmake
# 指定 CMake 的最低版本要求为 3.15，确保可以使用生成器表达式
cmake_minimum_required(VERSION 3.15)
project(Tutorial VERSION 1.0)
add_library(tutorial_compiler_flags INTERFACE)
target_compile_features(tutorial_compiler_flags INTERFACE cxx_std_11)

# 定义生成器表达式变量，用于检测编译器类型
set(gcc_like_cxx "$<COMPILE_LANG_AND_ID:CXX,ARMClang,AppleClang,Clang,GNU,LCC>")
set(msvc_cxx "$<COMPILE_LANG_AND_ID:CXX,MSVC>")

# 为接口库添加编译器选项, 使用生成器表达式根据编译器类型设置不同的编译选项
target_compile_options(tutorial_compiler_flags INTERFACE
    # 对于类 GCC 的编译器，启用以下警告选项：
    # -Wall：启用所有常见警告
    # -Wextra：启用额外警告
    # -Wshadow：警告变量遮蔽
    # -Wformat=2：检查格式化字符串的安全性
    # -Wunused：警告未使用的变量或函数
    "$<${gcc_like_cxx}:$<BUILD_INTERFACE:-Wall;-Wextra;-Wshadow;-Wformat=2;-Wunused>>"
    # 对于 MSVC 编译器，启用警告级别 3（-W3）
    "$<${msvc_cxx}:$<BUILD_INTERFACE:-W3>>"
)

configure_file(TutorialConfig.h.in TutorialConfig.h)
add_subdirectory(MathFunctions)
add_executable(Tutorial tutorial.cxx)
target_link_libraries(Tutorial PUBLIC MathFunctions tutorial_compiler_flags)
target_include_directories(Tutorial PUBLIC "${PROJECT_BINARY_DIR}")
```

## 六. 安装与测试
- **`add_custom_target`** 用于定义一个自定义目标，该目标可以执行一些与构建过程相关的任务（例如生成文档、运行脚本、打包发布文件等）。自定义目标不会直接参与构建过程，但可以通过 `cmake --build . --target 目标名称` 显式调用。
- **`message`** 用于在 `CMake` 配置或构建过程中输出信息。它可以用于调试、提示用户或记录日志。
- **`cmake -DCMAKE_INSTALL_PREFIX="../install" ..`**:  指定安装路径。
- **`ctest`** 是 `CMake` 的测试工具，用于运行项目中定义的测试用例。
  - `--output-on-failure`：在测试失败时输出详细信息。
  - `-R <正则表达式>`：运行名称匹配正则表达式的测试。
  - `-E <正则表达式>`：排除名称匹配正则表达式的测试。
  - `-j <数量>`：并行运行测试（指定线程数）。
  - `-V 或 --verbose`：显示详细输出。
  - `-N`：仅列出测试，不运行。
### 1. 主项目的`CMakeLists.txt`
```cmake
cmake_minimum_required(VERSION 3.15)
project(Tutorial VERSION 1.0)
add_library(tutorial_compiler_flags INTERFACE)
target_compile_features(tutorial_compiler_flags INTERFACE cxx_std_11)
set(gcc_like_cxx "$<COMPILE_LANG_AND_ID:CXX,ARMClang,AppleClang,Clang,GNU,LCC>")
set(msvc_cxx "$<COMPILE_LANG_AND_ID:CXX,MSVC>")
target_compile_options(tutorial_compiler_flags INTERFACE
    "$<${gcc_like_cxx}:$<BUILD_INTERFACE:-Wall;-Wextra;-Wshadow;-Wformat=2;-Wunused>>"
    "$<${msvc_cxx}:$<BUILD_INTERFACE:-W3>>"
)
configure_file(TutorialConfig.h.in TutorialConfig.h)
add_subdirectory(MathFunctions)
add_executable(Tutorial tutorial.cxx)
target_link_libraries(Tutorial PUBLIC MathFunctions tutorial_compiler_flags)
target_include_directories(Tutorial PUBLIC "${PROJECT_BINARY_DIR}")

# 安装配置：
install(TARGETS Tutorial DESTINATION bin)
install(FILES ${CMAKE_CURRENT_BINARY_DIR}/TutorialConfig.h DESTINATION include)

# 启用测试功能
enable_testing()
add_test(NAME Runs COMMAND Tutorial 25)
add_test(NAME Usage COMMAND Tutorial)

# 设置测试属性：
set_tests_properties(Usage PROPERTIES PASS_REGULAR_EXPRESSION "Usage.*number")
set_tests_properties(Runs PROPERTIES PASS_REGULAR_EXPRESSION "25 is 5")

function(do_test target arg result)
    add_test(NAME Comp${arg} COMMAND ${target} ${arg})
    set_tests_properties(Comp${arg}
        PROPERTIES PASS_REGULAR_EXPRESSION ${result}
    )
endfunction()
do_test(Tutorial 4 "4 is 2")
do_test(Tutorial 9 "9 is 3")
do_test(Tutorial 5 "5 is 2.236")
do_test(Tutorial 7 "7 is 2.645")
do_test(Tutorial 25 "25 is 5")
do_test(Tutorial -25 "-25 is (-nan|nan|0)")
do_test(Tutorial 0.0001 "0.0001 is 0.01")
```

### 2.子模块 `MathFunctions` 的 `CMakeLists.txt`
```cmake
add_library(MathFunctions MathFunctions.cxx)

target_include_directories(MathFunctions INTERFACE ${CMAKE_CURRENT_SOURCE_DIR})
option(USE_MYMATH "Use tutorial provided math implementation" ON)
if (USE_MYMATH)
    target_compile_definitions(MathFunctions PRIVATE "USE_MYMATH")
    add_library(SqrtLibrary STATIC mysqrt.cxx)
    target_link_libraries(SqrtLibrary PUBLIC tutorial_compiler_flags)
    target_link_libraries(MathFunctions PRIVATE SqrtLibrary)
endif ()

target_link_libraries(MathFunctions PUBLIC tutorial_compiler_flags)

set(installable_libs MathFunctions tutorial_compiler_flags)
install(TARGETS ${installable_libs} DESTINATION lib)
install(FILES MathFunctions.h DESTINATION include)
```
---

## 七. 安装与测试
- **`include(CTest)`**：启用测试功能，并支持高级测试模式。
- **`ctest -VV -D Experimental`**：运行测试并生成详细日志。
- 生成 `Makefile` 并测试的流程：
  - 使用 `cmake -G "MinGW Makefiles" ..` 生成构建系统。
  - 使用 `cmake --build .` 构建项目。
  - 使用 `ctest -VV -D Experimental` 运行测试。
  - `https://my.cdash.org/index.php?project=CMakeTutorial`, 可以查看测试结果
---

## 八. 添加系统特性检查
### 1. 子模块 `MathFunctions` 的 `CMakeLists.txt`
```cmake
add_library(MathFunctions MathFunctions.cxx)
target_include_directories(MathFunctionsINTERFACE ${CMAKE_CURRENT_SOURCE_DIR})
option(USE_MYMATH "Use tutorial provided math implementation" ON)

if (USE_MYMATH)
    target_compile_definitions(MathFunctions PRIVATE "USE_MYMATH")
    add_library(SqrtLibrary STATIC mysqrt.cxx)
    target_link_libraries(SqrtLibrary PUBLIC tutorial_compiler_flags)
    include(CheckCXXSourceCompiles)

    check_cxx_source_compiles("
        #include <cmath>
        int main() {
          std::log(1.0);
          return 0;
        } " HAVE_LOG)

    check_cxx_source_compiles("
        #include <cmath>
        int main() {
          std::exp(1.0);
          return 0;
        } " HAVE_EXP)

  if (HAVE_LOG)
      target_compile_definitions(SqrtLibrary PRIVATE "HAVE_LOG")
  endif()
  
  if (HAVE_EXP)
      target_compile_definitions(SqrtLibrary PRIVATE "HAVE_EXP")
  endif()

  target_link_libraries(MathFunctions PRIVATE SqrtLibrary)
endif ()


target_link_libraries(MathFunctions PUBLIC tutorial_compiler_flags)
set(installable_libs MathFunctions tutorial_compiler_flags)
if (TARGET SqrtLibrary)
    list(APPEND installable_libs SqrtLibrary)
endif ()
install(TARGETS ${installable_libs} DESTINATION lib)
install(FILES MathFunctions.h DESTINATION include)
```

### 2. 条件预编译
```c++
#if defined(HAVE_LOG) && defined(HAVE_EXP)
    double result = std::exp(std::log(x) * 0.5);
    std::cout << "Computing sqrt of " << x << " to be " << result << " using log and exp" << std::endl;
#else
    double result = x;
    for (int i = 0; i < 10; ++i) {
        if (result <= 0) {
            result = 0.1;
        }
        double delta = x - (result * result);
        result = result + 0.5 * delta / result;
        std::cout << "Computing sqrt of " << x << " to be " << result << std::endl;
    }
#endif
```
