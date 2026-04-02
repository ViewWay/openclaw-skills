# C Programming Skill

专业级 C 语言开发参考。结构：概念 → 代码示例 → 陷阱/注意 → 最佳实践。

---

## 1. 数据类型

### 概念

C 的基本类型大小因平台而异，使用 `stdint.h` 获得精确宽度类型。

### 代码示例

```c
#include <stdint.h>
#include <stddef.h>

// 精确宽度类型（推荐）
int32_t  count  = 100;
uint64_t size   = 0xFFFFFFFFFF;
intptr_t ptr_as_int = (intptr_t)some_ptr;

// 平台相关类型
size_t    array_size = sizeof(arr);  // 无符号，足够容纳任何对象大小
ptrdiff_t diff = &arr[5] - &arr[2]; // 指针差值，有符号

// 注意 char 可能 signed 也可能 unsigned（平台相关）
signed char   sc = -1;
unsigned char uc = 255;
```

### 陷阱

- `int` 不一定是 32 位（DSP 上常见 16 位）
- `char` 的符号性是实现定义的，**不要用 char 做算术**
- `size_t` 是无符号的，`size_t - 1` 不会变负，会回绕到极大值
- `int` 溢出是 **UB**，但 `unsigned int` 溢出是定义良好的（模 2^N）

### 最佳实践

- 优先使用 `int32_t`/`uint64_t` 等精确类型
- 表示大小/长度用 `size_t`，表示索引用 `size_t` 或 `ssize_t`
- 表示字节用 `uint8_t`，不要用 `char` 做字节操作（符号问题）

---

## 2. 运算符优先级与求值顺序

### 概念

C 的运算符优先级有许多反直觉之处，且函数参数求值顺序是**未指定的**。

### 代码示例

```c
// 经典陷阱：<< 优先级低于 +
int x = 1 << 2 + 3;    // 1 << (2+3) = 32，不是 (1<<2)+3 = 7

// 正确写法
int y = (1 << 2) + 3;  // 7

// . 高于 *（解引用结构体指针）
struct Point { int x, y; };
struct Point *p = get_point();
int val = (*p).x;   // 笨拙
int val2 = p->x;    // 正确，-> 就是 (*p).x 的语法糖

// 但 *p.x 是 *(p.x)，不是 (*p).x
// int bad = *p.x;  // 编译错误（如果 p 是指针）

// 参数求值顺序未指定
int a = 1;
printf("%d %d\n", a, ++a);  // 可能输出 "1 2" 或 "2 2"，都是合法的
```

### 陷阱

- `==` 高于 `&`：`if (flags & FLAG_MASK == 0)` → `if (flags & (FLAG_MASK == 0))`
- `,`（逗号运算符）优先级最低之一，但函数参数的逗号不是逗号运算符
- `sizeof` 的操作数如果是 VLA，会在运行时求值
- 序列点（sequence point）之间修改同一变量多次 → UB

### 最佳实践

- 不确定优先级时**加括号**，可读性 > 省括号
- 不要在同一个表达式中对同一变量读和写
- 用 `-Wall -Wextra -Wpedantic` 让编译器警告优先级问题

---

## 3. 指针

### 概念

指针是 C 的灵魂。需要区分：指针数组 vs 数组指针、函数指针、多级指针、const 修饰。

### 代码示例

```c
// 指针数组 vs 数组指针
int *arr[10];       // 10 个元素的数组，每个元素是 int*
int (*parr)[10];    // 指向 10 个 int 数组的指针

// 函数指针
int compare(const void *a, const void *b) { return *(int*)a - *(int*)b; }
void (*cmp_ptr)(const void*, const void*) = compare;
qsort(arr, 10, sizeof(int), cmp_ptr);

// typedef 简化函数指针
typedef int (*Comparator)(const void*, const void*);
Comparator cmp = compare;

// const 指针（从右往左读）
const int *p1;        // *p1 是 const，不能通过 p1 修改值
int *const p2;        // p2 本身是 const，不能指向别处
const int *const p3;  // 都不能改

// 多级指针
char **argv;          // 字符串数组（命令行参数）
int ***matrix;        // 二维动态数组的指针

// 指针算术（以指向类型大小为单位）
int arr[5] = {10, 20, 30, 40, 50};
int *p = arr;           // p 指向 arr[0]
int *q = p + 3;         // q 指向 arr[3]，偏移 3 * sizeof(int) 字节
ptrdiff_t d = q - p;    // d == 3
```

### 陷阱

- `int *p = malloc(5 * sizeof(int));` 然后 `p[5]` → 越界（0-4 有效）
- 函数指针声明难读：`void (*signal(int, void (*)(int)))(int)` — 用 typedef
- `NULL` 和 `0` 在函数重载时可能混淆（C++ 中用 `nullptr`）
- 解引用未初始化指针 → UB，指针不赋值就是野指针
- 指针算术只能在**同一数组**内进行，否则 UB

### 最佳实践

- 声明指针时**立即初始化**，哪怕是 `NULL`
- 函数指针用 `typedef` 提高可读性
- 复杂指针声明用 [cdecl.org](https://cdecl.org) 解读
- 释放后将指针置 `NULL`，避免悬垂指针

---

## 4. 数组与字符串

### 概念

数组名在大多数表达式中退化为指向首元素的指针，但 `sizeof` 和 `&` 操作符除外。

### 代码示例

```c
char str[] = "hello";
char *ptr  = "hello";    // ptr 指向字符串字面量（不可修改）

sizeof(str);   // 6（包含 '\0'）
sizeof(ptr);   // 8（64 位指针大小）
strlen(str);   // 5（不含 '\0'）

// 数组作为函数参数时退化为指针
void foo(int arr[]) { }    // 等价于 void foo(int *arr)
void foo(int arr[10]) { }  // 10 被忽略！仍然是 int*

// 多维数组
int matrix[3][4];
// matrix[i][j] 等价于 *(*(matrix + i) + j)

// 字符串安全操作
char dst[16];
snprintf(dst, sizeof(dst), "%s", src);  // 安全，不会溢出

// strlcpy（BSD/macOS，非标准但推荐）
strlcpy(dst, src, sizeof(dst));
```

### 陷阱

- `sizeof(arr)` 在函数参数中返回指针大小，不是数组大小 — **经典 Bug**
- `strcpy` 不检查长度，用 `strncpy`（但不保证 \0 终止）或 `strlcpy`/`snprintf`
- `strlen` 是 O(n)，不要在循环条件中调用
- 字符串字面量 `"hello"` 存在只读段，`ptr[0] = 'H'` → 崩溃
- `"hello"[0]` 是合法的，返回 `'h'`（数组下标操作符）

### 最佳实践

- 函数接收数组时**同时传长度**
- 拷贝字符串用 `snprintf(dst, sizeof(dst), "%s", src)` 最安全
- 用 `sizeof(arr)/sizeof(arr[0])` 计算数组元素数（仅在数组定义的作用域内有效）
- 定义数组时用 `char buf[BUF_SIZE] = {0}` 初始化为零

---

## 5. 结构体/联合体/位域

### 概念

结构体内存布局受对齐（alignment）和填充（padding）影响。

### 代码示例

```c
// 内存对齐示例
struct Bad {
    char   c;    // 1 byte + 7 padding
    double d;    // 8 bytes
    char   e;    // 1 byte + 7 padding
};  // 总共 24 bytes（64 位系统）

struct Good {
    double d;    // 8 bytes
    char   c;    // 1 byte + 1 padding
    char   e;    // 1 byte + 6 padding
};  // 总共 16 bytes

// 查看对齐
#include <stddef.h>
printf("align: %zu\n", _Alignof(struct Bad));  // 8

// 位域
struct Flags {
    unsigned int readable  : 1;
    unsigned int writable  : 1;
    unsigned int executable: 1;
    unsigned int reserved  : 29;
};

// 联合体：节省内存或类型双关
union Value {
    int   i;
    float f;
    char  bytes[4];
};

union Value v;
v.i = 0x41424344;
// v.bytes[0] 在小端系统上是 'D'（0x44），大端系统上是 'A'（0x41）

// C11 匿名结构体/联合体
struct Config {
    union {
        struct { int x, y; };
        struct { int width, height; };
    };
};
struct Config c;
c.x = 10;        // 通过匿名成员访问
c.width = 800;   // 同一块内存
```

### 陷阱

- 位域的布局、对齐、大小端都是实现定义的，**不要跨平台传输位域**
- `sizeof(struct)` 可能大于成员大小之和（padding）
- 联合体类型双关在 C 中合法（C99+），但在 C++ 中除 char 外是 UB
- 位域不能取地址（`&(v.readable)` 非法）

### 最佳实践

- 按大小降序排列结构体成员，减少 padding
- 需要精确布局时用 `__attribute__((packed))` 或 `#pragma pack`（注意对齐惩罚）
- 用 `static_assert(sizeof(struct) == expected)` 验证布局
- 网络传输用明确的序列化（htonl/ntohl），不要直接发送结构体

---

## 6. 内存管理

### 概念

C 是手动内存管理。`malloc`/`free` 是基础，`calloc` 清零，`realloc` 调整大小。

### 代码示例

```c
// 基本用法
int *arr = malloc(10 * sizeof(int));
if (!arr) { /* 处理分配失败 */ }
memset(arr, 0, 10 * sizeof(int));  // 或用 calloc
free(arr);
arr = NULL;  // 防止悬垂指针

// calloc：分配并清零
int *zeros = calloc(10, sizeof(int));

// realloc：调整大小（可能移动内存）
int *tmp = realloc(arr, 20 * sizeof(int));
if (!tmp) {
    /* realloc 失败，原 arr 仍然有效！不要直接 free */
    free(arr);
    return;
}
arr = tmp;

// 二维数组动态分配
int **matrix = malloc(rows * sizeof(int*));
for (int i = 0; i < rows; i++) {
    matrix[i] = malloc(cols * sizeof(int));
}
// 释放
for (int i = 0; i < rows; i++) free(matrix[i]);
free(matrix);
```

### 陷阱

- **内存泄漏**：`malloc` 后忘记 `free`，或指针覆盖前没释放
- **悬垂指针**：`free(p)` 后继续用 `p`
- **双重释放**：`free(p); free(p);` → UB/崩溃
- **`realloc` 失败处理**：不能直接 `arr = realloc(arr, size)`，失败时原指针丢失
- **`free(NULL)` 是安全的**（no-op），但不要依赖这点写出混乱代码
- **对齐要求**：`malloc` 返回的内存对齐到 `max_align_t`

### 最佳实践

- 每个分配对应一个释放，成对出现（考虑 `goto cleanup` 模式）
- `free` 后立即置 `NULL`
- 用 valgrind 或 ASan 检测内存问题：`gcc -fsanitize=address -g prog.c`
- 考虑自定义分配器模式：`create_foo()` / `destroy_foo()` 封装 malloc/free

```c
// goto cleanup 模式（Linux 内核风格）
int func(void) {
    int *a = malloc(100);
    if (!a) return -1;
    int *b = malloc(200);
    if (!b) { free(a); return -1; }
    int *c = malloc(300);
    if (!c) { free(b); free(a); return -1; }

    // ... 使用 a, b, c ...

    free(c);
    free(b);
    free(a);
    return 0;
}
```

---

## 7. 预处理器

### 概念

宏是文本替换，不是函数调用。`#define`、条件编译、`X-Macro` 技巧。

### 代码示例

```c
// 宏函数 — 必须加括号！
#define MAX(a, b)  ((a) > (b) ? (a) : (b))
#define SQUARE(x)  ((x) * (x))

// 条件编译
#ifdef DEBUG
    #define LOG(fmt, ...) fprintf(stderr, "[DEBUG] " fmt "\n", ##__VA_ARGS__)
#else
    #define LOG(fmt, ...) ((void)0)
#endif

// X-Macro 技巧：用宏生成枚举和字符串数组
#define COLOR_LIST \
    X(RED,   "#FF0000") \
    X(GREEN, "#00FF00") \
    X(BLUE,  "#0000FF")

enum Color { COLOR_LIST };
const char *color_names[] = { COLOR_LIST };
const char *color_hex[]   = { COLOR_LIST };

// 头文件 guard（现代写法）
#pragma once  // 非标准但广泛支持

// 传统写法
#ifndef FOO_H
#define FOO_H
// ...
#endif
```

### 陷阱

- `#define SQUARE(x) x * x` → `SQUARE(a+1)` 展开为 `a+1 * a+1` = `a + (1*a) + 1` ❌
- `#define MAX(a,b) (a>b?a:b)` → `MAX(i++, j++)` 会多次求值 `i++` ❌
- 宏不遵守作用域，没有类型检查
- `#ifdef` 检查是否定义（不管值），`#if defined(X)` 也可以
- `__LINE__`、`__FILE__`、`__func__` 在调试中很有用

### 最佳实践

- 宏函数每个参数和整体都要加括号
- 简单常量用 `enum` 或 `static const` 代替宏
- 内联函数代替宏函数（`static inline`）
- 用 `do { ... } while(0)` 包裹多语句宏

```c
// 多语句宏的正确写法
#define SWAP(a, b) do { \
    typeof(a) _tmp = (a); \
    (a) = (b); \
    (b) = _tmp; \
} while (0)
```

---

## 8. 链接

### 概念

`static` 限制作用域到当前翻译单元，`extern` 声明外部符号。

### 代码示例

```c
// file.h
#ifndef FILE_H
#define FILE_H
extern int global_counter;  // 声明，不分配存储
void increment(void);
#endif

// file.c
#include "file.h"
int global_counter = 0;     // 定义，分配存储
static int local_state = 0; // 仅本文件可见

static void helper(void) {  // 仅本文件可见
    local_state++;
}

void increment(void) {
    global_counter++;
    helper();
}
```

### 陷阱

- 头文件中定义变量（不加 `extern`）→ 每个 include 的 .c 都有一份副本 → 链接时 multiple definition
- `static` 全局变量在不同翻译单元中是**不同**的变量
- 函数声明不加 `static` 就是 `extern` 的（默认外部链接）
- 常见链接错误：`undefined reference`（声明了没定义）、`multiple definition`（定义了多次）

### 最佳实践

- 头文件只放声明，`.c` 文件放定义
- 内部函数/变量加 `static`
- 用 `static inline` 代替头文件中的函数定义

---

## 9. C 语言陷阱汇总

### 未定义行为（UB）— 最危险的 Bug 源

```c
// 有符号整数溢出 → UB
int x = INT_MAX;
x + 1;  // UB！但 (unsigned)(x + 1) 可以

// 移位超过位数 → UB
int y = 1 << 33;  // 如果 int 是 32 位 → UB

// 空指针解引用 → UB
int *p = NULL;
*p = 42;  // 崩溃

// 越界访问 → UB
int arr[5];
arr[5] = 0;  // UB

// 未初始化变量使用 → UB（值不确定）
int z;
printf("%d", z);  // UB

// 悬垂指针 → UB
int *q = malloc(sizeof(int));
free(q);
*q = 10;  // UB：use-after-free

// double free → UB
free(q);
free(q);  // UB

// 修改字符串字面量 → UB
char *s = "hello";
s[0] = 'H';  // UB（可能 SIGSEGV）
```

### 常见 Bug 模式

```c
// off-by-one
int arr[10];
for (int i = 0; i <= 10; i++) arr[i] = 0;  // 越界！应该 i < 10

// strcmp 返回值判断
if (strcmp(s1, s2) == 0) { /* 相等 */ }   // 正确
if (strcmp(s1, s2)) { /* 不等 */ }          // 正确
if (strcmp(s1, s2) > 0) { /* s1 > s2 */ }  // 正确
// ❌ 不要假设返回值只有 -1, 0, 1

// sizeof 陷阱
void foo(int arr[]) {
    size_t n = sizeof(arr) / sizeof(arr[0]);  // ❌ sizeof(int*)/sizeof(int) = 2（64位）
}

// signed/unsigned 比较
int i = -1;
unsigned int u = 5;
if (i < u) { /* 不会执行！i 被隐式转为 unsigned → 巨大值 */ }

// 整数提升
unsigned char a = 200;
unsigned char b = 100;
if (a + b < 300) { /* 不会执行！a+b 被提升为 int = 300 */ }

// 忘记 \0 终止
char buf[5];
buf[0] = 'h'; buf[1] = 'e'; buf[2] = 'l'; buf[3] = 'l'; buf[4] = 'o';
strlen(buf);  // UB：没有 \0 终止
```

### 并发问题

```c
// volatile ≠ 原子！
volatile int counter = 0;
// 两个线程同时 counter++ → 竞态条件（read-modify-write 不是原子的）
// 正确做法：用 pthread_mutex 或 atomic 操作

// 数据竞争
int shared = 0;
// 线程 A: shared = 1;
// 线程 B: printf("%d", shared);
// 没有同步 → 数据竞争 → UB
```

---

## 10. C 标准库要点

### string.h

```c
// 安全拷贝
snprintf(dst, sizeof(dst), "%s", src);  // 最推荐
strlcpy(dst, src, sizeof(dst));          // BSD/macOS

// 内存操作
memcpy(dst, src, n);    // 不能处理重叠区域
memmove(dst, src, n);   // 可以处理重叠
memset(buf, 0, sizeof(buf));
```

### stdlib.h

```c
// 安全的字符串转数字
long val = strtol(str, &endptr, 10);
if (endptr == str || *endptr != '\0') { /* 转换失败 */ }
// 避免 atoi（无错误检查）

// qsort
int cmp(const void *a, const void *b) {
    return (*(int*)a - *(int*)b);  // 注意：可能溢出！
}
// 安全比较
int cmp_safe(const void *a, const void *b) {
    int ia = *(const int*)a, ib = *(const int*)b;
    return (ia > ib) - (ia < ib);
}
```

### stdio.h

```c
// 安全的格式化
char buf[64];
snprintf(buf, sizeof(buf), "value: %d", x);  // 自动截断，保证 \0 终止

// 二进制读写
FILE *f = fopen("data.bin", "wb");
fwrite(arr, sizeof(int), count, f);
fclose(f);
```

---

## 11. C 高级特性

### 函数指针与回调

```c
// 回调模式
typedef void (*Callback)(int event, void *user_data);

void register_handler(Callback cb, void *user_data) {
    // 存储 cb 和 user_data
}

// 使用
void my_handler(int event, void *data) {
    printf("event %d, data: %s\n", event, (char*)data);
}
register_handler(my_handler, "context");
```

### C11 泛型选择

```c
#include <math.h>

#define cbrt(x) _Generic((x), \
    long double: cbrtl, \
    default: cbrt, \
    float: cbrtf \
)(x)

// 用法
double d = cbrt(27.0);   // 调用 cbrt
float f = cbrt(27.0f);   // 调用 cbrtf
```

### 可变参数

```c
#include <stdarg.h>

int sum(int count, ...) {
    va_list ap;
    va_start(ap, count);
    int total = 0;
    for (int i = 0; i < count; i++) {
        total += va_arg(ap, int);
    }
    va_end(ap);
    return total;
}
```

### restrict 关键字

```c
// 告诉编译器指针不重叠，允许优化
void add_arrays(int *restrict a, const int *restrict b, int n) {
    for (int i = 0; i < n; i++) {
        a[i] += b[i];  // 编译器可以放心向量化，因为 a 和 b 不重叠
    }
}
```

---

## 12. 嵌入式 C

### volatile 正确用法

```c
// ✅ 硬件寄存器
volatile uint32_t *UART_DR = (volatile uint32_t *)0x40011000;

// ✅ ISR 与主程序共享变量
volatile bool flag = false;

void ISR_Handler(void) {
    flag = true;  // ISR 中设置
}

void main_loop(void) {
    while (!flag) { /* wait */ }
    flag = false;
}

// ❌ volatile 不是锁！
volatile int counter = 0;
// counter++ 在多线程中仍然不安全（非原子）
```

### 位操作

```c
// 置位 (Set Bit)
REG |= (1 << BIT_POS);

// 清位 (Clear Bit)
REG &= ~(1 << BIT_POS);

// 取反 (Toggle Bit)
REG ^= (1 << BIT_POS);

// 读-修改-写（注意：非原子！）
uint32_t val = *REG_ADDR;
val |= (1 << BIT_POS);
*REG_ADDR = val;
```

### 寄存器映射

```c
// 方式 1：宏定义
#define GPIOA_BASE   0x40020000UL
#define GPIOA_MODER  (*(volatile uint32_t *)(GPIOA_BASE + 0x00))
#define GPIOA_ODR    (*(volatile uint32_t *)(GPIOA_BASE + 0x14))

// 方式 2：结构体映射（注意对齐和 volatile）
typedef struct {
    volatile uint32_t MODER;   // 0x00
    volatile uint32_t OTYPER;  // 0x04
    volatile uint32_t OSPEEDR; // 0x08
    volatile uint32_t PUPDR;   // 0x0C
    volatile uint32_t IDR;     // 0x10
    volatile uint32_t ODR;     // 0x14
} GPIO_TypeDef;

#define GPIOA ((GPIO_TypeDef *)0x40020000UL)
GPIOA->ODR |= (1 << 5);  // PA5 置高
```

### ISR 规则

- ISR 中**不能调用**非可重入函数（如 `malloc`/`printf`）
- ISR 应尽可能短，复杂处理放到主循环
- 共享变量用 `volatile`
- 必要时禁用中断保护临界区

---

## 编译与调试建议

```bash
# 推荐编译选项
gcc -std=c11 -Wall -Wextra -Wpedantic -Werror -O2 -g -fsanitize=address,undefined prog.c

# valgrind 检测
valgrind --leak-check=full --track-origins=yes ./a.out

# 查看预处理器输出
gcc -E prog.c

# 查看汇编
gcc -S -O2 prog.c
```
