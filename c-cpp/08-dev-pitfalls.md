# C/C++ 陷阱与开发经验 (Development Pitfalls)

> 来源：dev-pitfalls/SKILL.md + c-programming/SKILL.md + cpp-programming/SKILL.md 的陷阱部分
> 覆盖范围：C/C++ 通用陷阱、整数、浮点、字符串、内存、并发、安全相关

---

## 1. 整数陷阱

### 有符号整数溢出（UB）

```c
// ❌ UB
int a = INT_MAX;
int b = a + 1;

// ✅ 检查
if (a > INT_MAX - 1) { /* 处理溢出 */ }
```

### 无符号下溢

```c
// ❌ 死循环！unsigned 永远 >= 0
unsigned int len = 10;
for (unsigned int i = len - 1; i >= 0; i--) { /* ... */ }

// ✅
for (int i = (int)len - 1; i >= 0; i--) { ... }
```

### 整数提升

```c
unsigned char a = 200;
unsigned char b = 200;
if (a + b < 500) { /* a+b 提升为 int = 400 */ }
```

### 有符号/无符号混合比较

```cpp
// ❌ -1 变成极大值
int i = -1;
std::vector<int> v;
if (i < v.size()) { /* false */ }
```

### 大小端

```c
// ❌ 直接写 int 到网络
int32_t val = 0x12345678;
write(fd, &val, 4);

// ✅
int32_t net_val = htonl(val);
write(fd, &net_val, 4);
```

---

## 2. 浮点陷阱

### IEEE 754 精度

```cpp
// ❌ 0.1 + 0.2 != 0.3
if (0.1 + 0.2 == 0.3) { /* false */ }

// ✅
bool approx_equal(double a, double b, double eps = 1e-9) {
    return std::abs(a - b) <= eps * std::max(std::abs(a), std::abs(b));
}
```

### NaN 传播

```cpp
// ❌ NaN 与任何值比较都返回 false
double x = std::nan("");
if (x == x) { /* 永远 false */ }

// ✅
if (std::isnan(x)) { /* ... */ }
```

### 精度损失（大数吃小数）

```c
double big = 1e16;
double small = 1.0;
double result = big + small - big;  // 0.0
```

---

## 3. 字符串陷阱

### 缓冲区溢出

```c
// ❌
char buf[64];
gets(buf);  // 无长度检查

// ✅
fgets(buf, sizeof(buf), stdin);
```

### 编码问题

```c
// ❌ 默认编码不保证
FILE *f = fopen("data.txt", "r");

// ✅ 显式处理编码
// 跨平台时统一用 UTF-8
```

---

## 4. 内存陷阱

### Use-after-free

```cpp
// ❌
delete ptr;
*ptr = 42;

// ✅
ptr = nullptr;
if (ptr) *ptr = 42;
```

### 内存泄漏

```cpp
// ❌ 异常路径未释放
void* p = malloc(100);
if (error) return;

// ✅ RAII
std::unique_ptr<char[]> buf(new char[100]);
if (error) return;  // 自动释放
```

### sizeof(数组) vs sizeof(指针)

```c
// ❌ 函数参数退化为指针
void foo(int arr[]) {
    size_t n = sizeof(arr) / sizeof(arr[0]);  // sizeof(int*)/sizeof(int)
}

// ✅
void foo(int arr[], size_t n) { ... }
```

### 悬垂指针 / 野指针 / 双重释放

```c
int *p = malloc(sizeof(int));
free(p);
*p = 42;        // ❌ use-after-free

int *q;
*q = 42;        // ❌ 野指针

free(r);
free(r);        // ❌ double free

// ✅
free(p); p = NULL;
```

---

## 5. C++ 特有陷阱

### 虚析构缺失

```cpp
// ❌ 只调用 ~Base，Derived 的 data 泄漏
class Base { public: ~Base() {} };

// ✅
class Base { public: virtual ~Base() = default; };
```

### 对象切片

```cpp
// ❌ 值传递 → 切片
void process(Base b) { b.foo(); }

// ✅ 引用/指针传递
void process(Base& b) { b.foo(); }
```

### 迭代器失效

```cpp
// ❌
for (auto& elem : v) {
    if (elem == 3) v.push_back(6);
}

// ✅
for (auto it = v.begin(); it != v.end(); ) {
    if (*it == 3) it = v.erase(it);
    else ++it;
}
```

### 循环引用 shared_ptr

```cpp
// ❌ 内存泄漏
struct Node { std::shared_ptr<Node> next; std::shared_ptr<Node> prev; };

// ✅
struct Node { std::shared_ptr<Node> next; std::weak_ptr<Node> prev; };
```

### 静态初始化顺序

```cpp
// ❌ global_a 可能还没初始化
int global_b = global_a + 1;

// ✅ Meyers' Singleton
int& get_a() {
    static int a = compute();  // C++11 保证线程安全初始化
    return a;
}
```

### auto 陷阱

```cpp
auto x1 = {1};     // C++17 前：initializer_list，C++20：int
auto x2{1};        // C++17+：int
auto x3 = {1, 2};  // 始终是 initializer_list
```

### 隐式窄化转换

```cpp
int x = 3.14;    // x = 3，丢失小数
int y = 1e10;    // UB 或截断
int z{3.14};     // 编译错误！
```

### 宏的副作用

```c
#define SQUARE(x) ((x) * (x))
int a = SQUARE(i++);  // i++ 执行两次！

// ✅
static inline int square(int x) { return x * x; }
```

---

## 6. 并发陷阱

### volatile 不是锁

```cpp
// ❌ counter++ 不是原子操作
volatile int counter = 0;
counter++;

// ✅
std::atomic<int> counter{0};
counter.fetch_add(1, std::memory_order_relaxed);
```

### 数据竞争

```c
// ❌ 没有同步
int shared = 0;
// 线程 A: shared = 1;
// 线程 B: printf("%d", shared);

// ✅ 用 mutex 或 atomic
```

### 死锁

```cpp
// ❌ 不同顺序获取锁
// 线程1: lock(A) -> lock(B)
// 线程2: lock(B) -> lock(A)

// ✅ 统一顺序
void transfer(Account from, Account to) {
    auto first = from.id < to.id ? from : to;
    auto second = from.id < to.id ? to : from;
    lock(first); lock(second);
}
```

### 条件变量虚假唤醒

```cpp
// ❌
cv.wait(lk);

// ✅ 带谓词
cv.wait(lk, [] { return ready; });
```

---

## 7. 安全陷阱

### 命令注入

```c
// ❌
system(user_input);

// ✅
execvp(args[0], args);  // 参数列表，不经过 shell
```

### SQL 注入

```c
// ❌
sprintf(query, "SELECT * FROM users WHERE name = '%s'", user_input);

// ✅ 参数化查询
```

### 不安全的随机数

```c
// ❌ rand() 可预测
// ✅ arc4random() / getrandom() / /dev/urandom
```

---

## 8. 代码审查清单

- [ ] **边界条件** — 空、零、最大值、最小值、负数、空集合
- [ ] **错误处理** — 所有路径是否处理异常/错误码
- [ ] **资源释放** — 文件、连接、内存、锁、fd
- [ ] **并发安全** — 共享数据是否有保护，锁顺序是否一致
- [ ] **输入验证** — 长度、范围、格式、白名单
- [ ] **安全** — 注入、XSS、CSRF、权限检查、敏感数据不硬编码
- [ ] **性能** — N+1 查询、大循环内分配、算法复杂度
- [ ] **整数安全** — 溢出检查、符号转换、除零
- [ ] **浮点安全** — 不用 == 比较、金额用整数
