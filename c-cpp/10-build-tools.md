# 构建工具与项目工程 (Build Tools & Project Engineering)

> 来源：compiler-toolchain/SKILL.md 的构建系统部分 + cpp-programming/SKILL.md 的工具链部分
> 覆盖范围：CMake 深度教程、Make/Ninja/Bazel、包管理、IDE 集成、注释风格、代码风格、Femeter 项目风格参考

---

## 1. CMake 深度教程

### 现代 CMake 风格（target-based）

```cmake
cmake_minimum_required(VERSION 3.20)
project(MyProject VERSION 1.0 LANGUAGES C CXX)

# 设置 C/C++ 标准
set(CMAKE_C_STANDARD 17)
set(CMAKE_CXX_STANDARD 20)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)

# 库目标
add_library(mylib src/lib.cpp)
target_include_directories(mylib PUBLIC include)
target_compile_features(mylib PUBLIC cxx_std_20)
target_compile_options(mylib PRIVATE
    $<$<CONFIG:Release>:-O2 -flto=thin>
    $<$<CONFIG:Debug>:-Og -g3>
    -Wall -Wextra -Werror
)
target_compile_definitions(mylib PRIVATE MYLIB_EXPORTS)

# 可执行目标
add_executable(app src/main.cpp)
target_link_libraries(app PRIVATE mylib)
```

### 库类型

```cmake
add_library(mylib STATIC src/lib.cpp)        # 静态库 .a/.lib
add_library(mylib SHARED src/lib.cpp)        # 动态库 .so/.dll
add_library(mylib OBJECT src/lib.cpp)        # 对象库 .o
add_library(mylib INTERFACE src/lib.cpp)     # 接口库（无源文件，只传递属性）
```

### find_package / FetchContent

```cmake
# 系统已安装的库
find_package(cpr REQUIRED)
target_link_libraries(app PRIVATE cpr::cpr)

# 从网络拉取
include(FetchContent)
FetchContent_Declare(
    json
    GIT_REPOSITORY https://github.com/nlohmann/json
    GIT_TAG v3.11.3
)
FetchContent_MakeAvailable(json)
target_link_libraries(app PRIVATE nlohmann_json::nlohmann_json)

# add_subdirectory（本地第三方库）
add_subdirectory(third_party/freertos)
```

### 交叉编译

```cmake
# toolchain-aarch64.cmake
set(CMAKE_SYSTEM_NAME Linux)
set(CMAKE_SYSTEM_PROCESSOR aarch64)
set(CMAKE_C_COMPILER aarch64-linux-gnu-gcc)
set(CMAKE_CXX_COMPILER aarch64-linux-gnu-g++)
set(CMAKE_SYSROOT /opt/aarch64-rootfs)
set(CMAKE_FIND_ROOT_PATH_MODE_PROGRAM NEVER)
set(CMAKE_FIND_ROOT_PATH_MODE_LIBRARY ONLY)
set(CMAKE_FIND_ROOT_PATH_MODE_INCLUDE ONLY)

# 使用
cmake -DCMAKE_TOOLCHAIN_FILE=toolchain-aarch64.cmake ..
```

### CMakePresets.json (CMake 3.19+)

```json
{
    "version": 3,
    "configurePresets": [
        {
            "name": "debug",
            "binaryDir": "${sourceDir}/build/debug",
            "cacheVariables": {
                "CMAKE_BUILD_TYPE": "Debug",
                "CMAKE_CXX_STANDARD": "20"
            }
        },
        {
            "name": "release",
            "binaryDir": "${sourceDir}/build/release",
            "cacheVariables": {
                "CMAKE_BUILD_TYPE": "Release"
            }
        },
        {
            "name": "cross-arm",
            "binaryDir": "${sourceDir}/build/arm",
            "cacheVariables": {
                "CMAKE_TOOLCHAIN_FILE": "toolchain-arm.cmake"
            }
        }
    ]
}
```

```bash
cmake --preset debug
cmake --build --preset debug
```

### 测试

```cmake
enable_testing()
add_test(NAME MyTest COMMAND tests)

# Google Test
find_package(GTest REQUIRED)
add_executable(tests tests/test_main.cpp tests/test_utils.cpp)
target_link_libraries(tests PRIVATE mylib GTest::gtest_main)
add_test(NAME AllTests COMMAND tests)

# CTest
ctest --output-on-failure
```

### install / export

```cmake
install(TARGETS app RUNTIME DESTINATION bin)
install(TARGETS mylib LIBRARY DESTINATION lib ARCHIVE DESTINATION lib)
install(FILES include/mylib.h DESTINATION include)

# 导出供其他项目使用
install(EXPORT MyProjectTargets
    FILE MyProjectTargets.cmake
    NAMESPACE MyProject::
    DESTINATION lib/cmake/MyProject
)
```

### configure_file

```cmake
configure_file(config.h.in ${CMAKE_BINARY_DIR}/config.h)
target_include_directories(app PRIVATE ${CMAKE_BINARY_DIR})
```

---

## 2. Femeter 项目风格参考

Femeter 是 Rust 嵌入式项目，CMake 用于第三方库（FreeRTOS）。

### 风格要点

```
# 1. 嵌入式交叉编译：CMAKE_SYSTEM_NAME=Generic, CMAKE_C_COMPILER=arm-none-eabi-gcc
# 2. 第三方库用 add_subdirectory 引入
# 3. 目标分离：每个模块独立 target
# 4. 链接顺序：先应用后驱动（依赖顺序）
# 5. 编译选项集中管理
```

### 嵌入式 CMake 示例

```cmake
cmake_minimum_required(VERSION 3.20)
project(Firmware C ASM)

set(CMAKE_SYSTEM_NAME Generic)
set(CMAKE_C_COMPILER arm-none-eabi-gcc)
set(CMAKE_ASM_COMPILER arm-none-eabi-gcc)

# 集中管理编译选项
add_compile_options(
    -mcpu=cortex-m4 -mthumb
    -ffunction-sections -fdata-sections
    -Wall -Wextra -Wpedantic
)

# 第三方库
add_subdirectory(third_party/FreeRTOS)

# 应用模块（独立 target）
add_library(app_module src/app.c)
target_include_directories(app_module PUBLIC include)
target_link_libraries(app_module PUBLIC FreeRTOS::Kernel)

# 最终可执行文件
add_executable(firmware.elf src/main.c)
target_link_libraries(firmware.elf PRIVATE app_module)
target_link_options(firmware.elf PRIVATE
    -T ${CMAKE_SOURCE_DIR}/linker.ld
    -Wl,--gc-sections
    -Wl,-Map=output.map
)
```

---

## 3. 大型 C/C++ 项目 CMake 经验

### 推荐项目结构

```
project-root/
├── CMakeLists.txt          # 顶层
├── CMakePresets.json       # 预设
├── cmake/                  # CMake 模块
│   ├── CompilerWarnings.cmake
│   ├── Sanitizers.cmake
│   └── FindCustomLib.cmake
├── include/
│   └── project/
├── src/
│   ├── CMakeLists.txt
│   ├── module_a/
│   │   ├── CMakeLists.txt
│   │   ├── module_a.h
│   │   └── module_a.cpp
│   └── module_b/
│       ├── CMakeLists.txt
│       ├── module_b.h
│       └── module_b.cpp
├── tests/
│   ├── CMakeLists.txt
│   └── test_*.cpp
├── examples/
├── docs/
└── third_party/
```

### 模块化最佳实践

```cmake
# cmake/CompilerWarnings.cmake
function(set_project_warnings target)
    target_compile_options(${target} INTERFACE
        -Wall -Wextra -Wpedantic
        -Wshadow -Wconversion
        $<$<CXX_COMPILER_ID:GNU>:-Wnon-virtual-dtor>
    )
endfunction()

# cmake/Sanitizers.cmake
function(enable_sanitizers target)
    if(CMAKE_BUILD_TYPE STREQUAL "Debug")
        target_compile_options(${target} INTERFACE -fsanitize=address,undefined -fno-omit-frame-pointer)
        target_link_options(${target} INTERFACE -fsanitize=address,undefined)
    endif()
endfunction()
```

### 依赖管理策略

| 方式 | 适用场景 | 优点 | 缺点 |
|------|----------|------|------|
| `find_package` | 系统已安装 | 简单快速 | 需要预装 |
| `FetchContent` | 开源库 | 自动拉取 | 首次慢 |
| `add_subdirectory` | 本地/vendored | 完全控制 | 污染命名空间 |
| `vcpkg` | 跨平台 | 二进制缓存 | 需要安装 |
| `conan` | 企业/复杂依赖 | 版本管理 | 学习曲线 |

### 编译时间优化

```cmake
# UNITY_BUILD（合并编译单元）
set(CMAKE_UNITY_BUILD ON)
set(CMAKE_UNITY_BUILD_BATCH_SIZE 8)

# 预编译头
target_precompile_headers(mylib PRIVATE
    <vector> <string> <memory>
)

# ccache
set(CMAKE_C_COMPILER_LAUNCHER ccache)
set(CMAKE_CXX_COMPILER_LAUNCHER ccache)
```

---

## 4. Make / Makefile

### 基础

```makefile
CC = gcc
CFLAGS = -Wall -Wextra -g -O2
LDFLAGS = -lm
TARGET = myapp
SRCS = main.c utils.c parser.c
OBJS = $(SRCS:.c=.o)

all: $(TARGET)

$(TARGET): $(OBJS)
	$(CC) $(CFLAGS) -o $@ $^ $(LDFLAGS)

%.o: %.c
	$(CC) $(CFLAGS) -MMD -MP -c $< -o $@

-include $(OBJS:.o=.d)

clean:
	rm -f $(OBJS) $(TARGET) $(OBJS:.o=.d)
```

### 注意
- 缩进必须用 **Tab**
- `-MMD -MP` 自动生成头文件依赖
- `-include $(DEPS)` 包含依赖文件

---

## 5. Ninja

```bash
# CMake 使用 Ninja 后端（推荐）
cmake -G Ninja -B build -DCMAKE_BUILD_TYPE=Release
cmake --build build
```

> Ninja 是 CMake 的推荐后端，构建速度比 Make 快 2-10x。

---

## 6. Bazel

```python
# BUILD 文件
cc_library(
    name = "mylib",
    srcs = ["lib.cpp"],
    hdrs = ["lib.h"],
    deps = ["@nlohmann_json"],
)

cc_binary(
    name = "app",
    srcs = ["main.cpp"],
    deps = [":mylib"],
)

cc_test(
    name = "mylib_test",
    srcs = ["test.cpp"],
    deps = [":mylib", "@googletest"],
)
```

```bash
bazel build //:app
bazel test //...
bazel build //:app --remote_cache=grpc://cache.example.com:9092
```

---

## 7. 包管理器

### vcpkg

```bash
# 安装
git clone https://github.com/microsoft/vcpkg
./vcpkg/bootstrap-vcpkg.sh

# 安装库
./vcpkg install fmt
./vcpkg install spdlog

# CMake 集成
cmake -DCMAKE_TOOLCHAIN_FILE=/path/to/vcpkg/scripts/buildsystems/vcpkg.cmake ..
```

### Conan

```bash
# conanfile.txt
[requires]
fmt/10.0.0

[generators]
CMakeDeps
CMakeToolchain

# 使用
conan install .
cmake --preset conan-release
```

### 编译缓存

```bash
# ccache
export CC="ccache gcc"
export CXX="ccache g++"
ccache -s  # 查看命中率

# CMake 集成
cmake -DCMAKE_C_COMPILER_LAUNCHER=ccache \
      -DCMAKE_CXX_COMPILER_LAUNCHER=ccache ..

# sccache（支持远程缓存 S3/GCS）
export RUSTC_WRAPPER=sccache
```

---

## 8. IDE 集成

### VS Code + CMake Tools + clangd

```json
// .vscode/settings.json
{
    "cmake.configureOnOpen": true,
    "cmake.buildDirectory": "${workspaceFolder}/build",
    "clangd.arguments": ["--header-insertion=never", "--compile-commands-dir=${workspaceFolder}/build"]
}
```

### CLion
- 原生 CMake 支持，自动检测
- 内置调试器、代码分析
- 收费，但 C/C++ 开发体验最好

### Visual Studio
- 打开 CMakeLists.txt 即可
- 强大的调试器
- 仅 Windows

### Xcode
- `cmake -G Xcode ..` 生成 Xcode 项目
- macOS/iOS 开发首选

---

## 9. 注释与文档风格

### 文件头注释（C/C++）

```c
/* ================================================================== */
/*                                                                    */
/*  filename.c — 模块简述                                              */
/*                                                                    */
/*  硬件: XXX MCU (Flash/RAM/时钟)                                    */
/*  功能: 详细描述                                                      */
/*  依赖: 列出依赖的模块                                               */
/*                                                                    */
/*  数据流: 输入 → 处理 → 输出                                        */
/*                                                                    */
/*  (c) 2026 Project Name — Author                                    */
/* ================================================================== */
```

### 章节分隔

```c
/* ── 章节名称 ── */
```

### 函数注释（Doxygen 风格）

```c
/**
 * @brief 函数简述
 *
 * 详细描述（可选）
 *
 * @param[in]  name    参数描述
 * @param[out] result  输出参数描述
 * @return 返回值描述
 * @note 注意事项
 * @warning 警告信息
 * @see 相关函数
 */
```

### 行内注释

```c
/* 单行注释说明 */
```

### Rust 风格（参考 Femeter）

```rust
//! 模块文档（//!）
//! 详细描述

/// 函数文档（///）
/// 详细描述
///
/// # Arguments
/// * `param` - 参数描述
///
/// # Returns
/// 返回值描述
///
/// # Safety
/// 安全说明（unsafe 函数必须）
///
/// # Examples
/// ```
/// let result = function(arg);
/// ```
```

---

## 10. 代码风格

### clang-format 配置

```yaml
# .clang-format
BasedOnStyle: Google
IndentWidth: 4
ColumnLimit: 100
AllowShortFunctionsOnASingleLine: Inline
AllowShortIfStatementsOnASingleLine: false
BreakBeforeBraces: Attach
```

### clang-tidy 常用规则

```bash
clang-tidy src.cpp --checks='modernize-*,bugprone-*,performance-*,readability-*'
```

### 命名规范

| 类型 | 风格 | 示例 |
|------|------|------|
| 函数 | snake_case | `get_value()` |
| 变量 | snake_case | `my_variable` |
| 类/结构体 | PascalCase | `MyClass`, `TcpConnection` |
| 常量/宏 | UPPER_SNAKE_CASE | `MAX_SIZE`, `PI` |
| 成员变量 | m_ 前缀或 _ 后缀 | `m_count`, `count_` |
| 模板参数 | PascalCase 或单字母 | `typename T` |
