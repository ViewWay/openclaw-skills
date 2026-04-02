# C/C++ 面试高频题 (Interview Questions)

> 来源：c-cpp-tutorial/SKILL.md 的面试部分 + cpp-core-guidelines/SKILL.md 的关键规则
> 覆盖范围：指针、内存管理、OOP、模板、STL、并发、编译链接

---

## 1. 基础概念

### 指针 vs 引用

| 特性 | 指针 | 引用 |
|------|------|------|
| 可为空 | ✅ `nullptr` | ❌ 必须初始化 |
| 可重指向 | ✅ `p = &other` | ❌ |
| 需要解引用 | ✅ `*p` | ❌ 直接用 |
| 数组形式 | `int* arr[]` | `int& arr[]` ❌ |
| sizeof | 指针大小（8字节） | 绑定对象的大小 |

### malloc vs new

| 特性 | malloc | new |
|------|--------|-----|
| 语言 | C | C++ |
| 构造/析构 | 不调用 | 调用构造函数/析构函数 |
| 失败返回 | `NULL` | 抛出 `std::bad_alloc` |
| 大小 | `malloc(sizeof(T))` | `new T` |
| 重载 | 不可 | 可重载 `operator new` |
| 内存对齐 | `max_align_t` | `max_align_t` |

### struct vs class（C++）

唯一区别：默认访问权限。`struct` 默认 `public`，`class` 默认 `private`。

---

## 2. 内存管理

### 虚函数原理

- 每个有虚函数的类有一张 **vtable**（虚函数表）
- 每个对象有一个 **vptr**（虚表指针，指向类的 vtable）
- 虚函数调用通过 vptr 查表实现动态分派
- 代价：每个对象多一个指针（8字节），每次虚调用多一次间接寻址

### 深拷贝 vs 浅拷贝

```cpp
// 浅拷贝：只复制指针值
class Shallow {
    int* data;
    Shallow(const Shallow& o) : data(o.data) {}  // 共享同一块内存
};

// 深拷贝：复制指针指向的内容
class Deep {
    int* data;
    Deep(const Deep& o) : data(new int(*o.data)) {}  // 独立的内存
};
```

### 智能指针循环引用

```cpp
struct Node {
    std::shared_ptr<Node> next;
    std::shared_ptr<Node> prev;  // ❌ 循环引用，内存永不释放
};

// 解决
struct Node {
    std::shared_ptr<Node> next;
    std::weak_ptr<Node> prev;    // ✅ weak_ptr 不增加引用计数
};
```

### Rule of Five / Rule of Zero

- **Rule of Five**：如果定义了析构/拷贝构造/拷贝赋值中的一个，通常五个都要定义
- **Rule of Zero**：用智能指针和 STL，不需要手写任何特殊成员函数

---

## 3. OOP

### 虚析构为什么必须

```cpp
class Base { public: virtual ~Base() {} };    // ✅
class Base { public: ~Base() {} };            // ❌

Base* p = new Derived();
delete p;  // 有虚析构：先 ~Derived() 再 ~Base()
           // 无虚析构：只 ~Base()，Derived 资源泄漏
```

### 对象切片

```cpp
void process(Base b) { b.foo(); }   // ❌ 值传递，切片
void process(Base& b) { b.foo(); }  // ✅ 引用传递
void process(Base* b) { b->foo(); } // ✅ 指针传递
```

### 纯虚函数 vs 虚函数

- 纯虚函数 `= 0`：使类成为抽象类，派生类必须实现
- 虚函数：提供默认实现，派生类可以覆盖

### 多重继承与虚继承

```cpp
class A { public: int a; };
class B : public virtual A { public: int b; };
class C : public virtual A { public: int c; };
class D : public B, public C { };  // 只有一份 A
```

---

## 4. 模板

### 模板 vs 虚函数（多态）

| 特性 | 模板 | 虚函数 |
|------|------|--------|
| 多态类型 | 静态（编译期） | 动态（运行时） |
| 代码膨胀 | 每种类型一份 | 只有一份 |
| 性能 | 无虚表开销 | 有虚表开销 |
| 适用场景 | 类型安全、高性能 | 运行时多态 |

### SFINAE vs Concepts

```cpp
// SFINAE（C++17 前，难读）
template<typename T, std::enable_if_t<std::is_integral_v<T>, int> = 0>
void func(T val);

// Concepts（C++20，清晰）
template<std::integral T>
void func(T val);
```

---

## 5. STL

### vector 迭代器失效

```cpp
// push_back/insert：如果触发重分配，所有迭代器失效
// erase：被删元素及之后的迭代器失效
// map/set：仅被删元素的迭代器失效
// unordered_map：任何插入/删除都可能导致所有迭代器失效（rehash）
```

### map vs unordered_map

| 特性 | map | unordered_map |
|------|-----|---------------|
| 底层实现 | 红黑树 | 哈希表 |
| 查找 | O(log n) | 平均 O(1) |
| 有序 | ✅ | ❌ |
| 需要 `<` 运算符 | ✅ | 需要 `hash` |
| 内存开销 | 较大 | 较小 |

### vector vs deque vs list

| 特性 | vector | deque | list |
|------|--------|-------|------|
| 随机访问 | O(1) | O(1) | O(n) |
| 头部插入 | O(n) | O(1) | O(1) |
| 尾部插入 | 均摊 O(1) | O(1) | O(1) |
| 中间插入 | O(n) | O(n) | O(1) |
| 缓存友好 | ✅✅ | ✅ | ❌ |

---

## 6. 移动语义

### std::move 本质

```cpp
// std::move 只是 static_cast 到右值引用
template<typename T>
typename remove_reference<T>::type&& move(T&& t) noexcept {
    return static_cast<typename remove_reference<T>::type&&>(t);
}
// 它不做任何移动，真正的移动由移动构造/赋值完成
```

### 完美转发

```cpp
template<typename T>
void wrapper(T&& arg) {  // 万能引用（不是右值引用！）
    process(std::forward<T>(arg));  // 保持左值/右值属性
}
```

---

## 7. 并发

### 互斥锁 vs 原子操作

| 特性 | mutex | atomic |
|------|-------|--------|
| 适用范围 | 复杂临界区 | 简单操作（计数器等） |
| 开销 | 较大（系统调用） | 较小（CPU 指令） |
| 死锁风险 | 有 | 无 |
| 适用类型 | 任意 | 基本类型 |

### 条件变量为什么需要 while 循环

```cpp
// ❌ 虚假唤醒可能导致错误
cv.wait(lk);

// ✅ 带谓词防止虚假唤醒
cv.wait(lk, [] { return ready; });
```

### volatile vs atomic

```cpp
volatile int counter = 0;     // ❌ 不是线程安全的，counter++ 不是原子操作
std::atomic<int> counter{0};  // ✅ 线程安全
```

---

## 8. 编译链接

### 编译流程

```
源文件(.c/.cpp) → [预处理] → [编译] → [汇编] → [链接] → 可执行文件
```

### 头文件 vs 源文件

- 头文件（`.h`）：声明、内联函数、模板、常量
- 源文件（`.c/.cpp`）：定义、实现

### extern "C" 的作用

```cpp
// 告诉 C++ 编译器按 C 的方式链接（不进行 name mangling）
extern "C" {
    void c_function(int x);
}
```

### 常见链接错误

| 错误 | 原因 | 解决 |
|------|------|------|
| `undefined reference` | 声明了没定义 / 忘记链接 | 检查链接选项 |
| `multiple definition` | 头文件中定义了变量 | 加 `extern` 或 `inline` |
| 找不到 C++ 符号 | name mangling | 用 `extern "C"` |

---

## 9. 内存对齐

```c
struct Bad {
    char c;     // 1 + 7 padding
    double d;   // 8
    char e;     // 1 + 7 padding
};  // 24 bytes

struct Good {
    double d;   // 8
    char c;     // 1 + 1 padding
    char e;     // 1 + 6 padding
};  // 16 bytes
```

### 大端小端

```c
uint32_t val = 0x01020304;
uint8_t *bytes = (uint8_t*)&val;
// 小端：bytes[0] == 0x04（x86, ARM 默认）
// 大端：bytes[0] == 0x01（网络字节序）
// 转换：htonl/ntohl/htons/ntohs
```

---

## 10. C/C++ 区别

| 特性 | C | C++ |
|------|---|-----|
| 编程范式 | 过程式 | 多范式 |
| 内存管理 | malloc/free | new/delete + 智能指针 |
| 字符串 | char[] + string.h | std::string |
| 错误处理 | 返回值/errno | 异常 |
| 泛型 | void* / 宏 | 模板 |
| 函数重载 | ❌ | ✅ |
| 命名空间 | ❌ | ✅ |
| 类/继承 | ❌ | ✅ |
| 引用 | ❌ | ✅ |
| constexpr | C23 引入 | C++11 |
| auto | C23 引入 | C++11 |

---

## 11. 现代特性速问

### C++17 你用过什么？
- 结构化绑定、optional、variant、string_view、if constexpr、filesystem

### C++20 你知道什么？
- Concepts、Ranges、format、协程、modules、jthread、`<=>`

### RAII 是什么？
- 资源获取即初始化：构造时获取资源，析构时自动释放。C++ 资源管理的核心。

### 什么是对象切片？
- 派生类对象通过值传递给基类参数时，派生类部分被截断。

### 为什么不要用裸 new/delete？
- 异常不安全、容易忘记释放、不支持移动语义。用 make_unique/make_shared。

### 什么是 CRTP？
- 奇异递归模板模式：基类模板参数是派生类本身，用于静态多态。

---

## 12. 算法速查

### 排序算法复杂度

| 算法 | 平均 | 最坏 | 稳定 | 空间 |
|------|------|------|------|------|
| 快速排序 | O(n log n) | O(n²) | ❌ | O(log n) |
| 归并排序 | O(n log n) | O(n log n) | ✅ | O(n) |
| 堆排序 | O(n log n) | O(n log n) | ❌ | O(1) |
| 插入排序 | O(n²) | O(n²) | ✅ | O(1) |
| 计数排序 | O(n+k) | O(n+k) | ✅ | O(k) |

### 常见数据结构

| 结构 | 查找 | 插入 | 删除 | 适用 |
|------|------|------|------|------|
| 数组 | O(n) | O(n) | O(n) | 随机访问 |
| 链表 | O(n) | O(1) | O(1) | 频繁插入删除 |
| 哈希表 | O(1) | O(1) | O(1) | 快速查找 |
| 二叉搜索树 | O(log n) | O(log n) | O(log n) | 有序数据 |
| 红黑树 | O(log n) | O(log n) | O(log n) | 平衡有序 |
