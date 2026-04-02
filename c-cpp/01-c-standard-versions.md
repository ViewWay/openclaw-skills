# C 标准版本演进 (C Standard Versions)

> 来源：ISO/IEC 9899 各版本官方标准 + 编译器支持文档
> 覆盖范围：C89/C90 → C94 → C99 → C11 → C17 → C23 全版本对比

---

## C89/C90 (ANSI C / ISO/IEC 9899:1990)

第一个正式的 C 语言标准，奠定了 C 的基础。

### 特性
- K&R 风格与 ANSI 风格并存
- 基本类型：`auto`/`register`/`static`/`extern`/`signed`/`volatile`/`enum`/`void`
- `void*` 通用指针
- 函数原型（ANSI 新增，K&R 旧式仍允许）

### 限制
- 不允许 `//` 行注释
- 变量声明必须在块开头
- 无 `long long`、无 VLA、无 `stdint.h`
- 无 `inline`、无 `//` 注释、无 `snprintf`
- 无 `_Bool`/`bool`

---

## C94/C95 (ISO/IEC 9899:1990/Amd 1:1995)

C90 的修正案 1（Normative Amendment 1）。

### 新增特性
- **Digraphs**：`<:` → `[`、`:>` → `]`、`<%` → `{`、`%>` → `}`、`%:` → `#`、`%:%:` → `##`
- **Wide char 支持**：`wchar_t`、`<wchar.h>`、`<wctype.h>`
- 多字节字符函数增强

---

## C99 (ISO/IEC 9899:1999) — **使用最广泛**

C 语言最重要的更新之一，大部分现代 C 代码基于 C99。

### 核心新增特性

| 特性 | 说明 | 示例 |
|------|------|------|
| `//` 行注释 | 单行注释 | `// 这是注释` |
| `long long` | 64 位整数 | `long long x = 1LL << 60;` |
| `_Bool` / `bool` | 布尔类型 | `_Bool flag = 1;` / `#include <stdbool.h>` → `bool` |
| `_Complex` / `_Imaginary` | 复数支持 | `double complex z = 1.0 + 2.0i;` |
| VLA (变长数组) | 运行时确定大小（可选支持） | `int n = 10; int arr[n];` |
| 混合声明与代码 | 不再必须在块开头 | `for(int i=0; ...)` |
| 指定初始化器 | 按名初始化 | `struct s x = { .field = value };` |
| `restrict` 关键字 | 指针不重叠提示 | `void f(int *restrict a, const int *restrict b);` |
| Flexible array member | 结构体末尾柔性数组 | `struct s { int n; int data[]; };` |
| `snprintf` / `vsnprintf` | 安全格式化 | `snprintf(buf, sizeof(buf), "%d", x);` |
| `strtoll` / `strtoull` | long long 转换 | `long long v = strtoll(str, &end, 10);` |
| `stdint.h` / `inttypes.h` | 定宽整数类型 | `int8_t`/`uint64_t`/`PRId64` |
| `inline` | 内联函数 | `static inline int max(int a, int b);` |
| `__func__` | 预定义标识符 | `printf("%s", __func__);` |
| `_Pragma` | 运算符式 pragma | `_Pragma("GCC optimize (\"O2\")");` |
| 复合字面量 | 匿名对象 | `int *p = (int[]){1, 2, 3};` |
| `for` 循环初始化声明 | | `for(int i = 0; i < n; i++)` |
| `__VA_ARGS__` | 可变参数宏 | `#define LOG(fmt, ...) printf(fmt, __VA_ARGS__)` |
| `lldiv_t` / `llabs` / `lldiv` | long long 支持 | `llabs(x)` |
| 浮点 pragma | `FENV_ACCESS` / `FP_CONTRACT` | 精确控制浮点行为 |
| 空参数列表 | `int f(void)` 明确无参 | 区别于 K&R 的 `int f()` |

### 编译器支持
| 编译器 | 最低版本 | 备注 |
|--------|---------|------|
| GCC | 4.5+ | 默认 `-std=gnu99`（早期） |
| Clang | 3.0+ | 完整支持 |
| MSVC | 2013+ | 部分支持（VLA 不支持） |

---

## C11 (ISO/IEC 9899:2011)

### 核心新增特性

| 特性 | 说明 | 示例 |
|------|------|------|
| `_Static_assert` | 静态断言 | `_Static_assert(sizeof(int) >= 4, "int too small");` |
| `_Alignas` / `_Alignof` | 对齐控制 | `_Alignas(16) int x;` / `stdalign.h` 提供 `alignas`/`alignof` |
| `_Noreturn` | 不返回函数 | `_Noreturn void die(void) { exit(1); }` / `stdnoreturn.h` → `noreturn` |
| 匿名结构体和联合体 | 无名成员直接访问 | `struct { union { struct { int x, y; }; }; } s; s.x = 1;` |
| `threads.h` | C11 线程库（可选） | `thrd_t`、`mtx_t`、`cnd_t` — **大部分编译器不完整实现** |
| `stdatomic.h` | 原子操作 | `atomic_int counter; atomic_fetch_add(&counter, 1);` |
| `_Generic` | 泛型选择 | `#define cbrt(x) _Generic((x), double: cbrt, default: cbrtf)(x)` |
| `gets_s` / `strcpy_s` 等 | Bounds-checking 函数（Annex K，可选） | 大多数实现不提供 |
| `uchar.h` | UTF-16/UTF-32 字符类型 | `char16_t`、`char32_t` |
| `quick_exit` / `at_quick_exit` | 快速退出 | `quick_exit(EXIT_FAILURE);` |
| 匿名 struct/union | | 嵌套匿名访问 |

### 编译器支持
| 编译器 | 最低版本 |
|--------|---------|
| GCC | 4.9+ |
| Clang | 3.2+ |
| MSVC | 2015+（部分） |

---

## C17/C18 (ISO/IEC 9899:2018)

技术修正版本，**无新语言特性**。

### 变更内容
- 修复 C11 中的缺陷报告（Defect Reports）
- 澄清标准中模糊的措辞
- `aligned_alloc` 返回值的对齐要求更明确

### 编译器支持
| 编译器 | 最低版本 |
|--------|---------|
| GCC | 8+ |
| Clang | 7+ |

---

## C23 (ISO/IEC 9899:2024) — **最新标准**

C 语言现代化最大的一次更新，大量吸收 C++ 特性。

### 核心新增特性

| 特性 | 说明 | 示例 |
|------|------|------|
| `auto` 类型推断 | 类似 C++ auto | `auto x = 42;` / `auto p = malloc(100);` |
| `typeof` / `typeof_unqual` | 类型推导 | `typeof(x) y = x;` |
| `nullptr` | 替代 `NULL` | `int *p = nullptr;` |
| `constexpr` | 编译期常量 | `constexpr int SIZE = 1024;` |
| `true` / `false` / `bool` | 成为关键字，不再需要 `stdbool.h` | `bool flag = true;` |
| `[[attribute]]` | 属性语法 | `[[nodiscard]] int f(void);` / `[[maybe_unused]]` |
| `#embed` 指令 | 嵌入二进制文件 | `const unsigned char data[] = { #embed "logo.png" };` |
| 空初始化器 | 所有类型可用 `= {}` | `int x = {};` / `struct S s = {};` / `int *p = {};` |
| 二进制字面量 | `0b` 前缀 | `int mask = 0b10101100;` |
| 数字分隔符 | `'` 分隔 | `int million = 1'000'000;` |
| `_BitInt(N)` | 任意宽度整数 | `_BitInt(128) x;` |
| `static_assert` | 不需要下划线前缀 | `static_assert(sizeof(int) == 4);` |
| `alignof` | 不需要下划线前缀 | `size_t a = alignof(int);` |
| `constexpr` / `consteval` / `constinit` | 编译期求值 | |
| 未限定字符类型 | `uchar.h` 增强 | `char8_t` 等 |
| `nullptr_t` | nullptr 的类型 | |
| 去除 K&R 函数定义 | 不再允许旧式定义 | |
| 去除 `gets()` | 永久移除 | 用 `fgets` 替代 |
| `#warning` 标准化 | 标准警告指令 | `#warning "deprecated"` |

### 编译器支持
| 编译器 | 最低版本 |
|--------|---------|
| GCC | 14+（大部分特性） |
| Clang | 18+（大部分特性） |

---

## 版本对比总表

| 版本 | 年份 | 关键新增 | 编译器支持 | 推荐度 |
|------|------|----------|-----------|--------|
| **C89/C90** | 1990 | 首个标准 | 全部 | ⭐ 遗留维护 |
| **C94/C95** | 1995 | Digraphs、宽字符 | 全部 | ⭐ 与 C90 同 |
| **C99** | 1999 | `//`、`long long`、`stdint.h`、VLA、`inline`、指定初始化器 | GCC 4.5+, Clang 3.0+ | ⭐⭐⭐⭐⭐ **最广泛** |
| **C11** | 2011 | `_Static_assert`、`_Generic`、`atomic`、匿名 struct | GCC 4.9+, Clang 3.2+ | ⭐⭐⭐⭐ |
| **C17** | 2018 | 技术修正，无新特性 | GCC 8+, Clang 7+ | ⭐⭐⭐⭐ |
| **C23** | 2024 | `auto`、`nullptr`、`constexpr`、`#embed`、`bool` 关键字 | GCC 14+, Clang 18+ | ⭐⭐⭐⭐ 新项目 |

---

## 选择建议

| 场景 | 推荐标准 | 理由 |
|------|----------|------|
| **嵌入式** | C99 | 最广泛工具链支持；避免 C11 的 threads.h（实现不全） |
| **嵌入式（需断言）** | C11 | `_Static_assert`、`atomic` 值得拥有 |
| **系统编程** | C11 或 C17 | 现代特性 + 稳定 |
| **新项目** | C23 | 如果编译器支持，享受现代化语法 |
| **跨平台/最大兼容** | C99 | 最安全的跨编译器选择 |
| **遗留代码维护** | C89/C90 | 保持与旧代码一致 |
| **安全关键** | C11 + Annex K | bounds-checking 函数（如果编译器支持） |

---

## 编译器标准选项速查

```bash
# GCC / Clang
gcc -std=c90   # C89/C90
gcc -std=c99   # C99
gcc -std=c11   # C11
gcc -std=c17   # C17/C18
gcc -std=c23   # C23 (GCC 14+, Clang 18+)
gcc -std=c2x   # C23 预览版

# GNU 扩展版本（推荐开发时使用，但发布时用严格模式）
gcc -std=gnu99
gcc -std=gnu11
gcc -std=gnu17
gcc -std=gnu23

# MSVC
/clc          # 严格 C 模式
/std:c11
/std:c17
```
