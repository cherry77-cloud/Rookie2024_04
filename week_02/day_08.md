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

## 二. 练习一

- **`cmake -G "MinGW Makefiles" ..`**: 调用`CMake`程序, 指定生成器类型。告诉`CMake`使用`MinGW`的`make`工具来构建项目, 指向`CMakeLists.txt`文件所在的源码目录（当前目录的上一级）
- **`cmake --build .`**: 使用`CMake`的构建命令，自动选择合适的构建工具。`.`表示当前目录
- **`mingw32-make`**: 直接调用`MinGW`提供的`make`工具
- **`_cplusplus`** 是 `C++` 标准定义的宏，用于检测当前编译器使用的 `C++` 标准版本

```cmake
# 设置CMake的最低版本要求为3.10
cmake_minimum_required(VERSION 3.10)

# 创建一个名为Tutorial的项目, 在项目命令中设置项目版本号为1.0
project(Tutorial VERSION 1.0)

# 设置C++标准为11，并强制要求使用C++11
set(CMAKE_CXX_STANDARD 11)
set(CMAKE_CXX_STANDARD_REQUIRED ON)  # 强制要求使用C++11

# 设置一个字符串变量，内容为"Hello World"
set(STR_TEST "Hello World")

# 使用configure_file将TutorialConfig.h.in文件配置并复制为TutorialConfig.h
# 该步骤会将TutorialConfig.h.in中的占位符（如@Tutorial_VERSION_MAJOR@）替换为实际值
configure_file(TutorialConfig.h.in TutorialConfig.h)

# 添加一个名为Tutorial的可执行文件，源文件为tutorial.cxx
add_executable(Tutorial tutorial.cxx)

# 使用target_include_directories将${PROJECT_BINARY_DIR}添加到头文件搜索路径中
# 这样编译器可以找到生成的TutorialConfig.h文件
target_include_directories(Tutorial PUBLIC ${PROJECT_BINARY_DIR})

# 打印一些调试信息
message(STATUS "二进制目录: ${PROJECT_BINARY_DIR}")
message(STATUS "主版本号: ${Tutorial_VERSION_MAJOR}")
message(STATUS "次版本号: ${Tutorial_VERSION_MINOR}")
message(STATUS "测试字符串: ${STR_TEST}")
```
