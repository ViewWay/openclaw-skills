# Compiler & Toolchain Tuning (编译器与工具链调优)

> 资深工程师级别的编译器调优、构建优化与性能分析指南。涵盖 GCC、Clang/LLVM、JVM、Rust 及主流构建系统。

---

## 目录

- [GCC 调优](#gcc-调优)
- [LLVM/Clang 调优](#llvmclang-调优)
- [JVM 调优](#jvm-调优)
- [Rust 编译调优](#rust-编译调优)
- [构建系统](#构建系统)
- [性能分析工具](#性能分析工具)

---

## GCC 调优

### 编译选项体系

#### 概述

GCC 编译选项是 C/C++ 性能与安全优化的核心手段。合理组合优化级别、警告、安全加固与链接时优化，可将二进制性能提升 10-30% 同时显著减少安全隐患。

#### 关键选项

| 类别 | 选项 | 说明 |
|------|------|------|
| **优化级别** | `-O0` | 无优化，调试用 |
| | `-O1` | 基础优化，编译快 |
| | `-O2` | **推荐默认**，平衡速度与体积 |
| | `-O3` | 激进优化（自动向量化、循环展开），可能增大体积 |
| | `-Os` | 优化体积（嵌入式首选） |
| | `-Og` | 调试友好优化（`-O1` 子集，保留调试信息质量） |
| **警告** | `-Wall -Wextra` | 基础 + 额外警告 |
| | `-Werror` | 警告即错误 |
| | `-Wpedantic` | 严格 ISO C/C++ 合规 |
| | `-Wshadow` | 变量遮蔽检测 |
| | `-Wconversion` | 隐式类型转换警告 |
| | `-Wformat-security` | 格式化字符串安全 |
| | `-Wnull-dereference` | 空指针解引用 |
| | `-Wstack-protector` | 栈保护相关 |
| **调试** | `-g -g3` | 最大调试信息 |
| | `-gdwarf-5` | DWARF v5（支持更多类型信息） |
| | `-fno-omit-frame-pointer` | 保留帧指针（profiling 必需） |
| **安全加固** | `-fstack-protector-strong` | 栈溢出保护 |
| | `-D_FORTIFY_SOURCE=2` | 缓冲区溢出检测（需 `-O1`+） |
| | `-fPIE -pie` | 地址空间布局随机化（ASLR） |
| | `-Wl,-z,relro,-z,now` | 只读重定位 + 立即绑定 |
| **LTO** | `-flto` | 链接时优化，跨编译单元内联 |
| | `-flto=thin` | ThinLTO，更快链接，接近全 LTO 效果 |
| | `-flto -ffat-lto-objects` | 同时生成 LTO 与普通对象文件 |
| **PGO** | `-fprofile-generate` | 第一阶段：生成 profile |
| | `-fprofile-use` | 第二阶段：使用 profile 优化 |
| | `-fauto-profile=xxx.prof` | 使用 AutoFDO profile |
| **Sanitizer** | `-fsanitize=address` | 内存越界、use-after-free |
| | `-fsanitize=undefined` | 未定义行为检测（整数溢出、空指针等） |
| | `-fsanitize=thread` | 数据竞争检测（TSan） |
| | `-fsanitize=memory` | 未初始化内存读取（MSan） |
| | `-fsanitize=leak` | 内存泄漏检测 |

#### 实战示例

```bash
# 生产级 C/C++ 编译（推荐配置）
gcc -std=c17 -O2 -flto=thin \
    -Wall -Wextra -Werror -Wpedantic -Wshadow -Wconversion \
    -fstack-protector-strong -D_FORTIFY_SOURCE=2 -fPIE -pie \
    -Wl,-z,relro,-z,now \
    -o output source.c

# 调试构建
gcc -std=c17 -Og -g3 -gdwarf-5 -fno-omit-frame-pointer \
    -Wall -Wextra -Wshadow \
    -fsanitize=address,undefined -fno-omit-frame-pointer \
    -o output source.c

# PGO 三步优化
gcc -O2 -fprofile-generate -o app_prof source.c     # 1. instrumented build
./app_prof --run-typical-workload                       # 2. 收集 profile
gcc -O2 -fprofile-use -flto -o app source.c           # 3. 使用 profile 构建

# 嵌入式体积优化
arm-none-eabi-gcc -Os -flto -ffunction-sections -fdata-sections \
    -Wl,--gc-sections -fno-unwind-tables -fno-asynchronous-unwind-tables \
    -o firmware.elf main.c
```

#### 最佳实践

1. **CI/CD 中始终使用 `-Werror`**，但本地开发可先 `-Wall -Wextra`
2. **LTO + PGO** 是性能优化的黄金组合，大型项目收益可达 20%+
3. **Sanitizer 仅用于测试/开发**，生产构建禁用（`-fsanitize=none`）
4. **安全加固选项作为默认**，嵌入式等特殊场景按需调整
5. **`-fno-omit-frame-pointer`** 在 x86-64 上对性能影响极小（<1%），但显著提升 profiling 质量
6. **ThinLTO 优于全 LTO**：链接速度快 2-5x，优化效果接近

---

### GCC 内联汇编

#### 概述

GCC 内联汇编允许在 C/C++ 中嵌入汇编指令，用于性能关键路径、硬件访问和编译器无法表达的底层操作。扩展 asm 语法通过约束字符串告诉编译器操作数如何映射。

#### 关键语法

```c
// 基本内联汇编（无操作数）
asm volatile("cli");  // 禁止编译器优化 / 重排

// 扩展内联汇编
asm (
    "movl %1, %0\n\t"     // 指令模板
    : "=r"(dst)            // 输出操作数（= 表示写，r 表示寄存器）
    : "r"(src)             // 输入操作数
    : "cc"                 // clobber list（被修改的寄存器/标志）
);
```

**操作数约束：**

| 约束 | 含义 |
|------|------|
| `r` | 通用寄存器 |
| `m` | 内存操作数 |
| `i` | 立即数常量 |
| `a`/`b`/`c`/`d` | 特定寄存器（EAX/EBX/ECX/EDX） |
| `S`/`D` | ESI / EDI |
| `q` | a/b/c/d 中任一 |
| `0`-`9` | 与第 N 个操作数相同约束 |
| `=` | 输出（写） |
| `+` | 输入输出（读写） |
| `&` | 提前 clobber（输出在所有输入消费前写入） |

**修饰符：** `%b`/`%h`/`%w`/`%k`/`%q` 选择子寄存器大小（byte/word/dword/qword）。

#### 实战示例

```c
// 64 位原子 CAS（x86-64）
static inline int atomic_cas(volatile int *ptr, int old, int new_val) {
    int result;
    asm volatile (
        "lock; cmpxchgl %2, %1"
        : "=a"(result), "+m"(*ptr)
        : "r"(new_val), "0"(old)
        : "memory"
    );
    return result;
}

// RDTSC 读取时间戳计数器
static inline uint64_t rdtsc(void) {
    uint32_t lo, hi;
    asm volatile ("rdtsc" : "=a"(lo), "=d"(hi));
    return ((uint64_t)hi << 32) | lo;
}

// 编译器屏障（防止指令重排，不插入内存屏障）
#define compiler_barrier() asm volatile("" ::: "memory")

// 全内存屏障
#define memory_barrier() asm volatile("mfence" ::: "memory")
```

#### 最佳实践

1. **优先使用编译器内建**（`__builtin_ia32_*`、`<immintrin.h>`）而非手写内联汇编
2. **始终使用 `volatile`** 除非你确定编译器可以省略该 asm
3. **Clobber list 必须完整**：遗漏会导致难以排查的寄存器损坏 bug
4. **用 `+=m` 而非 `=m`** 当内存既是输入又是输出，避免编译器假设
5. **使用 `%[name]` 命名操作数** 提高可读性：

```c
asm ("add %[src], %[dst]"
     : [dst] "+r"(sum)
     : [src] "r"(val));
```

---

### GCC 属性

#### 概述

`__attribute__` 是 GCC 扩展语法，用于精细控制函数、变量、类型的编译行为——对齐、布局、链接符号、优化提示等。Clang 也支持大部分 GCC 属性。

#### 关键属性

| 属性 | 用途 | 示例 |
|------|------|------|
| `packed` | 取消结构体填充 | `struct __attribute__((packed)) hdr { ... };` |
| `aligned(n)` | 指定对齐 | `__attribute__((aligned(64)))` 用于 SIMD |
| `section("name")` | 指定段 | `__attribute__((section(".noinit")))` |
| `weak` | 弱符号（可被覆盖） | `void __attribute__((weak)) hook(void) { }` |
| `alias("sym")` | 符号别名 | `void __attribute__((alias("real_fn"))) wrapper(void);` |
| `constructor` | .init_array 执行 | 自动初始化，优先级 `(101)` |
| `destructor` | .fini_array 执行 | 自动清理 |
| `noreturn` | 函数不返回 | `_Noreturn` 或 `[[noreturn]]`（C11/C++11） |
| `always_inline` | 强制内联 | `static inline __attribute__((always_inline))` |
| `cold` | 冷路径提示 | 错误处理函数 |
| `hot` | 热路径提示 | 性能关键循环 |
| `unused` | 抑制未使用警告 | 条件编译中的工具函数 |
| `used` | 防止被优化删除 | 仅通过符号表引用的变量 |
| `visibility("default")` | 导出符号 | 配合 `-fvisibility=hidden` |

#### 实战示例

```c
// 共享库 API 导出控制
#if defined(BUILDING_DLL)
    #define API __attribute__((visibility("default")))
#else
    #define API
#endif

API int library_init(void);

// 对齐到缓存行（避免 false sharing）
typedef struct {
    int data;
    char pad[60];  // 填充到 64 字节
} __attribute__((aligned(64))) cache_aligned_t;

// 更现代的写法（C11 / C++11）
// C11: _Alignas(64)
// C++11: alignas(64)

// 弱符号实现可选钩子
void __attribute__((weak)) on_startup(void) {
    // 默认空实现，用户可定义同名函数覆盖
}

// 构造/析构自动执行
static void __attribute__((constructor(101))) early_init(void) {
    // 在 main() 之前执行
}
```

#### 最佳实践

1. **优先使用标准属性语法**：`[[nodiscard]]`、`[[maybe_unused]]`、`[[noreturn]]`（C++11/C23）
2. **`packed` 仅用于协议解析**，会影响访问性能（非对齐访问）
3. **`aligned(64)` 配合 `__builtin_prefetch`** 用于高性能数据结构
4. **弱符号用于插件架构**：核心库定义弱符号，插件提供强符号覆盖
5. **`visibility` 控制** 是减少共享库符号表、提升加载速度的标准做法

---

### GCC 链接

#### 概述

链接阶段决定了最终二进制的布局、符号解析和加载行为。理解静态/动态链接、链接脚本和符号可见性对构建高性能、可维护的二进制至关重要。

#### 关键选项

| 选项 | 说明 |
|------|------|
| `-static` | 全静态链接 |
| `-static-libgcc -static-libstdc++` | 仅静态链接运行时库 |
| `-Wl,--whole-archive -lfoo -Wl,--no-whole-archive` | 强制包含库中所有目标文件 |
| `-Wl,--gc-sections` | 裁剪未引用段（需 `-ffunction-sections -fdata-sections`） |
| `-Wl,-Map=output.map` | 生成链接映射文件 |
| `-Wl,--script=custom.ld` | 使用自定义链接脚本 |
| `-Wl,-rpath,$ORIGIN` | 运行时库搜索路径 |
| `-Wl,-z,relro,-z,now` | RELRO（只读重定位）+ BIND_NOW（立即绑定） |
| `-fvisibility=hidden` | 默认隐藏所有符号 |
| `-Wl,--version-script=versions.map` | 版本脚本控制符号导出 |

#### 实战示例

```bash
# 动态链接 + 符号裁剪
gcc -O2 -ffunction-sections -fdata-sections \
    -fvisibility=hidden \
    -Wl,--gc-sections -Wl,-Map=app.map \
    -o app main.c -lm -lpthread

# 自定义链接脚本（嵌入式）
gcc -T stm32f4.ld -nostartfiles -nostdlib -o firmware.elf startup.o main.o

# 共享库构建
gcc -shared -fPIC -fvisibility=hidden \
    -Wl,--version-script=libfoo.map \
    -o libfoo.so foo.c

# 版本脚本 libfoo.map
# {
#     global:
#         foo_init;
#         foo_process;
#     local:
#         *;
# };
```

#### 链接脚本要点

```ld
/* 最小化链接脚本示例 */
ENTRY(_start)

SECTIONS {
    . = 0x08000000;  /* Flash 起始地址 */

    .text : {
        *(.vectors)     /* 中断向量表放最前 */
        *(.text*)
        *(.rodata*)
    }

    .data : {
        *(.data*)
    } > RAM AT > FLASH

    .bss (NOLOAD) : {
        __bss_start = .;
        *(.bss*)
        *(COMMON)
        __bss_end = .;
    } > RAM

    _stack_top = ORIGIN(RAM) + LENGTH(RAM);
}
```

#### 最佳实践

1. **共享库务必用 `version-script` 或 `visibility`** 控制导出符号
2. **`-ffunction-sections -fdata-sections` + `--gc-sections`** 是减少二进制体积的标准组合
3. **`-Wl,-Map=app.map`** 在排查链接问题时不可或缺
4. **避免全静态链接**（`-static`），仅静态链接运行时库即可
5. **嵌入式使用链接脚本** 精确控制内存布局
6. **`--whole-archive`** 用于解决静态库中未被直接引用但需保留的符号（如全局构造函数）

---

### GCC 交叉编译

#### 概述

交叉编译是在一个平台上生成另一个平台可执行代码的过程。GCC 通过 `--target`、`--host`、`--build` 三元组和 sysroot 支持交叉编译。

#### 关键概念

| 概念 | 说明 |
|------|------|
| `--build` | 编译器运行的机器（通常省略） |
| `--host` | 编译器生成代码运行的机器 |
| `--target` | 编译器生成代码的目标机器（构建交叉编译器时） |
| `sysroot` | 目标系统的文件系统镜像（头文件、库） |
| `--sysroot=/path` | 指定 sysroot 路径 |
| `qemu-user` | 用户态模拟运行交叉编译产物 |

**工具链命名约定：** `<arch>-<vendor>-<os>-<abi>gcc`

| 目标 | 工具链前缀 |
|------|-----------|
| ARM bare-metal | `arm-none-eabi-` |
| ARM Linux | `arm-linux-gnueabihf-` |
| AArch64 Linux | `aarch64-linux-gnu-` |
| RISC-V | `riscv64-unknown-linux-gnu-` |

#### 实战示例

```bash
# 直接交叉编译
aarch64-linux-gnu-gcc -O2 --sysroot=/opt/aarch64-rootfs \
    -o app_aarch64 main.c

# CMake 交叉编译
# toolchain-aarch64.cmake:
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

# 用 QEMU user-mode 测试
qemu-aarch64 -L /opt/aarch64-rootfs ./app_aarch64
```

#### 最佳实践

1. **使用 crosstool-NG 或 buildroot** 构建自定义工具链
2. **sysroot 与工具链版本必须匹配**，否则链接时容易出现 ABI 不兼容
3. **CI 中用 QEMU user-mode 做交叉编译产物的冒烟测试**
4. **CMake toolchain file 是管理交叉编译配置的最佳方式**

---

## LLVM/Clang 调优

### Clang 与 GCC 对比

#### 概述

Clang 是 LLVM 前端，与 GCC 高度兼容但在诊断、编译速度和工具生态上有显著优势。两者在优化能力上各有胜负，取决于具体场景。

#### 对比

| 维度 | GCC | Clang |
|------|-----|-------|
| **诊断信息** | 足够但不够友好 | **更详细**，彩色高亮，Fix-It 提示 |
| **编译速度** | 较慢（尤其模板密集代码） | **更快**（2-3x），内存占用更低 |
| **优化质量** | 成熟，某些场景更优 | 持续改进，向量化常优于 GCC |
| **警告** | `-Wall -Wextra` | `-Wall -Wextra -Weverything` |
| **静态分析** | 无内置 | **clang-tidy** + clang-static-analyzer |
| **格式化** | 无 | **clang-format** |
| **调试信息** | DWARF | DWARF + **`-ftime-trace`** |
| **模块支持** | C20 模块 | C/C++ 模块 + ObjC 模块 |
| **ASan/UBSan** | 支持 | **更成熟**，报错更清晰 |
| **生态** | GNU 工具链标准 | Apple 默认，Google 内部主力 |

#### 实战示例

```bash
# Clang 超严格编译
clang++ -std=c++20 -O2 -Weverything -Wno-c++98-compat \
    -Werror -pedantic \
    -o app main.cpp

# 编译耗时分析（生成 time-trace JSON）
clang++ -O2 -ftime-trace -o app main.cpp
# 用 Chrome 打开 app.json 查看

# clang-tidy 检查
clang-tidy -p build main.cpp --checks='modernize-*,bugprone-*,performance-*'
```

#### 最佳实践

1. **CI 中同时用 GCC 和 Clang 编译**，捕获两家的警告
2. **开发阶段用 Clang** 获得更好的诊断，发布阶段对比两者性能
3. **`-ftime-trace`** 是排查编译慢的利器，直接在 Chrome DevTools 中查看
4. **`-Weverything` 配合白名单排除** 比逐步添加 `-Wxxx` 更彻底

---

### Clang 特有选项

#### 概述

Clang 在 GCC 兼容之外提供独特选项，包括模块系统、编译追踪和 C++ 运行时裁剪。

#### 关键选项

| 选项 | 说明 |
|------|------|
| `-fmodules` | 启用 C/C++ 模块（Clang 模块扩展） |
| `-fmodules-cache-path=<dir>` | 模块缓存路径 |
| `-ftime-trace` | 生成编译耗时 JSON（Chrome trace format） |
| `-fno-exceptions` | 禁用 C++ 异常（减体积，加速） |
| `-fno-rtti` | 禁用运行时类型信息 |
| `-fprofile-instr-generate` | LLVM PGO instrument（对应 GCC 的 `-fprofile-generate`） |
| `-fprofile-instr-use=<prof>` | LLVM PGO use |
| `-fcoverage-mapping` | 代码覆盖率映射 |
| `-Xclang -emit-llvm -S` | 输出 LLVM IR（`.ll`） |
| `-Xclang -disable-O0-optnone` | 即使 `-O0` 也保留 IR 优化信息 |

#### 实战示例

```bash
# C++ 减体积（无异常无 RTTI）
clang++ -std=c++20 -O2 -fno-exceptions -fno-rtti \
    -fvisibility=hidden -flto \
    -o minimal main.cpp

# 生成 LLVM IR 查看优化过程
clang++ -O2 -Xclang -disable-O0-optnone -Xclang -emit-llvm -S -o main.ll main.cpp

# 生成优化前后的 IR 对比
clang++ -O0 -Xclang -emit-llvm -S -o before.ll main.cpp
clang++ -O2 -Xclang -emit-llvm -S -o after.ll main.cpp
diff before.ll after.ll
```

#### 最佳实践

1. **`-fno-exceptions -fno-rtti`** 在嵌入式/游戏引擎中常用，可减少 5-15% 二进制体积
2. **`-ftime-trace` 应作为 CI 产物收集**，追踪编译时间回归
3. **生成 IR 是理解编译器行为的最佳方式**，比阅读汇编更直观

---

### LLVM IR

#### 概述

LLVM IR（Intermediate Representation）是 LLVM 的核心——所有前端语言编译为 IR，再由后端生成机器码。理解 IR 是编写 LLVM Pass 和调试优化问题的前提。

#### 格式

- **`.ll`**：可读文本格式
- **`.bc`**：二进制 bitcode 格式
- **转换**：`llvm-as file.ll -o file.bc` / `llvm-dis file.bc -o file.ll`

#### 关键工具

| 工具 | 用途 |
|------|------|
| `opt` | 运行优化 passes（`opt -O2 -S input.ll -o output.ll`） |
| `llc` | 后端代码生成（IR → 汇编/目标文件） |
| `llvm-mca` | 机器码分析器（吞吐量/延迟/流水线） |
| `lli` | 直接执行 IR（JIT） |
| `llvm-link` | 链接多个 IR 文件 |
| `llvm-ar` | 创建 LLVM bitcode 静态库 |

#### 实战示例

```bash
# 查看所有优化 passes
opt --passes='help' 2>&1 | less

# 运行特定 pass
opt -passes='inline,loop-vectorize' -S input.ll -o output.ll

# llvm-mca 分析吞吐量
cat hotloop.s | llvm-mca --mtriple=x86_64-unknown-linux-gnu \
    --mcpu=skylake

# 输出示例：
# Iterations:        100
# Total Cycles:      210
# Instructions:      400
# IPC:               1.90
# Block RThroughput: 2.1
```

#### 最佳实践

1. **用 `opt -O2 -S` 对比优化前后 IR** 定位性能问题
2. **`llvm-mca` 在写 SIMD 代码时极有用**——精确预测吞吐量
3. **自定义 Pass 用 C++ 编写**，参考 LLVM Pass Infrastructure 文档

---

### clang-tidy 规则

#### 概述

clang-tidy 是基于 Clang AST 的 C/C++ 静态分析工具，内置数百条规则，覆盖现代化改造、常见 bug、性能和可读性。

#### 关键规则类别

| 类别 | 前缀 | 重点关注 |
|------|------|---------|
| **现代化** | `modernize-*` | auto、nullptr、override、range-for、unique_ptr |
| **常见 Bug** | `bugprone-*` | 未检查返回值、错误 sizeof、赋值 vs 比较 |
| **性能** | `performance-*` | 不必要的拷贝、低效查找、auto-ref |
| **可读性** | `readability-*` | 命名、复杂度、else-after-return |
| **CERT/CPPCoreGuidelines** | `cert-*` / `cppcoreguidelines-*` | 安全与编码规范 |

#### 实战示例

```yaml
# .clang-tidy 配置文件
Checks: >
  -*,
  bugprone-*,
  modernize-*,
  performance-*,
  readability-*,
  -modernize-use-trailing-return-type,
  -readability-magic-numbers
WarningsAsErrors: ''
HeaderFilterRegex: 'src/.*\.h$'
CheckOptions:
  - key: modernize-use-override.IgnoreDestructors
    value: true
  - key: readability-identifier-naming.ClassCase
    value: CamelCase
  - key: performance-unnecessary-value-param.AllowedTypes
    value: 'std::vector'
```

```bash
# 运行检查
clang-tidy -p build src/*.cpp

# 自动修复
clang-tidy -p build src/*.cpp --fix

# 结合 CMake
cmake -DCMAKE_CXX_CLANG_TIDY="clang-tidy;-checks=modernize-*,bugprone-*" ..
```

#### 最佳实践

1. **先启用 `bugprone-*`**，收益最高、噪音最低
2. **`modernize-*` 配合 `--fix`** 可半自动迁移老旧代码到现代 C++
3. **`HeaderFilterRegex`** 限制检查范围，避免检查第三方头文件
4. **集成到 CI** 作为质量门禁，`-warnings-as-errors=*` 可选

---

### Sanitizer 组合

#### 概述

LLVM/Clang 的 Sanitizer 生态比 GCC 更成熟。合理组合可覆盖绝大多数内存安全和并发 bug。

#### 关键组合

| 组合 | 用途 | 适用场景 |
|------|------|---------|
| `address,undefined` | **首选组合** | 内存越界 + UB 检测 |
| `address,fuzzer` | 模糊测试 | 安全测试 |
| `thread` | 数据竞争 | 多线程程序（不能与其他 sanitizer 同时用） |
| `memory` | 未初始化内存 | 需要编译所有依赖（包括 libc） |
| `leak` | 内存泄漏 | 仅需加在最终链接 |

#### 实战示例

```bash
# 开发环境标配：ASan + UBSan
clang++ -std=c++20 -O1 -g \
    -fsanitize=address,undefined \
    -fno-omit-frame-pointer \
    -o app main.cpp

# 线程安全检测（单独使用）
clang++ -O1 -g -fsanitize=thread -o app_mt main.cpp

# 运行时控制
ASAN_OPTIONS=detect_leaks=1:detect_stack_use_after_return=1:halt_on_error=0 \
    UBSAN_OPTIONS=print_stacktrace=1 \
    ./app

# libFuzzer 集成
cat > fuzz_target.cpp << 'EOF'
extern "C" int LLVMFuzzerTestOneInput(const uint8_t *data, size_t size) {
    // 解析并测试
    parse_input(data, size);
    return 0;
}
EOF
clang++ -std=c++20 -O2 -g \
    -fsanitize=address,fuzzer \
    -o fuzzer fuzz_target.cpp
./fuzzer -max_len=4096 -jobs=4 corpus/
```

#### 最佳实践

1. **ASan + UBSan 作为默认开发配置**，覆盖率最高
2. **TSan 在发布前必须跑一轮**，数据竞争 bug 极难排查
3. **MSan 需要全程序编译**（包括依赖库），实用中受限
4. **`-O1` 是 Sanitizer 的推荐优化级别**——`-O0` 噪音太多，`-O2` 可能消除某些检测
5. **`ASAN_OPTIONS` / `UBSAN_OPTIONS`** 环境变量提供精细控制

---

## JVM 调优

### JVM 内存模型

#### 概述

JVM 内存分为堆、栈、元空间、直接内存和本地内存。理解各区域的作用与 GC 行为是调优的基础。

```
┌─────────────────────────────────────────────────┐
│                    JVM 内存                       │
├──────────────────┬──────────────────────────────┤
│   堆 (Heap)      │  Young Gen                    │
│   -Xms/-Xmx      │  ├─ Eden (80%)               │
│                  │  ├─ Survivor 0 (10%)          │
│                  │  └─ Survivor 1 (10%)          │
│                  │  Old Gen (Tenured)            │
├──────────────────┼──────────────────────────────┤
│   元空间          │  类元数据（替代 PermGen）      │
│   Metaspace      │  -XX:MetaspaceSize            │
├──────────────────┼──────────────────────────────┤
│   栈 (Stack)     │  线程私有，-Xss 控制          │
│   每线程一份      │  局部变量、方法帧              │
├──────────────────┼──────────────────────────────┤
│   直接内存         │  NIO DirectByteBuffer        │
│   Direct Memory  │  -XX:MaxDirectMemorySize      │
├──────────────────┼──────────────────────────────┤
│   本地内存         │  JNI、线程栈、GC 内部数据      │
│   Native Memory  │  -XX:MaxDirectMemorySize      │
└──────────────────┴──────────────────────────────┘
```

#### 关键参数

| 参数 | 说明 | 推荐值 |
|------|------|--------|
| `-Xms` | 初始堆大小 | 与 -Xms 相同 |
| `-Xmx` | 最大堆大小 | 物理内存 50-75% |
| `-XX:NewRatio` | Old/Young 比例（默认 2） | 1-2 |
| `-XX:SurvivorRatio` | Eden/Survivor 比例（默认 8） | 8 |
| `-XX:MetaspaceSize` | 元空间初始大小 | 256m |
| `-XX:MaxMetaspaceSize` | 元空间最大 | 512m-1g |
| `-Xss` | 线程栈大小 | 512k-1m |
| `-XX:MaxDirectMemorySize` | 直接内存上限 | 需 NIO 时设置 |

#### 实战示例

```bash
# 4GB 物理内存服务器，Web 应用
java -Xms2g -Xmx2g \
    -XX:+UseG1GC \
    -XX:MaxGCPauseMillis=200 \
    -XX:MetaspaceSize=256m -XX:MaxMetaspaceSize=512m \
    -Xss512k \
    -XX:+HeapDumpOnOutOfMemoryError \
    -XX:HeapDumpPath=/var/log/app/ \
    -Xlog:gc*:file=/var/log/app/gc.log:time,uptime,level,tags:filecount=5,filesize=50m \
    -jar app.jar

# 大内存低延迟（16GB 堆）
java -Xms16g -Xmx16g \
    -XX:+UseZGC \
    -XX:+ZGenerational \
    -Xlog:gc*:file=gc.log:time,uptime,level,tags \
    -jar app.jar
```

#### 最佳实践

1. **`-Xms` 与 `-Xmx` 设为相同**，避免运行时堆扩缩容
2. **堆大小不超过物理内存 75%**，留足 OS 缓存和直接内存
3. **线程数 × 栈大小 < 可用内存**，否则 `StackOverflowError` 或 OOM
4. **监控元空间增长**，类加载泄漏是常见的 Metaspace OOM 原因

---

### GC 算法

#### 概述

GC 算法选择直接影响吞吐量和延迟。JDK 9+ 默认 G1，JDK 21+ 引入分代 ZGC。

| GC | 适用场景 | 停顿目标 | 堆大小 |
|----|---------|---------|--------|
| **Serial** | 单线程小应用 | 长停顿 | < 1GB |
| **Parallel** | 吞吐量优先（批处理） | 中等停顿 | 1-8GB |
| **G1** | **通用推荐**，平衡吞吐/延迟 | 可控（200ms） | 1-32GB |
| **ZGC** | 超低延迟（交易/游戏） | < 1ms | 8GB-16TB |
| **Shenandoah** | 超低延迟替代方案 | < 10ms | 8GB+ |

#### 实战示例

```bash
# G1 GC（通用推荐）
java -XX:+UseG1GC \
    -XX:MaxGCPauseMillis=200 \
    -XX:G1HeapRegionSize=8m \      # 大堆用 16m/32m
    -XX:InitiatingHeapOccupancyPercent=45 \
    -XX:G1ReservePercent=10 \
    -jar app.jar

# ZGC（JDK 21+，分代模式）
java -XX:+UseZGC -XX:+ZGenerational \
    -Xlog:gc* \
    -jar app.jar

# GC 日志分析工具
# jdk Mission Control / GCViewer / GCEasy.io
```

#### 最佳实践

1. **大多数应用用 G1 就好**，除非有明确 <10ms 停顿需求
2. **ZGC（分代）是 JDK 21+ 的未来方向**，新项目可大胆使用
3. **不要手动调优 GC 参数**（除堆大小和 GC 选择外），先监控再调整
4. **`-XX:+AlwaysPreTouch`** 启动时预分配物理内存，减少运行时页面错误

---

### JVM 调优参数速查

#### 概述

以下是 JVM 生产环境常用参数，按类别分组。

#### 实战示例

```bash
# 生产环境推荐配置模板
JAVA_OPTS="
  # 内存
  -Xms${HEAP_SIZE} -Xmx${HEAP_SIZE}
  -XX:MetaspaceSize=256m -XX:MaxMetaspaceSize=512m
  -Xss512k

  # GC
  -XX:+UseG1GC
  -XX:MaxGCPauseMillis=200
  -XX:+AlwaysPreTouch

  # 可靠性
  -XX:+HeapDumpOnOutOfMemoryError
  -XX:HeapDumpPath=/var/log/app/heapdump.hprof
  -XX:+ExitOnOutOfMemoryError
  -XX:+ErrorFile=/var/log/app/hs_err_%p.log

  # 日志
  -Xlog:gc*:file=/var/log/app/gc.log:time,uptime,level,tags:filecount=5,filesize=50m

  # 性能
  -XX:+UseStringDeduplication     # G1 下去重字符串
  -XX:+OptimizeStringConcat       # 优化字符串拼接
"
java $JAVA_OPTS -jar app.jar
```

#### 最佳实践

1. **OOM 时自动 dump**：`-XX:+HeapDumpOnOutOfMemoryError` 是必须的
2. **`-XX:+ExitOnOutOfMemoryError`** 让进程快速退出，配合 systemd/k8s 重启
3. **GC 日志使用统一格式**（JDK9+ `-Xlog`），便于工具分析
4. **`UseStringDeduplication`** 在字符串密集型应用（JSON/XML 处理）中可节省 10-25% 堆

---

### JIT 编译

#### 概述

JVM 通过 JIT（Just-In-Time）编译将热点字节码编译为本地机器码。分层编译（Tiered Compilation）是默认策略。

#### 关键概念

| 层级 | 编译器 | 特点 |
|------|--------|------|
| 0 | 解释执行 | 启动快，无优化 |
| 1 | C1（Client） | 简单优化，编译快 |
| 2 | C1 + profiling | 收集 profile 数据 |
| 3 | C1 + profiling | |
| 4 | C2（Server） | 深度优化（内联、逃逸分析、循环优化） |
| - | GraalVM | Java 编写的 C2 替代，支持 AOT |

#### 实战示例

```bash
# 查看编译日志
java -XX:+PrintCompilation -jar app.jar

# 调整分层编译阈值
java -XX:CompileThreshold=1000 \          # 方法调用次数阈值
    -XX:CompileThresholdScaling=0.5 \     # 缩放因子（预热更快）
    -XX:TieredStopAtLevel=3 \            # 仅 C1，不启用 C2（更快启动）
    -jar app.jar

# GraalVM Native Image（AOT 编译）
native-image -H:+ReportExceptionStackTraces \
    -H:ReflectionConfigurationFiles=reflect-config.json \
    -jar app.jar
```

#### 最佳实践

1. **不要禁用分层编译**，`-XX:-TieredCompilation` 在 JDK 18+ 已移除
2. **预热期性能差是正常的**——关键路径可用 `-XX:CompileThresholdScaling=0.1` 加速
3. **GraalVM Native Image 适合 CLI 工具和 Serverless**，启动时间可从秒级降到毫秒级
4. **代码缓存不足会导致 C2 降级回解释执行**：`-XX:ReservedCodeCacheSize=256m`

---

### JVM 监控工具

#### 概述

| 工具 | 用途 | 使用方式 |
|------|------|---------|
| `jps` | 列出 Java 进程 | `jps -lv` |
| `jstat` | GC 统计 | `jstat -gcutil <pid> 1000` |
| `jinfo` | 查看/修改 JVM 参数 | `jinfo -flags <pid>` |
| `jmap` | 堆转储 | `jmap -dump:format=b,file=heap.hprof <pid>` |
| `jstack` | 线程转储 | `jstack <pid>` |
| `jcmd` | 统一诊断工具 | `jcmd <pid> GC.heap_info` |
| VisualVM | GUI 监控分析 | `jvisualvm` |
| JConsole | 基础 GUI 监控 | `jconsole` |
| JFR | 生产级飞行记录器 | `-XX:StartFlightRecording` |
| Arthas | 在线诊断（阿里） | `java -jar arthas-boot.jar` |

#### 实战示例

```bash
# jcmd 诊断（推荐，替代 jmap/jstack/jstat）
jcmd <pid> GC.heap_info
jcmd <pid> Thread.print
jcmd <pid> GC.run
jcmd <pid> VM.flags

# Java Flight Recorder（JDK 11+ 免费）
java -XX:StartFlightRecording=duration=60s,filename=recording.jfr \
    -jar app.jar

# Arthas 在线诊断
# 1. 附加到运行中进程
java -jar arthas-boot.jar
# 2. dashboard（实时面板）
# 3. thread -n 3（最忙的 3 个线程）
# 4. thread -b（检测死锁）
# 5. watch com.example.Service method params returnObj
# 6. profiler start（火焰图）
# 7. heapdump /tmp/heap.hprof
```

#### 最佳实践

1. **`jcmd` 是首选工具**，功能覆盖 jmap/jstack/jstat，且无需附加
2. **JFR 性能开销 < 1%**，可在生产环境长期运行
3. **Arthas 是生产环境在线诊断利器**，无需重启即可诊断
4. **定期采集线程转储**（3 次，间隔 5 秒），用于排查死锁和线程饥饿

---

### 类加载

#### 概述

JVM 类加载采用双亲委派模型（Parents Delegation Model），保证核心类不被篡改。自定义类加载器可实现热部署和模块隔离。

```
Bootstrap ClassLoader (rt.jar, --boot-class-path)
  └─ Extension ClassLoader (ext/*.jar, -Djava.ext.dirs)
      └─ Application ClassLoader (classpath, -cp/-classpath)
          └─ Custom ClassLoader (自定义)
```

#### 关键概念

| 概念 | 说明 |
|------|------|
| 双亲委派 | 加载类时先委托父加载器，父加载器无法加载时才自己加载 |
| `ClassLoader.loadClass()` | 实现双亲委派的核心方法 |
| `Class.forName()` | 显式加载并初始化类 |
| 热部署 | 自定义 ClassLoader 丢弃旧类、加载新类 |
| OSGi | 模块化框架，每个 bundle 有独立 ClassLoader |

#### 实战示例

```java
// 自定义类加载器（热部署基础）
public class HotSwapClassLoader extends ClassLoader {
    @Override
    protected Class<?> findClass(String name) throws ClassNotFoundException {
        byte[] classBytes = loadFromClassFile(name);
        return defineClass(name, classBytes, 0, classBytes.length);
    }
}

// 使用
HotSwapClassLoader loader = new HotSwapClassLoader();
Class<?> clazz = loader.loadClass("com.example.Service");
Object instance = clazz.getDeclaredConstructor().newInstance();
```

#### 最佳实践

1. **不要打破双亲委派**，除非有明确需求（热部署、模块隔离）
2. **`-Xlog:classload*=info`** 查看类加载详情，排查类冲突
3. **Metaspace OOM 通常由类加载泄漏引起**（如每次请求创建新 ClassLoader）
4. **OSGi 适用于插件架构**，但复杂度高，普通应用优先考虑 Java 9+ Module System

---

## Rust 编译调优

### cargo.toml 优化

#### 概述

Rust 的 release profile 通过 `cargo.toml` 配置。合理设置 LTO、codegen-units 和 strip 可显著优化二进制性能和体积。

#### 关键配置

```toml
[profile.release]
opt-level = 3          # 0/1/2/3/"z"(最小体积)/"s"(较小体积)
lto = "fat"          # false/"thin"/"fat"/true
codegen-units = 1     # 更好的优化（编译更慢）
panic = "abort"      # 去除 unwind 代码
strip = true          # 去除符号表
debug = 0            # 无调试信息
incremental = false   # 禁用增量编译（更一致的优化）

# 针对特定二进制的优化
[profile.release.package.my-bin]
opt-level = "z"
```

#### 实战示例

```bash
# 标准发布构建
cargo build --release

# 查看构建时间
cargo build --release --timings

# 最小化二进制
cargo build --release -Z build-std=std,panic_abort --target x86_64-unknown-linux-musl
```

#### 最佳实践

1. **`codegen-units = 1` + `lto = "fat"`** 是性能最优但编译最慢的组合
2. **`panic = "abort"`** 减少约 10-20% 二进制体积，但无法 catch panic
3. **`opt-level = "z"`** 用于嵌入式/WASM 场景
4. **CI 中使用 `sccache`** 加速重复编译

---

### cargo build 分析

#### 概述

Rust 生态提供了丰富的分析工具，用于构建时间分析、二进制体积优化和性能 profiling。

#### 关键工具

| 工具 | 安装 | 用途 |
|------|------|------|
| `cargo build --timings` | 内置 | 编译耗时分析（HTML 报告） |
| `cargo bloat` | `cargo install cargo-bloat` | 二进制体积分析（按 crate/函数） |
| `cargo flamegraph` | `cargo install cargo-flamegraph` | CPU 火焰图 |
| `cargo instruments` | 内置（macOS） | Instruments 集成 |
| `cargo profile` | `cargo install cargo-profile` | 编译时间 profiling |

#### 实战示例

```bash
# 编译耗时 HTML 报告
cargo build --timings --release
# 生成 target/cargo-timings/cargo-timing.html

# 二进制体积分析
cargo bloat --release --crates     # 按 crate 排列
cargo bloat --release --functions # 按函数排列

# CPU 火焰图
cargo flamegraph --release --bin myapp
# 生成 flamegraph.svg

# macOS Instruments
cargo instruments --release -t "Time Profiler" -- myapp
```

#### 最佳实践

1. **`--timings` 应在 CI 中定期检查**，防止编译时间回归
2. **`cargo bloat` 定位体积热点**，然后针对性优化（移除依赖、feature gate）
3. **火焰图定位性能热点**，先 profile 再优化

---

### Rust Lint

#### 概述

Clippy 是 Rust 的 lint 工具，比编译器警告更严格、更智能。`cargo-deny` 用于许可证和依赖安全审计。

#### 实战示例

```bash
# Clippy（严格模式）
cargo clippy -- -W clippy::all -W clippy::pedantic -D warnings

# cargo-deny（许可证 + 依赖审计）
cargo install cargo-deny
cargo deny check licenses bans sources
```

```toml
# clippy.toml 或 .clippy.toml
warn-on-all-wildcard-imports = true
disallowed-methods = [
    { path = "std::env::temp_dir", reason = "use tempdir crate instead" },
]
```

#### 最佳实践

1. **CI 中 `cargo clippy -- -D warnings`** 作为质量门禁
2. **逐步启用 `pedantic` 规则**，避免一次性大量警告
3. **`cargo deny` 在合规性要求高的项目中必须使用**

---

## 构建系统

### CMake

#### 概述

CMake 是 C/C++ 事实标准的跨平台构建系统。正确使用现代 CMake（target-based）可显著提升项目可维护性。

#### 关键模式

```cmake
# 最小 CMakeLists.txt（现代风格）
cmake_minimum_required(VERSION 3.20)
project(MyApp VERSION 1.0 LANGUAGES C CXX)

# 设置 C/C++ 标准
set(CMAKE_C_STANDARD 17)
set(CMAKE_CXX_STANDARD 20)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

# 库目标
add_library(mylib src/lib.cpp)
target_include_directories(mylib PUBLIC include)
target_compile_features(mylib PUBLIC cxx_std_20)
target_compile_options(mylib PRIVATE
    $<$<CONFIG:Release>:-O2 -flto=thin>
    $<$<CONFIG:Debug>:-Og -g3>
    -Wall -Wextra -Werror
)

# 可执行目标
add_executable(app src/main.cpp)
target_link_libraries(app PRIVATE mylib)

# 依赖管理
find_package(cpr REQUIRED)             # 系统已安装
# 或
include(FetchContent)
FetchContent_Declare(
    json
    GIT_REPOSITORY https://github.com/nlohmann/json
    GIT_TAG v3.11.3
)
FetchContent_MakeAvailable(json)
target_link_libraries(app PRIVATE nlohmann_json::nlohmann_json)

# 测试
enable_testing()
add_subdirectory(tests)

# 安装
install(TARGETS app RUNTIME DESTINATION bin)
install(TARGETS mylib LIBRARY DESTINATION lib)
```

#### 交叉编译工具链文件

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
```

#### 最佳实践

1. **使用 target-based API**（`target_compile_options`），避免全局 `add_compile_options`
2. **Generator expressions**（`$<CONFIG:Release>`）比 `if(CMAKE_BUILD_TYPE)` 更灵活
3. **`FetchContent` + `find_package` 优先**，避免 `add_subdirectory(external_lib)`
4. **`cmake --preset`** 管理多配置构建

---

### Make / Ninja

#### 概述

| 工具 | 特点 |
|------|------|
| Make | 经典，语法简单，并行 `-j$(nproc)` |
| Ninja | 专注速度，几乎被所有现代构建系统后端使用 |

#### 实战示例

```bash
# Make 并行构建
make -j$(nproc)

# CMake 使用 Ninja 后端（推荐）
cmake -G Ninja -B build -DCMAKE_BUILD_TYPE=Release
cmake --build build

# Make 自动依赖
%.o: %.c
	$(CC) $(CFLAGS) -MMD -MP -c $< -o $@
-include $(DEPS)
```

#### 最佳实践

1. **Ninja 是 CMake 的推荐后端**，构建速度比 Make 快 2-10x
2. **Make 的 `-j` 并行** 在大型项目中必须使用
3. **CCache + Ninja** 是快速迭代开发的标配

---

### Bazel

#### 概述

Bazel 是 Google 开源的构建系统，以增量构建、远程缓存和可复现构建著称。适合大型 monorepo。

#### 关键规则

```python
# WORKSPACE / MODULE.bazel
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

#### 实战示例

```bash
# 构建
bazel build //:app

# 测试
bazel test //...

# 远程缓存
bazel build //:app --remote_cache=grpc://cache.example.com:9092

# 查询依赖
bazel query "deps(//:app)" --output graph
```

#### 最佳实践

1. **Bazel 在 >1000 文件的项目中优势明显**，小项目用 CMake 即可
2. **远程缓存（remote cache）** 是 Bazel 最大的杀手锏，CI 速度可提升 5-10x
3. **`--config=clang-tidy`** 集成静态分析

---

### 构建缓存

#### 概述

| 工具 | 语言 | 特点 |
|------|------|------|
| ccache | C/C++ | 基于文件哈希，支持共享缓存目录 |
| sccache | C/C++/Rust | Rust 实现，支持 S3/GCS 远程缓存 |

#### 实战示例

```bash
# ccache
ccache gcc -O2 -c source.c
ccache -s  # 统计命中率

# 环境变量方式（无需修改构建命令）
export CC="ccache gcc"
export CXX="ccache g++"

# sccache（支持远程缓存）
export RUSTC_WRAPPER=sccache
export SCCACHE_CACHE_SIZE="10G"
export SCCACHE_S3_BUCKET=my-build-cache

# CMake 集成
cmake -DCMAKE_C_COMPILER_LAUNCHER=ccache \
    -DCMAKE_CXX_COMPILER_LAUNCHER=ccache ..
```

#### 最佳实践
1. **CI 中 ccache 是标配**，命中率通常 > 80%
2. **sccache 远程缓存** 在多机器构建时效果极佳
3. **设置合理的缓存上限**，避免磁盘占满

---

## 性能分析工具

### CPU 分析

#### 概述

| 工具 | 平台 | 采样方式 | 特点 |
|------|------|---------|------|
| perf | Linux | 硬件 PMU | 功能最全，`perf record/report` |
| Instruments | macOS | 系统调用 | Time Profiler / Allocations / Leaks |
| VTune | 跨平台 | 硬件 PMU | Intel 优化指南，热力图 |
| gprof | 跨平台 | 插桩 | 老旧但简单，需要重新编译 |
| FlameGraph | Linux | perf 脚本 | 可视化火焰图 |

#### 实战示例

```bash
# perf（Linux）
perf record -g --call-graph dwarf -F 99 ./app     # 采样
git diff --cached 2>/dev/null; perf report --stdio  # 文本报告
perf stat ./app                                     # 全局统计

# 火焰图
git clone https://github.com/brendangregg/FlameGraph
perf record -g -F 99 ./app
perf script | stackcollapse-perf.pl | flamegraph.pl > flame.svg

# VTune
gpu_vtune -collect hotspots ./app
vtune -report summary

# gprof
gcc -pg -O2 -o app source.c
./app
gprof app gmon.out > profile.txt
```

#### 最佳实践

1. **perf + FlameGraph 是 Linux 下 CPU profiling 的首选组合**
2. **采样频率 99Hz**（避免与系统定时器同步导致采样偏差）
3. **`--call-graph dwarf`** 比 `fp`（帧指针）更准确，但开销稍大
4. **VTune 提供最深入的微架构分析**（缓存未命中、分支预测、向量利用率）

---

### 内存分析

#### 概述

| 工具 | 用途 | 开销 |
|------|------|------|
| Valgrind (memcheck) | 内存错误检测 | 10-50x 慢 |
| Valgrind (massif) | 堆内存使用趋势 | 20x 慢 |
| Valgrind (callgrind) | 缓存/分支模拟分析 | 20-50x 慢 |
| AddressSanitizer | 内存错误检测 | 2x 慢 |
| Heaptrack | 堆分析（分配/泄漏） | 2x 慢 |

#### 实战示例

```bash
# Valgrind memcheck
valgrind --leak-check=full --show-leak-kinds=all \
    --track-origins=yes --verbose \
    ./app

# Valgrind massif（堆使用趋势）
valgrind --tool=massif ./app
ms_print massif.out.12345

# Valgrind callgrind + KCachegrind
callgrind ./app
kcachegrind callgrind.out.12345

# Heaptrack
heaptrack ./app
heaptrack_print heaptrack.app.12345.gz
```

#### 最佳实践

1. **开发阶段用 ASan**（快速、集成方便），发布前用 Valgrind 做深度检查
2. **massif 是排查内存泄漏和峰值内存的首选**
3. **callgrind + KCachegrind** 提供最直观的函数调用图和缓存分析
4. **Valgrind 不支持 SIMD 指令**，如果程序大量使用 AVX/SSE，用 ASan 代替

---

### 二进制分析

#### 概述

| 工具 | 用途 |
|------|------|
| `size` | 段大小（text/data/bss） |
| `nm` | 符号表 |
| `objdump -d` | 反汇编 |
| `readelf -a` | ELF 头/段/符号/重定位 |
| `strings` | 提取可读字符串 |
| `strip` | 去除符号表（减体积） |
| `ldd` | 共享库依赖 |
| `file` | 文件类型识别 |
| `binwalk` | 嵌入式二进制分析 |

#### 实战示例

```bash
# 段大小分析
size -A app
# text    data     bss     dec     hex filename
# 245760   4096    8192  258048   3f000 app

# 符号表
nm -C --size-sort app | tail -20      # 最大的符号
nm -D app                             # 动态符号

# 反汇编特定函数
objdump -d app --start-address=0x401000 --stop-address=0x401100 | less

# 共享库依赖
ldd app
ldd -r app  # 检查未解析符号

# Strip 优化体积
strip --strip-unneeded app            # 去除调试和非必要符号
strip --strip-all app                 # 去除所有符号

# 二进制体积优化全流程
# 1. 分析
cargo bloat --release --crates
# 2. 编译优化
CFLAGS="-Os -ffunction-sections -fdata-sections -flto"
# 3. 链接优化
LDFLAGS="-Wl,--gc-sections -Wl,-s"
# 4. Strip
strip --strip-unneeded app
```

#### 最佳实践

1. **`size -A` 是追踪二进制体积回归的最简单方法**
2. **`nm -C`（demangle）** 阅读性更好
3. **`strip --strip-unneeded`** 比 `--strip-all` 安全（保留动态符号）
4. **`ldd -r`** 检查共享库未解析符号，部署前必查
5. **`objdump -d` + `less`** 是快速阅读关键函数汇编的方式
6. **二进制体积优化顺序**：编译选项 → 链接裁剪 → strip → 最终检查
