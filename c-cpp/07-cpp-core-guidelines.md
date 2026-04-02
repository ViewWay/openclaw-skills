# C++ Core Guidelines Skill

> 基于 [C++ Core Guidelines](https://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines) 的中文速查参考，按原版章节组织，每条规则含编号、说明与正反代码示例。

---

## P: Philosophy (哲学)

### P.1 直接表达意图

代码应清晰表达设计意图，让读者无需猜测。

```cpp
// ❌ 不好：意图不明
vector<int> v;
// ...
int x = v.size() - 1;  // 最后一个元素的索引？还是 size 减一？

// ✅ 好：意图明确
int last_index = ssize(v) - 1;
```

### P.2 用 ISO C++ 标准写代码

使用标准 C++，避免编译器扩展和非标准特性。

```cpp
// ❌ 不好：编译器扩展
int x = __builtin_expect(flag, 0);

// ✅ 好：标准 C++
[[likely]] if (flag) { /* ... */ }
```

### P.3 表达意图时优先选择编译时检查而非运行时检查

能用编译器捕获的问题，不要留到运行时。

```cpp
// ❌ 不好：运行时检查
void use(int* p) {
    if (!p) throw std::invalid_argument{"null"};
    *p = 42;
}

// ✅ 好：编译时保证非空（使用 gsl::not_null 或引用）
void use(gsl::not_null<int*> p) {
    *p = 42;  // 编译器已保证 p 非空
}
```

### P.4 理想情况下，程序应该是静态类型安全的

避免使用 `void*`、联合体和 unchecked 下标。

```cpp
// ❌ 不好：void* 丢失类型信息
void* p = malloc(100);

// ✅ 好：类型安全
auto p = std::make_unique<char[]>(100);
```

### P.5 不要在编译时和运行时浪费时间

避免不必要的模板实例化和运行时检查。

```cpp
// ❌ 不好：不必要地重复实例化
for (int i = 0; i < n; ++i) {
    process<0>(data[i]);  // 每次迭代都实例化相同模板
}

// ✅ 好：一次实例化
template<int> void process(Data&);
template<> void process<0>(Data& d) { /* ... */ }
// 循环外调用
```

### P.6 不能在编译时检查的内容，应在运行时检查

运行时断言和检查不应被省略。

```cpp
// ❌ 不好：信任输入不做检查
int area(int w, int h) { return w * h; }

// ✅ 好：运行时检查
int area(int w, int h) {
    Expects(w > 0 && h > 0);
    return w * h;
}
```

### P.7 尽早捕获运行时错误

错误应在其产生处被检测。

```cpp
// ❌ 不好：错误传播很远才被发现
double read_value() {
    double v;
    std::cin >> v;  // 可能失败但不检查
    return v;
}

// ✅ 好：立即检查
double read_value() {
    double v;
    if (!(std::cin >> v)) throw std::runtime_error{"input failed"};
    return v;
}
```

### P.8 不要泄漏任何资源

使用 RAII 管理所有资源。

```cpp
// ❌ 不好：可能泄漏
void f() {
    int* p = new int[100];
    if (error) return;  // 泄漏！
    delete[] p;
}

// ✅ 好：RAII 自动释放
void f() {
    auto p = std::make_unique<int[]>(100);
    if (error) return;  // 自动释放
}
```

### P.9 不要浪费时间（pessimization）

不要引入不必要的性能开销。

```cpp
// ❌ 不好：不必要的拷贝
std::string result;
for (const auto& s : vec) {
    result = result + s;  // 每次都创建临时字符串
}

// ✅ 好：预分配 + 移动
std::string result;
result.reserve(estimated_size);
for (const auto& s : vec) {
    result += s;
}
```

### P.10 不要依赖未定义行为

UB 是不可预测的，必须避免。

```cpp
// ❌ 不好：有符号溢出是 UB
int a = INT_MAX;
int b = a + 1;  // UB！

// ✅ 好：检测溢出
int a = INT_MAX;
if (a > INT_MAX - 1) throw std::overflow_error{"overflow"};
int b = a + 1;
```

### P.11 不要写刻意的晦涩代码

代码应可读，不要炫技。

```cpp
// ❌ 不好：过度使用运算符重载
a() < b ? c : d = e;

// ✅ 好：清晰直白
if (a() < b) {
    d = e;
} else {
    c = e;
}
```

---

## In: Source Files (源文件)

### In.1 头文件不要包含 using 指令

头文件中的 `using` 会污染所有包含者的命名空间。

```cpp
// ❌ 不好：header.h
using namespace std;

// ✅ 好：header.h
#include <string>
// 使用 std::string 完整限定名
```

### In.2 .cpp 文件中不要在全局作用域使用 using 指令

```cpp
// ❌ 不好
using namespace std;
using namespace std::chrono;

// ✅ 好：使用 using 声明或限定名
using std::string;
using std::vector;
```

### In.3 使用 #include guard 或 #pragma once

防止多重包含。

```cpp
// ✅ 方式一：传统 include guard
#ifndef MY_HEADER_H
#define MY_HEADER_H
// ...
#endif // MY_HEADER_H

// ✅ 方式二：#pragma once（非标准但广泛支持）
#pragma once
// ...
```

### In.4 不要在头文件中使用 using namespace

同 In.1，这是最常见的头文件污染源。

```cpp
// ❌ 不好
// mylib.h
using namespace std;

// ✅ 好
// mylib.h
namespace mylib {
    using std::string;
}
```

### In.5 避免 include 顺序依赖

头文件应自包含，不依赖包含者的 include 顺序。

```cpp
// ❌ 不好：a.h 依赖 <string> 但没有 include
// b.h 包含了 <string>
#include "b.h"
#include "a.h"  // a.h 中用到 string，依赖 b.h 已包含

// ✅ 好：每个头文件自包含
// a.h
#ifndef A_H
#define A_H
#include <string>  // 自己包含需要的依赖
// ...
#endif
```

---

## NL: Naming and Layout (命名与布局)

### NL.1 命名规范

| 类别 | 风格 | 示例 |
|------|------|------|
| 类/结构体 | PascalCase | `class MyClass` |
| 函数 | camelCase | `void doWork()` |
| 变量 | camelCase | `int itemCount` |
| 常量/枚举 | UPPER_CASE | `constexpr int MAX_SIZE` |
| 模板参数 | TPascalCase | `template<typename TElement>` |
| 宏 | UPPER_CASE | `#define MY_MACRO` |
| 命名空间 | lowercase | `namespace mylib` |

### NL.2 声明式代码每行一个

```cpp
// ❌ 不好
int x, y, z;
int* p, q;  // q 是 int，不是 int*！

// ✅ 好
int x;
int y;
int z;
int* p;
int* q;
```

### NL.3 保持一致的缩进

```cpp
// ❌ 不好：缩进不一致
if (cond) {
    do_a();
  do_b();
}

// ✅ 好
if (cond) {
    do_a();
    do_b();
}
```

### NL.4 使用 if/else 和循环的大括号

始终使用大括号，即使只有一条语句。

```cpp
// ❌ 不好
if (cond)
    do_a();
    do_b();  // 看起来像条件内，实际不是

// ✅ 好
if (cond) {
    do_a();
}
do_b();
```

### NL.5 避免嵌套过深

嵌套超过 4 层时应重构。

```cpp
// ❌ 不好：深层嵌套
if (a) {
    if (b) {
        if (c) {
            if (d) {
                do_work();
            }
        }
    }
}

// ✅ 好：提前返回
if (!a) return;
if (!b) return;
if (!c) return;
if (!d) return;
do_work();
```

### NL.16 使用有意义的名字

```cpp
// ❌ 不好
auto x = compute(v, n);  // x 是什么？v 是什么？

// ✅ 好
auto average_score = compute_student_average(scores, count);
```

### NL.17 避免缩写

```cpp
// ❌ 不好
int usr_cnt;
void calc_prf();

// ✅ 好
int user_count;
void calculate_profit();
```

### NL.18 使用 ALL_CAPS 命名宏

```cpp
// ❌ 不好
#define max_value 100

// ✅ 好
#define MAX_VALUE 100
// 更好：用 constexpr
constexpr int MAX_VALUE = 100;
```

### NL.19 使用 PascalCase 命名类和类型

```cpp
// ❌ 不好
class data_processor {};
typedef int value_type;

// ✅ 好
class DataProcessor {};
using ValueType = int;
```

### NL.20 使用 camelCase 命名函数和变量

```cpp
// ❌ 不好
void Calculate_Total();
int ItemCount;

// ✅ 好
void calculateTotal();
int itemCount;
```

---

## SF: Streams (流)

### SF.1 需要时使用标准流

```cpp
// ✅ 好：使用 iostream 进行 I/O
#include <iostream>
std::cout << "Hello, " << name << "!\n";
```

### SF.2 不要用 printf/scanf

`printf` 缺乏类型安全。

```cpp
// ❌ 不好：类型不安全
printf("Value: %d\n", some_string);  // UB！

// ✅ 好：类型安全
std::cout << "Value: " << value << "\n";
// 或 C++20 format
std::cout << std::format("Value: {}\n", value);
```

### SF.7 不要在条件和循环条件中使用 >> 作为逻辑

```cpp
// ❌ 不好：容易混淆
while (stream >> x)  // 看起来像移位操作

// ✅ 好：明确意图
while (std::getline(stream, line)) {
    // ...
}
```

---

## I: Interfaces (接口)

### I.1 使接口明确且严格

```cpp
// ❌ 不好：参数含义不明
void draw(int x, int y, int w, int h, int c);

// ✅ 好：类型明确
void draw(Rectangle rect, Color color);
```

### I.2 避免全局变量

全局变量破坏封装性，导致不可预测的状态。

```cpp
// ❌ 不好
int global_counter;

// ✅ 好：封装在类中
class Counter {
    int count_ = 0;
public:
    void increment() { ++count_; }
    int get() const { return count_; }
};
```

### I.3 函数应该简单明了

```cpp
// ❌ 不好：做太多事
void process() {
    read_input();
    validate();
    transform();
    write_output();
    log_result();
}

// ✅ 好：拆分为清晰的步骤
void process_pipeline() {
    auto data = read_input();
    validate(data);
    transform(data);
    write_output(data);
}
```

### I.4 不要在没有明确需求的情况下使函数 virtual

```cpp
// ❌ 不好：不需要多态却用了 virtual
class Point {
    virtual double distance() const;  // Point 不需要被继承多态
};

// ✅ 好：只有需要多态时才用
class Shape {
public:
    virtual double area() const = 0;
    virtual ~Shape() = default;
};
```

### I.5 不应该在没有意义的情况下使类 virtual

只有需要通过基类接口使用派生类时才用多态。

### I.6 倾向于使用 const 和 constexpr

```cpp
// ❌ 不好
int max = 100;
void print(const std::string& s) { /* 不修改 s 但没标记 */ }

// ✅ 好
constexpr int MAX = 100;
void print(const std::string& s) { /* 标记 const */ }
```

### I.7 定义类层次结构时声明析构函数为 virtual

```cpp
// ❌ 不好：缺失虚析构函数
class Base {
public:
    virtual void foo();
    // ~Base() 非虚！通过基类指针删除会 UB
};

// ✅ 好
class Base {
public:
    virtual void foo();
    virtual ~Base() = default;
};
```

### I.8 倾向于使用 gsl::not_null 而不是裸指针表示非空

```cpp
// ❌ 不好：调用者可能传 null
void process(int* data);

// ✅ 好：编译期/运行期保证非空
void process(gsl::not_null<int*> data);
```

### I.11 原始指针传递仅表示单个对象

```cpp
// ❌ 不好：用裸指针传递数组
void process(int* data, int size);

// ✅ 好：用 span 传递数组
void process(gsl::span<int> data);
```

### I.12 声明一个指针参数仅用于指向非拥有的对象

```cpp
// ❌ 不好：所有权语义不明
void use(int* p);

// ✅ 好：明确非拥有
void use(int& r);           // 引用：非拥有、非空
void use(gsl::not_null<int*> p);  // 非拥有、非空
```

### I.13 传递序列时使用 gsl::span

```cpp
// ❌ 不好
void process(int* data, size_t count);

// ✅ 好
void process(gsl::span<const int> data);
```

### I.22 避免复杂的初始化

```cpp
// ❌ 不好：复杂的构造函数
Widget(const Config& cfg, const std::vector<int>& data, bool flag = false, int mode = 0);

// ✅ 好：使用工厂或 Builder
auto w = Widget::Builder(cfg).with_data(data).build();
```

### I.24 倾向于使用 unique_ptr 和 shared_ptr 表示所有权

```cpp
// ❌ 不好：裸指针表示所有权
class Owner {
    Widget* w_;
public:
    Owner() : w_(new Widget()) {}
    ~Owner() { delete w_; }  // 容易忘记
};

// ✅ 好
class Owner {
    std::unique_ptr<Widget> w_;
public:
    Owner() : w_(std::make_unique<Widget>()) {}
};
```

### I.25 倾向于抽象类作为接口

```cpp
// ✅ 好：纯抽象接口
class ISerializable {
public:
    virtual ~ISerializable() = default;
    virtual std::string serialize() const = 0;
    virtual void deserialize(std::string_view data) = 0;
};
```

### I.26 跨 DLL 的稳定 ABI 使用 Pimpl 惯用法

```cpp
// ✅ 好：Pimpl 隐藏实现
// widget.h
class Widget {
public:
    Widget();
    ~Widget();
    void draw();
private:
    class Impl;
    std::unique_ptr<Impl> impl_;
};

// widget.cpp
class Widget::Impl {
public:
    void draw() { /* 实现 */ }
};
```

### I.27 稳定库 ABI 使用稳定的命名约定

避免重命名导出符号，使用版本化命名空间。

### I.30 编译期不变量使用模板

```cpp
// ❌ 不好：运行时检查编译期可知的值
class Matrix {
    int rows_, cols_;
    void check(int r, int c) { if (r >= rows_ || c >= cols_) throw; }
};

// ✅ 好：模板参数表示编译期维度
template<int Rows, int Cols>
class Matrix {
    void check(int r, int c) { /* 静态断言可优化 */ }
};
```

### I.31 避免简单的 get/set 函数暴露内部数据

```cpp
// ❌ 不好
class Point {
    double x_, y_;
public:
    double getX() const { return x_; }
    void setX(double v) { x_ = v; }
};

// ✅ 好：提供有意义的操作
class Point {
    double x_, y_;
public:
    double x() const { return x_; }
    void translate(double dx, double dy);
    double distanceTo(const Point& other) const;
};
```

### I.33 使用 gsl::index 表示下标

```cpp
// ❌ 不好
void set_at(std::vector<int>& v, int i, int val);

// ✅ 好
void set_at(std::vector<int>& v, gsl::index i, int val);
```

---

## F: Functions (函数)

### F.1 函数应该简短

函数体不宜过长，通常不超过一屏。

### F.2 函数应该有单一出口（可选）

保持逻辑清晰，不要在函数中间随意 return。

```cpp
// ❌ 不好：多个出口散布各处
int compute(int x) {
    if (x < 0) return -1;
    // ... 很多代码
    if (x > 100) return -2;
    // ... 更多代码
    return x * 2;
}

// ✅ 好：逻辑分支清晰
int compute(int x) {
    if (x < 0) return error_negative;
    if (x > 100) return error_too_large;
    return x * 2;
}
```

### F.3 保持函数简单

每个函数只做一件事。

### F.4 如果函数做两件事，拆分为两个函数

```cpp
// ❌ 不好
void process_and_save(const Data& d) {
    auto result = process(d);
    save(result);
}

// ✅ 好
Data process(const Data& d);
void save(const Data& d);
```

### F.5 使用普通循环和简单循环变量

```cpp
// ❌ 不好：花哨的循环
for (auto it = std::rbegin(v); it != std::rend(v); std::advance(it, 2)) { /* ... */ }

// ✅ 好：简单清晰
for (int i = static_cast<int>(v.size()) - 1; i >= 0; i -= 2) { /* ... */ }
```

### F.6 保持函数参数少

理想情况下不超过 4 个参数，超过时考虑使用结构体。

```cpp
// ❌ 不好：参数太多
void create_window(const std::string& title, int x, int y, int w, int h, bool resizable, int style);

// ✅ 好：用结构体封装
struct WindowConfig {
    std::string title;
    int x, y, width, height;
    bool resizable = false;
    int style = 0;
};
void create_window(const WindowConfig& cfg);
```

### F.7 倾向于返回值

```cpp
// ❌ 不好：输出参数
bool find(const std::vector<int>& v, int target, int& result);

// ✅ 好：返回值
std::optional<int> find(const std::vector<int>& v, int target);
```

### F.8 倾向于纯函数

纯函数无副作用，更易测试和推理。

```cpp
// ❌ 不好：有副作用
int add_and_print(int a, int b) {
    int result = a + b;
    std::cout << result;
    return result;
}

// ✅ 好：纯函数
int add(int a, int b) {
    return a + b;
}
```

### F.15 普通函数参数使用 T* 或 T&

```cpp
// ❌ 不好
void process(int value);  // 总是拷贝

// ✅ 好
void process(const int& value);  // in 参数用引用
void process(int* out);          // out 参数用指针（或更好的方式）
```

### F.16 传 in 参数时，优先使用 const T&

```cpp
// ❌ 不好：不必要的拷贝
void print(std::string s);

// ✅ 好
void print(const std::string& s);
```

### F.17 传 in 参数时，小类型优先用 T

```cpp
// ✅ 好：小类型直接传值
void draw_point(Point p);       // Point 很小（通常 2 个 double）
void set_flag(bool flag);       // bool 只有 1 字节
```

### F.18 传 out 参数时，优先使用返回值

```cpp
// ❌ 不好
void get_next(std::vector<int>::iterator& it, int& value);

// ✅ 好
std::pair<std::vector<int>::iterator, int> get_next(std::vector<int>::iterator it);
// 或
std::optional<int> get_next(std::vector<int>::iterator& it);
```

### F.19 传 in/out 参数时，使用 T&

```cpp
// ✅ 好
void normalize(double& value);  // 修改 value 并返回
```

### F.20 输出参数使用 gsl::span

```cpp
// ❌ 不好
void get_data(int* buffer, int* count);

// ✅ 好
void get_data(gsl::span<int> buffer);
```

### F.21 返回拥有所有权的对象时，返回 unique_ptr\<T\>

```cpp
// ❌ 不好：裸指针所有权不明
Widget* createWidget();

// ✅ 好
std::unique_ptr<Widget> createWidget();
```

### F.22 返回共享所有权的对象时，返回 shared_ptr\<T\>

```cpp
// ✅ 好
std::shared_ptr<Resource> getResource();
```

### F.50 使用 lambda 进行复杂的初始化

```cpp
// ❌ 不好：复杂的初始化散布各处
Widget w;
w.setOption(A);
w.setOption(B);
w.configure(C);

// ✅ 好：lambda 初始化
Widget w = [&] {
    Widget tmp;
    tmp.setOption(A);
    tmp.setOption(B);
    tmp.configure(C);
    return tmp;
}();
```

### F.51 倾向于使用 lambda 而不是绑定

```cpp
// ❌ 不好：std::bind 难以阅读
auto fn = std::bind(&Foo::bar, obj, _1, 42);

// ✅ 好：lambda 清晰
auto fn = [&obj](auto arg) { obj.bar(arg, 42); };
```

### F.52 倾向于使用 lambda 而不是 std::bind

同 F.51。`std::bind` 有类型推导问题和性能开销，lambda 是更好的选择。

---

## C: Classes and Hierarchies (类与层次结构)

### C.1 用类表示概念

```cpp
// ✅ 好：类封装概念
class Date {
    int year_, month_, day_;
public:
    Date(int y, int m, int d);
    bool is_valid() const;
};
```

### C.2 用 struct 表示仅有数据的结构

```cpp
// ✅ 好：纯数据用 struct
struct Point {
    double x = 0.0;
    double y = 0.0;
};
```

### C.3 用 class 表示有不变量约束的

```cpp
// ✅ 好：有约束的用 class
class Temperature {
    double kelvin_;
public:
    explicit Temperature(double k) : kelvin_(k) {
        Expects(k >= 0);  // 绝对温度不能为负
    }
    double celsius() const { return kelvin_ - 273.15; }
};
```

### C.4 使函数成为成员函数，仅当它需要直接访问类的表示

```cpp
// ❌ 不好：不需要访问内部却做成成员
class Matrix {
    std::vector<double> data_;
    int rows_, cols_;
public:
    double determinant() const;  // 可以是自由函数
};

// ✅ 好：自由函数
double determinant(const Matrix& m);
```

### C.5 将辅助函数放在类定义中

辅助实现细节应放在类内部的匿名命名空间或作为私有成员。

### C.7 不要用 this 指针访问未初始化的成员

```cpp
// ❌ 不好：构造函数体执行前成员未初始化
class Foo {
    int x_;
public:
    Foo() { use(x_); }  // x_ 未初始化！
};

// ✅ 好：使用成员初始化列表
class Foo {
    int x_;
public:
    Foo() : x_(42) { use(x_); }
};
```

### C.8 倾向于在类内直接初始化成员

```cpp
// ❌ 不好
class Foo {
    std::string name_;
    std::vector<int> data_;
public:
    Foo() : name_(""), data_() {}
};

// ✅ 好：默认成员初始化
class Foo {
    std::string name_{"default"};
    std::vector<int> data_{};
public:
    Foo() = default;
};
```

### C.9 不要在构造函数中调用虚函数

```cpp
// ❌ 不好
class Base {
public:
    Base() { init(); }  // 不会调用派生类的 override！
    virtual void init() { /* ... */ }
};

// ✅ 好：使用工厂模式
class Base {
public:
    static std::unique_ptr<Base> create();  // 工厂
protected:
    virtual void init() = 0;
};
```

### C.11 让构造函数做简单的不变量建立

```cpp
// ❌ 不好：构造函数做复杂工作
class Database {
public:
    Database(const std::string& path) {
        open_connection();
        create_tables();
        load_cache();
        start_monitor();
    }
};

// ✅ 好：简单初始化，复杂工作交给方法
class Database {
public:
    explicit Database(const std::string& path) : path_(path) {
        Expects(!path.empty());
    }
    void initialize();  // 复杂初始化单独调用
};
```

### C.12 不要使用 set 函数来建立不变量

```cpp
// ❌ 不好
class Date {
    int y_, m_, d_;
public:
    Date() = default;
    void set_year(int y) { y_ = y; }
    void set_month(int m) { m_ = m; }
    void set_day(int d) { d_ = d; }
    // 不变量可能从未建立！
};

// ✅ 好
class Date {
    int y_, m_, d_;
public:
    Date(int y, int m, int d) {
        Expects(is_valid(y, m, d));
        y_ = y; m_ = m; d_ = d;
    }
};
```

### C.20 Rule of Five/Zero

如果一个类是资源句柄，需要完整的特殊成员函数。

```cpp
// ❌ 不好：只有析构函数
class FileHandle {
    FILE* fp_;
public:
    FileHandle(const char* path) : fp_(fopen(path, "r")) {}
    ~FileHandle() { if (fp_) fclose(fp_); }
    // 拷贝/移动操作会 double-close！
};

// ✅ 好：Rule of Five
class FileHandle {
    FILE* fp_ = nullptr;
public:
    explicit FileHandle(const char* path) : fp_(fopen(path, "r")) {}
    ~FileHandle() { if (fp_) fclose(fp_); }
    FileHandle(const FileHandle&) = delete;
    FileHandle& operator=(const FileHandle&) = delete;
    FileHandle(FileHandle&& other) noexcept : fp_(other.fp_) { other.fp_ = nullptr; }
    FileHandle& operator=(FileHandle&& other) noexcept {
        if (this != &other) {
            if (fp_) fclose(fp_);
            fp_ = other.fp_;
            other.fp_ = nullptr;
        }
        return *this;
    }
};
```

### C.21 默认操作保持一致

如果定义了析构函数，拷贝/移动操作应显式定义或 `=delete`。

```cpp
// ❌ 不好：只定义析构函数
class Foo {
    int* data_;
public:
    ~Foo() { delete[] data_; }  // 但拷贝操作会 double-delete
};

// ✅ 好：遵循 Rule of Five
class Foo {
    std::unique_ptr<int[]> data_;
public:
    ~Foo() = default;  // unique_ptr 自动处理
    Foo(const Foo&) = delete;
    Foo& operator=(const Foo&) = delete;
    Foo(Foo&&) = default;
    Foo& operator=(Foo&&) = default;
};
```

### C.22 使默认操作保持一致

拷贝操作语义应与移动操作一致。

### C.33 如果一个类是容器，则提供 initializer_list 构造函数

```cpp
// ✅ 好
class IntList {
    std::vector<int> data_;
public:
    IntList() = default;
    IntList(std::initializer_list<int> list) : data_(list) {}
};
IntList v{1, 2, 3, 4, 5};
```

### C.35 一个有默认操作的类可能需要用户定义的析构函数

当默认析构不够用时。

### C.36 析构函数不能失败

```cpp
// ❌ 不好
~Connection() {
    if (!close()) throw std::runtime_error{"close failed"};  // 绝对不要！
}

// ✅ 好
~Connection() noexcept {
    try { close(); } catch (...) {
        // 记录日志，但不要传播异常
        log_error("close failed in destructor");
    }
}
```

### C.37 不要使析构函数为纯虚函数

```cpp
// ❌ 不好
class Base {
public:
    virtual ~Base() = 0;  // 需要提供定义！
};

// ✅ 好
class Base {
public:
    virtual ~Base() = default;
};
```

### C.43 确保拷贝操作的参数是 const T&

```cpp
// ❌ 不好
Foo(Foo& other);  // 非 const，无法拷贝 const 对象

// ✅ 好
Foo(const Foo& other);
Foo& operator=(const Foo& other);
```

### C.44 尽可能让构造函数简单且非虚拟

构造函数中不应有复杂逻辑或虚函数调用。

### C.45 不要定义仅初始化数据成员的默认构造函数

```cpp
// ❌ 不好
struct Point {
    double x, y;
    Point() : x(0), y(0) {}  // 不必要的
};

// ✅ 好
struct Point {
    double x = 0, y = 0;  // 默认成员初始化
};
```

### C.46 声明 single-argument 构造函数为 explicit

```cpp
// ❌ 不好：隐式转换
class Widget {
    int size_;
public:
    Widget(int s) : size_(s) {}
};
Widget w = 42;  // 隐式转换！
void process(Widget w);
process(10);    // 隐式转换！

// ✅ 好
class Widget {
    int size_;
public:
    explicit Widget(int s) : size_(s) {}
};
Widget w{42};     // OK
// process(10);   // 编译错误
process(Widget{10});  // OK
```

### C.47 按成员声明顺序定义和初始化成员

```cpp
// ❌ 不好：初始化顺序与声明顺序不一致
class Foo {
    int a_;
    int b_;
public:
    Foo() : b_(2), a_(1) {}  // a_ 先于 b_ 初始化（按声明顺序）
};

// ✅ 好
class Foo {
    int a_;
    int b_;
public:
    Foo() : a_(1), b_(2) {}  // 与声明顺序一致
};
```

### C.48 优先使用 in-class 成员初始化器

```cpp
// ✅ 好
class Widget {
    std::string name_{"unnamed"};
    int count_{0};
    bool active_{false};
public:
    Widget() = default;
    Widget(std::string n) : name_(std::move(n)) {}
};
```

### C.49 倾向于初始化而不是赋值

```cpp
// ❌ 不好：先默认构造再赋值
class Foo {
    std::string name_;
public:
    Foo(const std::string& n) { name_ = n; }  // 两次操作
};

// ✅ 好：直接初始化
class Foo {
    std::string name_;
public:
    Foo(const std::string& n) : name_(n) {}  // 一次操作
};
```

### C.67 多态类应该抑制拷贝

```cpp
// ✅ 好
class Base {
public:
    Base() = default;
    Base(const Base&) = delete;
    Base& operator=(const Base&) = delete;
    Base(Base&&) = default;
    Base& operator=(Base&&) = default;
    virtual ~Base() = default;
    virtual void interface() = 0;
};
```

### C.120 使用类层次表示具有共同属性的一族概念

```cpp
// ✅ 好
class Shape {
public:
    virtual ~Shape() = default;
    virtual double area() const = 0;
    virtual void draw() const = 0;
};
```

### C.121 如果基类是多态的，则提供虚析构函数

同 I.7。

### C.122 使用虚函数来避免向下转型

```cpp
// ❌ 不好：向下转型
void process(Shape* s) {
    if (auto* c = dynamic_cast<Circle*>(s)) {
        c->draw_circle();
    } else if (auto* r = dynamic_cast<Rect*>(s)) {
        r->draw_rect();
    }
}

// ✅ 好：虚函数多态
class Shape {
public:
    virtual void draw() const = 0;
};
void process(const Shape& s) {
    s.draw();  // 自动分派
}
```

### C.126 抽象类不要有数据成员

```cpp
// ❌ 不好
class IShape {
    int color_;  // 接口不应有数据
public:
    virtual void draw() = 0;
};

// ✅ 好
class IShape {
public:
    virtual ~IShape() = default;
    virtual void draw() const = 0;
    virtual void set_color(int c) = 0;
    virtual int color() const = 0;
};
```

### C.127 含有虚函数的类需要虚析构函数

```cpp
// ❌ 不好
class Base { virtual void foo(); };  // 无虚析构

// ✅ 好
class Base {
public:
    virtual ~Base() = default;
    virtual void foo();
};
```

### C.128 虚函数应该指定 override

```cpp
// ❌ 不好：签名不匹配时不会报错
class Derived : public Base {
    void foo(int);  // 以为覆盖了 Base::foo()，实际不是
};

// ✅ 好
class Derived : public Base {
    void foo() override;  // 签名不匹配会编译错误
};
```

### C.129 制作虚函数 override 和 final

```cpp
// ✅ 好
class Derived final : public Base {
    void foo() override;     // 覆盖基类
    void bar() override final;  // 覆盖基类，禁止进一步覆盖
};
```

### C.131 避免无意义的虚函数调用

构造/析构期间调用虚函数不会多态分派。

### C.132 不要声明覆盖函数 final（除非有明确理由）

`final` 有其用途但不要过度使用。

### C.133 避免 protected 数据

```cpp
// ❌ 不好
class Base {
protected:
    int data_;  // 任何派生类都能随意修改
};

// ✅ 好：protected 函数 + private 数据
class Base {
    int data_;
protected:
    int data() const { return data_; }
    void set_data(int d) { data_ = d; }
};
```

### C.134 确保所有替代都是可替代的（LSP）

```cpp
// ❌ 不好：违反 LSP
class Rectangle {
public:
    virtual void set_width(int w) { width_ = w; }
    virtual void set_height(int h) { height_ = h; }
    virtual int area() const { return width_ * height_; }
};

class Square : public Rectangle {
public:
    void set_width(int w) override { width_ = height_ = w; }
    void set_height(int h) override { width_ = height_ = h; }
    // Square 不能替代 Rectangle！
};
```

### C.135 多态类不要用转换操作符

### C.136 多态类不要用比较操作符

```cpp
// ❌ 不好
class Base {
public:
    virtual bool operator==(const Base& other) const;
    // 比较可能被派生类覆盖，语义混乱
};
```

### C.145 访问多态对象通过指针和引用

```cpp
// ❌ 不好：对象切片
void process(Base b);  // 派生类对象传入会被切片

// ✅ 好
void process(const Base& b);  // 引用保留多态
void process(const Base* b);  // 指针保留多态
```

---

## R: Resource Management (资源管理)

### R.1 用对象管理资源（RAII）

```cpp
// ❌ 不好：手动管理
void f() {
    FILE* f = fopen("data.txt", "r");
    if (!f) return;
    char buf[1024];
    fread(buf, 1, 1024, f);
    fclose(f);
}

// ✅ 好：RAII
void f() {
    std::ifstream f("data.txt");
    char buf[1024];
    f.read(buf, 1024);
}  // 自动关闭
```

### R.2 在同一语句中不要混合使用 new 和裸指针

```cpp
// ❌ 不好：异常安全
process(std::shared_ptr<int>(new int(42)), compute());
// compute() 可能先于 shared_ptr 构造执行，导致内存泄漏

// ✅ 好
process(std::make_shared<int>(42), compute());
```

### R.3 裸指针不应拥有所有权

```cpp
// ❌ 不好
class Owner {
    Widget* widget_;  // 谁来 delete？
};

// ✅ 好
class Owner {
    std::unique_ptr<Widget> widget_;  // 所有权明确
};
```

### R.4 裸指针不应该用于传递所有权

```cpp
// ❌ 不好
Widget* create();  // 调用者需要 delete？谁拥有？

// ✅ 好
std::unique_ptr<Widget> create();
```

### R.5 倾向于使用 gsl::span 而不是裸指针和大小

```cpp
// ❌ 不好
void process(int* data, size_t count);

// ✅ 好
void process(gsl::span<int> data);
```

### R.6 使用 unique_ptr\<T\> 表示独占所有权

```cpp
// ✅ 好
auto widget = std::make_unique<Widget>();
// 唯一所有者，不可拷贝，可移动
```

### R.7 使用 shared_ptr\<T\> 表示共享所有权

```cpp
// ✅ 好
auto shared = std::make_shared<Widget>();
auto copy = shared;  // 引用计数 +1
```

### R.8 不需要共享时使用 unique_ptr

```cpp
// ❌ 不好
std::shared_ptr<int> p = std::make_shared<int>(42);  // 只有一个人用

// ✅ 好
auto p = std::make_unique<int>(42);
```

### R.10 避免使用 malloc/free

```cpp
// ❌ 不好
int* p = (int*)malloc(100 * sizeof(int));
free(p);

// ✅ 好
auto p = std::make_unique<int[]>(100);
```

### R.11 避免调用 new/delete 显式

```cpp
// ❌ 不好
auto p = new Widget();
// ...
delete p;

// ✅ 好
auto p = std::make_unique<Widget>();
```

### R.12 立即使用结果初始化智能指针

```cpp
// ❌ 不好
std::unique_ptr<int> p;
p.reset(new int(42));  // 不是异常安全的

// ✅ 好
auto p = std::make_unique<int>(42);
```

### R.13 每次执行直接使用 unique_ptr

### R.14-R.15 不要滥用 shared_ptr

共享所有权是最后手段，优先 unique_ptr。

### R.20 使用 unique_ptr 或 shared_ptr 表示所有权

### R.21 倾向于使用 unique_ptr\<T[]\> 或 vector\<T\>

```cpp
// ❌ 不好
int* arr = new int[n];

// ✅ 好
auto arr = std::make_unique<int[]>(n);
// 更好
std::vector<int> arr(n);
```

### R.22 使用 make_unique() 创建 unique_ptr

```cpp
// ❌ 不好
std::unique_ptr<Widget> p(new Widget());

// ✅ 好
auto p = std::make_unique<Widget>();
```

### R.23 使用 make_shared() 创建 shared_ptr

```cpp
// ❌ 不好
std::shared_ptr<Widget> p(new Widget());

// ✅ 好
auto p = std::make_shared<Widget>();
```

### R.24 使用 weak_ptr 打破循环引用

```cpp
// ✅ 好
struct Node {
    std::shared_ptr<Node> next;
    std::weak_ptr<Node> prev;  // 打破循环
};
```

### R.30 使用智能指针作为参数传递所有权

```cpp
// ✅ 好：传递所有权
void take(std::unique_ptr<Widget> w);
```

### R.31 如果需要共享语义，则传递 shared_ptr

```cpp
void share(std::shared_ptr<Resource> r);
```

### R.32 函数返回时使用 unique_ptr

```cpp
// ✅ 好
std::unique_ptr<Widget> create_widget() {
    return std::make_unique<Widget>();
}
```

### R.33 使用 shared_ptr 返回共享所有权的对象

### R.34 使用 weak_ptr 来观察 shared_ptr

```cpp
// ✅ 好
std::weak_ptr<Cache> cache_weak_;
void use_cache() {
    if (auto cache = cache_weak_.lock()) {
        cache->get("key");
    }
}
```

### R.35 使用 gsl::owner\<T*\> 标记拥有所有权的指针

```cpp
// ✅ 好
gsl::owner<FILE*> f = fopen("data.txt", "r");
// 明确表示 f 拥有该资源
```

### R.36 使用 gsl::span 表示连续序列

```cpp
void process(gsl::span<const double> data);
```

### R.37 不要传递指向容器元素的指针或引用

```cpp
// ❌ 不好：容器修改后指针失效
int* p = &vec[0];
vec.push_back(42);  // p 可能悬垂
*p = 10;            // UB！

// ✅ 好
size_t idx = 0;
vec.push_back(42);
vec[idx] = 10;  // 安全
```

### R.40 使用函数 try 块处理构造函数异常

```cpp
// ✅ 好
class Widget : public Base {
    std::string name_;
public:
    Widget(const std::string& n) try
        : Base(n.size()), name_(n)
    {
        // 构造函数体
    } catch (const std::exception& e) {
        std::cerr << "Widget construction failed: " << e.what() << "\n";
        throw;
    }
};
```

---

## ES: Expressions and Statements (表达式和语句)

### ES.1 代码应明确表达意图

```cpp
// ❌ 不好
bool b = !(a < b) && (c == d);

// ✅ 好
bool is_valid = (a >= b) && (c == d);
```

### ES.2 使用合适的类型

```cpp
// ❌ 不好
int count = vec.size();  // size_t → int 可能截断

// ✅ 好
auto count = vec.size();
// 或
std::ptrdiff_t count = static_cast<std::ptrdiff_t>(vec.size());
```

### ES.3 不要重复代码

### ES.5 保持作用域小

```cpp
// ❌ 不好
int i;
for (i = 0; i < n; ++i) { /* ... */ }
// i 在此仍可见

// ✅ 好
for (int i = 0; i < n; ++i) { /* ... */ }
// i 不再可见
```

### ES.6 声明名称时立即初始化

```cpp
// ❌ 不好
int x;
x = compute();

// ✅ 好
int x = compute();
```

### ES.7 保持常用和局部名称简短，不常用和非局部名称较长

```cpp
// ✅ 好
for (int i = 0; i < n; ++i) { /* i 足够 */ }
auto connection_pool = std::make_shared<ConnectionPool>();
```

### ES.8 避免看起来相似的名称

```cpp
// ❌ 不好
int data_count;
int data_cnt;
int dataCount;

// ✅ 好
int input_count;
int output_count;
int total_count;
```

### ES.9 避免使用 ALL_CAPS 名称（除宏外）

```cpp
// ❌ 不好
int MAX_SIZE;  // 看起来像宏

// ✅ 好
constexpr int MAX_SIZE = 100;  // 这是常量，但命名可能与宏冲突
// 更好
constexpr int kMaxSize = 100;
```

### ES.10 声明一个变量一次

```cpp
// ❌ 不好
int x = 0;
// ...
x = compute();
// ...
x = compute2();

// ✅ 好：每次使用不同名字
int initial = 0;
int result1 = compute();
int result2 = compute2();
```

### ES.11 使用 auto 避免多余的类型重复

```cpp
// ❌ 不好
std::unordered_map<std::string, std::vector<int>>::iterator it = map.begin();

// ✅ 好
auto it = map.begin();
```

### ES.12 不要重复命名空间名称

```cpp
// ❌ 不好
std::vector<std::string> vs = std::vector<std::string>{"a", "b"};

// ✅ 好
std::vector<std::string> vs{"a", "b"};
```

### ES.20 使用 ++i 而不是 i++

```cpp
// ❌ 不好（对于迭代器，后置++ 有额外开销）
for (auto it = v.begin(); it != v.end(); it++) { /* ... */ }

// ✅ 好
for (auto it = v.begin(); it != v.end(); ++it) { /* ... */ }
// 更好
for (auto& elem : v) { /* ... */ }
```

### ES.21 不要在循环条件中引入不必要的变量

```cpp
// ❌ 不好
for (int i = 0; i < compute_limit(); ++i) { /* compute_limit() 每次都调用 */ }

// ✅ 好
const int limit = compute_limit();
for (int i = 0; i < limit; ++i) { /* ... */ }
```

### ES.22 不要在无用变量上浪费空间

### ES.23 倾向于使用 {} 初始化器语法

```cpp
// ❌ 不好
int x(42);
std::vector<int> v(10);

// ✅ 好
int x{42};
std::vector<int> v{10};
// 避免 narrowing
// int y{3.14};  // 编译错误！
```

### ES.24 使用 unique_ptr 保持指针有效性

### ES.25 不要在 if 语句中定义变量（有争议，但需注意作用域）

### ES.26 不要用递归代替循环

```cpp
// ❌ 不好：简单任务用递归
int sum(int n) {
    if (n <= 0) return 0;
    return n + sum(n - 1);  // 可能栈溢出
}

// ✅ 好
int sum(int n) {
    int result = 0;
    for (int i = 1; i <= n; ++i) result += i;
    return result;
}
```

### ES.27 使用循环处理元素序列

```cpp
// ❌ 不好：手动索引
for (int i = 0; i < static_cast<int>(v.size()); ++i) {
    process(v[i]);
}

// ✅ 好：范围 for
for (const auto& elem : v) {
    process(elem);
}
```

### ES.28 避免使用 goto

```cpp
// ❌ 不好
for (...) {
    for (...) {
        if (error) goto cleanup;
    }
}
cleanup:
    release_resources();

// ✅ 好
try {
    for (...) {
        for (...) {
            if (error) throw std::runtime_error{"..."};
        }
    }
} catch (...) {
    release_resources();
}
```

### ES.30 不要用宏来操作程序文本

```cpp
// ❌ 不好
#define FIELD(name) name##_

// ✅ 好：使用模板或 constexpr
```

### ES.31 不要用宏来定义常量或函数

```cpp
// ❌ 不好
#define PI 3.14159265
#define SQUARE(x) ((x) * (x))

// ✅ 好
constexpr double PI = 3.14159265;
constexpr double square(double x) { return x * x; }
```

### ES.32 使用宏定义整个编译时函数（如果必须）

```cpp
// 如果必须用宏，确保完整
#define CHECK(expr) \
    do { if (!(expr)) { \
        std::cerr << "Check failed: " #expr << " at " << __FILE__ << ":" << __LINE__ << "\n"; \
        std::abort(); \
    } } while(0)
```

### ES.33 如果你必须使用宏，给它唯一的名字

```cpp
// ✅ 好：带项目前缀
#define MYLIB_VERSION_MAJOR 2
#define MYLIB_VERSION_MINOR 1
```

### ES.34 不要使用宏来定义符号常量

同 ES.31。

### ES.40 避免复杂的表达式

```cpp
// ❌ 不好
int result = a > b ? (c < d ? x : y) : (e > f ? z : w);

// ✅ 好：拆分
int result;
if (a > b) {
    result = (c < d) ? x : y;
} else {
    result = (e > f) ? z : w;
}
```

### ES.41 使用括号避免歧义

```cpp
// ❌ 不好
if (a & b != c)  // 实际是 a & (b != c)

// ✅ 好
if ((a & b) != c)
```

### ES.42 保持使用简单的声明语句

```cpp
// ❌ 不好
int* p = new int[10], *q = p + 5;

// ✅ 好
auto p = std::make_unique<int[]>(10);
int* q = p.get() + 5;
```

### ES.43 避免在表达式中有副作用

```cpp
// ❌ 不好
int i = ++a + a++;

// ✅ 好
++a;
int i = a + a;
++a;
```

### ES.44 倾向于使用命名变量而不是显式类型转换

```cpp
// ❌ 不好
auto y = static_cast<double>(static_cast<int>(x) + 1) / 2.0;

// ✅ 好
int x_int = static_cast<int>(x);
double y = (x_int + 1) / 2.0;
```

### ES.45 避免有魔力的常量（使用 constexpr）

```cpp
// ❌ 不好
if (status == 3) { /* retry */ }

// ✅ 好
constexpr int STATUS_RETRY = 3;
if (status == STATUS_RETRY) { /* retry */ }
```

### ES.46 避免窄化转换

```cpp
// ❌ 不好
int x = 3.14;  // 窄化！
char c = 1000; // 窄化！

// ✅ 好
int x = static_cast<int>(3.14);  // 显式，知道会发生什么
// 或
double x = 3.14;
```

### ES.47 使用 nullptr 而不是 0 或 NULL

```cpp
// ❌ 不好
int* p = 0;
int* q = NULL;

// ✅ 好
int* p = nullptr;
```

### ES.48 避免比较和布尔值之间的转换

```cpp
// ❌ 不好
if (strcmp(a, b)) { /* 相等时？不相等时？ */ }

// ✅ 好
if (std::string_view{a} == std::string_view{b}) { /* 相等 */ }
```

### ES.49 使用命名的转换

```cpp
// ❌ 不好
double d = (double)intValue;
int* p = (int*)voidPtr;

// ✅ 好
double d = static_cast<double>(intValue);
int* p = static_cast<int*>(voidPtr);
```

### ES.50 不要使用 C 风格转换

```cpp
// ❌ 不好
int x = (int)3.14;
Base* b = (Base*)derived_ptr;

// ✅ 好
int x = static_cast<int>(3.14);
Base* b = dynamic_cast<Base*>(derived_ptr);
```

### ES.55 避免需要范围检查的算术运算

### ES.56 只写标准库支持的算法

### ES.60 避免未同步的并发 new/delete

### ES.61 使用赋值初始化智能指针

```cpp
// ❌ 不好
std::shared_ptr<int> p(new int(42));

// ✅ 好
auto p = std::make_shared<int>(42);
```

### ES.62 不要在循环条件中比较指针

```cpp
// ❌ 不好
while (p != end) { /* ... */ }

// ✅ 好
while (p != std::end(container)) { /* ... */ }
// 或
for (auto& elem : container) { /* ... */ }
```

### ES.63 不要切片

```cpp
// ❌ 不好
class Base { /* ... */ };
class Derived : public Base { /* ... */ };
Derived d;
Base b = d;  // 切片！多态信息丢失

// ✅ 好
Derived d;
Base& b = d;  // 引用，保留多态
```

### ES.65 不要解引用无效指针

```cpp
// ❌ 不好
int* p = nullptr;
*p = 42;  // UB

// ✅ 好
if (p) *p = 42;
// 更好：使用引用或 gsl::not_null
```

### ES.70 倾向于 switch 而不是 if-else if

```cpp
// ❌ 不好
if (type == 1) { /* ... */ }
else if (type == 2) { /* ... */ }
else if (type == 3) { /* ... */ }

// ✅ 好
switch (type) {
    case 1: /* ... */ break;
    case 2: /* ... */ break;
    case 3: /* ... */ break;
    default: /* ... */ break;
}
```

### ES.71 倾向于范围 for 语句

```cpp
// ❌ 不好
for (auto it = v.begin(); it != v.end(); ++it) { process(*it); }

// ✅ 好
for (const auto& elem : v) { process(elem); }
```

### ES.72 倾向于 for 语句而不是 while 语句

```cpp
// ❌ 不好
int i = 0;
while (i < n) { process(v[i]); ++i; }

// ✅ 好
for (int i = 0; i < n; ++i) { process(v[i]); }
```

### ES.73 倾向于 while 当循环测试不是简单范围表达式

```cpp
// ✅ 好
while (std::getline(file, line)) {
    process(line);
}
```

### ES.74 倾向于 break 和 continue 而不是 goto

### ES.75 避免使用 do 语句

### ES.76 避免使用 goto

### ES.77 循环体中做最小量的工作

```cpp
// ❌ 不好
for (auto& item : items) {
    validate(item);
    transform(item);
    save(item);
    log(item);
    notify(item);
}

// ✅ 好：管道式处理
for (auto& item : items) {
    transform(item);
}
save_all(items);
log_all(items);
```

### ES.78 else 分支始终使用大括号

```cpp
// ❌ 不好
if (cond) {
    do_a();
} else
    do_b();

// ✅ 好
if (cond) {
    do_a();
} else {
    do_b();
}
```

### ES.79 循环和分支中使用大括号

同 NL.4。

### ES.84 不要声明没有初始化器的局部变量

```cpp
// ❌ 不好
int x;
std::string s;

// ✅ 好
int x = 0;
std::string s;
```

### ES.85 让空语句显式化

```cpp
// ❌ 不好
while (wait_for_event()) ;

// ✅ 好
while (wait_for_event()) {
    // intentionally empty
}
```

### ES.86 避免修改控制流的 lambda

### ES.87 避免使用会添加控制流的函数

### ES.100 不要进行有符号整数溢出的运算

```cpp
// ❌ 不好
int a = INT_MAX;
int b = a + 1;  // UB

// ✅ 好
if (a > INT_MAX - 1) handle_overflow();
int b = a + 1;
```

### ES.101 不要进行无符号整数溢出的运算

```cpp
// ❌ 不好
unsigned int a = 0;
unsigned int b = a - 1;  // 环绕，不是预期的行为

// ✅ 好
if (a == 0) handle_error();
unsigned int b = a - 1;
```

### ES.102 使用有符号算术运算

```cpp
// ✅ 好
int sum = 0;
for (int i = 0; i < n; ++i) sum += data[i];
```

### ES.103 不要溢出

### ES.104 不要执行下溢操作

### ES.106 不要尝试使用无效的下标

```cpp
// ❌ 不好
int x = v[i];  // i 可能越界

// ✅ 好
int x = v.at(i);  // 边界检查
// 或使用 gsl::span
```

### ES.107-ES.110 不要写入空指针、悬垂指针、越界数组

```cpp
// ❌ 不好
int* p = nullptr;
*p = 42;  // UB

// ✅ 好
if (p) *p = 42;
```

### ES.111 不要使用 C 风格数组

```cpp
// ❌ 不好
int arr[100];

// ✅ 好
std::array<int, 100> arr;
// 或
std::vector<int> arr(100);
```

### ES.112 使用 gsl::span 替代指针算术

```cpp
// ❌ 不好
void process(int* data, int size) {
    for (int i = 0; i < size; ++i) {
        use(data[i]);
    }
}

// ✅ 好
void process(gsl::span<int> data) {
    for (auto& elem : data) {
        use(elem);
    }
}
```

---

## Per: Performance (性能)

### Per.1 不要进行不成熟的优化

```cpp
// ❌ 不好：过早优化
// 还没测量就知道哪里是瓶颈
int sum = 0;
for (int i = 0; i < n; ++i) {
    sum += (i & 1) ? data[i] : 0;  // "优化"位运算
}

// ✅ 好：先写清晰的代码，再根据 profiling 优化
int sum = 0;
for (int i = 0; i < n; i += 2) {
    sum += data[i];
}
```

### Per.2-Per.4 不要不必要地优化/复杂化

先测量，再优化。

### Per.5 不要不必要地浪费空间

```cpp
// ❌ 不好
struct Flags {
    bool flag_a;
    bool flag_b;
    bool flag_c;
    bool flag_d;
};  // 4 字节，每个 bool 1 字节

// ✅ 好
enum class Flags : uint8_t {
    None = 0,
    A = 1 << 0,
    B = 1 << 1,
    C = 1 << 2,
    D = 1 << 3,
};
// 1 字节
```

### Per.6 不要进行不必要的分配

```cpp
// ❌ 不好：循环内分配
for (const auto& item : items) {
    std::string temp = format(item);  // 每次分配
    process(temp);
}

// ✅ 好：预分配或复用
std::string temp;
temp.reserve(256);
for (const auto& item : items) {
    temp.clear();
    format_to(temp, item);
    process(temp);
}
```

### Per.7 使用紧凑的数据表示

### Per.10 使用移动语义避免不必要的拷贝

```cpp
// ❌ 不好
std::vector<int> result;
for (auto v : source) {  // 拷贝
    result.push_back(v);
}

// ✅ 好
std::vector<int> result;
for (auto&& v : source) {  // 可能移动
    result.push_back(std::move(v));
}
// 更好
std::vector<int> result = std::move(source);
```

### Per.11 使用引用传递避免不必要的拷贝

同 F.16。

### Per.12 当优化需要时，使用 constexpr

```cpp
// ✅ 好
constexpr double PI = 3.14159265358979;
constexpr int factorial(int n) {
    return n <= 1 ? 1 : n * factorial(n - 1);
}
constexpr int f5 = factorial(5);  // 编译期计算
```

### Per.13 使用 constexpr 而不是宏定义常量

同 ES.31。

### Per.14-Per.15 用内存换取速度

```cpp
// ✅ 好：查找表替代计算
constexpr double SIN_TABLE[360] = { /* 预计算的 sin 值 */ };
double fast_sin(int degrees) {
    return SIN_TABLE[degrees % 360];
}
```

### Per.17 不要为速度牺牲可读性（可复用代码中）

### Per.18-Per.19 了解编译器能力和优化选项

```bash
# 常用优化选项
g++ -O2 -Wall -Wextra -std=c++20 program.cpp
```

### Per.30-Per.33 避免上下文切换和锁

见 Concurrency 章节。

---

## CP: Concurrency (并发)

### CP.1 假设你的代码会作为多线程运行

```cpp
// ❌ 不好：静态变量没有同步
int counter = 0;
void increment() { ++counter; }  // 数据竞争！

// ✅ 好
std::atomic<int> counter{0};
void increment() { counter.fetch_add(1, std::memory_order_relaxed); }
```

### CP.2 避免数据竞争

```cpp
// ❌ 不好
class Logger {
    std::string buffer_;
public:
    void log(const std::string& msg) {
        buffer_ += msg + "\n";  // 数据竞争
    }
};

// ✅ 好
class Logger {
    std::string buffer_;
    std::mutex mtx_;
public:
    void log(const std::string& msg) {
        std::lock_guard lock(mtx_);
        buffer_ += msg + "\n";
    }
};
```

### CP.3 最小化显式共享可写数据

### CP.4 倾向于使用任务而不是线程

```cpp
// ❌ 不好：手动管理线程
std::thread t([&]{ do_work(); });
t.join();

// ✅ 好：使用任务
auto future = std::async(std::launch::async, [&]{ return do_work(); });
auto result = future.get();
```

### CP.5 使用 futures 和 promises 传递结果

```cpp
// ✅ 好
std::future<int> compute_async() {
    std::promise<int> promise;
    auto future = promise.get_future();
    std::thread([p = std::move(promise)]() mutable {
        p.set_value(compute());
    }).detach();
    return future;
}
```

### CP.8 不要使用 volatile 来进行同步

```cpp
// ❌ 不好
volatile bool ready = false;
// 线程1
ready = true;
// 线程2
while (!ready) {}  // volatile 不保证原子性或内存序！

// ✅ 好
std::atomic<bool> ready{false};
// 线程1
ready.store(true, std::memory_order_release);
// 线程2
while (!ready.load(std::memory_order_acquire)) {}
```

### CP.9 使用 gsl::joining_thread

确保线程总是被 join。

### CP.20 使用 RAII 不要泄漏锁

```cpp
// ✅ 好
std::mutex mtx;
void safe_operation() {
    std::lock_guard<std::mutex> lock(mtx);
    // 临界区
}  // 自动释放
```

### CP.21 使用 lock_guard/unique_lock 管理锁

```cpp
// ✅ 好：条件变量需要 unique_lock
std::unique_lock<std::mutex> lock(mtx);
cond_var.wait(lock, []{ return ready; });
```

### CP.22 不要在持有锁时调用未知代码

```cpp
// ❌ 不好
std::lock_guard lock(mtx);
callback();  // 回调可能死锁！

// ✅ 好
{
    auto data = get_data_protected();
}
callback();  // 不持锁调用
```

### CP.23 保持锁粒度尽可能小

```cpp
// ❌ 不好
std::lock_guard lock(mtx);
auto data1 = get1();
process(data1);
auto data2 = get2();
process(data2);

// ✅ 好
int data1, data2;
{
    std::lock_guard lock(mtx);
    data1 = get1();
    data2 = get2();
}
process(data1);
process(data2);
```

### CP.24 不要在持有锁时阻塞

### CP.25 倾向于使用 lock_guard 和 unique_lock

同 CP.20-CP.21。

### CP.26 不要分离线程

```cpp
// ❌ 不好
std::thread t([&]{ do_work(); });
t.detach();  // 线程可能比 main 先结束

// ✅ 好
std::thread t([&]{ do_work(); });
t.join();
// 或使用 gsl::joining_thread
```

### CP.27 不要用 volatile 进行通信

同 CP.8。

### CP.31 使用原子变量传递数据

```cpp
// ✅ 好
std::atomic<int> shared_data{0};
shared_data.store(42, std::memory_order_release);
int val = shared_data.load(std::memory_order_acquire);
```

### CP.32 使用 shared_ptr 进行共享所有权访问

```cpp
// ✅ 好
std::shared_ptr<const Config> config;
void update_config() {
    config = std::make_shared<Config>(load_from_file());
}
void use_config() {
    auto cfg = std::atomic_load(&config);  // 线程安全读取
}
```

### CP.40-CP.44 最小化同步、避免死锁

```cpp
// ✅ 好：以固定顺序获取锁
void transfer(Account& from, Account& to, int amount) {
    // 总是按账户 ID 顺序加锁
    auto lock = std::scoped_lock(from.id < to.id ? from.mtx : to.mtx,
                                  from.id < to.id ? to.mtx : from.mtx);
    from.balance -= amount;
    to.balance += amount;
}
```

### CP.50 定义 mutex 与被保护的数据一起

```cpp
// ✅ 好
class ThreadSafeQueue {
    std::queue<int> queue_;
    mutable std::mutex mtx_;
public:
    void push(int val) {
        std::lock_guard lock(mtx_);
        queue_.push(val);
    }
};
```

### CP.51 倾向于 lock() 避免死锁

```cpp
// ✅ 好：一次获取多个锁，避免死锁
std::lock(mtx_a, mtx_b);
std::lock_guard lock_a(mtx_a, std::adopt_lock);
std::lock_guard lock_b(mtx_b, std::adopt_lock);
// 或更简洁
std::scoped_lock lock(mtx_a, mtx_b);
```

### CP.60 不要使用对参数类型产生数据竞争的 call()

### CP.61 使用 async 和 future 执行任务

```cpp
// ✅ 好
auto result = std::async(std::launch::async, [](int n) {
    return expensive_computation(n);
}, 42);
int value = result.get();
```

### CP.100 不要使用 lock-free 编程除非绝对必要

lock-free 编程极其复杂，容易出错。

### CP.101 对昂贵的初始化使用惰性求值

```cpp
// ✅ 好
class ExpensiveResource {
    static std::shared_ptr<ExpensiveResource> instance_;
    static std::once_flag flag_;
public:
    static std::shared_ptr<ExpensiveResource> get() {
        std::call_once(flag_, []{
            instance_ = std::make_shared<ExpensiveResource>();
        });
        return instance_;
    }
};
```

### CP.110 不要自己写 double-checked locking，使用 call_once

```cpp
// ❌ 不好：自写 DCLP（几乎不可能正确）
if (!instance) {
    std::lock_guard lock(mtx);
    if (!instance) {
        instance = new Singleton();
    }
}

// ✅ 好
std::call_once(flag, []{ instance = create(); });
```

---

## E: Error Handling (错误处理)

### E.1 开发一个策略用于错误处理

团队应统一错误处理策略：异常 vs 错误码 vs 断言。

### E.2- E.3 使用异常进行错误处理

```cpp
// ❌ 不好：错误码容易忽略
int result = do_work();
if (result != 0) {
    // 调用者可能忘记检查
}

// ✅ 好：异常强制处理
do_work();  // 失败时自动传播
```

### E.5 不要使用异常来绕过控制流

```cpp
// ❌ 不好：用异常做流程控制
try {
    for (auto& item : items) {
        if (item == target) throw item;
    }
} catch (const Item& found) {
    process(found);
}

// ✅ 好
for (auto& item : items) {
    if (item == target) {
        process(item);
        break;
    }
}
```

### E.6 使用 RAII 进行资源管理

同 R.1。

### E.7 在构造函数中抛出异常表示无效参数

```cpp
// ✅ 好
class Socket {
public:
    Socket(const std::string& address, int port) {
        if (port < 0 || port > 65535)
            throw std::invalid_argument{"invalid port: " + std::to_string(port)};
        // ...
    }
};
```

### E.8 使用自定义异常类型区分错误

```cpp
// ✅ 好
class NetworkError : public std::runtime_error {
public:
    using std::runtime_error::runtime_error;
};
class FileNotFoundError : public std::runtime_error {
public:
    using std::runtime_error::runtime_error;
};
```

### E.10 抛出异常而不是调用 abort

```cpp
// ❌ 不好
if (!ptr) std::abort();

// ✅ 好
if (!ptr) throw std::runtime_error{"null pointer"};
```

### E.11 使用 noexcept 函数

```cpp
// ✅ 好
void cleanup() noexcept {
    // 确保不会抛出异常
    resource_.reset();
}
```

### E.12 移动操作应 noexcept

```cpp
// ✅ 好
class Buffer {
    int* data_;
    size_t size_;
public:
    Buffer(Buffer&& other) noexcept
        : data_(other.data_), size_(other.size_) {
        other.data_ = nullptr;
        other.size_ = 0;
    }
    Buffer& operator=(Buffer&& other) noexcept { /* ... */ }
};
```

### E.13 不要用异常替代资源管理

### E.14 使用 purpose-built 异常类型

```cpp
// ✅ 好
class ValidationError : public std::runtime_error {
    std::string field_;
public:
    ValidationError(std::string field, std::string msg)
        : std::runtime_error(std::move(msg)), field_(std::move(field)) {}
    const std::string& field() const { return field_; }
};
```

### E.15 从 noexcept 函数中捕获异常

```cpp
// ✅ 好
void safe_func() noexcept {
    try {
        risky_operation();
    } catch (...) {
        // 处理所有异常，确保 noexcept 保证
    }
}
```

### E.16 析构函数、释放和 swap 绝对不能失败

```cpp
// ✅ 好
~MyClass() noexcept = default;
void swap(MyClass& other) noexcept {
    using std::swap;
    swap(data_, other.data_);
}
```

### E.17 不要编写没有 try 块的 throw 表达式

### E.18 最小化显式 try/catch

```cpp
// ❌ 不好：到处 try/catch
void a() {
    try { b(); }
    catch (...) { log("a failed"); }
}
void b() {
    try { c(); }
    catch (...) { log("b failed"); }
}

// ✅ 好：在最外层统一处理
void process() {
    try {
        a();
    } catch (const NetworkError& e) {
        handle_network_error(e);
    } catch (const std::exception& e) {
        log_error(e.what());
    }
}
```

### E.19 使用 RAII 替代范围 try 块

### E.20-E.24 避免直接操作原始内存

```cpp
// ❌ 不好
void* raw = operator new(100);
Widget* w = new(raw) Widget{};

// ✅ 好
auto w = std::make_unique<Widget>();
```

### E.25 如果函数不能抛出异常，使用 noexcept

同 E.11。

### E.26 如果无法恢复，使用 noexcept failure

### E.27 如果不能抛异常，模拟 RAII

```cpp
// ✅ 好：错误码 + RAII
struct Error {
    int code;
    std::string message;
};
class Result {
    std::variant<Data, Error> value_;
public:
    explicit Result(Error e) : value_(std::move(e)) {}
    explicit Result(Data d) : value_(std::move(d)) {}
    bool has_error() const { return std::holds_alternative<Error>(value_); }
    const Data& get() const { return std::get<Data>(value_); }
};
```

### E.28 不要返回错误码

同 E.2-E.3。

### E.29 使用 error_code 或 error_condition 返回可恢复错误

```cpp
// ✅ 好：当异常不可用时
std::error_code try_read(File& f, Buffer& buf) {
    if (auto ec = f.read(buf.data(), buf.size())) return ec;
    return {};
}
```

### E.30 不要使用异常规范

```cpp
// ❌ 不好
void func() throw(std::runtime_error);

// ✅ 好
void func();  // 不限制
// 或
void func() noexcept;  // 承诺不抛
```

### E.31 使用 gsl::narrow 或 gsl::narrow_cast

```cpp
// ❌ 不好
int x = static_cast<int>(long_value);  // 可能截断但不报错

// ✅ 好
int x = gsl::narrow<int>(long_value);  // 截断时抛异常
// 或
int x = gsl::narrow_cast<int>(long_value);  // 显式标注意图
```

### E.40 逐步完善错误处理

先让代码正确，再逐步添加错误处理。

### E.41 调用者应考虑异常的可能

```cpp
// ✅ 好
void process_all(const std::vector<File>& files) {
    for (const auto& f : files) {
        try {
            process(f);
        } catch (const FileError& e) {
            log_error(e);
            continue;  // 继续处理其他文件
        }
    }
}
```

---

## Co: Contracts (契约) — C++20+

### Co.1 假设契约会被检查

契约（contract）用于定义和检查接口的前置条件、后置条件和不变量。

### Co.2 使用契约防止未定义行为

```cpp
// ✅ 好
int safe_divide(int a, int b) [[pre: b != 0]] {
    return a / b;
}
```

### Co.3 倾向于 static_assert 静态检查不变量

```cpp
// ✅ 好
template<typename T>
class NumericVector {
    static_assert(std::is_arithmetic_v<T>, "T must be arithmetic");
    // ...
};
```

### Co.4 使用契约为运行时检查定义预期

```cpp
// ✅ 好
double sqrt_positive(double x)
    [[pre: x >= 0]]
    [[post r: r >= 0]]
{
    return std::sqrt(x);
}
```

### Co.5 倾向于 expects() 验证前置条件

```cpp
// ✅ 好
void set_age(int age)
    [[expects: age >= 0 && age <= 150]]
{
    age_ = age;
}
```

### Co.6 倾向于 ensures() 验证后置条件

```cpp
// ✅ 好
class Vector {
    int* data_;
    size_t size_;
public:
    int& at(size_t i)
        [[expects: i < size_]]
        [[ensures ret: ret == data_[i]]]
    {
        return data_[i];
    }
};
```

### Co.7 使用 assert() 检查不变量

```cpp
// ✅ 好
class BoundedBuffer {
    std::vector<int> buffer_;
    size_t head_ = 0, tail_ = 0, count_ = 0;
public:
    void push(int val) {
        assert(count_ < buffer_.size());
        buffer_[tail_] = val;
        tail_ = (tail_ + 1) % buffer_.size();
        ++count_;
        assert(count_ <= buffer_.size());
    }
};
```

---

## GSL: Guidelines Support Library

GSL 是 C++ Core Guidelines 配套的支持库，提供类型安全的替代。

### gsl::span\<T\> — 替代指针+大小

```cpp
// ❌ 不好
void process(int* data, size_t count);

// ✅ 好
#include <gsl/span>
void process(gsl::span<int> data);
void process(gsl::span<const int> data);  // 只读
```

### gsl::not_null\<T\> — 替代非空裸指针

```cpp
// ❌ 不好
void use(int* p);  // 可能传 null

// ✅ 好
void use(gsl::not_null<int*> p);  // 编译/运行时保证非空
```

### gsl::owner\<T*\> — 标记拥有所有权的指针

```cpp
// ✅ 好
gsl::owner<char*> buffer = new char[1024];
// ... 使用 ...
delete[] buffer;
```

### gsl::unique_ptr / gsl::shared_ptr — 别名

```cpp
// GSL 提供的别名，语义更明确
gsl::unique_ptr<Widget> w = gsl::make_unique<Widget>();
```

### gsl::string_span — 字符串视图

```cpp
void process(gsl::czstring<> str);   // C 风格字符串
void process(gsl::zstring<> str);    // 可修改的 C 字符串
```

### gsl::index / gsl::dim — 下标/维度类型

```cpp
// ✅ 好：明确的下标类型
void set(gsl::span<int> data, gsl::index i, int value) {
    data[i] = value;
}
```

### gsl::finally — RAII 清理

```cpp
// ✅ 好
void process_file(const std::string& path) {
    FILE* f = fopen(path.c_str(), "r");
    if (!f) throw std::runtime_error{"cannot open"};
    auto cleanup = gsl::finally([f] { fclose(f); });
    // ... 使用 f ...
}  // 自动关闭
```

### gsl::narrow / gsl::narrow_cast — 安全转换

```cpp
// ❌ 不好
int x = static_cast<int>(large_double);  // 静默截断

// ✅ 好
int x = gsl::narrow<int>(large_double);      // 截断时抛异常
int x = gsl::narrow_cast<int>(large_double); // 显式标注，不检查
```

### gsl::byte — 字节数据

```cpp
// ✅ 好
void write_bytes(gsl::span<gsl::byte> buffer);
// 防止对字节进行算术运算
```

---

## 快速参考

| 场景 | 推荐做法 |
|------|---------|
| 所有权 | `unique_ptr` > `shared_ptr` > 裸指针（仅观察） |
| 非空参数 | `gsl::not_null<T*>` 或引用 |
| 数组/序列 | `gsl::span<T>` > `vector<T>` > 原生数组 |
| 错误处理 | 异常 > `std::expected` > 错误码 |
| 常量 | `constexpr` > `const` > `#define` |
| 转换 | `static_cast` > 命名转换 > C 风格转换 |
| 多态 | 虚函数 + `override` + 虚析构函数 |
| 并步 | `std::scoped_lock` > `std::lock_guard` > 手动锁 |
| 初始化 | `{}` 初始化 > `()` 初始化 |
| 类型推导 | `auto` 减少重复，明确语义处写全类型 |

---

> **参考**: [C++ Core Guidelines](https://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines)  
> **GSL 实现**: [Microsoft GSL](https://github.com/Microsoft/GSL) | [GSL-lite](https://github.com/gsl-lite/gsl-lite)
