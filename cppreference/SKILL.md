# C/C++ 标准库 API 速查 / C/C++ Standard Library Quick Reference

> 中英双语速查手册，涵盖 C 标准库与 C++ 标准库核心 API。
> Bilingual quick reference covering C and C++ standard library core APIs.
>
> 参考：cppreference.com / cppreference.cn / c.biancheng.net

---

# 目录 / Table of Contents

- [C 标准库](#c-标准库-c-standard-library)
  - [<string.h> 字符串操作](#stringh--cstring-字符串操作--string-operations)
  - [<stdlib.h> 通用工具](#stdlibh--cstdlib-通用工具--general-utilities)
  - [<stdio.h> 输入输出](#stdioh--cstdio-输入输出--inputoutput)
  - [<math.h> 数学](#mathh--cmath-数学--mathematics)
  - [<ctype.h> 字符分类](#ctypeh--cctype-字符分类--character-classification)
  - [<time.h> 时间](#timeh--ctime-时间--time)
  - [<signal.h> 信号](#signalh--csignal-信号--signals)
  - [<assert.h> 断言](#asserth--cassert-断言--assertions)
  - [<errno.h> 错误码](#errnoh--cerrno-错误码--error-codes)
  - [<stdint.h> 定宽整数](#stdinth--cstdint-定宽整数--fixed-width-integers)
  - [<stddef.h> 标准定义](#stddefh--cstddef-标准定义--standard-definitions)
  - [<stdbool.h> 布尔](#stdboolh--cstdbool-布尔--boolean)
  - [<inttypes.h> 格式化宏](#inttypesh--cinttypes-格式化宏--formatting-macros)
  - [<limits.h> 极限值](#limitsh--climits-极限值--limits)
  - [<float.h> 浮点极限](#floath--cfloat-浮点极限--floating-point-limits)
- [C++ 标准库](#c-标准库-c-standard-library-1)
  - [<string> 字符串](#string-字符串--strings)
  - [<vector> 动态数组](#vector-动态数组--dynamic-arrays)
  - [<list>/<forward_list> 链表](#list--forward_list-链表--linked-lists)
  - [<deque> 双端队列](#deque-双端队列--double-ended-queue)
  - [<map>/<unordered_map> 关联容器](#map--unordered_map-关联容器--associative-containers)
  - [<set>/<unordered_set>](#set--unordered_set-集合--sets)
  - [<array>/<span>](#array--span-固定数组与视图--fixed-array-and-view)
  - [<algorithm> 算法](#algorithm-算法--algorithms)
  - [<numeric> 数值算法](#numeric-数值算法--numeric-algorithms)
  - [<memory> 内存](#memory-内存--memory)
  - [<functional> 函数对象](#functional-函数对象--function-objects)
  - [<thread> 线程](#thread-线程--threading)
  - [<mutex> 互斥量](#mutex-互斥量--mutexes)
  - [<condition_variable> 条件变量](#condition_variable-条件变量--condition-variables)
  - [<future> 异步](#future-异步--futures)
  - [<atomic> 原子操作](#atomic-原子操作--atomic-operations)
  - [<chrono> 时间](#chrono-时间--time)
  - [<optional>/<variant>/<any>](#optional--variant--any-可选类型--optional-types)
  - [<tuple> 元组](#tuple-元组--tuples)
  - [<filesystem> 文件系统](#filesystem-文件系统-c17--filesystem-c17)
  - [<format> 格式化](#format-格式化-c20--format-c20)
  - [<regex> 正则表达式](#regex-正则表达式--regular-expressions)
  - [<iostream>/<fstream>/<sstream>](#iostream--fstream--sstream-流--io-streams)
  - [<stdexcept> 异常](#stdexcept-异常--exceptions)
  - [<type_traits> 类型特性](#type_traits-类型特性--type-traits)
  - [<utility>](#utility-通用工具--utility)
  - [<bitset>](#bitset-位集--bitsets)
  - [<valarray> 数值数组](#valarray-数值数组--numeric-arrays)
- [开发经验与调优](#开发经验与调优-development-tips)

---

# C 标准库 (C Standard Library / C 标准库)

---

## <string.h> / <cstring> 字符串操作 / String Operations

### memcpy / 内存拷贝 / Memory Copy

| 项目 / Item | 内容 |
|---|---|
| **头文件 / Header** | `<string.h>` / `<cstring>` |
| **原型 / Prototype** | `void *memcpy(void *dest, const void *src, size_t n);` |
| **说明 / Description** | 从 `src` 复制 `n` 字节到 `dest`。源和目标内存区域不可重叠。Copies `n` bytes from `src` to `dest`. Regions must not overlap. |
| **参数 / Parameters** | `dest` - 目标指针 / destination pointer; `src` - 源指针 / source pointer; `n` - 字节数 / byte count |
| **返回值 / Return** | `dest` 指针 |
| **复杂度 / Complexity** | O(n) |
| **安全 / Safety** | ⚠️ 内存重叠时行为未定义，重叠区域用 `memmove` |

```c
char src[] = "hello";
char dst[6];
memcpy(dst, src, 6); // dst == "hello"
```

### memmove / 安全内存拷贝 / Safe Memory Copy

| 项目 / Item | 内容 |
|---|---|
| **头文件 / Header** | `<string.h>` / `<cstring>` |
| **原型 / Prototype** | `void *memmove(void *dest, const void *src, size_t n);` |
| **说明 / Description** | 从 `src` 复制 `n` 字节到 `dest`，源和目标可以重叠。Copies `n` bytes; handles overlapping regions. |
| **参数 / Parameters** | 同 memcpy |
| **返回值 / Return** | `dest` |
| **复杂度 / Complexity** | O(n) |

```c
char buf[] = "abcdef";
memmove(buf + 2, buf, 4); // buf == "ababcd"
```

### memset / 内存填充 / Memory Set

| 项目 / Item | 内容 |
|---|---|
| **头文件 / Header** | `<string.h>` / `<cstring>` |
| **原型 / Prototype** | `void *memset(void *dest, int c, size_t n);` |
| **说明 / Description** | 将 `dest` 的前 `n` 字节设为 `c`（转为 unsigned char）。Sets first `n` bytes of `dest` to `c`. |
| **参数 / Parameters** | `dest` - 目标; `c` - 填充值; `n` - 字节数 |
| **返回值 / Return** | `dest` |
| **复杂度 / Complexity** | O(n) |

```c
int arr[10];
memset(arr, 0, sizeof(arr)); // 全部置零 / zero-fill
```

### memcmp / 内存比较 / Memory Compare

| 项目 / Item | 内容 |
|---|---|
| **头文件 / Header** | `<string.h>` / `<cstring>` |
| **原型 / Prototype** | `int memcmp(const void *s1, const void *s2, size_t n);` |
| **说明 / Description** | 比较前 `n` 字节。Compares first `n` bytes. 返回 <0/0/>0 表示 s1 小于/等于/大于 s2。 |
| **返回值 / Return** | 负数/0/正数 |
| **复杂度 / Complexity** | O(n) |

### memchr / 内存搜索 / Memory Character Search

| 项目 / Item | 内容 |
|---|---|
| **头文件 / Header** | `<string.h>` / `<cstring>` |
| **原型 / Prototype** | `void *memchr(const void *s, int c, size_t n);` |
| **说明 / Description** | 在前 `n` 字节中查找 `c`。Searches for `c` in first `n` bytes. |
| **返回值 / Return** | 找到的指针，未找到返回 NULL |

### strcpy / 字符串拷贝 / String Copy

| 项目 / Item | 内容 |
|---|---|
| **头文件 / Header** | `<string.h>` / `<cstring>` |
| **原型 / Prototype** | `char *strcpy(char *dest, const char *src);` |
| **说明 / Description** | 将 `src`（含 `\0`）复制到 `dest`。Copies `src` including null terminator to `dest`. |
| **安全 / Safety** | ⚠️ 缓冲区溢出风险，推荐 `strncpy` 或 `snprintf` |

```c
char dst[20];
strcpy(dst, "hello");
```

### strncpy / 有限字符串拷贝 / Bounded String Copy

| 项目 / Item | 内容 |
|---|---|
| **头文件 / Header** | `<string.h>` / `<cstring>` |
| **原型 / Prototype** | `char *strncpy(char *dest, const char *src, size_t n);` |
| **说明 / Description** | 最多复制 `n` 字节。如果 `strlen(src) < n`，剩余字节填充 `\0`。Copies up to `n` bytes. |
| **安全 / Safety** | ⚠️ 不保证以 `\0` 结尾（当 `strlen(src) >= n` 时） |

### strlcpy / 安全字符串拷贝 / Safe String Copy (BSD)

| 项目 / Item | 内容 |
|---|---|
| **头文件 / Header** | `<string.h>` (BSD / macOS) |
| **原型 / Prototype** | `size_t strlcpy(char *dest, const char *src, size_t size);` |
| **说明 / Description** | 最多复制 `size-1` 字节，始终保证 `\0` 结尾。返回 `strlen(src)`。Guaranteed null-termination. |
| **安全 / Safety** | ✅ 安全，返回完整源串长度便于截断检测 |

```c
char buf[8];
strlcpy(buf, "hello world", sizeof(buf)); // buf == "hello w", 返回 11
```

### strcat / 字符串连接 / String Concatenation

| 项目 / Item | 内容 |
|---|---|
| **头文件 / Header** | `<string.h>` / `<cstring>` |
| **原型 / Prototype** | `char *strcat(char *dest, const char *src);` |
| **说明 / Description** | 将 `src` 追加到 `dest` 末尾。Appends `src` to `dest`. |
| **安全 / Safety** | ⚠️ 缓冲区溢出风险，推荐 `strncat` 或 `snprintf` |

### strncat / 有限字符串连接 / Bounded String Concatenation

| 项目 / Item | 内容 |
|---|---|
| **头文件 / Header** | `<string.h>` / `<cstring>` |
| **原型 / Prototype** | `char *strncat(char *dest, const char *src, size_t n);` |
| **说明 / Description** | 最多追加 `n` 字节，始终以 `\0` 结尾。Appends up to `n` bytes, always null-terminated. |

### strlcat / 安全字符串连接 / Safe String Concatenation (BSD)

| 项目 / Item | 内容 |
|---|---|
| **头文件 / Header** | `<string.h>` (BSD / macOS) |
| **原型 / Prototype** | `size_t strlcat(char *dest, const char *src, size_t size);` |
| **说明 / Description** | 安全连接，返回尝试创建的字符串总长度。Safe concatenation, returns total intended length. |

### strlen / 字符串长度 / String Length

| 项目 / Item | 内容 |
|---|---|
| **头文件 / Header** | `<string.h>` / `<cstring>` |
| **原型 / Prototype** | `size_t strlen(const char *s);` |
| **说明 / Description** | 返回字符串长度（不含 `\0`）。Returns length excluding null terminator. |
| **返回值 / Return** | 字符串长度 / string length |
| **复杂度 / Complexity** | O(n) |

### strcmp / 字符串比较 / String Compare

| 项目 / Item | 内容 |
|---|---|
| **头文件 / Header** | `<string.h>` / `<cstring>` |
| **原型 / Prototype** | `int strcmp(const char *s1, const char *s2);` |
| **说明 / Description** | 按字典序比较。返回 <0/0/>0。Lexicographic comparison. |
| **复杂度 / Complexity** | O(n) |

### strncmp / 有限字符串比较 / Bounded String Compare

| 项目 / Item | 内容 |
|---|---|
| **原型 / Prototype** | `int strncmp(const char *s1, const char *s2, size_t n);` |
| **说明 / Description** | 最多比较前 `n` 个字符。Compares up to `n` characters. |

### strchr / 字符查找（正向）/ Find Character (Forward)

| 项目 / Item | 内容 |
|---|---|
| **原型 / Prototype** | `char *strchr(const char *s, int c);` |
| **说明 / Description** | 查找 `c` 第一次出现的位置。Finds first occurrence of `c`. |

### strrchr / 字符查找（反向）/ Find Character (Reverse)

| 项目 / Item | 内容 |
|---|---|
| **原型 / Prototype** | `char *strrchr(const char *s, int c);` |
| **说明 / Description** | 查找 `c` 最后一次出现的位置。Finds last occurrence of `c`. |

### strstr / 子串查找 / Substring Search

| 项目 / Item | 内容 |
|---|---|
| **原型 / Prototype** | `char *strstr(const char *haystack, const char *needle);` |
| **说明 / Description** | 查找 `needle` 在 `haystack` 中首次出现的位置。Finds first occurrence of substring. |
| **返回值 / Return** | 匹配位置的指针，未找到返回 NULL |
| **复杂度 / Complexity** | O(n*m) 最坏情况 |

```c
const char *s = "hello world";
char *p = strstr(s, "world"); // p 指向 "world"
```

### strtok / 字符串分割 / String Tokenize

| 项目 / Item | 内容 |
|---|---|
| **原型 / Prototype** | `char *strtok(char *str, const char *delim);` |
| **说明 / Description** | 按 `delim` 分割字符串。首次调用传字符串，后续传 NULL。Tokenizes string by delimiters. |
| **安全 / Safety** | ⚠️ 修改原字符串，非线程安全。多线程用 `strtok_r` |

```c
char s[] = "one,two,three";
char *tok = strtok(s, ",");
while (tok) {
    printf("%s\n", tok);
    tok = strtok(NULL, ",");
}
```

### strerror / 错误码转字符串 / Error Code to String

| 项目 / Item | 内容 |
|---|---|
| **原型 / Prototype** | `char *strerror(int errnum);` |
| **说明 / Description** | 返回描述错误码的字符串。Returns string describing error code. |
| **安全 / Safety** | ⚠️ 非线程安全，多线程用 `strerror_r` |

---

## <stdlib.h> / <cstdlib> 通用工具 / General Utilities

### malloc / 内存分配 / Memory Allocation

| 项目 / Item | 内容 |
|---|---|
| **头文件 / Header** | `<stdlib.h>` / `<cstdlib>` |
| **原型 / Prototype** | `void *malloc(size_t size);` |
| **说明 / Description** | 分配 `size` 字节未初始化内存。Allocates `size` bytes of uninitialized memory. |
| **返回值 / Return** | 指针或 NULL（失败时） |
| **复杂度 / Complexity** | O(1) ~ O(n) 取决于实现 |
| **安全 / Safety** | ⚠️ 分配失败返回 NULL，需检查；不初始化内存 |

```c
int *p = malloc(100 * sizeof(int));
if (!p) { /* 处理错误 */ }
free(p); p = NULL;
```

### calloc / 分配并清零 / Allocate and Zero

| 项目 / Item | 内容 |
|---|---|
| **原型 / Prototype** | `void *calloc(size_t nmemb, size_t size);` |
| **说明 / Description** | 分配 `nmemb * size` 字节并初始化为零。Allocates and zero-initializes. |

```c
int *arr = calloc(100, sizeof(int)); // 100 个 int，全零
```

### realloc / 重新分配 / Reallocate

| 项目 / Item | 内容 |
|---|---|
| **原型 / Prototype** | `void *realloc(void *ptr, size_t size);` |
| **说明 / Description** | 调整已分配内存块大小。保留原数据（min(old, new) 字节）。Resizes memory block, preserves data. |
| **安全 / Safety** | ⚠️ 失败返回 NULL 但原指针仍有效，应使用临时变量接收 |

```c
int *tmp = realloc(p, 200 * sizeof(int));
if (tmp) p = tmp;
else { /* 保留 p，处理错误 */ }
```

### free / 释放内存 / Free Memory

| 项目 / Item | 内容 |
|---|---|
| **原型 / Prototype** | `void free(void *ptr);` |
| **说明 / Description** | 释放 `malloc`/`calloc`/`realloc` 分配的内存。Frees allocated memory. |
| **安全 / Safety** | ⚠️ 不可释放栈内存、已释放内存（double free）、NULL 可安全释放 |

### atoi / 字符串转整数 / String to Integer

| 项目 / Item | 内容 |
|---|---|
| **原型 / Prototype** | `int atoi(const char *str);` |
| **说明 / Description** | 将字符串转为 int。转换失败行为未定义。Converts string to int. |
| **安全 / Safety** | ⚠️ 无错误检测，推荐 `strtol` |

### strtol / 字符串转长整数 / String to Long

| 项目 / Item | 内容 |
|---|---|
| **原型 / Prototype** | `long strtol(const char *str, char **endptr, int base);` |
| **说明 / Description** | 将字符串转为 long，支持指定进制和错误检测。Converts with base and error detection. |
| **参数 / Parameters** | `str` - 字符串; `endptr` - 存储第一个未转换字符的位置（可 NULL）; `base` - 进制（0=自动检测, 8/10/16） |
| **返回值 / Return** | 转换结果，溢出时为 LONG_MAX/MIN 并设 errno |

```c
char *end;
long val = strtol("123abc", &end, 10); // val=123, end 指向 "abc"
```

### strtoul / 字符串转无符号长整数 / String to Unsigned Long

| 项目 / Item | 内容 |
|---|---|
| **原型 / Prototype** | `unsigned long strtoul(const char *str, char **endptr, int base);` |

### strtod / 字符串转双精度浮点 / String to Double

| 项目 / Item | 内容 |
|---|---|
| **原型 / Prototype** | `double strtod(const char *str, char **endptr);` |

### strtof / 字符串转浮点 / String to Float

| 项目 / Item | 内容 |
|---|---|
| **原型 / Prototype** | `float strtof(const char *str, char **endptr);` |

### rand / 随机数 / Random Number

| 项目 / Item | 内容 |
|---|---|
| **原型 / Prototype** | `int rand(void);` |
| **说明 / Description** | 返回 [0, RAND_MAX] 伪随机整数。Returns pseudo-random integer in [0, RAND_MAX]. |
| **安全 / Safety** | ⚠️ 质量低，不适用于密码学。现代替代：arc4random 或 <random> |

### srand / 设置随机种子 / Seed Random Generator

| 项目 / Item | 内容 |
|---|---|
| **原型 / Prototype** | `void srand(unsigned int seed);` |
| **说明 / Description** | 设置 rand 的种子。通常 `srand(time(NULL))`。 |

### arc4random / 安全随机数 / Secure Random (BSD)

| 项目 / Item | 内容 |
|---|---|
| **头文件 / Header** | `<stdlib.h>` (BSD / macOS) |
| **原型 / Prototype** | `uint32_t arc4random(void);` |
| **说明 / Description** | 返回 [0, 2^32-1] 伪随机数，无需手动播种。Returns pseudo-random uint32, auto-seeded. |

### qsort / 快速排序 / Quick Sort

| 项目 / Item | 内容 |
|---|---|
| **原型 / Prototype** | `void qsort(void *base, size_t nmemb, size_t size, int (*compar)(const void *, const void *));` |
| **说明 / Description** | 对数组排序。Sorts array using comparison function. |
| **参数 / Parameters** | `base` - 数组起始; `nmemb` - 元素数; `size` - 元素大小; `compar` - 比较函数（返回 <0/0/>0） |
| **复杂度 / Complexity** | O(n log n) 平均 |

```c
int cmp(const void *a, const void *b) {
    return (*(int*)a - *(int*)b);
}
int arr[] = {3, 1, 4, 1, 5};
qsort(arr, 5, sizeof(int), cmp);
```

### bsearch / 二分搜索 / Binary Search

| 项目 / Item | 内容 |
|---|---|
| **原型 / Prototype** | `void *bsearch(const void *key, const void *base, size_t nmemb, size_t size, int (*compar)(const void *, const void *));` |
| **说明 / Description** | 在已排序数组中二分查找。Binary search in sorted array. |
| **返回值 / Return** | 匹配元素的指针，未找到返回 NULL |
| **复杂度 / Complexity** | O(log n) |
| **前提 / Requirement** | 数组必须已排序 / Array must be sorted |

### abs / labs / llabs / 绝对值 / Absolute Value

| 项目 / Item | 内容 |
|---|---|
| **原型 / Prototype** | `int abs(int j);` / `long labs(long j);` / `long long llabs(long long j);` |
| **说明 / Description** | 返回绝对值。Returns absolute value. |
| **安全 / Safety** | ⚠️ `abs(INT_MIN)` 结果未定义（溢出） |

### div / ldiv / 整数除法 / Integer Division

| 项目 / Item | 内容 |
|---|---|
| **原型 / Prototype** | `div_t div(int numer, int denom);` / `ldiv_t ldiv(long numer, long denom);` |
| **说明 / Description** | 同时计算商和余数。Computes quotient and remainder. |
| **返回值 / Return** | `div_t { quot, rem }` 结构体 |

### exit / 程序退出 / Program Exit

| 项目 / Item | 内容 |
|---|---|
| **原型 / Prototype** | `void exit(int status);` / `_Exit(int status);` |
| **说明 / Description** | 正常终止程序，调用 atexit 注册的函数并刷新缓冲区。`_Exit` 不调用清理函数。 |

### abort / 异常终止 / Abnormal Termination

| 项目 / Item | 内容 |
|---|---|
| **原型 / Prototype** | `void abort(void);` |
| **说明 / Description** | 异常终止程序，生成 core dump。Abnormal termination, raises SIGABRT. |

### atexit / 退出注册 / Exit Registration

| 项目 / Item | 内容 |
|---|---|
| **原型 / Prototype** | `int atexit(void (*func)(void));` |
| **说明 / Description** | 注册程序正常退出时调用的函数。Registers function to be called on normal exit. |

### system / 执行系统命令 / Execute System Command

| 项目 / Item | 内容 |
|---|---|
| **原型 / Prototype** | `int system(const char *command);` |
| **说明 / Description** | 执行 shell 命令。Executes shell command. |
| **安全 / Safety** | ⚠️ 命令注入风险，避免拼接用户输入 |

### getenv / 获取环境变量 / Get Environment Variable

| 项目 / Item | 内容 |
|---|---|
| **原型 / Prototype** | `char *getenv(const char *name);` |
| **说明 / Description** | 获取环境变量值。返回的指针不应被修改或释放。Returns environment variable value. |
| **返回值 / Return** | 环境变量值的指针，不存在返回 NULL |

---

## <stdio.h> / <cstdio> 输入输出 / Input/Output

### printf / 格式化输出 / Formatted Output

| 项目 / Item | 内容 |
|---|---|
| **头文件 / Header** | `<stdio.h>` / `<cstdio>` |
| **原型 / Prototype** | `int printf(const char *format, ...);` |
| **说明 / Description** | 格式化输出到 stdout。Writes formatted output to stdout. |
| **返回值 / Return** | 输出的字符数，出错返回负数 |
| **常用格式 / Common Formats** | `%d` int, `%ld` long, `%lld` long long, `%u` unsigned, `%f` double, `%.2f` 保留2位, `%e` 科学计数, `%x` 十六进制, `%o` 八进制, `%s` 字符串, `%c` 字符, `%p` 指针, `%%` 百分号, `%zu` size_t |

```c
printf("Name: %s, Age: %d, PI: %.2f\n", "Alice", 30, 3.14159);
```

### fprintf / 文件格式化输出 / File Formatted Output

| 项目 / Item | 内容 |
|---|---|
| **原型 / Prototype** | `int fprintf(FILE *stream, const char *format, ...);` |
| **说明 / Description** | 格式化输出到文件流。Writes formatted output to stream. |

### sprintf / 字符串格式化 / String Format

| 项目 / Item | 内容 |
|---|---|
| **原型 / Prototype** | `int sprintf(char *str, const char *format, ...);` |
| **说明 / Description** | 格式化输出到字符串。Writes formatted output to string buffer. |
| **安全 / Safety** | ⚠️ 缓冲区溢出风险，推荐 `snprintf` |

### snprintf / 安全字符串格式化 / Safe String Format

| 项目 / Item | 内容 |
|---|---|
| **原型 / Prototype** | `int snprintf(char *str, size_t size, const char *format, ...);` |
| **说明 / Description** | 最多写入 `size-1` 字节，始终 `\0` 结尾。返回需要的总长度。Writes up to `size-1` bytes, always null-terminated. |
| **安全 / Safety** | ✅ 安全 |

```c
char buf[32];
int needed = snprintf(buf, sizeof(buf), "value=%d", 42);
if (needed >= sizeof(buf)) { /* 被截断 */ }
```

### scanf / 格式化输入 / Formatted Input

| 项目 / Item | 内容 |
|---|---|
| **原型 / Prototype** | `int scanf(const char *format, ...);` |
| **说明 / Description** | 从 stdin 格式化读取。Reads formatted input from stdin. |
| **返回值 / Return** | 成功匹配的项数 |
| **安全 / Safety** | ⚠️ 缓冲区溢出风险，建议用宽度限定符 `%99s` |

```c
int n;
scanf("%d", &n);
char name[100];
scanf("%99s", name); // 限制宽度
```

### fopen / 打开文件 / Open File

| 项目 / Item | 内容 |
|---|---|
| **原型 / Prototype** | `FILE *fopen(const char *pathname, const char *mode);` |
| **说明 / Description** | 打开文件。Opens a file. |
| **模式 / Modes** | `"r"` 读, `"w"` 写（截断）, `"a"` 追加, `"r+"` 读+写, `"w+"` 写+读（截断）, `"a+"` 追加+读, `"rb"`/`"wb"` 二进制模式 |
| **返回值 / Return** | FILE 指针，失败返回 NULL |

```c
FILE *f = fopen("data.txt", "r");
if (!f) { perror("fopen"); return 1; }
```

### fclose / 关闭文件 / Close File

| 项目 / Item | 内容 |
|---|---|
| **原型 / Prototype** | `int fclose(FILE *stream);` |
| **说明 / Description** | 关闭文件流，刷新缓冲区。Closes file stream, flushes buffers. |

### fread / 读取数据块 / Read Block

| 项目 / Item | 内容 |
|---|---|
| **原型 / Prototype** | `size_t fread(void *ptr, size_t size, size_t nmemb, FILE *stream);` |
| **说明 / Description** | 从文件读取 `nmemb` 个大小为 `size` 的元素。Reads array of elements from file. |
| **返回值 / Return** | 实际读取的元素数 |

```c
int arr[10];
size_t n = fread(arr, sizeof(int), 10, f);
```

### fwrite / 写入数据块 / Write Block

| 项目 / Item | 内容 |
|---|---|
| **原型 / Prototype** | `size_t fwrite(const void *ptr, size_t size, size_t nmemb, FILE *stream);` |
| **说明 / Description** | 向文件写入 `nmemb` 个大小为 `size` 的元素。Writes array of elements to file. |

### fseek / 文件定位 / File Seek

| 项目 / Item | 内容 |
|---|---|
| **原型 / Prototype** | `int fseek(FILE *stream, long offset, int whence);` |
| **说明 / Description** | 设置文件位置。Sets file position. |
| **whence** | `SEEK_SET` 文件开头, `SEEK_CUR` 当前位置, `SEEK_END` 文件末尾 |

### ftell / 获取文件位置 / Get File Position

| 项目 / Item | 内容 |
|---|---|
| **原型 / Prototype** | `long ftell(FILE *stream);` |
| **说明 / Description** | 返回当前文件位置。Returns current file position. |

### rewind / 重置文件位置 / Rewind File

| 项目 / Item | 内容 |
|---|---|
| **原型 / Prototype** | `void rewind(FILE *stream);` |
| **说明 / Description** | 等价于 `fseek(stream, 0, SEEK_SET); clearerr(stream);` |

### fflush / 刷新缓冲区 / Flush Buffer

| 项目 / Item | 内容 |
|---|---|
| **原型 / Prototype** | `int fflush(FILE *stream);` |
| **说明 / Description** | 刷新输出缓冲区。Flushing stdout for output. `fflush(NULL)` 刷新所有流。 |

### fgets / 读取一行 / Read Line

| 项目 / Item | 内容 |
|---|---|
| **原型 / Prototype** | `char *fgets(char *s, int size, FILE *stream);` |
| **说明 / Description** | 最多读取 `size-1` 个字符，包含 `\n`，始终 `\0` 结尾。Reads at most `size-1` chars including newline. |
| **返回值 / Return** | 成功返回 `s`，EOF 或错误返回 NULL |
| **安全 / Safety** | ✅ 安全 |

```c
char line[256];
while (fgets(line, sizeof(line), f)) {
    printf("%s", line);
}
```

### fputs / 写入字符串 / Write String

| 项目 / Item | 内容 |
|---|---|
| **原型 / Prototype** | `int fputs(const char *s, FILE *stream);` |
| **说明 / Description** | 写入字符串到文件（不追加 `\n`）。Writes string to stream. |

### fgetc / 读取单个字符 / Read Single Character

| 项目 / Item | 内容 |
|---|---|
| **原型 / Prototype** | `int fgetc(FILE *stream);` |
| **说明 / Description** | 读取下一个字符。返回 int 以区分 EOF。Reads next character as int. |
| **返回值 / Return** | 字符（unsigned char 转为 int），EOF 表示结束或错误 |

### fputc / 写入单个字符 / Write Single Character

| 项目 / Item | 内容 |
|---|---|
| **原型 / Prototype** | `int fputc(int c, FILE *stream);` |
| **说明 / Description** | 写入一个字符到文件。Writes a character to stream. |

### ungetc / 回退字符 / Push Back Character

| 项目 / Item | 内容 |
|---|---|
| **原型 / Prototype** | `int ungetc(int c, FILE *stream);` |
| **说明 / Description** | 将字符推回流，下次读取会先读到它。Pushes character back to stream. |

### feof / 检测文件结束 / Check End of File

| 项目 / Item | 内容 |
|---|---|
| **原型 / Prototype** | `int feof(FILE *stream);` |
| **说明 / Description** | 检测是否到达文件末尾。Returns true if EOF flag is set. |

### ferror / 检测错误 / Check Error

| 项目 / Item | 内容 |
|---|---|
| **原型 / Prototype** | `int ferror(FILE *stream);` |
| **说明 / Description** | 检测是否发生错误。Returns true if error flag is set. |

### remove / 删除文件 / Remove File

| 项目 / Item | 内容 |
|---|---|
| **原型 / Prototype** | `int remove(const char *pathname);` |
| **说明 / Description** | 删除文件或空目录。Deletes file or empty directory. |

### rename / 重命名 / Rename File

| 项目 / Item | 内容 |
|---|---|
| **原型 / Prototype** | `int rename(const char *oldpath, const char *newpath);` |
| **说明 / Description** | 重命名或移动文件。Renames or moves a file. |

### popen / 管道打开 / Pipe Open

| 项目 / Item | 内容 |
|---|---|
| **原型 / Prototype** | `FILE *popen(const char *command, const char *type);` |
| **说明 / Description** | 创建管道并执行命令。Creates pipe to/from command. |
| **type** | `"r"` 读取命令输出, `"w"` 写入命令输入 |

```c
FILE *fp = popen("ls -la", "r");
char line[256];
while (fgets(line, sizeof(line), fp)) puts(line);
pclose(fp);
```

---

## <math.h> / <cmath> 数学 / Mathematics

### fabs / 绝对值（浮点）/ Absolute Value (Float)

| 项目 / Item | 内容 |
|---|---|
| **头文件 / Header** | `<math.h>` / `<cmath>` |
| **原型 / Prototype** | `double fabs(double x);` / `float fabsf(float);` / `long double fabsl(long double);` |
| **说明 / Description** | 返回浮点绝对值。Returns floating-point absolute value. |

### fmod / 浮点取模 / Float Modulo

| 项目 / Item | 内容 |
|---|---|
| **原型 / Prototype** | `double fmod(double x, double y);` |
| **说明 / Description** | 返回 x/y 的浮点余数。Returns floating-point remainder of x/y. |

### floor / ceil / trunc / round / 取整函数 / Rounding Functions

| 项目 / Item | 内容 |
|---|---|
| **原型 / Prototype** | `double floor(double x);` `double ceil(double x);` `double trunc(double x);` `double round(double x);` |
| **说明 / Description** | `floor` 向下取整 / round down; `ceil` 向上取整 / round up; `trunc` 向零取整 / toward zero; `round` 四舍五入 / round to nearest |

```c
floor(3.7)  // 3.0
ceil(3.2)   // 4.0
trunc(-3.7) // -3.0
round(3.5)  // 4.0
```

### sqrt / 平方根 / Square Root

| 项目 / Item | 内容 |
|---|---|
| **原型 / Prototype** | `double sqrt(double x);` |
| **说明 / Description** | 返回非负平方根。Returns non-negative square root. |
| **注意 / Note** | 负数返回 NaN |

### pow / 幂运算 / Power

| 项目 / Item | 内容 |
|---|---|
| **原型 / Prototype** | `double pow(double base, double exponent);` |
| **说明 / Description** | 返回 base^exponent。Returns base raised to exponent. |

### exp / log / log2 / log10 / 指数与对数 / Exponential and Logarithm

| 项目 / Item | 内容 |
|---|---|
| **原型 / Prototype** | `double exp(double x);` `double log(double x);` `double log2(double x);` `double log10(double x);` |
| **说明 / Description** | `exp` e^x; `log` 自然对数 ln(x); `log2` log₂(x); `log10` log₁₀(x) |

### sin / cos / tan / 三角函数 / Trigonometric Functions

| 项目 / Item | 内容 |
|---|---|
| **原型 / Prototype** | `double sin(double x);` `double cos(double x);` `double tan(double x);` |
| **说明 / Description** | 参数为弧度。Arguments in radians. |

### asin / acos / atan / atan2 / 反三角函数 / Inverse Trigonometric

| 项目 / Item | 内容 |
|---|---|
| **原型 / Prototype** | `double asin(double x);` `double acos(double x);` `double atan(double y, double x);` |
| **说明 / Description** | `asin` [-1,1]→[-π/2,π/2]; `acos` [-1,1]→[0,π]; `atan2`(y,x) 返回 atan(y/x) 并正确处理象限 |

### isnan / isinf / isfinite / 浮点分类 / Float Classification

| 项目 / Item | 内容 |
|---|---|
| **头文件 / Header** | `<math.h>` (C99) |
| **原型 / Prototype** | `int isnan(double x);` `int isinf(double x);` `int isfinite(double x);` |
| **说明 / Description** | 检测 NaN、无穷大、有限值。Detects NaN, infinity, finite values. |

### copysign / 复制符号 / Copy Sign

| 项目 / Item | 内容 |
|---|---|
| **原型 / Prototype** | `double copysign(double x, double y);` |
| **说明 / Description** | 返回 x 的绝对值与 y 的符号组合。Returns magnitude of x with sign of y. |

### 数学常量 / Math Constants

| 常量 / Constant | 值 / Value | 说明 |
|---|---|---|
| `M_PI` | 3.14159265358979323846 | π |
| `M_E` | 2.71828182845904523536 | e |
| `HUGE_VAL` | double 正无穷 | 正无穷大 |
| `INFINITY` | float/double 正无穷 | 正无穷大 |
| `NAN` | NaN | 非数值 |

---

## <ctype.h> / <cctype> 字符分类 / Character Classification

| 函数 / Function | 说明 / Description | 示例 |
|---|---|---|
| `int isdigit(int c)` | 数字 0-9 / decimal digit | `isdigit('5')` → true |
| `int isxdigit(int c)` | 十六进制数字 0-9,a-f,A-F / hex digit | `isxdigit('A')` → true |
| `int isalpha(int c)` | 字母 a-z,A-Z / alphabetic | `isalpha('K')` → true |
| `int isalnum(int c)` | 字母或数字 / alphanumeric | `isalnum('3')` → true |
| `int isspace(int c)` | 空白字符 / whitespace | `isspace(' ')` → true |
| `int isupper(int c)` | 大写字母 / uppercase | `isupper('A')` → true |
| `int islower(int c)` | 小写字母 / lowercase | `islower('a')` → true |
| `int isprint(int c)` | 可打印字符 / printable | `isprint('!')` → true |
| `int ispunct(int c)` | 标点符号 / punctuation | `ispunct(',')` → true |
| `int iscntrl(int c)` | 控制字符 / control | `iscntrl('\n')` → true |
| `int isgraph(int c)` | 图形字符（可打印且非空格）/ graphic | `isgraph('A')` → true |
| `int toupper(int c)` | 转大写 / to uppercase | `toupper('a')` → 'A' |
| `int tolower(int c)` | 转小写 / to lowercase | `tolower('A')` → 'a' |

**注意 / Note:** 参数应为 unsigned char 或 EOF。传入负值 char（如中文）是未定义行为。

---

## <time.h> / <ctime> 时间 / Time

### time / 获取当前时间 / Get Current Time

| 项目 / Item | 内容 |
|---|---|
| **头文件 / Header** | `<time.h>` / `<ctime>` |
| **原型 / Prototype** | `time_t time(time_t *tloc);` |
| **说明 / Description** | 返回当前日历时间（秒数，自 Epoch 1970-01-01）。Returns current calendar time. |

### localtime / 本地时间分解 / Local Time Decomposition

| 项目 / Item | 内容 |
|---|---|
| **原型 / Prototype** | `struct tm *localtime(const time_t *timep);` |
| **说明 / Description** | 将 time_t 转为本地时间的 struct tm。Converts to local time broken-down form. |
| **安全 / Safety** | ⚠️ 返回静态缓冲区，非线程安全。多线程用 `localtime_r` |

### gmtime / UTC 时间分解 / UTC Time Decomposition

| 项目 / Item | 内容 |
|---|---|
| **原型 / Prototype** | `struct tm *gmtime(const time_t *timep);` |
| **说明 / Description** | 转为 UTC 时间的 struct tm。Converts to UTC time. |

### mktime / 时间合成 / Make Time

| 项目 / Item | 内容 |
|---|---|
| **原型 / Prototype** | `time_t mktime(struct tm *tm);` |
| **说明 / Description** | 将 struct tm 转为 time_t。Converts broken-down time to time_t. |

### strftime / 时间格式化 / Format Time

| 项目 / Item | 内容 |
|---|---|
| **原型 / Prototype** | `size_t strftime(char *s, size_t max, const char *format, const struct tm *tm);` |
| **常用格式 / Formats** | `%Y` 年(4位), `%m` 月(01-12), `%d` 日(01-31), `%H` 时(00-23), `%M` 分, `%S` 秒, `%F` `%Y-%m-%d`, `%T` `%H:%M:%S`, `%c` 本地日期时间 |

```c
time_t now = time(NULL);
struct tm *t = localtime(&now);
char buf[64];
strftime(buf, sizeof(buf), "%Y-%m-%d %H:%M:%S", t);
printf("%s\n", buf); // 2024-01-15 14:30:00
```

### clock / 进程 CPU 时间 / Process CPU Time

| 项目 / Item | 内容 |
|---|---|
| **原型 / Prototype** | `clock_t clock(void);` |
| **说明 / Description** | 返回进程使用的 CPU 时间。Returns CPU time used. |
| **常量** | `CLOCKS_PER_SEC` 每秒时钟数 |

```c
clock_t start = clock();
// ... 工作 ...
double elapsed = (double)(clock() - start) / CLOCKS_PER_SEC;
```

### struct tm / 分解时间结构 / Broken-down Time Structure

```c
struct tm {
    int tm_sec;   // 秒 [0,60]
    int tm_min;   // 分 [0,59]
    int tm_hour;  // 时 [0,23]
    int tm_mday;  // 日 [1,31]
    int tm_mon;   // 月 [0,11] ⚠️ 从0开始
    int tm_year;  // 年-1900
    int tm_wday;  // 星期 [0,6] 0=周日
    int tm_yday;  // 一年中的第几天 [0,365]
    int tm_isdst; // 夏令时标志
};
```

---

## <signal.h> / <csignal> 信号 / Signals

| 函数 / Function | 说明 / Description |
|---|---|
| `void (*signal(int sig, void (*handler)(int)))(int);` | 设置信号处理函数 / Set signal handler |
| `int sigaction(int sig, const struct sigaction *act, struct sigaction *oldact);` | POSIX 信号处理（推荐）/ POSIX signal handling (recommended) |
| `int kill(pid_t pid, int sig);` | 向进程发送信号 / Send signal to process |
| `int raise(int sig);` | 向自身发送信号 / Send signal to self |
| `unsigned int alarm(unsigned int seconds);` | 设置定时 SIGALRM / Schedule SIGALRM |

### 常用信号 / Common Signals

| 信号 / Signal | 值 / Value | 说明 / Description |
|---|---|---|
| `SIGINT` | 2 | 中断 (Ctrl+C) / Interrupt |
| `SIGTERM` | 15 | 终止请求 / Termination |
| `SIGKILL` | 9 | 强制终止（不可捕获）/ Kill (uncatchable) |
| `SIGSEGV` | 11 | 段错误 / Segmentation fault |
| `SIGPIPE` | 13 | 管道破裂（写已关闭的管道）/ Broken pipe |
| `SIGCHLD` | 20 | 子进程状态变化 / Child status change |
| `SIGALRM` | 14 | 定时器到期 / Alarm clock |

```c
#include <signal.h>
#include <stdio.h>
#include <unistd.h>

volatile sig_atomic_t running = 1;
void handler(int sig) { running = 0; }

int main() {
    signal(SIGINT, handler);
    while (running) { pause(); }
    printf("退出\n");
}
```

---

## <assert.h> / <cassert> 断言 / Assertions

| 项目 / Item | 内容 |
|---|---|
| **头文件 / Header** | `<assert.h>` / `<cassert>` |
| **宏 / Macro** | `void assert(scalar expression);` |
| **说明 / Description** | 表达式为 0 时打印诊断信息并调用 `abort()`。Aborts if expression is false. |
| **禁用 / Disable** | 定义 `NDEBUG` 宏可禁用所有 assert。`#define NDEBUG` before include disables all asserts. |

```c
#include <assert.h>
void process(int *p) {
    assert(p != NULL); // p 为 NULL 时终止
    // ...
}
```

---

## <errno.h> / <cerrno> 错误码 / Error Codes

| 项目 / Item | 内容 |
|---|---|
| **头文件 / Header** | `<errno.h>` / `<cerrno>` |
| **变量 / Variable** | `int errno;` — 由库函数设置的全局错误码 |
| **函数** | `void perror(const char *s);` — 打印 errno 对应的错误信息到 stderr |
| **函数** | `char *strerror(int errnum);` — 返回错误描述字符串 |

### 常见错误码 / Common Error Codes

| 错误码 / Code | 值 / Value | 说明 / Description |
|---|---|---|
| `EINVAL` | 22 | 无效参数 / Invalid argument |
| `ENOMEM` | 12 | 内存不足 / Out of memory |
| `ENOENT` | 2 | 文件或目录不存在 / No such file |
| `EACCES` | 13 | 权限不足 / Permission denied |
| `EINTR` | 4 | 被信号中断 / Interrupted system call |
| `EAGAIN` / `EWOULDBLOCK` | 35 / 11 | 资源暂时不可用 / Resource temporarily unavailable |
| `EPERM` | 1 | 操作不允许 / Operation not permitted |

```c
#include <errno.h>
FILE *f = fopen("nonexistent.txt", "r");
if (!f) {
    perror("fopen"); // fopen: No such file or directory
    printf("errno=%d\n", errno); // errno=2
}
```

---

## <stdint.h> / <cstdint> 定宽整数 / Fixed-Width Integers

| 类型 / Type | 说明 / Description |
|---|---|
| `int8_t` ~ `int64_t` | 有符号 8/16/32/64 位整数 / Signed fixed-width |
| `uint8_t` ~ `uint64_t` | 无符号 8/16/32/64 位整数 / Unsigned fixed-width |
| `intptr_t` / `uintptr_t` | 可容纳指针的整数 / Integer type holding a pointer |
| `size_t` | sizeof 返回的无符号类型 / Unsigned type from sizeof |
| `ptrdiff_t` | 指针差值的有符号类型 / Signed pointer difference |
| `intmax_t` / `uintmax_t` | 最大宽度整数 / Maximum width integer |

### 极限宏 / Limit Macros

| 宏 / Macro | 说明 / Description |
|---|---|
| `INT8_MIN` / `INT8_MAX` | int8_t 最小/最大值 |
| `INT64_MIN` / `INT64_MAX` | int64_t 范围: -2^63 ~ 2^63-1 |
| `UINT64_MAX` | 2^64 - 1 = 18446744073709551615 |
| `SIZE_MAX` | size_t 最大值 |

---

## <stddef.h> / <cstddef> 标准定义 / Standard Definitions

| 项目 / Item | 内容 |
|---|---|
| `size_t` | sizeof 运算符结果的无符号整数类型 |
| `ptrdiff_t` | 两个指针差值的有符号整数类型 |
| `offsetof(type, member)` | 结构体成员偏移量（字节） |
| `NULL` | 空指针常量 |
| `max_align_t` | 最大对齐要求的类型 |

---

## <stdbool.h> / <cstdbool> 布尔 / Boolean

| 项目 / Item | 内容 |
|---|---|
| `bool` | 布尔类型（展开为 `_Bool`） |
| `true` | 展开为 `1` |
| `false` | 展开为 `0` |
| `__bool_true_false_are_defined` | 展开为 `1` |

> C++ 中 `bool`/`true`/`false` 是内置关键字，无需此头文件。

---

## <inttypes.h> / <cinttypes> 格式化宏 / Formatting Macros

用于 `printf`/`scanf` 打印 `int64_t` 等定宽类型。

| 宏 / Macro | 用途 / Usage | 示例 |
|---|---|---|
| `PRId64` | printf int64_t | `printf("%" PRId64, val);` |
| `PRIu64` | printf uint64_t | `printf("%" PRIu64, val);` |
| `PRIX64` | printf uint64_t 十六进制大写 | `printf("%" PRIX64, val);` |
| `SCNd64` | scanf int64_t | `scanf("%" SCNd64, &val);` |

---

## <limits.h> / <climits> 极限值 / Limits

| 宏 / Macro | 说明 / Description |
|---|---|
| `CHAR_BIT` | char 的位数（通常 8） |
| `INT_MIN` / `INT_MAX` | int 范围: -32768 ~ 32767 (16位) 或 -2^31 ~ 2^31-1 (32位) |
| `LONG_MAX` | long 最大值 |
| `LLONG_MAX` | long long 最大值: 2^63-1 |
| `MB_LEN_MAX` | 多字节字符最大字节数 |
| `UCHAR_MAX` | unsigned char 最大值: 255 |
| `USHRT_MAX` | unsigned short 最大值: 65535 |
| `UINT_MAX` | unsigned int 最大值 |
| `ULONG_MAX` | unsigned long 最大值 |

---

## <float.h> / <cfloat> 浮点极限 / Floating-Point Limits

| 宏 / Macro | 说明 / Description |
|---|---|
| `FLT_MIN` / `FLT_MAX` | float 最小/最大正规值 |
| `DBL_MIN` / `DBL_MAX` | double 最小/最大正规值 |
| `FLT_EPSILON` | float 最小可表示差值（≈1.19e-7） |
| `DBL_EPSILON` | double 最小可表示差值（≈2.22e-16） |
| `LDBL_DIG` | long double 十进制精度位数 |
| `FLT_DIG` / `DBL_DIG` | 十进制精度位数（6 / 15） |
| `DECIMAL_DIG` | 最宽类型十进制精度 |

---

# C++ 标准库 (C++ Standard Library / C++ 标准库)

---

## <string> 字符串 / Strings

### std::string 构造与基本操作 / Construction & Basics

| 操作 / Operation | 说明 / Description | 示例 |
|---|---|---|
| `string s;` | 默认构造，空字符串 | |
| `string s("hello");` | C 字符串构造 | |
| `string s(5, 'a');` | 5 个 'a' → "aaaaa" | |
| `string s2 = s;` | 拷贝构造 | |
| `string s3 = std::move(s);` | 移动构造 / Move construct | |
| `s = "world";` | 赋值 / Assign | |
| `s.empty()` | 是否为空 / Is empty | `s.empty()` → false |
| `s.size()` / `s.length()` | 字符数 / Character count | |
| `s.capacity()` | 已分配容量 / Allocated capacity | |
| `s.reserve(100)` | 预留容量（不改变 size）/ Reserve capacity | |
| `s.shrink_to_fit()` | 释放多余容量 / Release excess capacity (C++11) | |
| `s.c_str()` | 返回 C 字符串（const char*） | |
| `s.data()` | 返回可修改的字符数组指针 (C++17) | |

### 连接与比较 / Concatenation & Comparison

| 操作 / Operation | 说明 / Description |
|---|---|
| `s1 + s2` | 连接 / Concatenation |
| `s += "x"` | 追加 / Append |
| `s.append("xyz")` | 追加字符串 |
| `s.push_back('a')` | 追加单个字符 |
| `s1 == s2` / `s1 != s2` | 相等比较 / Equality |
| `s1 < s2` / `s1 > s2` | 字典序比较 / Lexicographic |
| `s1.compare(s2)` | 比较函数（<0/0/>0） |

### 查找 / Searching

| 操作 / Operation | 说明 / Description | 复杂度 |
|---|---|---|
| `s.find("ab")` | 查找子串首次出现位置 / First occurrence | O(n*m) |
| `s.rfind("ab")` | 查找子串最后出现位置 / Last occurrence | O(n*m) |
| `s.find_first_of("aeiou")` | 查找任一字符首次出现 | O(n) |
| `s.find_last_of("aeiou")` | 查找任一字符最后出现 | O(n) |
| `s.find_first_not_of("abc")` | 查找不在集合中的首个字符 | O(n) |
| `s.substr(pos, len)` | 提取子串 / Extract substring | O(n) |
| **返回值** | `string::npos` 表示未找到 | |

### 修改 / Modification

| 操作 / Operation | 说明 / Description |
|---|---|
| `s.insert(pos, "str")` | 在 pos 插入 |
| `s.erase(pos, len)` | 删除从 pos 开始的 len 个字符 |
| `s.replace(pos, len, "new")` | 替换 |
| `s.clear()` | 清空 |

### 字符串转换 / String Conversion (C++11)

| 函数 / Function | 说明 / Description |
|---|---|
| `std::stoi(s)` | 字符串→int |
| `std::stol(s)` | 字符串→long |
| `std::stoll(s)` | 字符串→long long |
| `std::stoul(s)` | 字符串→unsigned long |
| `std::stod(s)` | 字符串→double |
| `std::stof(s)` | 字符串→float |
| `std::to_string(42)` | 数值→string |

### std::string_view (C++17) / 字符串视图

| 项目 / Item | 内容 |
|---|---|
| **头文件 / Header** | `<string_view>` |
| **说明 / Description** | 非拥有字符串视图，零拷贝引用字符串片段。Non-owning reference to string. |

| 操作 / Operation | 说明 / Description |
|---|---|
| `string_view sv("hello");` | 构造 |
| `sv.substr(pos, len)` | 子串视图 / Sub-view |
| `sv.starts_with("he")` | 前缀匹配 (C++20) |
| `sv.ends_with("lo")` | 后缀匹配 (C++20) |
| `sv.find("ll")` | 查找 |
| `sv.compare(sv2)` | 比较 |
| `sv.size()` / `sv.length()` | 长度 |
| `sv.data()` | 底层数据指针 |
| `sv.empty()` | 是否为空 |

```cpp
void process(std::string_view sv) {
    // 无拷贝地处理字符串片段
    if (sv.starts_with("http")) { /* ... */ }
}
process("https://example.com");
std::string s = "hello";
process(s); // 隐式转换
```

---

## <vector> 动态数组 / Dynamic Arrays

### 构造与基本操作 / Construction & Basics

| 操作 / Operation | 说明 / Description | 复杂度 |
|---|---|---|
| `vector<int> v;` | 默认构造，空 | O(1) |
| `vector<int> v(100);` | 100 个默认值元素 | O(n) |
| `vector<int> v(10, 42);` | 10 个值为 42 | O(n) |
| `vector<int> v{1,2,3};` | 初始化列表 (C++11) | O(n) |
| `vector<int> v2(v);` | 拷贝构造 | O(n) |
| `vector<int> v3(std::move(v));` | 移动构造 | O(1) |

### 元素访问 / Element Access

| 操作 / Operation | 说明 / Description |
|---|---|
| `v[i]` | 下标访问（无边界检查）/ No bounds check |
| `v.at(i)` | 带边界检查，越界抛 `out_of_range` / Bounds checked |
| `v.front()` | 第一个元素 |
| `v.back()` | 最后一个元素 |
| `v.data()` | 底层数组指针 |
| `v.size()` | 元素个数 |
| `v.capacity()` | 已分配容量 |
| `v.empty()` | 是否为空 |

### 修改 / Modification

| 操作 / Operation | 说明 / Description | 复杂度 | 迭代器失效 |
|---|---|---|---|
| `v.push_back(x)` | 末尾添加 | 均摊 O(1) | 可能全部失效 |
| `v.emplace_back(args...)` | 原地构造末尾元素 (C++11) | 均摊 O(1) | 可能全部失效 |
| `v.pop_back()` | 移除末尾元素 | O(1) | 仅 end() 失效 |
| `v.insert(pos, x)` | 在 pos 处插入 | O(n) | pos 及之后失效 |
| `v.erase(pos)` | 删除 pos 处元素 | O(n) | pos 及之后失效 |
| `v.resize(n)` | 调整大小（新元素默认值） | O(n) | 可能全部失效 |
| `v.reserve(n)` | 预留容量（不改变 size） | O(n) 最坏 | 不失效 |
| `v.shrink_to_fit()` | 释放多余容量 (C++11) | O(n) | 不失效 |
| `v.clear()` | 清空所有元素 | O(n) | 全部失效 |
| `v.swap(v2)` | 交换内容 | O(1) | 引用其他容器失效 |

### 迭代器 / Iterators

| 操作 / Operation | 说明 / Description |
|---|---|
| `v.begin()` / `v.end()` | 正向迭代器 |
| `v.rbegin()` / `v.rend()` | 反向迭代器 |
| `v.cbegin()` / `v.cend()` | const 迭代器 |

### 迭代器失效规则 / Iterator Invalidation Rules

- `push_back` / `emplace_back`：若发生重分配（`size == capacity`），所有迭代器/指针/引用失效
- `insert`：插入点及之后的所有失效
- `erase`：删除点及之后的所有失效
- `pop_back`：仅 `end()` 失效
- `resize`：若增大则可能全部失效
- `reserve` / `shrink_to_fit`：可能全部失效

### 二维 vector / 2D Vector

```cpp
vector<vector<int>> matrix(3, vector<int>(4, 0)); // 3×4 全零
matrix[1][2] = 42;
for (auto& row : matrix) {
    for (auto& val : row) {
        cout << val << " ";
    }
    cout << "\n";
}
```

---

## <list> / <forward_list> 链表 / Linked Lists

| 操作 / Operation | list | forward_list | 说明 / Description |
|---|---|---|---|
| `push_back(x)` | ✅ | ❌ | 末尾添加 |
| `push_front(x)` | ✅ | ✅ | 头部添加 O(1) |
| `pop_back()` | ✅ | ❌ | 移除末尾 |
| `pop_front()` | ✅ | ✅ | 移除头部 O(1) |
| `insert(pos, x)` | ✅ | ✅ | 插入 O(1) |
| `erase(pos)` | ✅ | ✅ | 删除 O(1) |
| `splice(pos, other)` | ✅ | ✅ | 转移节点 O(1) |
| `sort()` | ✅ | ✅ | 排序 O(n log n) |
| `unique()` | ✅ | ✅ | 去除连续重复 O(n) |
| `merge(other)` | ✅ | ✅ | 合并有序链表 O(n) |
| `reverse()` | ✅ | ✅ | 反转 O(n) |
| `remove(val)` | ✅ | ✅ | 移除所有等于 val 的元素 O(n) |

> 链表特点：任意位置插入/删除 O(1)，不支持随机访问，缓存不友好。

---

## <deque> 双端队列 / Double-Ended Queue

| 操作 / Operation | 说明 / Description | 复杂度 |
|---|---|---|
| `push_front(x)` / `push_back(x)` | 头/尾添加 | O(1) |
| `pop_front()` / `pop_back()` | 头/尾移除 | O(1) |
| `d[i]` / `d.at(i)` | 随机访问 | O(1) |
| `d.front()` / `d.back()` | 首/尾元素 | O(1) |
| `d.insert(pos, x)` | 插入 | O(n) |
| `d.size()` / `d.empty()` | 大小/判空 | O(1) |

> deque 内存为分段连续，支持 O(1) 首尾操作和随机访问。中间插入仍为 O(n)。

---

## <map> / <unordered_map> 关联容器 / Associative Containers

### std::map（有序映射）/ Ordered Map

| 操作 / Operation | 说明 / Description | 复杂度 |
|---|---|---|
| `m[key] = val` | 插入或更新（不存在时默认构造）/ Insert or update | O(log n) |
| `m.at(key)` | 访问，不存在抛异常 / Access with bounds check | O(log n) |
| `m.insert({key, val})` | 插入，返回 pair<iterator,bool> | O(log n) |
| `m.emplace(key, val)` | 原地插入 (C++11) | O(log n) |
| `m.erase(key)` | 按键删除 | O(log n) |
| `m.erase(it)` | 按迭代器删除 | O(1) 均摊 |
| `m.find(key)` | 查找，返回迭代器 | O(log n) |
| `m.count(key)` | 计数（0 或 1） | O(log n) |
| `m.contains(key)` | 是否存在 (C++20) | O(log n) |
| `m.lower_bound(key)` | 第一个 >= key 的元素 | O(log n) |
| `m.upper_bound(key)` | 第一个 > key 的元素 | O(log n) |
| `m.equal_range(key)` | 等价范围 [lower, upper) | O(log n) |
| `m.size()` / `m.empty()` | 大小/判空 | O(1) |

### std::unordered_map（无序映射）/ Unordered Map

| 操作 / Operation | 说明 / Description | 复杂度 |
|---|---|---|
| 同 map | 大部分接口相同 | **O(1) 平均**, O(n) 最坏 |
| `m.bucket_count()` | 桶数量 | O(1) |
| `m.load_factor()` | 负载因子 | O(1) |
| `m.rehash(n)` | 重新设置桶数量 | O(n) |
| `m.max_load_factor(f)` | 设置最大负载因子 | O(1) |

### map vs unordered_map 对比

| 特性 / Feature | map | unordered_map |
|---|---|---|
| 底层实现 / Implementation | 红黑树 / Red-black tree | 哈希表 / Hash table |
| 有序 / Ordered | ✅ 按键排序 | ❌ 无序 |
| 查找复杂度 / Lookup | O(log n) | O(1) 平均 |
| 内存开销 / Memory | 较低 | 较高（桶开销） |
| 需求 / Requirements | `<` 运算符 | `==` 运算符 + hash |

```cpp
// 遍历 map（按键序）
std::map<string, int> ages;
ages["Alice"] = 30;
ages["Bob"] = 25;
for (const auto& [name, age] : ages) {
    cout << name << ": " << age << "\n";
}
```

---

## <set> / <unordered_set> 集合 / Sets

| 操作 / Operation | set | unordered_set | 复杂度 |
|---|---|---|---|
| `s.insert(val)` | ✅ | ✅ | O(log n) / O(1) |
| `s.erase(val)` | ✅ | ✅ | O(log n) / O(1) |
| `s.find(val)` | ✅ | ✅ | O(log n) / O(1) |
| `s.count(val)` | ✅ | ✅ | O(log n) / O(1) |
| `s.contains(val)` | ✅ | ✅ (C++20) | O(log n) / O(1) |
| `s.lower_bound(val)` | ✅ | ❌ | O(log n) |
| `s.upper_bound(val)` | ✅ | ❌ | O(log n) |

```cpp
std::set<int> s = {3, 1, 4, 1, 5}; // {1, 3, 4, 5}
if (s.contains(3)) { /* 存在 */ }
```

---

## <array> / <span> 固定数组与视图 / Fixed Array and View

### std::array (C++11)

| 操作 / Operation | 说明 / Description |
|---|---|
| `array<int, 5> a = {1,2,3,4,5};` | 构造（大小为编译期常量） |
| `a[i]` / `a.at(i)` | 元素访问 |
| `a.front()` / `a.back()` | 首/尾元素 |
| `a.size()` | 元素个数（编译期常量） |
| `a.fill(val)` | 填充所有元素 |
| `a.swap(a2)` | 交换 |
| `a.begin()` / `a.end()` | 迭代器 |

### std::span (C++20)

| 项目 / Item | 内容 |
|---|---|
| **头文件 / Header** | `<span>` |
| **说明 / Description** | 非拥有连续序列视图。Non-owning view over contiguous sequence. |

| 操作 / Operation | 说明 / Description |
|---|---|
| `span<int> s(arr);` | 从数组构造 |
| `span<int> s(arr, 5);` | 从指针+大小构造 |
| `s.first(3)` / `s.last(2)` | 前/后 N 个元素的子视图 |
| `s.subspan(1, 3)` | 子视图 |
| `s.size()` / `s.size_bytes()` | 元素数/字节数 |
| `s[i]` / `s.front()` / `s.back()` | 元素访问 |
| `s.empty()` | 是否为空 |

```cpp
void process(std::span<const int> data) {
    for (auto v : data) cout << v << " ";
}
int arr[] = {1, 2, 3, 4, 5};
process(arr);        // 整个数组
process({arr, 3});   // 前3个
```

---

## <algorithm> 算法 / Algorithms

### 排序 / Sorting

| 算法 / Algorithm | 说明 / Description | 复杂度 |
|---|---|---|
| `sort(first, last)` | 排序（不保证稳定）/ Sort (not stable) | O(n log n) |
| `sort(first, last, comp)` | 自定义比较排序 | O(n log n) |
| `stable_sort(first, last)` | 稳定排序 / Stable sort | O(n log²n) |
| `partial_sort(first, middle, last)` | 部分排序，前 middle 个为最小元素 | O(n log n) |
| `nth_element(first, nth, last)` | 第 nth 个元素归位，前小后大 | O(n) 平均 |
| `is_sorted(first, last)` | 是否已排序 (C++11) | O(n) |

```cpp
vector<int> v = {3, 1, 4, 1, 5, 9};
sort(v.begin(), v.end()); // {1, 1, 3, 4, 5, 9}
sort(v.begin(), v.end(), greater<>()); // 降序 {9, 5, 4, 3, 1, 1}
```

### 搜索 / Searching

| 算法 / Algorithm | 说明 / Description | 复杂度 |
|---|---|---|
| `find(first, last, val)` | 查找值 | O(n) |
| `find_if(first, last, pred)` | 查找满足条件的元素 | O(n) |
| `find_if_not(first, last, pred)` | 查找不满足条件的元素 (C++11) | O(n) |
| `search(haystack, h_end, needle, n_end)` | 子序列搜索 | O(n*m) |
| `binary_search(first, last, val)` | 二分查找是否存在（需已排序） | O(log n) |
| `lower_bound(first, last, val)` | 第一个 >= val 的位置 | O(log n) |
| `upper_bound(first, last, val)` | 第一个 > val 的位置 | O(log n) |
| `equal_range(first, last, val)` | 等价范围 [lower, upper) | O(log n) |

```cpp
auto it = find_if(v.begin(), v.end(), [](int x) { return x > 4; });
auto pos = lower_bound(v.begin(), v.end(), 4);
```

### 变换 / Transforming

| 算法 / Algorithm | 说明 / Description |
|---|---|
| `for_each(first, last, fn)` | 对每个元素执行函数 |
| `transform(first, last, dest, fn)` | 变换元素到目标 |
| `replace(first, last, old, new)` | 替换值 |
| `replace_if(first, last, pred, new)` | 条件替换 |
| `fill(first, last, val)` | 填充值 |
| `fill_n(first, n, val)` | 填充前 n 个 |
| `generate(first, last, gen)` | 用生成器填充 |
| `remove(first, last, val)` | 移除值（不改变 size！） |
| `remove_if(first, last, pred)` | 条件移除 |
| `unique(first, last)` | 去除连续重复 |
| `reverse(first, last)` | 反转 |
| `rotate(first, mid, last)` | 旋转 |
| `shuffle(first, last, rng)` | 随机洗牌 (C++11) |
| `sample(first, last, out, n, rng)` | 随机采样 (C++17) |

> ⚠️ `remove`/`unique` 不实际删除元素，需配合 `erase`：
> ```cpp
> v.erase(remove(v.begin(), v.end(), 3), v.end()); // erase-remove 惯用法
> ```

### 数值统计 / Numeric Queries

| 算法 / Algorithm | 说明 / Description | 复杂度 |
|---|---|---|
| `count(first, last, val)` | 计数 | O(n) |
| `count_if(first, last, pred)` | 条件计数 | O(n) |
| `min(a, b)` / `max(a, b)` | 最小/最大值 | O(1) |
| `minmax(a, b)` | 同时返回 min 和 max (C++11) | O(1) |
| `min_element(first, last)` | 最小元素迭代器 | O(n) |
| `max_element(first, last)` | 最大元素迭代器 | O(n) |
| `clamp(val, lo, hi)` | 限制在 [lo, hi] 范围 (C++17) | O(1) |

### 累积 / Accumulation

| 算法 / Algorithm | 说明 / Description |
|---|---|
| `accumulate(first, last, init)` | 求和（可自定义运算） |
| `inner_product(first1, last1, first2, init)` | 内积 |
| `partial_sum(first, last, dest)` | 部分和 |
| `adjacent_difference(first, last, dest)` | 相邻差 |
| `reduce(first, last, init)` | 可并行求和 (C++17) |
| `transform_reduce(first, last, init, ...)` | 变换后并行累积 (C++17) |

```cpp
int sum = accumulate(v.begin(), v.end(), 0);
int product = accumulate(v.begin(), v.end(), 1, multiplies<>());
```

### 集合操作 / Set Operations（需已排序）

| 算法 / Algorithm | 说明 / Description |
|---|---|
| `set_intersection(...)` | 交集 |
| `set_union(...)` | 并集 |
| `set_difference(...)` | 差集 |
| `set_symmetric_difference(...)` | 对称差集 |
| `merge(first1, last1, first2, last2, dest)` | 合并有序序列 |
| `includes(first1, last1, first2, last2)` | 是否包含 |

### 堆操作 / Heap Operations

| 算法 / Algorithm | 说明 / Description | 复杂度 |
|---|---|---|
| `make_heap(first, last)` | 构造最大堆 | O(n) |
| `push_heap(first, last)` | 入堆（先 push_back） | O(log n) |
| `pop_heap(first, last)` | 出堆到 last-1（再 pop_back） | O(log n) |
| `sort_heap(first, last)` | 堆排序 | O(n log n) |
| `is_heap(first, last)` | 是否为堆 (C++11) | O(n) |

```cpp
vector<int> v = {3, 1, 4, 1, 5};
make_heap(v.begin(), v.end()); // 最大堆
v.push_back(9);
push_heap(v.begin(), v.end());
pop_heap(v.begin(), v.end());
v.pop_back(); // 取出最大值
```

### 排列 / Permutation

| 算法 / Algorithm | 说明 / Description |
|---|---|
| `next_permutation(first, last)` | 下一个字典序排列（返回 false 表示已是最后一个） |
| `prev_permutation(first, last)` | 上一个字典序排列 |

### 分区 / Partition

| 算法 / Algorithm | 说明 / Description |
|---|---|
| `partition(first, last, pred)` | 满足条件的放前面（不保序） |
| `stable_partition(first, last, pred)` | 分区且保持相对顺序 |
| `partition_point(first, last, pred)` | 分区点（已分区时使用） |
| `is_partitioned(first, last, pred)` | 是否已分区 (C++11) |

### 其他 / Miscellaneous

| 算法 / Algorithm | 说明 / Description |
|---|---|
| `any_of(first, last, pred)` | 任一满足 (C++11) |
| `all_of(first, last, pred)` | 全部满足 (C++11) |
| `none_of(first, last, pred)` | 全部不满足 (C++11) |
| `mismatch(first1, last1, first2)` | 首个不同的位置 |
| `equal(first1, last1, first2)` | 两个范围是否相等 |
| `lexicographical_compare(...)` | 字典序比较 |
| `swap(a, b)` | 交换两个值 |
| `iter_swap(it1, it2)` | 交换两个迭代器指向的值 |

---

## <numeric> 数值算法 / Numeric Algorithms

| 算法 / Algorithm | 说明 / Description | 复杂度 |
|---|---|---|
| `iota(first, last, val)` | 填充递增序列 val, val+1, ... | O(n) |
| `accumulate(first, last, init)` | 累加/累积 | O(n) |
| `inner_product(f1, l1, f2, init)` | 内积 | O(n) |
| `partial_sum(first, last, dest)` | 部分和 | O(n) |
| `adjacent_difference(f, l, dest)` | 相邻差 | O(n) |
| `gcd(a, b)` | 最大公约数 (C++17) | O(log(min(a,b))) |
| `lcm(a, b)` | 最小公倍数 (C++17) | O(log(min(a,b))) |
| `midpoint(a, b)` | 中点，避免溢出 (C++20) | O(1) |
| `lerp(a, b, t)` | 线性插值 (C++20) | O(1) |

```cpp
vector<int> v(10);
iota(v.begin(), v.end(), 1); // {1, 2, 3, ..., 10}
cout << gcd(12, 8) << endl;   // 4
cout << lcm(12, 8) << endl;   // 24
```

---

## <memory> 内存 / Memory

### unique_ptr / 独占智能指针 / Exclusive Smart Pointer

| 操作 / Operation | 说明 / Description |
|---|---|
| `auto p = make_unique<T>(args...)` | 创建（C++14） |
| `auto p = make_unique<T[]>(n)` | 创建数组 |
| `p.reset()` | 释放并置空 |
| `p.release()` | 释放所有权，返回裸指针 |
| `p.swap(p2)` | 交换 |
| `p.get()` | 获取裸指针 |
| `*p` / `p->` | 解引用 |
| `p.get_deleter()` | 获取删除器 |
| `bool(p)` | 是否持有资源 |

```cpp
auto p = make_unique<int>(42);
cout << *p; // 42
// 自定义删除器
auto deleter = [](FILE* f) { if(f) fclose(f); };
unique_ptr<FILE, decltype(deleter)> fp(fopen("a.txt","r"), deleter);
```

### shared_ptr / 共享智能指针 / Shared Smart Pointer

| 操作 / Operation | 说明 / Description |
|---|---|
| `auto p = make_shared<T>(args...)` | 创建（一次分配） |
| `p.use_count()` | 引用计数 |
| `p.reset()` | 释放引用 |
| `p.unique()` | 是否唯一持有 (C++17 前) |
| `p.get()` / `*p` / `p->` | 同 unique_ptr |

```cpp
class Node : public enable_shared_from_this<Node> {
public:
    shared_ptr<Node> getPtr() { return shared_from_this(); }
};
auto p = make_shared<Node>();
```

### weak_ptr / 弱引用 / Weak Reference

| 操作 / Operation | 说明 / Description |
|---|---|
| `weak_ptr<T> wp(sp)` | 从 shared_ptr 构造 |
| `wp.lock()` | 尝试提升为 shared_ptr（可能为空） |
| `wp.expired()` | 原对象是否已释放 |
| `wp.reset()` | 释放弱引用 |
| `wp.use_count()` | 关联的 shared_ptr 引用计数 |

> 用途：打破 shared_ptr 循环引用 / Breaking shared_ptr circular references.

### allocator / 分配器

| 操作 / Operation | 说明 / Description |
|---|---|
| `allocator<T> a;` | 构造分配器 |
| `a.allocate(n)` | 分配 n 个未初始化的 T |
| `a.deallocate(p, n)` | 释放 |
| `a.construct(p, args...)` | 原地构造 (C++17 前) |
| `a.destroy(p)` | 析构 (C++17 前) |

---

## <functional> 函数对象 / Function Objects

| 项目 / Item | 内容 |
|---|---|
| **头文件 / Header** | `<functional>` |

### std::function / 通用可调用包装 / General Callable Wrapper

```cpp
#include <functional>
function<int(int, int)> add = [](int a, int b) { return a + b; };
function<int(int, int)> add2 = plus<int>();
cout << add(2, 3); // 5
```

### std::bind / 绑定参数 / Bind Arguments

```cpp
auto half = bind(divides<double>(), placeholders::_1, 2.0);
cout << half(10); // 5
```

### std::invoke / 调用可调用对象 (C++17)

```cpp
invoke(add, 2, 3); // 等同于 add(2, 3)
```

### 仿函数 / Functors

| 仿函数 / Functor | 操作 |
|---|---|
| `plus<T>` | a + b |
| `minus<T>` | a - b |
| `multiplies<T>` | a * b |
| `divides<T>` | a / b |
| `modulus<T>` | a % b |
| `negate<T>` | -a |
| `equal_to<T>` | a == b |
| `not_equal_to<T>` | a != b |
| `greater<T>` | a > b |
| `less<T>` | a < b |
| `greater_equal<T>` | a >= b |
| `less_equal<T>` | a <= b |
| `logical_and<T>` | a && b |
| `logical_or<T>` | a \|\| b |
| `logical_not<T>` | !a |

### std::hash / 哈希函数

```cpp
size_t h = hash<string>{}("hello");
```

### std::reference_wrapper / 引用包装

```cpp
auto ref_cref = ref(x);  // reference_wrapper
auto r = cref(x);         // const reference_wrapper
```

---

## <thread> 线程 / Threading

| 项目 / Item | 内容 |
|---|---|
| **头文件 / Header** | `<thread>` |
| **编译** | 需链接 `-pthread`（Linux） |

### std::thread

| 操作 / Operation | 说明 / Description |
|---|---|
| `thread t(fn, args...)` | 创建线程 |
| `t.join()` | 等待线程结束 |
| `t.detach()` | 分离线程（后台运行） |
| `t.joinable()` | 是否可 join |
| `t.get_id()` | 线程 ID |
| `thread::hardware_concurrency()` | 硬件线程数 |

### std::jthread (C++20)

| 操作 / Operation | 说明 / Description |
|---|---|
| `jthread t(fn, args...)` | 创建（析构时自动 join） |
| `t.get_stop_token()` | 获取停止令牌 |
| `t.request_stop()` | 请求停止 |

### this_thread 命名空间

| 函数 / Function | 说明 / Description |
|---|---|
| `this_thread::get_id()` | 当前线程 ID |
| `this_thread::yield()` | 让出时间片 |
| `this_thread::sleep_for(100ms)` | 休眠一段时间 |
| `this_thread::sleep_until(tp)` | 休眠到时间点 |

```cpp
#include <thread>
#include <iostream>

void worker(int id) {
    for (int i = 0; i < 3; ++i) {
        std::cout << "Thread " << id << ": " << i << "\n";
    }
}

int main() {
    std::thread t1(worker, 1);
    std::thread t2(worker, 2);
    t1.join();
    t2.join();
}
```

---

## <mutex> 互斥量 / Mutexes

| 项目 / Item | 内容 |
|---|---|
| **头文件 / Header** | `<mutex>` |

### 互斥量类型 / Mutex Types

| 类型 / Type | 说明 / Description |
|---|---|
| `mutex` | 基本互斥量 |
| `recursive_mutex` | 可递归锁定 |
| `timed_mutex` | 支持超时锁定 |
| `recursive_timed_mutex` | 递归 + 超时 |
| `shared_mutex` | 读写锁（C++17） |

### RAII 锁 / RAII Locks

| 类型 / Type | 说明 / Description | 用法 |
|---|---|---|
| `lock_guard<mutex>` | 作用域锁，构造时锁定，析构时解锁 | `lock_guard<mutex> lk(m);` |
| `unique_lock<mutex>` | 灵活锁，支持延迟锁定、手动解锁、移动 | `unique_lock<mutex> lk(m);` |
| `scoped_lock<mutex>` | 多锁同时锁定，防死锁 (C++17) | `scoped_lock lk(m1, m2);` |
| `shared_lock<shared_mutex>` | 共享（读）锁 (C++14) | `shared_lock<shared_mutex> lk(rw);` |

### 其他 / Miscellaneous

| 操作 / Operation | 说明 / Description |
|---|---|
| `call_once(flag, fn)` | 只执行一次 |
| `once_flag flag;` | call_once 的标志 |

```cpp
mutex m;
void safe_print(string msg) {
    lock_guard<mutex> lk(m);
    cout << msg << endl;
}
```

---

## <condition_variable> 条件变量 / Condition Variables

| 项目 / Item | 内容 |
|---|---|
| **头文件 / Header** | `<condition_variable>` |

| 操作 / Operation | 说明 / Description |
|---|---|
| `cv.wait(lk)` | 释放锁并等待通知 |
| `cv.wait(lk, pred)` | 带谓词等待（推荐）/ Wait with predicate (recommended) |
| `cv.wait_for(lk, dur)` | 超时等待 |
| `cv.wait_until(lk, tp)` | 定时等待 |
| `cv.notify_one()` | 唤醒一个等待线程 |
| `cv.notify_all()` | 唤醒所有等待线程 |

### 生产者-消费者模式 / Producer-Consumer Pattern

```cpp
mutex m;
condition_variable cv;
queue<int> q;
bool done = false;

void producer() {
    for (int i = 0; i < 10; ++i) {
        {
            lock_guard<mutex> lk(m);
            q.push(i);
        }
        cv.notify_one();
    }
    lock_guard<mutex> lk(m);
    done = true;
    cv.notify_all();
}

void consumer() {
    while (true) {
        unique_lock<mutex> lk(m);
        cv.wait(lk, []{ return !q.empty() || done; });
        if (q.empty() && done) break;
        int val = q.front();
        q.pop();
        lk.unlock();
        cout << "Got: " << val << "\n";
    }
}
```

---

## <future> 异步 / Futures

| 项目 / Item | 内容 |
|---|---|
| **头文件 / Header** | `<future>` |

| 类型 / Type | 说明 / Description |
|---|---|
| `promise<T>` | 异步结果提供者 / Result provider |
| `future<T>` | 异步结果获取者 / Result receiver |
| `shared_future<T>` | 可复制的 future / Copyable future |
| `packaged_task<R(Args...)>` | 包装可调用对象为异步任务 |

| 操作 / Operation | 说明 / Description |
|---|---|
| `async(launch::async, fn, args...)` | 异步执行，返回 future |
| `async(fn, args...)` | 自动选择策略 |
| `fut.get()` | 获取结果（阻塞直到就绪） |
| `fut.wait()` | 等待就绪 |
| `fut.wait_for(dur)` | 超时等待 |
| `fut.wait_for(dur)` | 返回 `future_status` |
| `prom.set_value(val)` | 设置结果 |
| `prom.set_exception(ex)` | 设置异常 |

```cpp
future<int> f = async(launch::async, [](int x) {
    this_thread::sleep_for(1s);
    return x * x;
}, 42);
cout << f.get(); // 1764 (等待1秒后)
```

---

## <atomic> 原子操作 / Atomic Operations

| 项目 / Item | 内容 |
|---|---|
| **头文件 / Header** | `<atomic>` |

| 操作 / Operation | 说明 / Description |
|---|---|
| `atomic<T> a(val)` | 原子变量 |
| `a.load(order)` | 原子读取 |
| `a.store(val, order)` | 原子写入 |
| `a.exchange(val, order)` | 原子交换 |
| `a.compare_exchange_weak/exp(expected, desired, ...)` | CAS 操作 |
| `a.fetch_add(delta, order)` | 原子加 |
| `a.fetch_sub(delta, order)` | 原子减 |
| `a.fetch_and(val, order)` | 原子与 |
| `a.fetch_or(val, order)` | 原子或 |
| `a.fetch_xor(val, order)` | 原子异或 |
| `a++` / `a--` / `++a` / `--a` | 原子自增/自减 |

### 内存序 / Memory Order

| 内存序 / Order | 说明 / Description |
|---|---|
| `memory_order_relaxed` | 无序，仅保证原子性 |
| `memory_order_consume` | 数据依赖排序（极少使用） |
| `memory_order_acquire` | 获取语义（读操作） |
| `memory_order_release` | 释放语义（写操作） |
| `memory_order_acq_rel` | 获取+释放 |
| `memory_order_seq_cst` | 顺序一致性（默认）/ Sequential consistency (default) |

### atomic_flag / 自旋锁原语

```cpp
atomic_flag lock = ATOMIC_FLAG_INIT;
void critical_section() {
    while (lock.test_and_set(memory_order_acquire)) {}
    // ... 临界区 ...
    lock.clear(memory_order_release);
}
```

---

## <chrono> 时间 / Time

| 项目 / Item | 内容 |
|---|---|
| **头文件 / Header** | `<chrono>` |

### duration / 时间段

```cpp
using namespace std::chrono;
auto d1 = 100ms;       // milliseconds
auto d2 = 5s;          // seconds
auto d3 = duration_cast<milliseconds>(d2); // 5000ms
auto d4 = 1h + 30min;  // 1小时30分钟
```

### time_point / 时间点

```cpp
auto now = system_clock::now();
auto tp = steady_clock::now();
auto since_epoch = now.time_since_epoch();
```

### 时钟 / Clocks

| 时钟 / Clock | 说明 / Description |
|---|---|
| `system_clock` | 系统时钟（可能跳变）/ System wall clock |
| `steady_clock` | 单调时钟（不会跳变）/ Monotonic clock |
| `high_resolution_clock` | 最高精度时钟 / Highest resolution |

### sleep

```cpp
this_thread::sleep_for(100ms);
this_thread::sleep_until(system_clock::now() + 1s);
```

---

## <optional> / <variant> / <any> 可选类型 / Optional Types

### std::optional (C++17)

| 项目 / Item | 内容 |
|---|---|
| **头文件 / Header** | `<optional>` |

| 操作 / Operation | 说明 / Description |
|---|---|
| `optional<int> o;` | 空 optional |
| `optional<int> o = 42;` | 含值 |
| `optional<int> o = nullopt;` | 置空 |
| `o.has_value()` | 是否有值 |
| `o.value()` | 获取值（无值时抛异常） |
| `o.value_or(default)` | 获取值或默认值 |
| `*o` | 获取值（无值时未定义） |
| `o.reset()` | 清除值 |
| `o.emplace(args...)` | 原地构造值 |

```cpp
optional<string> find_user(int id) {
    if (id == 1) return "Alice";
    return nullopt;
}
auto name = find_user(1).value_or("Unknown");
```

### std::variant (C++17)

| 项目 / Item | 内容 |
|---|---|
| **头文件 / Header** | `<variant>` |

| 操作 / Operation | 说明 / Description |
|---|---|
| `variant<int, string, double> v = 42;` | 构造 |
| `holds_alternative<int>(v)` | 是否持有指定类型 |
| `get<int>(v)` | 获取值（类型不匹配抛异常） |
| `get_if<int>(&v)` | 获取指针（类型不匹配返回 nullptr） |
| `visit(visitor, v)` | 类型安全访问 |
| `v.index()` | 当前类型的索引 |

```cpp
variant<int, string> v = "hello";
visit([](auto&& arg) { cout << arg; }, v);
```

### std::any

| 项目 / Item | 内容 |
|---|---|
| **头文件 / Header** | `<any>` |

| 操作 / Operation | 说明 / Description |
|---|---|
| `any a = 42;` | 存储任意类型 |
| `a = "hello"s;` | 更改类型 |
| `any_cast<int>(a)` | 获取值（类型不匹配抛异常） |
| `any_cast<int>(&a)` | 安全获取指针 |
| `a.has_value()` | 是否有值 |
| `a.reset()` | 清除 |
| `a.type()` | type_info |

---

## <tuple> 元组 / Tuples

| 项目 / Item | 内容 |
|---|---|
| **头文件 / Header** | `<tuple>` |

| 操作 / Operation | 说明 / Description |
|---|---|
| `auto t = make_tuple(1, "hi", 3.14);` | 创建元组 |
| `get<0>(t)` | 获取第 0 个元素 |
| `tuple_element<0,decltype(t)>::type` | 第 0 个元素类型 |
| `tuple_size<decltype(t)>::value` | 元素个数 |
| `tie(a, b, c) = t;` | 解包到变量 |
| `tuple_cat(t1, t2)` | 连接元组 |
| `apply(fn, t)` | 将元组作为参数调用函数 (C++17) |
| `forward_as_tuple(args...)` | 转发引用元组 |

### 结构化绑定 (C++17) / Structured Bindings

```cpp
auto [x, y, z] = make_tuple(1, "hi", 3.14);
auto& [key, value] = *map.begin(); // 引用绑定
```

---

## <filesystem> 文件系统 (C++17) / Filesystem (C++17)

| 项目 / Item | 内容 |
|---|---|
| **头文件 / Header** | `<filesystem>` |
| **命名空间** | `std::filesystem` |

### path / 路径

| 操作 / Operation | 说明 / Description |
|---|---|
| `path p("/usr/local/bin");` | 构造路径 |
| `p /= "app";` | 追加路径组件 |
| `p.filename()` | 文件名: "app" |
| `p.extension()` | 扩展名: ".txt" |
| `p.stem()` | 文件名（不含扩展名） |
| `p.parent_path()` | 父路径 |
| `p.string()` | 字符串表示 |
| `p.c_str()` | C 字符串 |
| `p.is_absolute()` | 是否绝对路径 |

### 文件操作 / File Operations

| 操作 / Operation | 说明 / Description |
|---|---|
| `exists(p)` | 是否存在 |
| `is_directory(p)` | 是否是目录 |
| `is_regular_file(p)` | 是否是普通文件 |
| `file_size(p)` | 文件大小（字节） |
| `last_write_time(p)` | 最后修改时间 |
| `create_directory(p)` | 创建目录 |
| `create_directories(p)` | 递归创建目录（类似 mkdir -p） |
| `remove(p)` | 删除文件或空目录 |
| `remove_all(p)` | 递归删除（类似 rm -rf） |
| `rename(old, new)` | 重命名/移动 |
| `copy(src, dst)` | 复制文件 |
| `current_path()` | 当前工作目录 |
| `temp_directory_path()` | 临时目录 |
| `space(p)` | 磁盘空间信息 |

### 遍历目录 / Directory Iteration

```cpp
for (auto& entry : directory_iterator("/tmp")) {
    cout << entry.path() << "\n";
}
for (auto& entry : recursive_directory_iterator("/home")) {
    if (entry.is_directory()) continue;
    cout << entry.path() << " (" << entry.file_size() << " bytes)\n";
}
```

---

## <format> 格式化 (C++20) / Format (C++20)

| 项目 / Item | 内容 |
|---|---|
| **头文件 / Header** | `<format>` (C++20) / `<print>` (C++23) |

| 函数 / Function | 说明 / Description |
|---|---|
| `format("Hello {}", name)` | 格式化字符串 |
| `format_to(out, "...", args...)` | 格式化到输出迭代器 |
| `format_to_n(out, n, "...", args...)` | 格式化最多 n 个字符 |
| `println("Hello {}", name)` | 格式化并输出到 stdout (C++23) |

### 格式说明符 / Format Specifiers

| 说明符 / Specifier | 说明 / Description | 示例 |
|---|---|---|
| `{}` | 默认格式 | `format("{}", 42)` → "42" |
| `{:d}` | 十进制整数 | |
| `{:f}` | 浮点 | `format("{:f}", 3.14)` → "3.140000" |
| `{:.2f}` | 保留2位小数 | `format("{:.2f}", 3.14159)` → "3.14" |
| `{:x}` | 十六进制小写 | `format("{:x}", 255)` → "ff" |
| `{:X}` | 十六进制大写 | |
| `{:o}` | 八进制 | |
| `{:b}` | 二进制 (C++23) | |
| `{:s}` | 字符串 | |
| `{:p}` | 指针 | |
| `{:e}` | 科学计数 | |
| `{:>10}` | 右对齐，宽10 | |
| `{:<10}` | 左对齐 | |
| `{:^10}` | 居中 | |
| `{:0>5}` | 零填充右对齐 | `format("{:0>5}", 42)` → "00042" |
| `{:+d}` | 显示正负号 | `format("{:+d}", 42)` → "+42" |

---

## <regex> 正则表达式 / Regular Expressions

| 项目 / Item | 内容 |
|---|---|
| **头文件 / Header** | `<regex>` |

| 类型 / Type | 说明 / Description |
|---|---|
| `regex` | 正则表达式（char） |
| `wregex` | 正则表达式（wchar_t） |
| `smatch` | 匹配结果（string） |
| `cmatch` | 匹配结果（const char*） |

| 操作 / Operation | 说明 / Description |
|---|---|
| `regex_match(s, m, r)` | 全串匹配 |
| `regex_search(s, m, r)` | 搜索匹配 |
| `regex_replace(s, r, fmt)` | 替换匹配 |

### 正则语法 / Regex Syntax

| 语法 / Syntax | 说明 / Description |
|---|---|
| `.` | 任意字符（除换行） |
| `^` / `$` | 行首/行尾 |
| `*` / `+` / `?` | 0+ / 1+ / 0或1 |
| `{n,m}` | n 到 m 次 |
| `[abc]` / `[^abc]` | 字符集/取反 |
| `\d` / `\w` / `\s` | 数字/单词字符/空白 |
| `\D` / `\W` / `\S` | 取反 |
| `( )` | 捕获组 |
| `\|` | 或 |
| `\b` | 单词边界 |
| `(?P<name>...)` | 命名捕获 (C++11) |

```cpp
regex r("\\d{4}-\\d{2}-\\d{2}");
smatch m;
if (regex_search("Date: 2024-01-15", m, r)) {
    cout << m[0]; // 2024-01-15
}
```

---

## <iostream> / <fstream> / <sstream> 流 / IO Streams

### 标准流 / Standard Streams

| 流 / Stream | 说明 / Description |
|---|---|
| `cin` | 标准输入 |
| `cout` | 标准输出 |
| `cerr` | 标准错误（无缓冲） |
| `clog` | 标准错误（有缓冲） |
| `wcin` / `wcout` | 宽字符版本 |

### 文件流 / File Streams

| 类型 / Type | 说明 / Description |
|---|---|
| `ifstream` | 输入文件流 |
| `ofstream` | 输出文件流 |
| `fstream` | 输入输出文件流 |

```cpp
ifstream fin("input.txt");
ofstream fout("output.txt");
string line;
while (getline(fin, line)) {
    fout << line << "\n";
}
```

### 字符串流 / String Streams

| 类型 / Type | 说明 / Description |
|---|---|
| `istringstream` | 输入字符串流（读） |
| `ostringstream` | 输出字符串流（写） |
| `stringstream` | 双向字符串流 |

```cpp
ostringstream oss;
oss << "Value: " << 42 << ", PI: " << 3.14;
string s = oss.str(); // "Value: 42, PI: 3.14"

istringstream iss("10 20 30");
int a, b, c;
iss >> a >> b >> c;
```

### 操纵符 / Manipulators

| 操纵符 / Manipulator | 说明 / Description | 示例 |
|---|---|---|
| `setw(n)` | 设置宽度（仅下一次有效） | `cout << setw(10) << 42;` |
| `setprecision(n)` | 设置精度 | `cout << setprecision(2) << 3.14;` |
| `fixed` | 定点表示 | `cout << fixed << 3.14;` |
| `scientific` | 科学计数 | |
| `hex` / `oct` / `dec` | 进制 | `cout << hex << 255;` |
| `boolalpha` | bool 输出 true/false | `cout << boolalpha << true;` |
| `left` / `right` | 对齐方式 | |
| `endl` | 换行并刷新 | |
| `ws` | 跳过空白（输入） | `cin >> ws;` |

---

## <stdexcept> 异常 / Exceptions

| 项目 / Item | 内容 |
|---|---|
| **头文件 / Header** | `<stdexcept>` / `<exception>` |

| 异常类 / Exception | 基类 / Base | 说明 / Description |
|---|---|---|
| `exception` | - | 所有标准异常基类，`what()` 返回描述 |
| `logic_error` | exception | 逻辑错误（可预防） |
| `invalid_argument` | logic_error | 无效参数 |
| `domain_error` | logic_error | 域错误（数学） |
| `length_error` | logic_error | 超出最大长度 |
| `out_of_range` | logic_error | 越界 |
| `runtime_error` | exception | 运行时错误（不可预防） |
| `range_error` | runtime_error | 范围错误 |
| `overflow_error` | runtime_error | 溢出 |
| `underflow_error` | runtime_error | 下溢 |
| `bad_alloc` | exception | 内存分配失败 (`<new>`) |
| `bad_cast` | exception | dynamic_cast 失败 (`<typeinfo>`) |
| `bad_typeid` | exception | typeid 应用于空指针 |

```cpp
try {
    throw invalid_argument("age must be positive");
} catch (const exception& e) {
    cerr << "Error: " << e.what() << endl;
}
```

---

## <type_traits> 类型特性 / Type Traits

| 项目 / Item | 内容 |
|---|---|
| **头文件 / Header** | `<type_traits>` |

### 类型查询 / Type Queries

| 特性 / Trait | 说明 / Description |
|---|---|
| `is_integral_v<T>` | 是否整数类型 |
| `is_floating_point_v<T>` | 是否浮点类型 |
| `is_pointer_v<T>` | 是否指针 |
| `is_array_v<T>` | 是否数组 |
| `is_class_v<T>` | 是否类类型 |
| `is_enum_v<T>` | 是否枚举 |
| `is_same_v<T, U>` | 是否相同类型 |
| `is_base_of_v<Base, Derived>` | 是否基类 |
| `is_convertible_v<From, To>` | 是否可转换 |
| `is_const_v<T>` / `is_volatile_v<T>` | 是否 const/volatile |
| `is_reference_v<T>` | 是否引用 |
| `is_nothrow_constructible_v<T, Args...>` | 是否 noexcept 构造 |

### 类型变换 / Type Transformations

| 特性 / Trait | 说明 / Description |
|---|---|
| `remove_const_t<T>` | 移除 const |
| `remove_reference_t<T>` | 移除引用 |
| `add_pointer_t<T>` | 添加指针 |
| `decay_t<T>` | 衰退（数组→指针，函数→指针，去 cv） |
| `enable_if_t<cond, T>` | 条件启用类型 |
| `conditional_t<cond, T, F>` | 条件选择 |
| `void_t<Types...>` | void 类型 (C++17) |

### 常用模式 / Common Patterns

```cpp
// SFINAE 约束 / SFINAE constraint
template<typename T, enable_if_t<is_integral_v<T>, int> = 0>
void process(T val) { /* ... */ }

// 编译期 if / Compile-time if (C++17)
if constexpr (is_pointer_v<T>) {
    cout << *val;
} else {
    cout << val;
}
```

---

## <utility> 通用工具 / Utility

| 项目 / Item | 内容 |
|---|---|
| **头文件 / Header** | `<utility>` |

| 操作 / Operation | 说明 / Description |
|---|---|
| `pair<T,U>` | 二元组 |
| `make_pair(a, b)` | 创建 pair |
| `move(x)` | 转为右值引用 |
| `forward<T>(x)` | 完美转发 |
| `swap(a, b)` | 交换两个值 |
| `exchange(obj, new_val)` | 替换并返回旧值 (C++14) |
| `index_sequence<Ns...>` | 编译期整数序列 |
| `make_index_sequence<N>` | 创建 0..N-1 序列 (C++14) |
| `in_place` | 原地构造标签 (C++17) |
| `in_place_type<T>` | 原地构造指定类型 |
| `in_place_index<I>` | 原地构造指定索引 |

```cpp
auto p = make_pair(1, "hello");
auto [a, b] = p; // 结构化绑定
```

---

## <bitset> 位集 / Bitsets

| 项目 / Item | 内容 |
|---|---|
| **头文件 / Header** | `<bitset>` |

| 操作 / Operation | 说明 / Description |
|---|---|
| `bitset<8> b(42);` | 从整数构造（42 = 00101010） |
| `bitset<8> b("1010");` | 从字符串构造 |
| `b.set()` | 全部置 1 |
| `b.set(pos)` | 指定位置 1 |
| `b.reset()` | 全部置 0 |
| `b.reset(pos)` | 指定位置 0 |
| `b.flip()` | 全部取反 |
| `b.flip(pos)` | 指定位取反 |
| `b.test(pos)` | 测试指定位 |
| `b[pos]` | 访问指定位 |
| `b.count()` | 1 的个数 |
| `b.size()` | 总位数 |
| `b.any()` | 是否有 1 |
| `b.none()` | 是否全 0 |
| `b.all()` | 是否全 1 |
| `b.to_string()` | 转为字符串 |
| `b.to_ulong()` / `to_ullong()` | 转为整数 |
| `b & b2` / `b \| b2` / `b ^ b2` / `~b` | 位运算 |

```cpp
bitset<8> flags;
flags.set(2);
flags.set(5);
cout << flags; // 00100100
cout << flags.count(); // 2
```

---

## <valarray> 数值数组 / Numeric Arrays

| 项目 / Item | 内容 |
|---|---|
| **头文件 / Header** | `<valarray>` |
| **说明 / Description** | 为数值计算优化的数组容器。Array container optimized for numeric computation. |

| 操作 / Operation | 说明 / Description |
|---|---|
| `valarray<double> v(10, 5);` | 5 个 10.0 |
| `v + v2` / `v * v2` | 逐元素运算 |
| `v.sum()` | 求和 |
| `v.min()` / `v.max()` | 最小/最大值 |
| `v.shift(n)` | 逻辑移位 |
| `v.cshift(n)` | 循环移位 |
| `v.apply(fn)` | 逐元素应用函数 |
| `v[slice(1, 3, 2)]` | 切片：从索引1开始，3个元素，步长2 |

---

# 开发经验与调优 (Development Tips / 开发经验与调优)

---

## 编译优化 / Compiler Optimization

### 优化级别 / Optimization Levels

| 级别 / Level | 说明 / Description |
|---|---|
| `-O0` | 无优化，最快编译，利于调试 |
| `-O1` | 基本优化，减小代码大小 |
| `-O2` | **推荐** 平衡速度与大小 |
| `-O3` | 激进优化（向量化、循环展开），可能增大代码 |
| `-Os` | 优化大小（嵌入式常用） |
| `-Og` | 调试友好优化（GCC） |

### 链接时优化 (LTO) / Link-Time Optimization

```bash
# 编译和链接都加 -flto
gcc -O2 -flto -c a.c
gcc -O2 -flto -c b.c
gcc -O2 -flto a.o b.o -o prog

# ThinLTO (Clang，更快)
clang -O2 -flto=thin ...
```

### PGO / Profile-Guided Optimization

```bash
# 1. 编译插桩版本
gcc -O2 -fprofile-generate -c prog.c
gcc -O2 -fprofile-generate prog.o -o prog

# 2. 运行收集 profile
./prog  # 运行典型工作负载

# 3. 用 profile 重新编译
gcc -O2 -fprofile-use -c prog.c
gcc -O2 -fprofile-use prog.o -o prog
```

---

## 内存优化 / Memory Optimization

### 小对象优化 (SSO) / Small String Optimization

- `std::string` 通常内联存储 15-22 字节（取决于实现），避免短字符串的堆分配
- `std::string` typically stores 15-22 bytes inline to avoid heap allocation for short strings

### reserve 避免重分配 / Avoid Reallocation

```cpp
vector<int> v;
v.reserve(1000); // 预分配，避免多次重分配
for (int i = 0; i < 1000; ++i)
    v.push_back(i);
```

### emplace_back vs push_back

```cpp
vector<pair<int,int>> v;
v.push_back(make_pair(1, 2));   // 构造临时对象再移动
v.emplace_back(1, 2);           // 原地构造，更高效
```

### move 语义 / Move Semantics

```cpp
string a = "hello";
string b = a;       // 拷贝
string c = move(a); // 移动，a 变为有效但未指定状态
```

### 自定义分配器 / Custom Allocator

```cpp
template<typename T>
struct PoolAllocator {
    using value_type = T;
    PoolAllocator() noexcept {}
    template<typename U> PoolAllocator(const PoolAllocator<U>&) noexcept {}
    T* allocate(size_t n) { /* 从内存池分配 */ }
    void deallocate(T* p, size_t) noexcept { /* 归还内存池 */ }
};
vector<int, PoolAllocator<int>> v;
```

---

## 并发调优 / Concurrency Tuning

### 线程池设计模式 / Thread Pool Pattern

```cpp
class ThreadPool {
    vector<thread> workers;
    queue<function<void()>> tasks;
    mutex queue_mutex;
    condition_variable cv;
    bool stop;
public:
    ThreadPool(size_t threads);
    template<class F> void enqueue(F&& f);
    ~ThreadPool(); // join 所有线程
};
```

### false sharing 避免 / Avoid False Sharing

```cpp
// 错误：两个原子变量在同一缓存行
struct Bad { atomic<int> a; atomic<int> b; };

// 正确：用 padding 分离
struct Good {
    alignas(64) atomic<int> a;
    alignas(64) atomic<int> b;
};
```

### 批量处理减少锁竞争 / Batch Processing

```cpp
// 不好的做法：每次操作都加锁
for (auto& item : items) {
    lock_guard<mutex> lk(m);
    process(item);
}

// 好的做法：批量加锁
{
    lock_guard<mutex> lk(m);
    for (auto& item : items) process(item);
}
```

---

## 调试技巧 / Debugging Tips

### Sanitizers

```bash
# 地址检查（内存错误）
gcc -g -fsanitize=address prog.c

# 未定义行为检查
gcc -g -fsanitize=undefined prog.c

# 线程竞争检查
gcc -g -fsanitize=thread prog.c

# 内存泄漏检查
valgrind --leak-check=full ./prog

# 多个 sanitizer 组合
gcc -g -fsanitize=address,undefined prog.c
```

### gdb 常用命令 / Common GDB Commands

| 命令 / Command | 说明 / Description |
|---|---|
| `break main` / `b main.c:42` | 设置断点 |
| `run` / `r` | 运行程序 |
| `next` / `n` | 下一行（不进入函数） |
| `step` / `s` | 下一行（进入函数） |
| `print x` / `p x` | 打印变量 |
| `print *arr@10` | 打印数组前10个元素 |
| `finish` | 运行到当前函数返回 |
| `watch var` | 设置观察点 |
| `backtrace` / `bt` | 打印调用栈 |
| `frame N` / `f N` | 切换栈帧 |
| `info threads` | 列出线程 |
| `thread N` | 切换线程 |
| `continue` / `c` | 继续运行 |

### Core Dump 分析

```bash
ulimit -c unlimited  # 启用 core dump
./prog               # 崩溃后生成 core 文件
gdb ./prog core      # 分析
(gdb) bt             # 查看崩溃位置
(gdb) info frame     # 当前帧信息
(gdb) print var      # 查看变量
```

---

## 代码风格 / Code Style

### clang-format 常用配置 / Common Configuration

```yaml
# .clang-format
BasedOnStyle: Google
IndentWidth: 4
ColumnLimit: 100
AllowShortFunctionsOnASingleLine: Inline
AllowShortIfStatementsOnASingleLine: false
BreakBeforeBraces: Attach
```

### clang-tidy 常用检查 / Common Checks

```bash
clang-tidy src.cpp --checks='modernize-*,bugprone-*,performance-*,readability-*'
```

### 命名规范 / Naming Conventions（建议）

| 类型 / Type | 风格 / Style | 示例 |
|---|---|---|
| 变量/函数 | snake_case | `my_variable`, `get_value()` |
| 类/结构体 | PascalCase | `MyClass`, `TcpConnection` |
| 常量/宏 | UPPER_SNAKE_CASE | `MAX_SIZE`, `PI` |
| 成员变量 | m_ 前缀或 _ 后缀 | `m_count`, `count_` |
| 模板参数 | PascalCase 或单字母 | `typename T`, `typename Iterator` |

---

## 平台差异 / Platform Differences

### Windows vs Linux

| 项目 / Feature | Windows | Linux/macOS |
|---|---|---|
| 换行符 / Line ending | `\r\n` (CRLF) | `\n` (LF) |
| 路径分隔符 / Path separator | `\` | `/` |
| 动态库 / Shared lib | `.dll` + `.lib` | `.so` / `.dylib` |
| 静态库 / Static lib | `.lib` | `.a` |
| 大小写敏感 / Case sensitivity | 不敏感 | 敏感 |
| `size_t` | 32/64 位 | 64 位 |

### 大端 vs 小端 / Endianness

```cpp
// 检测字节序 / Detect endianness
uint32_t val = 0x01020304;
uint8_t *bytes = (uint8_t*)&val;
if (bytes[0] == 0x04) cout << "Little Endian\n";
else cout << "Big Endian\n";
```

### 32 位 vs 64 位

| 项目 / Feature | 32 位 | 64 位 |
|---|---|---|
| 指针大小 | 4 字节 | 8 字节 |
| `long` | 4 字节 (Windows/Linux) | 4 字节 (Windows) / 8 字节 (Linux) |
| `size_t` | 4 字节 | 8 字节 |
| `int` | 4 字节 | 4 字节 |

> 跨平台建议：使用 `<cstdint>` 定宽类型而非 `int`/`long`。

---

> 📖 本文档基于 C11 / C++20 标准，部分 API 标注了引入版本。
> This document is based on C11/C++20 standards. API introduction versions are noted where applicable.
