# C++ 核心特性 (C++ Fundamentals)

> 来源：cpp-programming/SKILL.md + c-cpp-tutorial/SKILL.md 的 C++ 部分整合
> 覆盖范围：类与对象、继承多态、模板、STL 容器与算法、智能指针、移动语义、Lambda、异常与 RAII、设计模式

---

## 1. 类与对象

### Rule of Five

```cpp
class Buffer {
public:
    explicit Buffer(size_t size) : size_(size), data_(new char[size]) {}
    ~Buffer() { delete[] data_; }

    // 拷贝构造
    Buffer(const Buffer& other) : size_(other.size_), data_(new char[other.size_]) {
        std::memcpy(data_, other.data_, size_);
    }

    // 拷贝赋值
    Buffer& operator=(const Buffer& other) {
        if (this != &other) {
            delete[] data_;
            size_ = other.size_;
            data_ = new char[other.size_];
            std::memcpy(data_, other.data_, size_);
        }
        return *this;
    }

    // 移动构造
    Buffer(Buffer&& other) noexcept : size_(other.size_), data_(other.data_) {
        other.size_ = 0; other.data_ = nullptr;
    }

    // 移动赋值
    Buffer& operator=(Buffer&& other) noexcept {
        if (this != &other) {
            delete[] data_;
            size_ = other.size_; data_ = other.data_;
            other.size_ = 0; other.data_ = nullptr;
        }
        return *this;
    }

private:
    size_t size_;
    char* data_;
};

// Rule of Zero：用智能指针和 STL，编译器自动生成正确实现
class BufferV2 {
public:
    BufferV2(size_t size) : data_(std::make_unique<char[]>(size)), size_(size) {}
private:
    std::unique_ptr<char[]> data_;
    size_t size_;
};
```

### 陷阱
- 自定义析构后编译器不再生成移动操作（退化为拷贝）
- 移动构造函数未 `noexcept` → `std::vector` 扩容时会拷贝而非移动
- 构造函数初始化列表中成员顺序应与声明顺序一致

### 最佳实践
- 优先 **Rule of Zero**：用 `unique_ptr`/`shared_ptr`/`vector`/`string` 管理资源
- 移动操作标记 `noexcept`
- 析构函数要么是 `public virtual`，要么是 `protected non-virtual`

---

## 2. 继承与多态

```cpp
class Shape {
public:
    virtual ~Shape() = default;  // 虚析构必须！
    virtual double area() const = 0;
    virtual void draw() const = 0;
};

class Circle : public Shape {
public:
    explicit Circle(double r) : radius_(r) {}
    double area() const override { return M_PI * radius_ * radius_; }
    void draw() const override { std::cout << "Circle r=" << radius_ << "\n"; }
private:
    double radius_;
};

std::unique_ptr<Shape> s = std::make_unique<Circle>(5.0);
s->draw();
```

### 陷阱
- **虚析构缺失**：`delete base_ptr` 只调用基类析构 → 派生类资源泄漏
- **对象切片**：`Shape s = Circle(5.0);` 切片掉派生类部分
- **构造/析构中调用虚函数**：调用的是当前类版本

### 最佳实践
- 多态对象通过指针/引用传递
- 用 `override` 标记每个覆盖的虚函数
- 用 `final` 防止进一步继承或覆盖

---

## 3. 模板

```cpp
// 基本函数模板
template<typename T>
T max_value(T a, T b) { return (a > b) ? a : b; }

// 类模板
template<typename T, size_t N>
class Stack {
public:
    void push(const T& val) { /* ... */ }
    T pop() { /* ... */ }
private:
    T data_[N];
    size_t top_ = 0;
};

// C++20 Concepts
template<typename T>
concept Numeric = std::integral<T> || std::floating_point<T>;

template<Numeric T>
T safe_divide(T a, T b) {
    if (b == T{0}) throw std::invalid_argument("division by zero");
    return a / b;
}

// SFINAE（C++17 前）
template<typename T, std::enable_if_t<std::is_integral_v<T>, int> = 0>
T process(T val) { return val * 2; }
```

### 陷阱
- 模板错误信息极其冗长 → C++20 Concepts 大幅改善
- 模板声明和实现放同一头文件

### 最佳实践
- C++20 优先用 Concepts 约束
- 复杂逻辑用 `if constexpr` 分发到普通函数

---

## 4. STL 容器

### 选择指南

```
需要随机访问？
├── 是 → vector（默认首选）/ deque
└── 否
    ├── 频繁头部插入删除？→ deque/list
    ├── 需要 key-value 查找？→ map/unordered_map
    ├── 需要自动排序？→ set/map
    └── 其他 → vector（通常还是最快）
```

### 常用操作

```cpp
// 遍历时删除（正确方式）
for (auto it = v.begin(); it != v.end(); ) {
    if (*it % 2 == 0) it = v.erase(it);
    else ++it;
}

// C++20 erase_if
std::erase_if(v, [](int x) { return x % 2 == 0; });

// reserve vs resize
v.reserve(1000);  // 不改变 size，只改变 capacity
v.resize(1000);   // 改变 size
```

### 陷阱
- **迭代器失效**：`vector` 插入/删除后所有迭代器失效；`map` 仅删除的那个失效
- `vector<bool>` 不存储真正的 `bool`，元素不可取地址
- `unordered_map` 迭代顺序不稳定

### 最佳实践
- 默认用 `vector`
- 用 `emplace_back` 代替 `push_back`
- 提前 `reserve` 已知大小

---

## 5. STL 算法

```cpp
std::vector<int> v = {5, 2, 8, 2, 1, 8, 3};

// 去重
std::sort(v.begin(), v.end());
v.erase(std::unique(v.begin(), v.end()), v.end());

// 查找
auto it = std::find_if(v.begin(), v.end(), [](int x) { return x > 4; });

// 二分查找（需要已排序）
bool found = std::binary_search(v.begin(), v.end(), 5);
auto lb = std::lower_bound(v.begin(), v.end(), 3);

// 累积
int sum = std::accumulate(v.begin(), v.end(), 0);

// transform
std::transform(v.begin(), v.end(), out.begin(), [](int x) { return x * 2; });
```

---

## 6. 智能指针

```cpp
// unique_ptr：独占所有权，零开销
auto ptr = std::make_unique<int>(42);

// 自定义 deleter
auto file_closer = [](FILE* f) { if (f) fclose(f); };
std::unique_ptr<FILE, decltype(file_closer)> fp(fopen("test.txt", "r"), file_closer);

// shared_ptr：共享所有权
auto sp1 = std::make_shared<std::string>("hello");
auto sp2 = sp1;

// weak_ptr：打破循环引用
class Node {
public:
    std::shared_ptr<Node> next;
    std::weak_ptr<Node> prev;
};
if (auto sp = wp.lock()) { /* 对象仍存在 */ }
```

### 陷阱
- **循环引用**：用 `weak_ptr` 打破
- `make_shared` 优于 `new`：异常安全，一次分配
- `shared_ptr` 拷贝有开销（原子引用计数）

### 最佳实践
- 默认用 `unique_ptr`，需要共享时才用 `shared_ptr`
- 总是用 `make_unique`/`make_shared`

---

## 7. 右值引用与移动语义

```cpp
// std::move 本质：static_cast 到右值引用
std::string a = "hello";
std::string b = std::move(a);  // a 变为有效但未指定状态

// 完美转发
template<typename T, typename... Args>
std::unique_ptr<T> make_unique_custom(Args&&... args) {
    return std::unique_ptr<T>(new T(std::forward<Args>(args)...));
}
```

### 陷阱
- `std::move` 后对象处于"有效但未指定"状态
- 对 `const` 对象 `std::move` 无效
- 局部变量返回时不需要 `std::move`（NRVO 会优化）

### 最佳实践
```cpp
// 正确：返回局部变量不 move
std::vector<int> create() {
    std::vector<int> v = {1, 2, 3};
    return v;  // 不要 return std::move(v)
}

// 正确：值参数可以 move
void sink(std::string s) {
    internal_str = std::move(s);
}
```

---

## 8. Lambda 表达式

```cpp
// 基本用法
auto add = [](int a, int b) { return a + b; };

// 捕获列表
auto f1 = [x, y]() { return x + y; };       // 值捕获
auto f2 = [&x, &y]() { return x + y; };     // 引用捕获
auto f3 = [=]() { return x + y; };          // 全部值捕获
auto f4 = [&]() { x += y; };                // 全部引用捕获

// mutable lambda
auto counter = [count = 0]() mutable { return ++count; };

// C++14 泛型 Lambda
auto print = [](const auto& x) { std::cout << x << "\n"; };

// C++20 模板 Lambda
auto cast = []<typename T>(T val) -> int { return static_cast<int>(val); };
```

### 陷阱
- `[&]` 捕获局部引用 → lambda 超出作用域后悬垂引用
- lambda 不能直接递归

### 最佳实践
- 捕获列表尽量明确，避免 `[=]`/`[&]`

---

## 9. 异常处理与 RAII

```cpp
// RAII 锁
std::mutex mtx;
{
    std::lock_guard<std::mutex> lock(mtx);  // 构造时加锁
    // 临界区
}  // 析构时自动解锁

// 异常安全保证
// 基本保证：异常后对象有效
// 强异常保证：操作要么完全成功，要么完全回滚
// noexcept 保证：不抛异常
```

### 陷阱
- 析构函数默认 `noexcept`，析构中抛异常 → `std::terminate()`
- 析构函数不要抛异常

### 最佳实践
- 用 RAII 管理所有资源（文件、锁、socket）
- 关键路径标记 `noexcept`

---

## 10. 设计模式

### Pimpl 惯用法

```cpp
// header.h
class Widget {
public:
    Widget();
    ~Widget();
    void do_something();
private:
    class Impl;
    std::unique_ptr<Impl> impl_;
};

// widget.cpp
class Widget::Impl {
public:
    void do_something() { /* ... */ }
    std::vector<int> data_;
};
Widget::Widget() : impl_(std::make_unique<Impl>()) {}
Widget::~Widget() = default;
void Widget::do_something() { impl_->do_something(); }
```

### CRTP（奇异递归模板模式）

```cpp
template<typename Derived>
class Counter {
    int count_ = 0;
public:
    void increment() { ++count_; }
    int count() const { return count_; }
};

class MyClass : public Counter<MyClass> {
    // 自动获得 count() 和 increment()
};
```

---

## 11. 性能优化

```cpp
// reserve 预分配
std::vector<int> v;
v.reserve(10000);

// emplace_back vs push_back
v.emplace_back(42);       // 就地构造
v.push_back(42);          // 可能触发拷贝/移动

// 缓存友好：SoA vs AoS
// SoA（Structure of Arrays）— 缓存友好
struct ParticlesSoA {
    std::vector<float> x, y, z;
    std::vector<float> vx, vy, vz;
    std::vector<float> mass;
};
```

---

## 12. C/C++ 混合编程

```cpp
// C++ 调用 C 函数
extern "C" {
    void c_function(int x);
}

// C 调用 C++ 函数：用 extern "C" 包裹
// cpp_wrapper.cpp
extern "C" void my_cpp_func(int x) { /* C++ 实现 */ }

// c_wrapper.h
#ifdef __cplusplus
extern "C" {
#endif
void my_cpp_func(int x);
#ifdef __cplusplus
}
#endif
```

---

## 快速参考卡片

| 场景 | 推荐 |
|---|---|
| 默认容器 | `std::vector` |
| 键值映射 | `std::unordered_map`（或 `std::map` 需要有序） |
| 字符串参数 | `std::string_view`（只读）/ `std::string`（需要所有权） |
| 动态分配 | `std::make_unique<T>()` |
| 共有所有权 | `std::make_shared<T>()` |
| 线程安全计数 | `std::atomic<T>` |
| 互斥锁 | `std::lock_guard` / `std::scoped_lock` |
| 错误处理 | 异常 / `std::expected`（C++23） |
| 回调 | Lambda 或 `std::function` |
| 配置常量 | `constexpr` / `inline constexpr` |
| 可选值 | `std::optional` |
| 多类型值 | `std::variant` |
