# C 语言基础 (C Fundamentals)

> 来源：c-programming/SKILL.md + c-cpp-tutorial/SKILL.md 的 C 部分整合
> 覆盖范围：数据类型、指针、数组、结构体、内存管理、预处理器、链接、文件操作、链表、位运算

---

## 1. 数据类型

### 基本类型

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

// char 可能 signed 也可能 unsigned（平台相关）
signed char   sc = -1;
unsigned char uc = 255;
```

### 整数类型对比

| 类型 | 位数（典型） | 范围 | printf 格式 |
|------|-------------|------|-------------|
| `char` | 8 | -128 ~ 127 或 0 ~ 255 | `%hhd` / `%hhu` |
| `short` | 16 | -32768 ~ 32767 | `%hd` |
| `int` | 32 | -2³¹ ~ 2³¹-1 | `%d` |
| `long` | 32/64 | 平台相关 | `%ld` |
| `long long` | 64 | -2⁶³ ~ 2⁶³-1 | `%lld` |
| `size_t` | 64（64位平台） | 无符号 | `%zu` |

### 浮点类型对比

| 类型 | 位数 | 有效数字 | printf 格式 |
|------|------|---------|-------------|
| `float` | 32 | 6~7 位 | `%f` / `%g` |
| `double` | 64 | 15~16 位 | `%lf` / `%g` |
| `long double` | 80/128 | 18~19+ 位 | `%Lf` |

### 陷阱
- `int` 不一定是 32 位（DSP 上常见 16 位）
- `char` 的符号性是实现定义的，**不要用 char 做算术**
- `size_t` 是无符号的，`size_t - 1` 不会变负，会回绕到极大值
- `int` 溢出是 **UB**，但 `unsigned int` 溢出是定义良好的（模 2^N）
- 有符号/无符号混合比较：`int i = -1; if (i < (unsigned)5)` → false

### 最佳实践
- 优先使用 `int32_t`/`uint64_t` 等精确类型
- 表示大小/长度用 `size_t`，表示索引用 `size_t` 或 `ssize_t`
- 表示字节用 `uint8_t`，不要用 `char` 做字节操作

---

## 2. 运算符优先级

### 经典陷阱

```c
// << 优先级低于 +
int x = 1 << 2 + 3;    // 1 << (2+3) = 32，不是 (1<<2)+3 = 7
int y = (1 << 2) + 3;  // 7

// == 高于 &
if (flags & FLAG_MASK == 0)  // → if (flags & (FLAG_MASK == 0))
if ((flags & FLAG_MASK) == 0)  // 正确

// 参数求值顺序未指定
int a = 1;
printf("%d %d\n", a, ++a);  // 可能输出 "1 2" 或 "2 2"，都是合法的
```

### 最佳实践
- 不确定优先级时**加括号**
- 不要在同一个表达式中对同一变量读和写
- 用 `-Wall -Wextra -Wpedantic` 让编译器警告

---

## 3. 指针

### 核心概念

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

// 指针算术（以指向类型大小为单位）
int arr[5] = {10, 20, 30, 40, 50};
int *p = arr;           // p 指向 arr[0]
int *q = p + 3;         // q 指向 arr[3]
ptrdiff_t d = q - p;    // d == 3
```

### 函数指针与回调

```c
typedef void (*Callback)(int event, void *user_data);

void register_handler(Callback cb, void *user_data) { /* ... */ }

void my_handler(int event, void *data) {
    printf("event %d, data: %s\n", event, (char*)data);
}
register_handler(my_handler, "context");
```

### 陷阱
- `int *p = malloc(5 * sizeof(int));` 然后 `p[5]` → 越界
- 函数指针声明难读 → 用 `typedef`
- `NULL` 和 `0` 在函数重载时可能混淆（C++ 中用 `nullptr`）
- 解引用未初始化指针 → UB
- 指针算术只能在**同一数组**内进行，否则 UB

### 最佳实践
- 声明指针时**立即初始化**，哪怕是 `NULL`
- 释放后将指针置 `NULL`，避免悬垂指针
- 复杂指针声明用 [cdecl.org](https://cdecl.org) 解读

---

## 4. 数组与字符串

```c
char str[] = "hello";
char *ptr  = "hello";    // ptr 指向字符串字面量（不可修改）

sizeof(str);   // 6（包含 '\0'）
sizeof(ptr);   // 8（64 位指针大小）
strlen(str);   // 5（不含 '\0'）

// 数组作为函数参数时退化为指针
void foo(int arr[]) { }    // 等价于 void foo(int *arr)

// 安全字符串操作
char dst[16];
snprintf(dst, sizeof(dst), "%s", src);  // 最推荐
strlcpy(dst, src, sizeof(dst));          // BSD/macOS

// 字符串转数字（安全方式）
char *endptr;
long val = strtol("123abc", &endptr, 10);
if (*endptr != '\0') { /* 转换失败 */ }
```

### 陷阱
- `sizeof(arr)` 在函数参数中返回指针大小 — **经典 Bug**
- `strcpy` 不检查长度，用 `snprintf` 或 `strlcpy`
- `strlen` 是 O(n)，不要在循环条件中调用
- 字符串字面量存在只读段，`ptr[0] = 'H'` → 崩溃

### 最佳实践
- 函数接收数组时**同时传长度**
- 用 `sizeof(arr)/sizeof(arr[0])` 计算元素数（仅数组定义作用域内有效）
- `atoi` 无法检测错误，用 `strtol`

---

## 5. 结构体/联合体/位域

```c
// 内存对齐示例
struct Bad {
    char   c;    // 1 byte + 7 padding
    double d;    // 8 bytes
    char   e;    // 1 byte + 7 padding
};  // 24 bytes

struct Good {
    double d;    // 8 bytes
    char   c;    // 1 byte + 1 padding
    char   e;    // 1 byte + 6 padding
};  // 16 bytes

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
- 位域布局、对齐、大小端都是实现定义的，**不要跨平台传输位域**
- `sizeof(struct)` 可能大于成员大小之和
- 联合体类型双关在 C 中合法（C99+），但在 C++ 中除 char 外是 UB
- 位域不能取地址

### 最佳实践
- 按大小降序排列结构体成员，减少 padding
- 网络传输用明确的序列化（htonl/ntohl），不要直接发送结构体
- 用 `static_assert(sizeof(struct) == expected)` 验证布局

---

## 6. 内存管理

```c
// 基本用法
int *arr = malloc(10 * sizeof(int));
if (!arr) { /* 处理分配失败 */ }
memset(arr, 0, 10 * sizeof(int));
free(arr);
arr = NULL;

// calloc：分配并清零
int *zeros = calloc(10, sizeof(int));

// realloc：调整大小（可能移动内存）
int *tmp = realloc(arr, 20 * sizeof(int));
if (!tmp) { free(arr); return; }  // realloc 失败时原 arr 仍有效
arr = tmp;

// goto cleanup 模式（Linux 内核风格）
int func(void) {
    int *a = malloc(100);
    if (!a) return -1;
    int *b = malloc(200);
    if (!b) { free(a); return -1; }
    // ... 使用 a, b ...
    free(b); free(a);
    return 0;
}
```

### 陷阱
- **内存泄漏**：`malloc` 后忘记 `free`
- **悬垂指针**：`free(p)` 后继续用 `p`
- **双重释放**：`free(p); free(p);` → UB
- **`realloc` 失败处理**：不能直接 `arr = realloc(arr, size)`
- **`free(NULL)` 是安全的**（no-op）

### 最佳实践
- 每个分配对应一个释放，成对出现
- `free` 后立即置 `NULL`
- 用 valgrind 或 ASan 检测：`gcc -fsanitize=address -g prog.c`

---

## 7. 预处理器

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

// X-Macro 技巧
#define MSG_TABLE \
    X(MSG_LOGIN,    "LOGIN") \
    X(MSG_LOGOUT,   "LOGOUT") \
    X(MSG_DATA,     "DATA")

typedef enum {
    #define X(id, str) id,
    MSG_TABLE
    #undef X
    MSG_COUNT
} MsgType;

static const char *msg_names[] = {
    #define X(id, str) str,
    MSG_TABLE
    #undef X
};

// 头文件 guard
#pragma once  // 非标准但广泛支持

// 多语句宏
#define SWAP(a, b) do { \
    typeof(a) _tmp = (a); \
    (a) = (b); \
    (b) = _tmp; \
} while (0)
```

### 陷阱
- `#define SQUARE(x) x * x` → `SQUARE(a+1)` = `a+1 * a+1` ❌
- `#define MAX(a,b) (a>b?a:b)` → `MAX(i++, j++)` 多次求值 ❌
- 宏不遵守作用域，没有类型检查

### 最佳实践
- 简单常量用 `enum` 或 `static const` 代替宏
- 内联函数代替宏函数（`static inline`）

---

## 8. 链接

```c
// file.h
extern int global_counter;  // 声明，不分配存储
void increment(void);

// file.c
int global_counter = 0;     // 定义
static int local_state = 0; // 仅本文件可见
static void helper(void) {  // 仅本文件可见
    local_state++;
}
```

### 陷阱
- 头文件中定义变量（不加 `extern`）→ multiple definition
- `static` 全局变量在不同翻译单元中是**不同**的变量
- 常见错误：`undefined reference`、`multiple definition`

### 最佳实践
- 头文件只放声明，`.c` 文件放定义
- 内部函数/变量加 `static`

---

## 9. 链表

```c
typedef struct Node {
    int data;
    struct Node *next;
} Node;

// 反转链表（迭代法）
Node *reverse(Node *head) {
    Node *prev = NULL, *curr = head, *next = NULL;
    while (curr) {
        next = curr->next;
        curr->next = prev;
        prev = curr;
        curr = next;
    }
    return prev;
}

// 快慢指针检测环
int has_cycle(Node *head) {
    Node *slow = head, *fast = head;
    while (fast && fast->next) {
        slow = slow->next;
        fast = fast->next->next;
        if (slow == fast) return 1;
    }
    return 0;
}
```

---

## 10. 位运算

```c
#define SET_BIT(x, n)   ((x) |= (1U << (n)))
#define CLEAR_BIT(x, n) ((x) &= ~(1U << (n)))
#define TOGGLE_BIT(x, n) ((x) ^= (1U << (n)))
#define CHECK_BIT(x, n)  (((x) >> (n)) & 1U)

// 交换两个数（不用临时变量）
a ^= b; b ^= a; a ^= b;

// 快速乘除 2 的幂
x << 1;   // x * 2
x >> 1;   // x / 2
```

---

## 11. C 高级特性

### C11 泛型选择

```c
#define cbrt(x) _Generic((x), \
    long double: cbrtl, \
    default: cbrt, \
    float: cbrtf \
)(x)
```

### 可变参数

```c
#include <stdarg.h>
int sum(int count, ...) {
    va_list ap;
    va_start(ap, count);
    int total = 0;
    for (int i = 0; i < count; i++)
        total += va_arg(ap, int);
    va_end(ap);
    return total;
}
```

### restrict 关键字

```c
void add_arrays(int *restrict a, const int *restrict b, int n) {
    for (int i = 0; i < n; i++)
        a[i] += b[i];  // 编译器可以放心向量化
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

// ❌ volatile 不是锁！counter++ 在多线程中仍然不安全
```

### 寄存器映射

```c
typedef struct {
    volatile uint32_t MODER;
    volatile uint32_t OTYPER;
    volatile uint32_t IDR;
    volatile uint32_t ODR;
} GPIO_TypeDef;

#define GPIOA ((GPIO_TypeDef *)0x40020000UL)
GPIOA->ODR |= (1 << 5);  // PA5 置高
```

### ISR 规则
- ISR 中**不能调用**非可重入函数（如 `malloc`/`printf`）
- ISR 应尽可能短
- 共享变量用 `volatile`

---

## 13. 排序

```c
#include <stdlib.h>

int cmp_desc(const void *a, const void *b) {
    return (*(int *)b - *(int *)a);
}

int arr[] = {5, 2, 8, 1, 9};
int n = sizeof(arr) / sizeof(arr[0]);
qsort(arr, n, sizeof(int), cmp_desc);
```

| 场景 | 推荐算法 | 时间复杂度 |
|------|---------|-----------|
| 通用排序 | `qsort` | O(n log n) 平均 |
| 基本有序 | 插入排序 | O(n) ~ O(n²) |
| 稳定排序 | 归并排序 | O(n log n) |
| 实时系统 | 堆排序 | O(n log n)，最坏也是 |

---

## 14. 编译与调试建议

```bash
# 推荐编译选项
gcc -std=c11 -Wall -Wextra -Wpedantic -Werror -O2 -g -fsanitize=address,undefined prog.c

# valgrind 检测
valgrind --leak-check=full --track-origins=yes ./a.out

# 查看预处理/汇编
gcc -E prog.c    # 预处理
gcc -S -O2 prog.c # 汇编
```
