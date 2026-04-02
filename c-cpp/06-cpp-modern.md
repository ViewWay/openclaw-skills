# C++11~23 现代特性 (Modern C++ Features)

> 来源：cpp-programming/SKILL.md 的现代特性部分 + c-cpp-tutorial/SKILL.md 的 C++17/20/23 部分
> 覆盖范围：C++11/14/17/20/23 各版本新特性，侧重实战用法

---

## C++11 — 现代 C++ 基石

### auto & decltype

```cpp
auto x = 42;              // int
auto y = 42.0;            // double
auto z = "hello";         // const char*
decltype(x) w = x;        // int

// 返回类型推导（C++14）
auto add(int a, int b) { return a + b; }
```

### 范围 for 循环

```cpp
std::vector<int> v = {1, 2, 3};
for (const auto& elem : v) {
    std::cout << elem << " ";
}
```

### nullptr

```cpp
int *p = nullptr;  // 替代 NULL 和 0
void f(int);       // 重载 1
void f(int*);      // 重载 2
f(nullptr);        // 调用重载 2，明确
f(0);              // 调用重载 1，歧义风险
```

### 强类型枚举 (enum class)

```cpp
enum class Color { Red, Green, Blue };
Color c = Color::Red;
// 不会隐式转换为 int，不会污染命名空间
```

### static_assert

```cpp
static_assert(sizeof(int) >= 4, "int too small");
static_assert(std::is_integral_v<T>, "T must be integral");
```

### constexpr

```cpp
constexpr int factorial(int n) {
    return n <= 1 ? 1 : n * factorial(n - 1);
}
constexpr int val = factorial(5);  // 编译期 = 120
static_assert(val == 120);
```

### 统一初始化 (brace initialization)

```cpp
int x{42};
std::vector<int> v{1, 2, 3};
std::map<std::string, int> m{{"a", 1}, {"b", 2}};

// 防止窄化转换
// int x{3.14};  // 编译错误！
int y = 3.14;     // OK（截断为 3）
```

### 右值引用与移动语义

```cpp
std::string a = "hello";
std::string b = std::move(a);  // 移动，a 变为有效但未指定状态
```

### Lambda

```cpp
auto add = [](int a, int b) { return a + b; };
std::sort(v.begin(), v.end(), [](int a, int b) { return a > b; });
```

### 智能指针

```cpp
auto up = std::make_unique<int>(42);     // C++14
auto sp = std::make_shared<std::string>("hello");
```

### 其他 C++11 特性

| 特性 | 说明 |
|------|------|
| `override` / `final` | 虚函数覆盖控制 |
| `= default` / `= delete` | 显式默认/删除特殊成员 |
| `noexcept` | 不抛异常声明 |
| `[[deprecated]]` | 废弃标记 |
| `long long` | 64 位整数 |
| `<chrono>` | 时间库 |
| `<random>` | 随机数库 |
| `<tuple>` | 元组 |
| `<unordered_map/set>` | 哈希容器 |
| 委托构造 | 构造函数调用构造函数 |
| 继承构造 | `using Base::Base;` |
| 用户定义字面量 | `operator"" _km` |

---

## C++14 — 完善

### 泛型 Lambda

```cpp
auto print = [](const auto& x) { std::cout << x << "\n"; };
print(42);        // int
print("hello");   // const char*
```

### 返回类型推导

```cpp
auto add(int a, int b) { return a + b; }
auto divide(int a, int b) -> decltype(a / b) { return a / b; }
```

### make_unique

```cpp
auto p = std::make_unique<int>(42);          // C++11 没有！
auto arr = std::make_unique<int[]>(100);
```

### 初始化 Lambda 捕获

```cpp
auto task = [w = std::move(pw)]() { w->do_something(); };
auto counter = [count = 0]() mutable { return ++count; };
```

### 其他 C++14 特性

| 特性 | 说明 |
|------|------|
| `std::make_unique` | unique_ptr 工厂函数 |
| `std::exchange` | 替换并返回旧值 |
| `std::integer_sequence` | 编译期整数序列 |
| `std::quoted` | 引号转义 I/O |
| `<shared_mutex>` | 读写锁 |
| `[[deprecated("reason")]]` | 带原因的废弃 |
| 二进制字面量 `0b1010` | |
| 数字分隔符 `1'000'000` | |

---

## C++17 — 重大更新

### 结构化绑定

```cpp
auto [key, value] = *map.begin();
auto [it, inserted] = map.insert({"key", 42});
auto& [a, b] = pair_ref;
auto [x, y, z] = std::tuple{1, 2.0, "hello"};
```

### std::optional

```cpp
std::optional<int> find_user(int id) {
    if (id == 0) return std::nullopt;
    return 42;
}
auto result = find_user(0);
if (result) { std::cout << *result; }
int val = result.value_or(-1);  // 默认值
```

### std::variant

```cpp
std::variant<int, double, std::string> v = "hello";
v = 42;
std::visit([](auto&& arg) { std::cout << arg << "\n"; }, v);
```

### std::string_view

```cpp
void process(std::string_view sv) { /* 不分配内存 */ }
process("hello");
std::string s = "world";
process(s);
process(s.substr(1, 3));  // 返回 string_view，零拷贝
```

### if constexpr

```cpp
template<typename T>
std::string to_string_impl(const T& val) {
    if constexpr (std::is_integral_v<T>) {
        return std::to_string(val);
    } else {
        return val;  // 假设 T 本身就是 string
    }
}
```

### std::filesystem

```cpp
namespace fs = std::filesystem;
for (auto& entry : fs::directory_iterator("/tmp")) {
    std::cout << entry.path() << " " << entry.file_size() << "\n";
}
fs::copy(src, dst, fs::copy_options::recursive);
```

### 折叠表达式

```cpp
template<typename... Args>
auto sum(Args... args) {
    return (args + ...);  // 一元右折叠
}

template<typename... Args>
auto all(Args... args) {
    return (... && args);  // 一元左折叠
}
```

### inline 变量

```cpp
// 头文件中定义（不再需要 extern）
inline constexpr int MAX_SIZE = 1024;
```

### std::any

```cpp
std::any a = 42;
a = std::string("hello");
std::any_cast<std::string>(a);
```

### 其他 C++17 特性

| 特性 | 说明 |
|------|------|
| `std::scoped_lock` | 多锁同时锁定 |
| `std::shared_mutex` | 读写锁 |
| `std::invoke` | 通用可调用对象调用 |
| `std::apply` | 元组展开调用 |
| `std::byte` | 独立于 char 的字节类型 |
| `std::launder` | 指针优化屏障 |
| `[[nodiscard]]` | 丢弃返回值警告 |
| `[[maybe_unused]]` | 抑制未使用警告 |
| 嵌套命名空间 `namespace A::B::C {}` | |
| 类模板参数推导 `std::pair p{1, 2.0};` | |
| `auto` 参数模板 | |
| `std::clamp` / `std::gcd` / `std::lcm` | 数学工具 |
| `<execution>` 并行算法 | |
| `map::try_emplace` / `map::insert_or_assign` | |

---

## C++20 — 另一次重大更新

### Concepts

```cpp
#include <concepts>

template<typename T>
concept Numeric = std::integral<T> || std::floating_point<T>;

template<Numeric T>
T square(T x) { return x * x; }

// 或者用 requires
template<typename T>
    requires std::integral<T>
T multiply(T a, T b) { return a * b; }

// 标准库 concepts
// std::integral, std::floating_point, std::same_as
// std::derived_from, std::convertible_to, std::invocable
```

### Ranges

```cpp
#include <ranges>
namespace rv = std::views;

auto result = numbers
    | rv::filter([](int n) { return n % 2 == 0; })
    | rv::transform([](int n) { return n * 2; })
    | rv::take(10);

// range 适配器
rv::iota(1, 100)          // 生成序列
rv::take(n)               // 取前 n 个
rv::drop(n)               // 跳过前 n 个
rv::filter(pred)          // 过滤
rv::transform(fn)         // 变换
rv::reverse               // 反转
rv::distinct              // 去重
rv::split(delimiter)      // 分割
rv::join                  // 连接
rv::zip(r1, r2)           // 拉链（C++23）
```

### std::format

```cpp
#include <format>
std::string s = std::format("Hello, {}! You are {} years old.", name, age);
std::format("{0} + {0} = {2}", 1, 2, 2);      // "1 + 1 = 2"
std::format("{:>10}", "right");                 // 右对齐
std::format("{:0>5d}", 42);                     // "00042"
std::format("{:.2f}", 3.14159);                 // "3.14"
std::format("{:#x}", 255);                      // "0xff"
std::format_to(buf, "{} + {} = {}", a, b, a+b); // 写入缓冲区
```

### 协程 (Coroutines)

```cpp
#include <coroutine>

Task<int> async_compute() {
    int a = co_await some_async_op();
    int b = co_await another_async_op();
    co_return a + b;
}

Generator<int> range(int start, int end) {
    for (int i = start; i < end; ++i)
        co_yield i;
}
```

### std::jthread (C++20)

```cpp
std::jthread t([](std::stop_token st) {
    while (!st.stop_requested()) {
        // 工作...
    }
});  // 析构时自动 join，支持协作式取消
t.request_stop();
```

### Modules (C++20，实验性)

```cpp
// module.cppm
export module mymodule;
export int add(int a, int b) { return a + b; }

// main.cpp
import mymodule;
int main() { return add(1, 2); }
```

### 三路比较运算符 `<=>`

```cpp
struct Point {
    int x, y;
    auto operator<=>(const Point&) const = default;  // 自动生成所有比较
};
Point a{1, 2}, b{1, 3};
if (a < b) { /* ... */ }
```

### 其他 C++20 特性

| 特性 | 说明 |
|------|------|
| `std::span` | 非拥有数组视图 |
| `constexpr` 扩展 | `vector`, `string` 等可用于 constexpr |
| `consteval` | 强制编译期求值 |
| `std::contains` | 容器查找（map/set/string） |
| `std::starts_with` / `ends_with` | 前缀/后缀检查 |
| `std::assume_aligned` | 对齐提示 |
| `[[likely]]` / `[[unlikely]]` | 分支预测提示 |
| `std::is_constant_evaluated()` | 判断是否在编译期 |
| `std::source_location` | 源码位置信息 |
| `char8_t` | UTF-8 字符类型 |
| `std::make_shared_for_overwrite` | 延迟构造 |
| `std::atomic<shared_ptr>` | |
| `std::atomic_ref` | 引用类型原子 |
| `std::barrier` / `std::latch` | 线程同步原语 |
| `noexcept` 作为类型系统的一部分 | |

---

## C++23 — 最新特性

### std::print / std::println

```cpp
std::print("Hello, {}!\n", "world");
std::println("x = {}, y = {}", x, y);  // 自动换行
```

### std::expected

```cpp
std::expected<int, std::string> parse_int(std::string_view sv) {
    try {
        return std::stoi(std::string(sv));
    } catch (...) {
        return std::unexpected("invalid integer");
    }
}

auto result = parse_int("42");
if (result) {
    std::cout << *result;  // 42
} else {
    std::cerr << result.error();
}
```

### std::ranges 增强

```cpp
// ranges::to — 转换为容器
auto result = numbers
    | std::views::filter([](int n) { return n % 2 == 0; })
    | std::views::take(5)
    | std::ranges::to<std::vector>();
```

### 其他 C++23 特性

| 特性 | 说明 |
|------|------|
| `std::generator` | 协程生成器（LWG 协程提案） |
| `std::move_only_function` | 仅可移动的函数包装 |
| `std::flat_map` / `std::flat_set` | 平坦关联容器 |
| `std::mdspan` | 多维数组视图 |
| `std::out_ptr` / `std::inout_ptr` | 智能指针与 C API 交互 |
| `std::print` / `std::println` | 格式化输出 |
| `std::expected<T, E>` | 错误处理替代方案 |
| `#warning` 标准化 | |
| `auto(*f)() -> int` | 显式对象成员函数 |
| 多维下标运算符 `operator[]` | |
| `std::start_lifetime_as` | 对象模型操作 |
| `constexpr` 进一步扩展 | `unique_ptr`, `optional` 等可用于 constexpr |

---

## 版本特性对比总表

| 特性 | C++11 | C++14 | C++17 | C++20 | C++23 |
|------|-------|-------|-------|-------|-------|
| `auto` | ✅ | | | | |
| Lambda | ✅ | 泛型 | | 模板 | |
| `constexpr` | ✅ | 放宽 | 扩展 | 大幅扩展 | 更扩展 |
| 智能指针 | `shared/unique` | `make_unique` | | | |
| `nullptr` | ✅ | | | | |
| 右值引用/移动 | ✅ | | | | |
| 结构化绑定 | | | ✅ | | |
| `optional/variant` | | | ✅ | | |
| `string_view` | | | ✅ | | |
| `if constexpr` | | | ✅ | | |
| `filesystem` | | | ✅ | | |
| `std::format` | | | | ✅ | 扩展 |
| Concepts | | | | ✅ | |
| Ranges | | | | ✅ | 增强 |
| 协程 | | | | ✅ | 增强 |
| `<=>` | | | | ✅ | |
| `expected` | | | | | ✅ |
| `print` | | | | | ✅ |
| `flat_map/set` | | | | | ✅ |
| `generator` | | | | | ✅ |
| Modules | | | | 实验性 | 改进 |

---

## 使用建议

| 场景 | 推荐标准 |
|------|----------|
| 新项目 | **C++20**（Concepts + Ranges + format） |
| 需要最大兼容 | C++17 |
| 嵌入式/系统编程 | C++17 或 C++11（工具链限制） |
| 学习现代 C++ | 从 C++17 开始，再学 C++20 |
| 遗留代码升级 | 逐步引入 C++11 特性，再迁移到更高版本 |
