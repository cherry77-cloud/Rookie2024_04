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

```cmake
cmake_minimum_required(VERSION 3.10)

project(Tutorial VERSION 1.0)
set(CMAKE_CXX_STANDARD 11)
set(CMAKE_CXX_STANDARD_REQUIRED True)

configure_file(TutorialConfig.h.in TutorialConfig.h)
add_executable(Tutorial tutorial.cxx)

target_include_directories(Tutorial PUBLIC ${PROJECT_BINARY_DIR})

# 在TutorialConfig.h.in中编辑
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

# 为可执行文件添加头文件包含目录
target_include_directories(Tutorial PUBLIC
    "${PROJECT_BINARY_DIR}"      # 包含生成的 TutorialConfig.h
    "${PROJECT_SOURCE_DIR}/MathFunctions"  # 包含自定义数学库头文件
)
```

### 3. 子模块 `MathFunctions` 的 `CMakeLists.txt`
```bash
# 创建名为 MathFunctions 的静态库，仅包含 MathFunctions.cxx 源文件
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

### 3. 代码中的条件编译

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
