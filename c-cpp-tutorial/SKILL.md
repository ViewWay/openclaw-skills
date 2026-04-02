# C/C++ 开发实战教程

> 中文为主，关键术语附英文。风格参考 c.biancheng.net，侧重实战经验和性能调优。
> 每个主题遵循：**概念 → 代码示例 → 常见错误 → 最佳实践**。

---

# 第一部分：C 语言开发实战

---

## 一、C 语言入门

### 1.1 第一个 C 程序

```c
#include <stdio.h>  // 预处理：引入标准 I/O 头文件

int main(void) {    // 程序入口，返回 int
    printf("Hello, World!\n");  // 打印到标准输出
    return 0;       // 返回 0 表示成功
}
```

**解析：**
- `#include` 是预处理指令，在编译前将头文件内容插入
- `main` 是程序入口，标准签名 `int main(void)` 或 `int main(int argc, char *argv[])`
- `printf` 是标准库函数，需要 `<stdio.h>`
- `\n` 是转义字符，表示换行

**编译运行：**
```bash
gcc -Wall -o hello hello.c && ./hello
```

### 1.2 编译流程

C 源码到可执行文件经历四个阶段：

```
源文件(.c) → [预处理] → [编译] → [汇编] → [链接] → 可执行文件
```

| 阶段 | gcc 参数 | 输出 | 说明 |
|------|---------|------|------|
| 预处理 | `-E` | `.i` | 展开宏、处理 `#include`、删除注释 |
| 编译 | `-S` | `.s` | 生成汇编代码 |
| 汇编 | `-c` | `.o` | 生成目标文件（机器码） |
| 链接 | （默认） | 可执行文件 | 合并目标文件和库 |

```bash
gcc -E hello.c -o hello.i     # 预处理
gcc -S hello.i -o hello.s     # 编译为汇编
gcc -c hello.s -o hello.o     # 汇编为目标文件
gcc hello.o -o hello          # 链接为可执行文件
```

### 1.3 gcc 编译常用选项

```bash
# 基础编译
gcc -c main.c              # 只编译不链接，生成 main.o
gcc -o myapp main.c        # 指定输出文件名

# 警告（实战中务必开启）
gcc -Wall main.c           # 开启常见警告
gcc -Wextra main.c         # 额外警告
gcc -Werror main.c         # 警告视为错误
gcc -Wpedantic main.c      # 严格 ISO C 标准

# 调试
gcc -g main.c              # 生成调试信息（gdb 用）
gcc -O0 main.c             # 不优化（调试用）
gcc -DDEBUG main.c         # 定义宏 DEBUG

# 优化
gcc -O1                    # 基本优化
gcc -O2                    # 推荐生产环境
gcc -O3                    # 激进优化（可能引入问题）
gcc -Os                    # 优化代码大小

# 标准版本
gcc -std=c11 main.c        # 指定 C11 标准
gcc -std=c99 main.c        # 指定 C99 标准

# 包含路径和库
gcc -I./include main.c     # 添加头文件搜索路径
gcc -L./lib -lm -lpthread  # 添加库路径和链接库
```

### 1.4 Makefile 基础

```makefile
# 变量定义
CC = gcc
CFLAGS = -Wall -Wextra -g -O2
LDFLAGS = -lm
TARGET = myapp
SRCS = main.c utils.c parser.c
OBJS = $(SRCS:.c=.o)

# 默认目标
all: $(TARGET)

# 链接
$(TARGET): $(OBJS)
	$(CC) $(CFLAGS) -o $@ $^ $(LDFLAGS)

# 编译（自动规则）
%.o: %.c
	$(CC) $(CFLAGS) -c $< -o $@

# 清理
clean:
	rm -f $(OBJS) $(TARGET)

# 依赖头文件
main.o: main.h utils.h
utils.o: utils.h
```

**常见错误：**
- Makefile 缩进必须用 **Tab**，不能用空格
- 忘记清理 `.o` 文件导致链接旧目标文件

### 1.5 IDE 选择

| IDE | 优点 | 缺点 | 推荐场景 |
|-----|------|------|---------|
| VS Code | 轻量、插件丰富、跨平台 | 需要手动配置调试 | 日常开发首选 |
| CLion | CMake 深度集成、智能提示 | 收费、较重 | 大型 C++ 项目 |
| VS | 调试强大、Windows 生态 | 仅 Windows、体积大 | Windows 平台开发 |
| Vim/Neovim | 高效、服务器可用 | 学习曲线陡 | 服务器/终端开发 |

---

## 二、数据类型深入

### 2.1 整数类型对比

| 类型 | 位数（典型） | 范围 | printf 格式 |
|------|-------------|------|-------------|
| `char` | 8 | -128 ~ 127 或 0 ~ 255 | `%hhd` / `%hhu` |
| `short` | 16 | -32768 ~ 32767 | `%hd` |
| `int` | 32 | -2³¹ ~ 2³¹-1 | `%d` |
| `long` | 32/64 | 平台相关 | `%ld` |
| `long long` | 64 | -2⁶³ ~ 2⁶³-1 | `%lld` |
| `size_t` | 64（64位平台） | 无符号，表示大小 | `%zu` |

> **最佳实践：** 需要固定大小时用 `<stdint.h>` 的 `int32_t`、`uint64_t` 等。

### 2.2 浮点类型对比

| 类型 | 位数 | 有效数字 | 范围（绝对值） | printf 格式 |
|------|------|---------|---------------|-------------|
| `float` | 32 | 6~7 位 | ~3.4e38 | `%f` / `%g` |
| `double` | 64 | 15~16 位 | ~1.8e308 | `%lf` / `%g` |
| `long double` | 80/128 | 18~19+ 位 | 平台相关 | `%Lf` |

**常见错误：**
```c
// ❌ 用 == 比较浮点数
if (a == 0.3) { ... }

// ✅ 用 eps 比较
#include <math.h>
#define EPS 1e-9
if (fabs(a - 0.3) < EPS) { ... }
```

### 2.3 有符号 vs 无符号陷阱

```c
// ❌ 有符号溢出是未定义行为（UB）
int a = INT_MAX;
int b = a + 1;  // UB！结果不可预测

// ✅ 无符号溢出是取模（定义良好）
unsigned int ua = UINT_MAX;
unsigned int ub = ua + 1;  // 结果为 0，合法

// ❌ 混合运算的陷阱
int x = -1;
unsigned int y = 1;
if (x < y) {  // ⚠️ 结果为 false！x 被隐式转为 unsigned
    printf("x < y\n");  // 不会执行
}
```

**最佳实践：**
- 除非需要取模语义，否则不要用 `unsigned` 做算术
- `size_t` 用于大小和索引，混合运算时注意隐式转换
- 开启 `-Wsign-compare` 警告

### 2.4 sizeof 与内存对齐

```c
struct Foo {
    char   a;   // 1 字节
    int    b;   // 4 字节
    short  c;   // 2 字节
};

// 实际布局（64位系统，默认对齐）：
// [a][pad][pad][pad][bbbb][cc][pad][pad] = 12 字节
printf("%zu\n", sizeof(struct Foo));  // 输出 12（或 8，取决于平台）

// 紧凑排列：取消对齐
struct __attribute__((packed)) FooPacked {
    char   a;   // 1
    int    b;   // 4
    short  c;   // 2
};
// [a][bbbb][cc] = 7 字节
printf("%zu\n", sizeof(struct FooPacked));  // 输出 7
```

**最佳实践：**
- 按大小降序排列成员，减少 padding
- 网络协议等需要精确布局时用 `packed`
- 对齐访问更快（CPU 友好）

### 2.5 枚举

```c
// C 风格枚举（本质是 int）
enum Color { RED = 0, GREEN = 1, BLUE = 2 };
enum Color c = RED;  // 实际上是 int，可以赋任意 int 值

// 更好的做法：用前缀避免命名冲突
enum ErrorCode {
    ERR_NONE = 0,
    ERR_FILE_NOT_FOUND,
    ERR_PERMISSION_DENIED,
    ERR_OUT_OF_MEMORY
};
```

---

## 三、指针精通

### 3.1 指针基础

```c
int a = 42;
int *p = &a;       // p 存储 a 的地址
printf("%d\n", *p); // 解引用：输出 42
*p = 100;           // 通过指针修改 a
```

### 3.2 const 指针（面试高频）

```c
const int *p1;        // 指向 const int，不能通过 *p1 修改值，但 p1 可以指向别的
int *const p2 = &a;   // const 指针，p2 不能指向别的，但可以通过 *p2 修改值
const int *const p3 = &a;  // 都不能改

// 记忆技巧：const 在 * 左边修饰值，在 * 右边修饰指针
```

### 3.3 函数指针与回调

```c
// 函数指针声明
int (*cmp_func)(const void *, const void *);

// 使用 qsort 的经典场景
int cmp_int(const void *a, const void *b) {
    return (*(int *)a - *(int *)b);
}

int arr[] = {5, 2, 8, 1, 9};
qsort(arr, 5, sizeof(int), cmp_int);  // 传递函数名即函数指针
```

**函数指针数组：**
```c
typedef void (*handler_t)(int);
handler_t handlers[256] = {0};  // 256 个函数指针，类似中断向量表
handlers['A'] = handle_add;
handlers['D'] = handle_delete;
handlers[ch] = handlers[ch] ? handlers[ch] : handle_default;
```

### 3.4 动态内存分配

```c
// malloc 分配未初始化内存
int *p = (int *)malloc(10 * sizeof(int));
if (p == NULL) {  // 必须检查！
    fprintf(stderr, "malloc failed\n");
    exit(EXIT_FAILURE);
}

// calloc 分配并清零
int *q = (int *)calloc(10, sizeof(int));

// realloc 调整大小
int *r = (int *)realloc(p, 20 * sizeof(int));
if (r == NULL) {
    free(p);  // realloc 失败时原指针仍有效，需要手动释放
    exit(EXIT_FAILURE);
}
// p 现在可能已失效，使用 r

free(r);  // 释放内存
r = NULL; // 最佳实践：释放后置 NULL
```

### 3.5 常见指针 Bug

```c
// 1. 悬垂指针（Dangling Pointer）
int *p = (int *)malloc(sizeof(int));
free(p);
*p = 42;  // ❌ 使用已释放内存，UB

// 2. 野指针（Wild Pointer）
int *q;     // 未初始化
*q = 42;    // ❌ UB，指向随机地址

// 3. 内存泄漏（Memory Leak）
void leak() {
    int *p = (int *)malloc(sizeof(int));
    *p = 42;
    // 忘记 free(p)，每次调用泄漏 4 字节
}

// 4. 双重释放（Double Free）
int *r = (int *)malloc(sizeof(int));
free(r);
free(r);  // ❌ UB，可能导致堆损坏
```

**最佳实践：**
- `free` 后立即置 `NULL`：`free(p); p = NULL;`
- 使用 `valgrind` 检测内存问题：`valgrind --leak-check=full ./myapp`
- 每个 `malloc` 都要有对应的 `free`（谁分配谁释放）

---

## 四、数组与字符串

### 4.1 字符串操作陷阱

```c
char src[] = "hello";
char dst[10];

// ❌ strcpy 不检查目标缓冲区大小
strcpy(dst, src);  // 如果 src 长度 >= 10，缓冲区溢出

// ✅ 安全替代方案
snprintf(dst, sizeof(dst), "%s", src);  // 推荐方式
```

**strlen vs sizeof 陷阱：**
```c
char s1[] = "hello";
char *s2 = "hello";

sizeof(s1);  // 6（包含 '\0'，因为 s1 是数组）
strlen(s1);  // 5（不包含 '\0'）
sizeof(s2);  // 8（指针大小，64位系统）
strlen(s2);  // 5
```

### 4.2 字符串与数字转换

```c
// ❌ atoi 无法检测错误
int n = atoi("123abc");  // 返回 123，无错误提示

// ✅ strtol 可以检测错误
char *endptr;
long val = strtol("123abc", &endptr, 10);
if (*endptr != '\0') {
    printf("转换失败：'%s' 不是纯数字\n", endptr);
}

// 数字转字符串
char buf[32];
snprintf(buf, sizeof(buf), "%d", 42);
```

---

## 五、结构体与联合

### 5.1 结构体内存对齐

```c
// 按大小降序排列减少 padding
struct Good {
    double d;    // 8
    int    i;    // 4
    short  s;    // 2
    char   c;    // 1
};  // 总大小 16（无 padding 浪费）

struct Bad {
    char   c;    // 1 + 3 padding
    double d;    // 8
    short  s;    // 2 + 2 padding
    int    i;    // 4
};  // 总大小 24（浪费 8 字节 padding）
```

### 5.2 大小端转换（union 技巧）

```c
union EndianTest {
    uint32_t i;
    uint8_t  c[4];
};

void check_endian(void) {
    union EndianTest test;
    test.i = 0x01020304;
    if (test.c[0] == 0x04) {
        printf("小端序（Little Endian）\n");
    } else {
        printf("大端序（Big Endian）\n");
    }
}
```

---

## 六、预处理器

### 6.1 宏函数陷阱

```c
// ❌ 常见宏函数错误
#define SQUARE(x) x * x
SQUARE(3 + 1);  // 展开为 3 + 1 * 3 + 1 = 7，不是 16！

// ✅ 加括号
#define SQUARE(x) ((x) * (x))
SQUARE(3 + 1);  // ((3+1)*(3+1)) = 16 ✓

// ❌ 副作用问题
#define MAX(a, b) ((a) > (b) ? (a) : (b))
int i = 5, j = 10;
int m = MAX(i++, j++);  // i++ 被执行两次！
// ✅ 用 inline 函数代替
static inline int max_int(int a, int b) {
    return a > b ? a : b;
}
```

### 6.2 X-Macro 技巧（消除重复代码）

```c
// 定义消息类型列表（只维护一份）
#define MSG_TABLE \
    X(MSG_LOGIN,    "LOGIN") \
    X(MSG_LOGOUT,   "LOGOUT") \
    X(MSG_DATA,     "DATA") \
    X(MSG_HEARTBEAT,"HEARTBEAT")

// 自动生成枚举
typedef enum {
    #define X(id, str) id,
    MSG_TABLE
    #undef X
    MSG_COUNT
} MsgType;

// 自动生成字符串数组
static const char *msg_names[] = {
    #define X(id, str) str,
    MSG_TABLE
    #undef X
};

// 使用
printf("消息类型: %s\n", msg_names[MSG_LOGIN]);  // "LOGIN"
```

### 6.3 调试宏

```c
#define LOG(fmt, ...) \
    fprintf(stderr, "[%s:%d %s] " fmt "\n", \
            __FILE__, __LINE__, __func__, ##__VA_ARGS__)

LOG("value = %d, name = %s", 42, "test");
// 输出：[main.c:25 main] value = 42, name = test
```

---

## 七、文件操作

### 7.1 配置文件解析（INI 格式）

```c
// 简单 INI 解析器
int parse_ini(const char *filename, void (*callback)(const char *key, const char *val)) {
    FILE *f = fopen(filename, "r");
    if (!f) return -1;

    char line[512];
    while (fgets(line, sizeof(line), f)) {
        // 去除注释和空白行
        char *p = strchr(line, '#');
        if (p) *p = '\0';
        // 去除尾部空白
        size_t len = strlen(line);
        while (len > 0 && isspace((unsigned char)line[len-1])) line[--len] = '\0';
        if (len == 0) continue;

        char *eq = strchr(line, '=');
        if (eq) {
            *eq = '\0';
            char *key = line;
            char *val = eq + 1;
            // 去除 key 和 val 首尾空白
            while (*key && isspace((unsigned char)*key)) key++;
            while (*val && isspace((unsigned char)*val)) val++;
            callback(key, val);
        }
    }
    fclose(f);
    return 0;
}
```

---

## 八、链表

### 8.1 单链表核心操作

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
        if (slow == fast) return 1;  // 有环
    }
    return 0;  // 无环
}

// 安全释放整个链表
void free_list(Node *head) {
    while (head) {
        Node *tmp = head->next;
        free(head);
        head = tmp;
    }
}
```

---

## 九、排序算法

### 9.1 qsort 使用

```c
#include <stdlib.h>

int cmp_desc(const void *a, const void *b) {
    return (*(int *)b - *(int *)a);  // 降序
}

int arr[] = {5, 2, 8, 1, 9, 3};
int n = sizeof(arr) / sizeof(arr[0]);
qsort(arr, n, sizeof(int), cmp_desc);

// 按字符串长度排序
int cmp_strlen(const void *a, const void *b) {
    return strlen(*(const char **)a) - strlen(*(const char **)b);
}
```

### 9.2 排序算法选择指南

| 场景 | 推荐算法 | 时间复杂度 |
|------|---------|-----------|
| 通用排序 | `qsort`（快排变体） | O(n log n) 平均 |
| 基本有序 | 插入排序 | O(n) ~ O(n²) |
| 范围小的整数 | 计数排序 | O(n + k) |
| 稳定排序 | 归并排序 | O(n log n) |
| 实时系统 | 堆排序 | O(n log n)，最坏也是 |

---

## 十、位运算

### 10.1 常用位操作

```c
// 置位（Set bit n）
#define SET_BIT(x, n)   ((x) |= (1U << (n)))

// 清位（Clear bit n）
#define CLEAR_BIT(x, n) ((x) &= ~(1U << (n)))

// 取反（Toggle bit n）
#define TOGGLE_BIT(x, n) ((x) ^= (1U << (n)))

// 判断（Check bit n）
#define CHECK_BIT(x, n)  (((x) >> (n)) & 1U)

// 判断奇偶
if (x & 1) { /* 奇数 */ }

// 交换两个数（不用临时变量）
a ^= b; b ^= a; a ^= b;

// 快速乘除 2 的幂
x << 1;   // x * 2
x >> 1;   // x / 2
```

---

## 十一、C 语言项目实战经验

### 11.1 模块化设计

```
project/
├── include/          # 公共头文件
│   ├── utils.h
│   └── parser.h
├── src/              # 源文件
│   ├── main.c
│   ├── utils.c
│   └── parser.c
├── tests/            # 测试
│   └── test_utils.c
├── Makefile
└── README.md
```

**头文件守卫：**
```c
#ifndef UTILS_H
#define UTILS_H

// 声明，不定义

#endif // UTILS_H

// 或更简洁：
#pragma once
```

### 11.2 错误处理策略

```c
// 推荐：返回错误码 + 输出参数
typedef enum {
    ERR_OK = 0,
    ERR_NULL_PTR,
    ERR_OUT_OF_RANGE,
    ERR_IO
} ErrorCode;

ErrorCode divide(int a, int b, double *result) {
    if (result == NULL) return ERR_NULL_PTR;
    if (b == 0) return ERR_OUT_OF_RANGE;
    *result = (double)a / b;
    return ERR_OK;
}

// 使用
double res;
ErrorCode err = divide(10, 3, &res);
if (err != ERR_OK) {
    LOG("错误: %d", err);
}
```

### 11.3 内存泄漏检测

```bash
# 编译时加 -g
gcc -g -o myapp main.c

# 使用 valgrind 检测
valgrind --leak-check=full --show-leak-kinds=all ./myapp

# AddressSanitizer（更快，编译期检测）
gcc -fsanitize=address -g -o myapp main.c
./myapp
```

---

# 第二部分：C++ 开发实战

---

## 一、C++ 入门

### 1.1 C vs C++ 区别

| 特性 | C | C++ |
|------|---|-----|
| 编程范式 | 过程式 | 多范式（过程/面向对象/泛型） |
| 内存管理 | malloc/free | new/delete + 智能指针 |
| 字符串 | char[] + string.h | std::string |
| 错误处理 | 返回值/errno | 异常 + 错误码 |
| 泛型 | void* / 宏 | 模板（template） |
| 函数重载 | ❌ | ✅ |
| 命名空间 | ❌ | ✅ |

### 1.2 编译

```bash
g++ -std=c++20 -Wall -Wextra -O2 -o app main.cpp
```

### 1.3 引用（Reference）

```c++
int a = 42;
int &ref = a;      // 必须初始化，不能为 NULL
ref = 100;          // a 也变成 100

// const 引用可以绑定到右值
const int &r = 42;  // 合法

// 实战：参数传递
void swap(int &a, int &b) {  // 用引用避免拷贝
    int tmp = a; a = b; b = tmp;
}
```

### 1.4 constexpr（编译期计算）

```cpp
constexpr int factorial(int n) {
    return n <= 1 ? 1 : n * factorial(n - 1);
}
constexpr int val = factorial(5);  // 编译期就算好了，值为 120
static_assert(val == 120);         // 编译期断言
```

---

## 二、面向对象

### 2.1 Rule of Five

```cpp
class Buffer {
public:
    // 1. 构造函数
    explicit Buffer(size_t size) : size_(size), data_(new char[size]) {}

    // 2. 虚析构函数
    virtual ~Buffer() { delete[] data_; }

    // 3. 拷贝构造（深拷贝）
    Buffer(const Buffer &other) : size_(other.size_), data_(new char[other.size_]) {
        std::memcpy(data_, other.data_, size_);
    }

    // 4. 拷贝赋值
    Buffer &operator=(const Buffer &other) {
        if (this != &other) {  // 自赋值检查
            delete[] data_;
            size_ = other.size_;
            data_ = new char[size_];
            std::memcpy(data_, other.data_, size_);
        }
        return *this;
    }

    // 5. 移动构造
    Buffer(Buffer &&other) noexcept : size_(other.size_), data_(other.data_) {
        other.size_ = 0;
        other.data_ = nullptr;
    }

    // 6. 移动赋值
    Buffer &operator=(Buffer &&other) noexcept {
        if (this != &other) {
            delete[] data_;
            size_ = other.size_;
            data_ = other.data_;
            other.size_ = 0;
            other.data_ = nullptr;
        }
        return *this;
    }

private:
    size_t size_;
    char *data_;
};
```

**最佳实践：**
- 如果定义了析构函数/拷贝构造/拷贝赋值中的任何一个，通常五个都要定义
- 如果不需要拷贝和移动：`= delete`（Rule of Zero）
- 优先用 `std::vector`/`std::string` 管理内存，自动满足 Rule of Zero

---

## 三、继承与多态

### 3.1 虚函数与 vtable

```cpp
class Animal {
public:
    virtual void speak() const { std::cout << "...\n"; }
    virtual ~Animal() = default;  // 虚析构必须！
};

class Dog : public Animal {
public:
    void speak() const override { std::cout << "汪汪！\n"; }
};

// 多态使用
Animal *a = new Dog();
a->speak();     // 输出 "汪汪！"（运行时绑定）
delete a;       // 正确调用 Dog 的析构函数
```

**虚析构为什么必须：**
```cpp
// ❌ 没有虚析构
class Base { public: ~Base() { std::cout << "~Base\n"; } };
class Derived : public Base { public: ~Derived() { std::cout << "~Derived\n"; } };

Base *p = new Derived();
delete p;  // 只调用 ~Base()！Derived 的资源泄漏！

// ✅ 加 virtual
class Base { public: virtual ~Base() { std::cout << "~Base\n"; } };
// delete p; 会先调用 ~Derived() 再调用 ~Base()
```

---

## 四、模板编程

### 4.1 函数模板与类模板

```cpp
// 函数模板
template<typename T>
T max_val(T a, T b) {
    return (a > b) ? a : b;
}

// 类模板
template<typename T, size_t N>
class Stack {
public:
    void push(const T &val) {
        if (top_ < N) data_[top_++] = val;
    }
    T pop() { return data_[--top_]; }
    bool empty() const { return top_ == 0; }
private:
    T data_[N];
    size_t top_ = 0;
};

Stack<int, 1024> s;  // 最大 1024 个 int
```

### 4.2 C++20 Concepts

```cpp
#include <concepts>

// 定义 concept
template<typename T>
concept Numeric = std::integral<T> || std::floating_point<T>;

// 约束模板
template<Numeric T>
T add(T a, T b) {
    return a + b;
}

// 或者用 requires
template<typename T>
    requires std::integral<T>
T multiply(T a, T b) { return a * b; }
```

---

## 五、STL 容器实战

### 5.1 容器选择指南

```
需要随机访问？
├── 是 → vector（默认首选）/ deque
└── 否
    ├── 频繁头部插入删除？→ deque/list
    ├── 需要 key-value 查找？→ map/unordered_map
    ├── 需要自动排序？→ set/map
    ├── 需要去重？→ set
    └── 其他 → vector（通常还是最快）
```

### 5.2 vector 迭代器失效

```cpp
std::vector<int> v = {1, 2, 3, 4, 5};

// ❌ 在遍历时删除元素
for (auto it = v.begin(); it != v.end(); ++it) {
    if (*it == 3) {
        v.erase(it);  // it 失效！
    }
}

// ✅ erase 返回下一个有效迭代器
for (auto it = v.begin(); it != v.end(); ) {
    if (*it == 3) {
        it = v.erase(it);
    } else {
        ++it;
    }
}

// ✅ C++20 std::erase_if（更简洁）
std::erase_if(v, [](int x) { return x == 3; });
```

### 5.3 map vs unordered_map

```cpp
std::map<std::string, int> ordered;       // 红黑树，有序，O(log n)
std::unordered_map<std::string, int> hash; // 哈希表，无序，平均 O(1)

// 选择标准：
// 需要有序 → map
// 只需要查找 → unordered_map（通常更快）
// 需要 lower_bound/upper_bound → map
// key 是自定义类型且没有好哈希 → map

// 自定义 key（需要提供 < 或 hash）
struct Person {
    std::string name;
    int age;
    bool operator<(const Person &o) const { return name < o.name; }
};
std::map<Person, std::string> contacts;
```

---

## 六、STL 算法实战

### 6.1 常用模式

```cpp
std::vector<int> v = {5, 2, 8, 2, 1, 8, 3};

// 去重（先排序再 unique）
std::sort(v.begin(), v.end());
v.erase(std::unique(v.begin(), v.end()), v.end());
// v = {1, 2, 3, 5, 8}

// 查找满足条件的元素
auto it = std::find_if(v.begin(), v.end(), [](int x) { return x > 4; });

// 二分查找（需要已排序）
bool found = std::binary_search(v.begin(), v.end(), 5);
auto lb = std::lower_bound(v.begin(), v.end(), 3);  // 第一个 >= 3

// 累积
int sum = std::accumulate(v.begin(), v.end(), 0);
// 带初始值和自定义操作
int product = std::accumulate(v.begin(), v.end(), 1, std::multiplies<int>());

// transform
std::vector<int> doubled(v.size());
std::transform(v.begin(), v.end(), doubled.begin(), [](int x) { return x * 2; });
```

---

## 七、智能指针实战

### 7.1 选择指南

```
需要独占所有权？→ unique_ptr（默认选择）
需要共享所有权？→ shared_ptr（小心循环引用）
需要观察但不拥有？→ weak_ptr / 裸指针（非拥有）
```

### 7.2 unique_ptr

```cpp
// 创建
auto p = std::make_unique<int>(42);

// 自定义删除器（资源管理）
auto file_closer = [](FILE *f) { if (f) fclose(f); };
std::unique_ptr<FILE, decltype(file_closer)> fp(fopen("test.txt", "r"), file_closer);

// 工厂模式（返回 unique_ptr）
std::unique_ptr<Animal> create_animal(const std::string &type) {
    if (type == "dog") return std::make_unique<Dog>();
    if (type == "cat") return std::make_unique<Cat>();
    return nullptr;
}
// 使用
auto pet = create_animal("dog");
pet->speak();
```

### 7.3 shared_ptr 循引用问题

```cpp
struct Node {
    std::shared_ptr<Node> next;
    std::weak_ptr<Node>   prev;  // 用 weak_ptr 打破循环
    int value;
};

auto a = std::make_shared<Node>();
auto b = std::make_shared<Node>();
a->next = b;  // shared_ptr
b->prev = a;  // weak_ptr，不增加引用计数
```

**为什么 `make_unique`/`make_shared` 优于 `new`：**
- 异常安全（避免 new 后、智能指针构造前抛异常导致泄漏）
- 减少内存分配次数（shared_ptr 合并控制块和对象为一次分配）
- 性能更好

---

## 八、移动语义与完美转发

### 8.1 std::move

```cpp
std::string a = "hello";
std::string b = std::move(a);  // a 的资源"移动"到 b，a 变为有效但未指定状态
// a 现在可能为空，不要再使用 a
```

**关键理解：** `std::move` 本身不做任何移动，它只是将左值转换为右值引用（`static_cast<T&&>`），真正的移动由移动构造/赋值完成。

### 8.2 完美转发

```cpp
// 工厂函数：将参数完美转发给构造函数
template<typename T, typename... Args>
std::unique_ptr<T> make(Args &&... args) {
    return std::make_unique<T>(std::forward<Args>(args)...);
}

auto p = make<std::string>(5, 'a');  // 转发为 string(5, 'a')
```

---

## 九、Lambda 表达式

### 9.1 实战示例

```cpp
// C++14 泛型 lambda
auto print = [](const auto &x) { std::cout << x << " "; };
std::vector<int> v = {1, 2, 3};
std::for_each(v.begin(), v.end(), print);

// C++14 初始化捕获
std::unique_ptr<Widget> pw = std::make_unique<Widget>();
auto task = [w = std::move(pw)]() { w->do_something(); };

// mutable lambda（可修改捕获的值）
int count = 0;
auto counter = [count]() mutable { return ++count; };
counter();  // 1
counter();  // 2

// 作为线程参数
std::thread t([](int id) {
    std::cout << "Thread " << id << "\n";
}, 42);
t.join();
```

---

## 十、C++ 并发编程实战

### 10.1 基本模式

```cpp
#include <thread>
#include <mutex>
#include <condition_variable>
#include <future>

// 互斥锁
std::mutex mtx;
int shared_data = 0;

void safe_increment() {
    std::lock_guard<std::mutex> lock(mtx);  // RAII 自动加解锁
    ++shared_data;
}

// 生产者-消费者
std::queue<int> q;
std::mutex q_mtx;
std::condition_variable cv;
bool done = false;

void producer() {
    for (int i = 0; i < 10; ++i) {
        {
            std::lock_guard<std::mutex> lock(q_mtx);
            q.push(i);
        }
        cv.notify_one();
    }
    done = true;
    cv.notify_all();
}

void consumer() {
    while (true) {
        std::unique_lock<std::mutex> lock(q_mtx);
        cv.wait(lock, [] { return !q.empty() || done; });
        while (!q.empty()) {
            std::cout << q.front() << " ";
            q.pop();
        }
        if (done) break;
    }
}

// async / future
auto future = std::async(std::launch::async, [](int n) {
    int sum = 0;
    for (int i = 1; i <= n; ++i) sum += i;
    return sum;
}, 100);
int result = future.get();  // 阻塞等待结果
```

**死锁避免：**
- 始终以相同顺序获取多个锁
- 使用 `std::scoped_lock`（C++17）同时锁定多个互斥量
- 使用 `std::lock` + `std::lock_guard` 的 adopt_lock

---

## 十一、C++17/20/23 新特性实战

### 11.1 C++17 亮点

```cpp
// 结构化绑定
auto [key, value] = *map.begin();
auto [it, inserted] = map.insert({"key", 42});

// optional
std::optional<int> find_user(int id) {
    if (id == 0) return std::nullopt;
    return 42;
}
auto result = find_user(0);
if (result) { std::cout << *result; }

// string_view（零拷贝字符串引用）
void process(std::string_view sv) { /* 不分配内存 */ }
process("hello");
std::string s = "world";
process(s);

// if constexpr
template<typename T>
std::string to_string_impl(const T &val) {
    if constexpr (std::is_integral_v<T>) {
        return std::to_string(val);
    } else {
        return val;
    }
}

// 文件系统
namespace fs = std::filesystem;
for (auto &entry : fs::directory_iterator("/tmp")) {
    std::cout << entry.path() << " " << entry.file_size() << "\n";
}
```

### 11.2 C++20 亮点

```cpp
// Concepts
template<std::integral T>
T square(T x) { return x * x; }

// Ranges
#include <ranges>
auto result = std::views::filter(v, [](int x) { return x > 3; })
            | std::views::transform([](int x) { return x * 2; });

// std::format
#include <format>
std::string s = std::format("Hello, {}! You are {} years old.", name, age);
```

---

## 十二、C++ 项目实战经验

### 12.1 CMake 实战

```cmake
cmake_minimum_required(VERSION 3.20)
project(MyProject VERSION 1.0 LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 20)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)

# 添加编译选项
add_compile_options(-Wall -Wextra -Wpedantic)

# 库
add_library(utils src/utils.cpp)
target_include_directories(utils PUBLIC include)

# 可执行文件
add_executable(app src/main.cpp)
target_link_libraries(app PRIVATE utils)

# 测试（Google Test）
find_package(GTest REQUIRED)
add_executable(tests tests/test_main.cpp)
target_link_libraries(tests PRIVATE utils GTest::gtest_main)
enable_testing()
add_test(NAME AllTests COMMAND tests)
```

### 12.2 pimpl 惯用法（隐藏实现）

```cpp
// Widget.h
class Widget {
public:
    Widget();
    ~Widget();
    Widget(const Widget &) = delete;
    Widget &operator=(const Widget &) = delete;
    void do_something();
private:
    class Impl;         // 前置声明
    std::unique_ptr<Impl> pImpl;  // 指针，不完整类型
};

// Widget.cpp
#include "Widget.h"
class Widget::Impl {
public:
    void do_something() { /* 具体实现 */ }
    int internal_data = 42;
};

Widget::Widget() : pImpl(std::make_unique<Impl>()) {}
Widget::~Widget() = default;  // 需要 Impl 完整定义，所以放在 .cpp
void Widget::do_something() { pImpl->do_something(); }
```

**好处：** 修改实现不影响头文件，减少编译依赖，隐藏私有成员。

### 12.3 性能优化策略

```bash
# 1. 先测量，再优化
perf record ./myapp          # 采样分析
perf report                  # 查看热点

# 2. 编译器优化
gcc -O2 -march=native        # 针对当前 CPU 优化

# 3. 常见优化点
# - 减少 malloc/new 调用（对象池、预留空间）
# - 减少虚函数调用（CRTP 替代虚函数）
# - 避免不必要的拷贝（移动语义、引用传递）
# - 缓存友好（数据局部性、结构体数组 SoA vs AoS）
# - 并行化（多线程、SIMD）
```

---

## 十三、C/C++ 混合编程

```cpp
// C++ 代码
extern "C" {
    void c_function(int x);  // 声明 C 函数
}

// 在 C 中调用 C++ 函数：用 extern "C" 包裹
// cpp_wrapper.cpp
extern "C" void my_cpp_func(int x) {
    // C++ 实现
}

// c_wrapper.h（纯 C 头文件）
#ifdef __cplusplus
extern "C" {
#endif
void my_cpp_func(int x);
#ifdef __cplusplus
}
#endif
```

**常见链接错误：**
- `undefined reference` → 忘记链接对应的 `.o` 或库
- `multiple definition` → 头文件中定义了变量（应声明 `extern`）
- C++ name mangling 导致找不到符号 → 用 `extern "C"`

---

## 十四、C/C++ 面试高频题速查

| 问题 | 关键点 |
|------|--------|
| 指针 vs 引用 | 指针可为 NULL、可重指向；引用必须初始化、不可重指向 |
| malloc vs new | new 调用构造函数/析构函数，malloc 只分配内存 |
| struct vs class | C++ 中默认 public vs private；C 中无 class |
| 虚函数原理 | vptr 指向 vtable，每个类一张表，每个对象一个 vptr |
| 深拷贝 vs 浅拷贝 | 深拷贝复制指针指向的内容；浅拷贝只复制指针值 |
| 智能指针循环引用 | 用 weak_ptr 打破；shared_ptr 引用计数永远不为 0 |
| std::move 语义 | 只是 static_cast 到右值引用；不移动，只是允许移动 |
| 多线程安全 | 临界区保护共享数据；避免数据竞争 |
| 内存对齐 | CPU 按字访问效率高；struct 成员按大小对齐；可 packed |
| 大端小端 | 网络传输需统一字节序（网络序=大端）；用 htonl/ntohl 转换 |

---

## 附录：常用编译/调试命令速查

```bash
# 编译
gcc -Wall -Wextra -g -O2 -std=c11 -o app main.c
g++ -Wall -Wextra -g -O2 -std=c++20 -o app main.cpp

# 调试
gdb ./app                    # 启动 gdb
gdb -tui ./app              # gdb TUI 模式
lldb ./app                  # macOS 默认调试器

# 内存检查
valgrind --leak-check=full ./app
gcc -fsanitize=address -g -o app main.c && ./app

# 性能分析
perf record -g ./app
perf report
gprof ./app gmon.out

# 静态分析
cppcheck --enable=all .
clang-tidy main.cpp
```
