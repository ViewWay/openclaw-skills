# C++ Programming Skill

专业级 C++ 开发参考。结构：概念 → 代码示例 → 陷阱/注意 → 最佳实践。

---

## 1. 类与对象

### 概念

C++ 类的核心是构造、析构、拷贝、移动。遵循 **Rule of Five**（C++11+）或 **Rule of Zero**。

### 代码示例

```cpp
// Rule of Five
class Buffer {
public:
    // 构造
    Buffer(size_t size) : size_(size), data_(new char[size]) {}

    // 析构
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
        other.size_ = 0;
        other.data_ = nullptr;
    }

    // 移动赋值
    Buffer& operator=(Buffer&& other) noexcept {
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
    char* data_;
};

// Rule of Zero：用智能指针和 STL，不需要手写任何特殊成员函数
class BufferV2 {
public:
    BufferV2(size_t size) : data_(std::make_unique<char[]>(size)), size_(size) {}
    // 编译器自动生成正确的拷贝/移动/析构
private:
    std::unique_ptr<char[]> data_;
    size_t size_;
};
```

### 陷阱

- 自定义析构后编译器不再生成移动操作（退化为拷贝）
- 拷贝赋值忘记自赋值检查 → `delete[] data_` 后 `data_` 已失效
- 移动构造函数未 `noexcept` → `std::vector` 扩容时会拷贝而非移动
- 构造函数初始化列表中成员顺序应与声明顺序一致（否则有警告）

### 最佳实践

- 优先 **Rule of Zero**：用 `unique_ptr`/`shared_ptr`/`vector`/`string` 管理资源
- 必须手动管理资源时，实现 Rule of Five
- 移动操作标记 `noexcept`
- 析构函数要么是 `public virtual`，要么是 `protected non-virtual`

---

## 2. 继承与多态

### 概念

虚函数实现运行时多态。`override` 和 `final` 防止意外覆盖。

### 代码示例

```cpp
class Shape {
public:
    virtual ~Shape() = default;  // 虚析构！必须有
    virtual double area() const = 0;  // 纯虚函数
    virtual void draw() const { std::cout << "Shape\n"; }
    virtual void draw() const = 0;  // 纯虚
};

class Circle : public Shape {
public:
    Circle(double r) : radius_(r) {}
    double area() const override { return M_PI * radius_ * radius_; }
    void draw() const override { std::cout << "Circle r=" << radius_ << "\n"; }
private:
    double radius_;
};

// 使用
std::unique_ptr<Shape> s = std::make_unique<Circle>(5.0);
s->draw();      // Circle r=5
std::cout << s->area() << "\n";
```

### 陷阱

- **虚析构缺失**：`delete base_ptr` 只调用基类析构，派生类资源泄漏
- **对象切片**：`Shape s = Circle(5.0);` 切片掉派生类部分，虚函数调用基类版本
- **构造/析构中调用虚函数**：调用的是当前类版本（构造期间 vtable 未完成）
- **多重继承菱形问题**：用虚继承解决

### 最佳实践

- 基类析构总是 `virtual` 或 `protected`
- 多态对象通过指针/引用传递，永远不要值传递
- 用 `override` 标记每个覆盖的虚函数
- 用 `final` 防止进一步继承或覆盖

---

## 3. 模板

### 概念

泛型编程基础。SFINAE + Concepts（C++20）约束模板参数。

### 代码示例

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

// 模板特化
template<>
class Stack<const char*> {
    // 针对 const char* 的特殊实现
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

template<typename T, std::enable_if_t<std::is_floating_point_v<T>, int> = 0>
T process(T val) { return val + 0.5; }
```

### 陷阱

- 模板错误信息极其冗长难读 → C++20 Concepts 大幅改善
- 模板实例化在头文件中 → 编译时间增加，考虑显式实例化
- 模板递归无终止条件 → 编译错误或栈溢出
- `typename` vs `class` 在模板参数中等价，但 `typename` 用于依赖类型：`typename T::iterator`

### 最佳实践

- C++20 优先用 Concepts 约束
- 模板声明和实现放同一头文件（或 .hpp）
- 模板代码保持简单，复杂逻辑用 `if constexpr` 分发到普通函数

---

## 4. STL 容器

### 概念

选择正确容器是性能关键。迭代器失效是常见 Bug 源。

### 代码示例

```cpp
// 容器选择指南
std::vector<int> v;                    // 默认选择，连续内存
std::deque<int> dq;                    // 双端队列，两端 O(1)
std::list<int> lst;                    // 链表，中间插入 O(1)（但缓存不友好）
std::map<int, std::string> m;          // 有序映射，O(log n)
std::unordered_map<int, std::string> um;// 哈希映射，平均 O(1)

// 遍历时删除（正确方式）
std::vector<int> v = {1, 2, 3, 4, 5};
for (auto it = v.begin(); it != v.end(); ) {
    if (*it % 2 == 0) {
        it = v.erase(it);  // erase 返回下一个有效迭代器
    } else {
        ++it;
    }
}

// C++20 erase_if
std::erase_if(v, [](int x) { return x % 2 == 0; });
```

### 陷阱

- **迭代器失效**：`vector` 插入/删除后所有迭代器失效；`map`/`unordered_map` 仅删除的那个失效
- `vector<bool>` 不是 `vector<bool>` — 它是特化的，不存储真正的 `bool`，元素不可取地址
- `unordered_map` 迭代顺序不稳定（rehash 后变化）
- `reserve` 不改变 `size`，只改变 `capacity`；`resize` 改变 `size`

### 最佳实践

- 默认用 `vector`，只有明确需要时才选其他
- 用 `emplace_back` 代替 `push_back`（就地构造，减少拷贝）
- 提前 `reserve` 已知大小，避免多次重分配
- 用 `at()` 代替 `[]` 获取越界检查（调试期）

---

## 5. 智能指针

### 概念

RAII 管理动态内存，杜绝裸 `new`/`delete`。

### 代码示例

```cpp
// unique_ptr：独占所有权，零开销
auto ptr = std::make_unique<int>(42);
std::unique_ptr<int[]> arr = std::make_unique<int[]>(100);

// 自定义 deleter
auto file_closer = [](FILE* f) { if (f) fclose(f); };
std::unique_ptr<FILE, decltype(file_closer)> fp(fopen("test.txt", "r"), file_closer);

// shared_ptr：共享所有权，引用计数
auto sp1 = std::make_shared<std::string>("hello");
auto sp2 = sp1;  // 引用计数 = 2

// weak_ptr：打破循环引用
class Node {
public:
    std::shared_ptr<Node> next;
    std::weak_ptr<Node> prev;  // 用 weak_ptr 避免循环引用
};

// 从 weak_ptr 获取 shared_ptr
std::weak_ptr<Node> wp;
if (auto sp = wp.lock()) {
    // 对象仍然存在，使用 sp
}
```

### 陷阱

- **循环引用**：`shared_ptr` A 持有 `shared_ptr` B，B 持有 A → 内存泄漏，用 `weak_ptr` 打破
- `make_shared` 优于 `new`：一次分配（控制块+对象），异常安全
- `shared_ptr` 拷贝有开销（原子引用计数），不要无谓拷贝
- `unique_ptr` 不可拷贝但可移动：`std::move(ptr)`
- 裸 `new` 传给 `shared_ptr` 构造函数可能泄漏（参数求值顺序）

### 最佳实践

- 默认用 `unique_ptr`，需要共享时才用 `shared_ptr`
- 总是用 `make_unique`/`make_shared`
- 函数返回时用值返回智能指针（移动语义）
- 工厂函数返回 `unique_ptr`，让调用者决定所有权

---

## 6. 右值引用与移动语义

### 概念

`std::move` 不移动任何东西，它只是将左值转为右值引用。真正的"移动"由移动构造/赋值完成。

### 代码示例

```cpp
// std::move 本质
template<typename T>
typename std::remove_reference<T>::type&& move(T&& t) noexcept {
    return static_cast<typename std::remove_reference<T>::type&&>(t);
}

// 完美转发
template<typename T, typename... Args>
std::unique_ptr<T> make_unique_custom(Args&&... args) {
    return std::unique_ptr<T>(new T(std::forward<Args>(args)...));
}

// 万能引用（forwarding reference）
template<typename T>
void wrapper(T&& arg) {  // T&& 不是右值引用！是万能引用
    process(std::forward<T>(arg));
}

// 调用
int x = 10;
wrapper(x);       // T = int&, arg 是左值引用
wrapper(10);      // T = int, arg 是右值引用
wrapper(std::move(x));  // T = int, arg 是右值引用
```

### 陷阱

- `std::move` 后的对象处于"有效但未指定"状态，不要假设其值
- 对 `const` 对象 `std::move` 无效（会匹配拷贝而非移动）
- 万能引用只在模板参数推导时成立：`void f(T&&)` 是万能引用，`void f(int&&)` 不是
- `auto&&` 也是万能引用

### 最佳实践

- 局部变量返回时不需要 `std::move`（NRVO 会优化，加了反而阻止 NRVO）
- 函数参数是值类型时，在函数体内 `std::move` 它

```cpp
// 正确
std::vector<int> create() {
    std::vector<int> v = {1, 2, 3};
    return v;  // 不要 return std::move(v)，会阻止 NRVO
}

// 正确
void sink(std::string s) {
    internal_str = std::move(s);  // s 是值类型，可以 move
}
```

---

## 7. Lambda 表达式

### 概念

匿名函数对象，支持捕获列表。

### 代码示例

```cpp
// 基本用法
auto add = [](int a, int b) { return a + b; };

// 捕获列表
int x = 10, y = 20;
auto f1 = [x, y]() { return x + y; };       // 值捕获（拷贝）
auto f2 = [&x, &y]() { return x + y; };     // 引用捕获
auto f3 = [=]() { return x + y; };          // 全部值捕获
auto f4 = [&]() { x += y; };                // 全部引用捕获
auto f5 = [=, &y]() { return x + y; };      // 默认值捕获，y 引用捕获

// mutable：修改值捕获的变量
auto counter = [count = 0]() mutable { return ++count; };
counter();  // 1
counter();  // 2

// 泛型 Lambda（C++14）
auto print = [](const auto& x) { std::cout << x << "\n"; };
print(42);
print("hello");

// C++20 模板 Lambda
auto cast = []<typename T>(T val) -> int { return static_cast<int>(val); };
```

### 陷阱

- `[&]` 捕获局部引用 → lambda 超出作用域后悬垂引用
- `[=]` 默认值捕获可能导致意外的大拷贝
- `mutable` lambda 不是纯函数，注意副作用
- lambda 不能直接递归（需要用 `std::function` 或 Y 组合子）

### 最佳实践

- 捕获列表尽量明确，避免 `[=]`/`[&]`
- 短 lambda 内联使用，复杂逻辑提取为命名函数
- 优先用 `const auto&` 捕获避免拷贝

---

## 8. 异常处理与 RAII

### 概念

RAII（资源获取即初始化）是 C++ 资源管理的核心。异常安全保证分三级。

### 代码示例

```cpp
// RAII 锁
std::mutex mtx;
{
    std::lock_guard<std::mutex> lock(mtx);  // 构造时加锁
    // 临界区
}  // 析构时自动解锁，即使抛异常

// 异常安全保证
class SafeVector {
public:
    // 基本保证：异常后对象处于有效状态
    void push_back(const std::string& s) {
        data_.push_back(s);
    }

    // 强异常保证：操作要么完全成功，要么完全回滚
    void set_all(const std::string& s) {
        auto copy = data_;  // 先拷贝
        for (auto& e : copy) e = s;  // 修改拷贝
        data_ = std::move(copy);  // noexcept 操作提交
    }

    // noexcept 保证：不抛异常
    void clear() noexcept { data_.clear(); }
};

// noexcept 关键函数
void critical_func() noexcept {
    // 如果这里抛异常 → std::terminate()
}
```

### 陷阱

- 析构函数默认 `noexcept`（C++11+），析构中抛异常 → `std::terminate()`
- 构造函数中 `new` 失败会抛 `std::bad_alloc`
- `catch(...)` 捕获所有异常，但不知道类型
- 异常与裸指针混合 → 泄漏（RAII 解决）

### 最佳实践

- 析构函数不要抛异常
- 关键路径标记 `noexcept`（移动操作、析构、swap）
- 用 RAII 管理所有资源（文件、锁、socket、数据库连接）

---

## 9. C++ 现代（C++11~C++23）

### 核心现代特性

```cpp
// auto & decltype
auto x = 42;              // int
auto y = 42.0;            // double
auto z = "hello";         // const char*
decltype(x) w = x;        // int

// 结构化绑定（C++17）
auto [key, value] = *map.begin();
auto& [a, b] = pair_ref;

// std::optional（C++17）
std::optional<int> find_user(const std::string& name);
auto result = find_user("Alice");
if (result) {
    std::cout << *result << "\n";
}

// std::variant（C++17）
std::variant<int, double, std::string> v = "hello";
v = 42;
std::visit([](auto&& arg) { std::cout << arg << "\n"; }, v);

// std::string_view（C++17）— 不拥有字符串，零拷贝
void process(std::string_view sv) {
    // 可以接受 string, const char*, string_view
}

// constexpr（C++11/14/20）
constexpr int factorial(int n) {
    return n <= 1 ? 1 : n * factorial(n - 1);
}
static_assert(factorial(5) == 120);

// consteval（C++20）— 必须在编译期求值
consteval int square(int n) { return n * n; }

// std::format（C++20）
std::string s = std::format("Hello, {}! You are {} years old.", name, age);

// Ranges（C++20）
auto result = numbers | std::views::filter([](int n) { return n % 2 == 0; })
                       | std::views::transform([](int n) { return n * 2; })
                       | std::views::take(10);

// 协程（C++20）
Task<int> async_compute() {
    int a = co_await some_async_op();
    int b = co_await another_async_op();
    co_return a + b;
}
```

---

## 10. C++ 陷阱汇总

### 对象切片

```cpp
class Base { virtual void foo() { std::cout << "Base\n"; } };
class Derived : public Base { void foo() override { std::cout << "Derived\n"; } };

void process(Base b) { b.foo(); }  // ❌ 值传递 → 切片
void process(Base& b) { b.foo(); } // ✅ 引用传递 → 多态
void process(Base* b) { b->foo(); } // ✅ 指针传递 → 多态
```

### 虚析构缺失

```cpp
class Base { public: ~Base() {} };  // ❌ 非 virtual
class Derived : public Base { int* data = new int[100]; ~Derived() { delete[] data; } };

Base* p = new Derived();
delete p;  // ❌ 只调用 ~Base，data 泄漏！
```

### 迭代器失效

```cpp
std::vector<int> v = {1, 2, 3, 4, 5};
for (auto& elem : v) {
    if (elem == 3) v.push_back(6);  // ❌ push_back 可能使所有迭代器失效
}

// ✅ 正确
for (auto it = v.begin(); it != v.end(); ) {
    if (*it == 3) {
        it = v.insert(it, 6);  // insert 返回新插入元素的迭代器
        ++it; ++it;             // 跳过新插入的和原来的 3
    } else {
        ++it;
    }
}
```

### 静态初始化顺序

```cpp
// file_a.cpp
extern int global_a;
int global_a = compute();

// file_b.cpp
extern int global_a;
int global_b = global_a + 1;  // ❌ global_a 可能还没初始化！

// ✅ 解决：Meyers' Singleton（函数内 static）
int& get_a() {
    static int a = compute();  // C++11 保证线程安全初始化
    return a;
}
int global_b = get_a() + 1;
```

### 浮点比较

```cpp
// ❌ 直接比较
if (a == b) { }

// ✅ epsilon 比较
bool approx_equal(double a, double b, double eps = 1e-9) {
    return std::abs(a - b) <= eps * std::max(std::abs(a), std::abs(b));
}
```

### auto 陷阱

```cpp
auto x1 = {1};     // C++17 前：std::initializer_list<int>，C++20：int
auto x2{1};        // C++17 前：std::initializer_list<int>，C++17+：int
auto x3 = {1, 2};  // 始终是 std::initializer_list<int>

// auto 推导为引用类型需要 &
const std::map<std::string, int> m;
auto it = m.begin();  // std::map<std::string, int>::const_iterator（const 对象）
auto& ref = *it;      // const std::pair<const std::string, int>&
```

### 隐式转换

```cpp
// 窄化转换
int x = 3.14;    // x = 3，丢失小数部分
int y = 1e10;    // UB 或截断

// bool → int
bool b = true;
int z = b + 1;   // z = 2

// 解决：用 {-Werror=narrowing} 或 brace initialization（会报错）
int x{3.14};     // 编译错误！
```

---

## 11. 设计模式

### RAII 资源管理

```cpp
class FileGuard {
public:
    FileGuard(const char* path, const char* mode)
        : fp_(fopen(path, mode)) {
        if (!fp_) throw std::runtime_error("Cannot open file");
    }
    ~FileGuard() { if (fp_) fclose(fp_); }
    FILE* get() const { return fp_; }
    // 禁止拷贝
    FileGuard(const FileGuard&) = delete;
    FileGuard& operator=(const FileGuard&) = delete;
private:
    FILE* fp_;
};
```

### Pimpl 惯用法

```cpp
// header.h
class Widget {
public:
    Widget();
    ~Widget();
    void do_something();
private:
    class Impl;        // 前向声明
    std::unique_ptr<Impl> impl_;  // 隐藏实现细节
};

// widget.cpp
class Widget::Impl {
public:
    void do_something() { /* ... */ }
    std::vector<int> data_;  // 实现细节不需要在头文件中
};

Widget::Widget() : impl_(std::make_unique<Impl>()) {}
Widget::~Widget() = default;  // 必须在 .cpp 中定义（Impl 完整类型可见）
void Widget::do_something() { impl_->do_something(); }
```

### CRTP（奇异递归模板模式）

```cpp
template<typename Derived>
class Counter {
public:
    void increment() { ++count_; }
    int count() const { return count_; }
private:
    int count_ = 0;
};

class MyClass : public Counter<MyClass> {
    // 自动获得 count() 和 increment()
};
```

---

## 12. C++ 并发

### 概念

C++11 提供标准线程库。正确使用互斥锁和原子操作。

### 代码示例

```cpp
#include <thread>
#include <mutex>
#include <condition_variable>
#include <atomic>
#include <future>

// 基本线程
std::mutex mtx;
int shared = 0;

void increment(int n) {
    for (int i = 0; i < n; i++) {
        std::lock_guard<std::mutex> lock(mtx);
        shared++;
    }
}

std::thread t1(increment, 100000);
std::thread t2(increment, 100000);
t1.join();
t2.join();

// 条件变量（注意虚假唤醒）
std::mutex queue_mtx;
std::condition_variable cv;
std::queue<int> q;
std::atomic<bool> done{false};

void producer() {
    for (int i = 0; i < 10; i++) {
        {
            std::lock_guard<std::mutex> lock(queue_mtx);
            q.push(i);
        }
        cv.notify_one();
    }
    done = true;
    cv.notify_one();
}

void consumer() {
    while (true) {
        std::unique_lock<std::mutex> lock(queue_mtx);
        cv.wait(lock, [] { return !q.empty() || done.load(); });  // 用 lambda 防止虚假唤醒
        while (!q.empty()) {
            int val = q.front();
            q.pop();
            std::cout << val << "\n";
        }
        if (done && q.empty()) break;
    }
}

// async
auto future = std::async(std::launch::async, [](int a, int b) {
    return a + b;
}, 10, 20);
int result = future.get();  // 阻塞等待结果

// 原子操作
std::atomic<int> counter{0};
counter.fetch_add(1, std::memory_order_relaxed);  // 无锁递增
```

### 陷阱

- **条件变量虚假唤醒**：必须用 `while` 循环或带谓词的 `wait`
- **死锁**：两个线程以不同顺序获取两把锁 → 解决：始终按固定顺序加锁，或用 `std::lock`
- **数据竞争**：未保护的共享变量并发读写 → UB
- `std::thread` 析构前必须 `join()` 或 `detach()`，否则 `std::terminate()`
- `std::shared_mutex` 适合读多写少场景

### 最佳实践

- 优先使用 `std::lock_guard`/`std::scoped_lock`（C++17），避免手动 lock/unlock
- 尽量缩小临界区范围
- 考虑用 `std::atomic` 替代 mutex（简单计数器等）
- 用 ThreadSanitizer 检测：`-fsanitize=thread`

---

## 13. 性能优化

### 核心技巧

```cpp
// reserve 预分配
std::vector<int> v;
v.reserve(10000);  // 避免多次重分配

// emplace_back vs push_back
v.emplace_back(42);       // 就地构造，不拷贝
v.push_back(42);          // 可能触发拷贝/移动

// 移动语义减少拷贝
std::vector<std::string> build_strings() {
    std::vector<std::string> result;
    result.reserve(100);
    for (int i = 0; i < 100; i++) {
        result.emplace_back(std::to_string(i));
    }
    return result;  // NRVO 或移动
}

// 小对象优化（SSO）— string 内部缓冲
std::string s1 = "short";  // 存在内部缓冲区，不分配堆内存
std::string s2(100, 'x');  // 超过 SSO 阈值，分配堆内存

// 缓存友好：SoA vs AoS
// AoS（Array of Structures）— 缓存不友好
struct ParticleAoS {
    float x, y, z;      // 位置
    float vx, vy, vz;   // 速度
    float mass;
};
std::vector<ParticleAoS> particles;

// SoA（Structure of Arrays）— 缓存友好
struct ParticlesSoA {
    std::vector<float> x, y, z;
    std::vector<float> vx, vy, vz;
    std::vector<float> mass;
};
// 只更新位置时，只需加载 x/y/z，缓存命中率更高

// constexpr 编译期计算
constexpr int fib(int n) {
    if (n <= 1) return n;
    int a = 0, b = 1;
    for (int i = 2; i <= n; i++) {
        int tmp = a + b;
        a = b;
        b = tmp;
    }
    return b;
}
static_assert(fib(10) == 55);
```

---

## 14. 工具链

### 编译选项

```bash
# 推荐编译选项
g++ -std=c++20 -Wall -Wextra -Wpedantic -Werror -O2 -g prog.cpp -o prog

# Sanitizers
g++ -fsanitize=address -g prog.cpp    # 内存错误检测
g++ -fsanitize=undefined -g prog.cpp   # 未定义行为检测
g++ -fsanitize=thread -g prog.cpp      # 数据竞争检测
g++ -fsanitize=memory -g prog.cpp      # 未初始化内存读取

# 组合使用
g++ -fsanitize=address,undefined -g prog.cpp

# clang-tidy 静态分析
clang-tidy prog.cpp -- -std=c++20

# clang-format 格式化
clang-format -i prog.cpp
```

### CMake 基本配置

```cmake
cmake_minimum_required(VERSION 3.20)
project(MyProject CXX)

set(CMAKE_CXX_STANDARD 20)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

add_executable(prog main.cpp)

# Debug 模式启用 sanitizers
if(CMAKE_BUILD_TYPE STREQUAL "Debug")
    target_compile_options(prog PRIVATE -fsanitize=address,undefined -g)
endif()
```

### 调试

```bash
# gdb
gdb ./prog
(gdb) break main
(gdb) run
(gdb) print variable
(gdb) bt              # backtrace
(gdb) info locals     # 局部变量

# lldb
lldb ./prog
(lldb) b main
(lldb) run
(lldb) p variable
(lldb) bt
```

---

## 快速参考卡片

| 场景 | 推荐 |
|---|---|
| 默认容器 | `std::vector` |
| 键值映射 | `std::unordered_map`（或 `std::map` 需要有序） |
| 字符串参数 | `std::string_view`（只读）/ `std::string`（需要所有权） |
| 动态分配 | `std::make_unique<T>()` |
| 共享所有权 | `std::make_shared<T>()` |
| 线程安全计数 | `std::atomic<T>` |
| 互斥锁 | `std::lock_guard` / `std::scoped_lock` |
| 错误处理 | 异常（应用）/ `std::expected`（C++23，库）/ 错误码（系统编程） |
| 回调 | Lambda 或 `std::function` |
| 配置常量 | `constexpr` / `inline constexpr`（C++17） |
| 编译期计算 | `constexpr` 函数 / `consteval`（C++20） |
| 类型擦除 | `std::any` / `std::function` |
| 可选值 | `std::optional` |
| 多类型值 | `std::variant` |
