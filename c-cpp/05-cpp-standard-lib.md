# C++ 标准库 API 速查 (C++ Standard Library Quick Reference)

> 来源：cppreference/SKILL.md 的 C++ 标准库部分
> 覆盖范围：<string> <vector> <map> <set> <algorithm> <memory> <functional> <optional> <variant> <filesystem> <format> <regex> <iostream> <stdexcept> <type_traits> <utility> <chrono> <thread> <mutex> <atomic> <future>
> 完整版本见：~/.openclaw/workspace/skills/cppreference/SKILL.md

---

## <string> — 字符串

```cpp
std::string s = "hello";
s.size() / s.length()          // 长度
s.empty()                       // 是否为空
s.c_str()                       // C 风格字符串
s.substr(pos, len)             // 子串
s.find(str) / s.rfind(str)     // 查找
s.insert(pos, str)             // 插入
s.erase(pos, len)              // 删除
s.replace(pos, len, str)       // 替换
s.compare(str)                 // 比较
s.starts_with(str) / s.ends_with(str)  // C++20
s + " world"                   // 拼接
stoi(s) / stoll(s) / stod(s)   // 转数字
to_string(42)                  // 数字转字符串
```

### string_view (C++17)

```cpp
std::string_view sv = "hello";  // 零拷贝字符串引用
sv.substr(1, 3);                // 返回 string_view
// 可以接受 string, const char*, string_view
void process(std::string_view sv);
```

---

## <vector> — 动态数组

```cpp
std::vector<int> v = {1, 2, 3};
v.push_back(4);            // 末尾添加
v.emplace_back(5);         // 原地构造
v.pop_back();              // 末尾删除
v.insert(it, val);         // 插入
v.erase(it);               // 删除
v.size() / v.capacity()    // 大小/容量
v.empty()                  // 是否为空
v.reserve(100);            // 预分配
v.resize(100);             // 调整大小
v.shrink_to_fit();         // 释放多余容量
v.clear();                 // 清空
v[n] / v.at(n)             // 访问（at 有越界检查）
v.front() / v.back()       // 首尾元素
v.data();                  // 原始指针
```

---

## <map> / <unordered_map> — 关联容器

```cpp
std::map<std::string, int> m;
m["hello"] = 1;
m.insert({"world", 2});
m.emplace("foo", 3);
m.find("hello")       // 返回迭代器
m.count("hello")      // 0 或 1
m.erase("hello");
m.contains("hello")   // C++20
m.lower_bound(key);   // 第一个 >= key
m.upper_bound(key);   // 第一个 > key

std::unordered_map<std::string, int> um;  // 哈希，平均 O(1)
um.bucket_count()     // 桶数量
um.load_factor()      // 负载因子
um.max_load_factor(0.75);
um.rehash(100);       // 预分配桶
```

### 选择标准
- 需要有序 → `map`
- 只需要查找 → `unordered_map`（通常更快）

---

## <set> / <unordered_set> — 集合

```cpp
std::set<int> s = {3, 1, 4, 1, 5};
s.insert(2);
s.erase(3);
s.find(1);
s.count(1);        // 0 或 1
s.contains(2);     // C++20
s.lower_bound(2);
```

---

## <algorithm> — 算法

```cpp
// 排序
std::sort(begin, end);
std::stable_sort(begin, end);
std::partial_sort(begin, middle, end);

// 查找
std::find(begin, end, val);
std::find_if(begin, end, pred);
std::binary_search(begin, end, val);
std::lower_bound(begin, end, val);
std::upper_bound(begin, end, val);

// 变换
std::transform(in_begin, in_end, out_begin, fn);
std::for_each(begin, end, fn);
std::replace(begin, end, old_val, new_val);
std::remove(begin, end, val);          // 移除（不改变 size）
std::erase(v, val);                     // C++20，真正删除

// 数值
std::accumulate(begin, end, init);
std::count(begin, end, val);
std::count_if(begin, end, pred);
std::min_element(begin, end);
std::max_element(begin, end);
std::minmax_element(begin, end);

// 去重
std::unique(begin, end);  // 前提：已排序

// 堆
std::make_heap(begin, end);
std::push_heap(begin, end);
std::pop_heap(begin, end);
std::sort_heap(begin, end);

// C++20 ranges
#include <ranges>
auto result = v | std::views::filter(pred)
               | std::views::transform(fn)
               | std::views::take(10);
```

---

## <memory> — 内存

```cpp
std::make_unique<T>(args...)      // unique_ptr
std::make_shared<T>(args...)      // shared_ptr
std::weak_ptr<T>                  // 弱引用
std::allocate_shared<T>(alloc)    // 自定义分配器
std::owner_less<T>                // 用于关联容器
std::get_deleter(ptr)             // 获取删除器
```

---

## <functional> — 函数对象

```cpp
std::function<int(int,int)> f = [](int a, int b) { return a + b; };
std::bind(&func, args...)         // 绑定参数
std::placeholders::_1             // 占位符
std::plus<int>() / std::minus<int>()  // 算术
std::greater<int>() / std::less<int>() // 比较
std::logical_and<int>()           // 逻辑
std::hash<int>                    // 哈希函数
std::reference_wrapper<T>         // 引用包装（可用于容器）
```

---

## <optional> / <variant> / <any> — 可选类型

```cpp
// std::optional (C++17)
std::optional<int> maybe_val;
maybe_val = 42;
if (maybe_val) { use(*maybe_val); }
maybe_val.value_or(0);   // 默认值
maybe_val.has_value();    // C++17
maybe_val.reset();

// std::variant (C++17)
std::variant<int, double, std::string> v = "hello";
v = 42;
std::visit([](auto&& arg) { std::cout << arg; }, v);
std::get<int>(v);        // 按类型取（可能 throw）
std::get_if<int>(&v);    // 按类型取（返回指针）
v.index();               // 当前类型索引

// std::any
std::any a = 42;
a = std::string("hello");
std::any_cast<std::string>(a);
```

---

## <tuple> — 元组

```cpp
auto t = std::make_tuple(1, "hello", 3.14);
std::get<0>(t);  // 1
std::get<1>(t);  // "hello"
auto [a, b, c] = t;  // 结构化绑定（C++17）
std::tuple_size_v<decltype(t)>;  // 3
```

---

## <filesystem> — 文件系统 (C++17)

```cpp
namespace fs = std::filesystem;
fs::path p = "/tmp/test.txt";
fs::exists(p);
fs::is_regular_file(p);
fs::is_directory(p);
fs::file_size(p);
fs::create_directory(p);
fs::create_directories("a/b/c");
fs::copy(src, dst);
fs::remove(p);
fs::remove_all(dir);
fs::rename(old, new_name);
fs::current_path();

for (auto& entry : fs::directory_iterator("/tmp")) {
    std::cout << entry.path() << " " << entry.file_size() << "\n";
}

// 递归遍历
for (auto& entry : fs::recursive_directory_iterator("/tmp")) {
    // entry.is_directory(), entry.is_regular_file(), etc.
}
```

---

## <format> — 格式化 (C++20)

```cpp
#include <format>
std::string s = std::format("Hello, {}!", "world");
std::string s2 = std::format("{0} + {0} = {}", 1, 2);       // "1 + 1 = 2"
std::string s3 = std::format("{:>10}", "right");             // 右对齐
std::string s4 = std::format("{:0>5d}", 42);                 // "00042"
std::string s5 = std::format("{:.2f}", 3.14159);             // "3.14"
std::string s6 = std::format("{:#x}", 255);                  // "0xff"

// C++23: std::print
std::print("Hello, {}!\n", "world");
```

---

## <regex> — 正则表达式

```cpp
std::regex re(R"(\d+)");
std::smatch m;
std::string s = "abc123def";
if (std::regex_search(s, m, re)) {
    std::cout << m[0] << "\n";  // "123"
}

// 替换
std::string result = std::regex_replace(s, re, "NUM");
```

---

## <iostream> — IO 流

```cpp
std::cout << "hello" << std::endl;
std::cerr << "error" << std::endl;
std::cin >> x;

// 文件流
std::ifstream ifs("input.txt");
std::ofstream ofs("output.txt");
std::fstream fs("data.txt", std::ios::in | std::ios::out);

// 字符串流
std::stringstream ss;
ss << 42 << " hello";
int n; std::string s;
ss >> n >> s;
```

---

## <stdexcept> — 异常

```cpp
throw std::runtime_error("msg");
throw std::logic_error("msg");
throw std::invalid_argument("msg");
throw std::out_of_range("msg");
throw std::bad_alloc();  // 内存分配失败
// std::exception 基类
try { /* ... */ }
catch (const std::exception& e) { std::cerr << e.what(); }
```

---

## <type_traits> — 类型特性

```cpp
std::is_integral_v<T>
std::is_floating_point_v<T>
std::is_pointer_v<T>
std::is_reference_v<T>
std::is_const_v<T>
std::is_same_v<T, U>
std::is_base_of_v<Base, Derived>
std::enable_if_t<cond, T>
std::remove_reference_t<T>
std::remove_cv_t<T>
std::decay_t<T>
static_assert(std::is_integral_v<T>, "T must be integral");
```

---

## <utility> — 通用工具

```cpp
std::pair<int, std::string> p = {1, "hello"};
p.first / p.second;

std::swap(a, b);
std::move(x);                    // 转为右值引用
std::forward<T>(arg);           // 完美转发
std::exchange(obj, new_val);    // 替换并返回旧值
```

---

## <bitset> — 位集

```cpp
std::bitset<8> b("10101010");
b.test(0);      // 第 0 位
b.set(3);       // 置位
b.reset(3);     // 清位
b.flip(3);      // 取反
b.count();      // 1 的个数
b.any(); b.none(); b.all();
std::cout << b;  // "10101010"
```

---

> 📖 完整 API 文档详见 cppreference/SKILL.md，包含每个函数/类的详细参数、返回值、复杂度和代码示例。
