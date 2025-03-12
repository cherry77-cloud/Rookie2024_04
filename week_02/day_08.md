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

- `cmake_minimum_required`: 指定 `CMake` 的最低版本要求，确保项目使用的 `CMake` 功能与版本兼容。
- `project`: 定义项目名称，并隐式启用默认编程语言（`C/C++`）。设置变量 `PROJECT_NAME` 为项目名称。自动检测系统默认的 `C/C++` 编译器。`project(<PROJECT-NAME> [LANGUAGES] [<language>...])`
- `add_executable`: 生成一个可执行文件目标，指定其名称和源文件。`add_executable(<target> [<source>...])`

```bash
# TODO 1: Set the minimum required version of CMake to be 3.10
cmake_minimum_required(VERSION 3.10)

# TODO 2: Create a project named Tutorial
project(Tutorial)

# TODO 3: Add an executable called Tutorial to the project
add_executable(Tutorial tutorial.cxx)
```

```bash
# 配置CMake项目
cmake -G "MinGW Makefiles" ..
# cmake: 调用CMake程序
# -G "MinGW Makefiles": 指定生成器类型。告诉CMake使用MinGW的make工具来构建项目
# ..: 指向CMakeLists.txt文件所在的源码目录（当前目录的上一级）

cmake --build .
# cmake --build .: 使用CMake的构建命令，自动选择合适的构建工具。.表示当前目录

mingw32-make
# mingw32-make: 直接调用MinGW提供的make工具（Windows上不能直接使用make命令）
```
---

