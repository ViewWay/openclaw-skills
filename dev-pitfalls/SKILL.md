# 开发经验与常见陷阱

实战经验速查手册。每个陷阱按 **问题描述 → 错误示例 → 正确做法 → 工具检测** 组织。

---

## 一、通用编程陷阱

### 1.1 整数

#### 有符号整数溢出（未定义行为）

**问题：** 有符号整数溢出在 C/C++ 中是 UB，编译器可能激进优化导致逻辑错误。

```c
// ❌ 错误
int a = INT_MAX;
int b = a + 1; // UB，结果不可预测

// ✅ 正确
if (a > INT_MAX - 1) { /* 处理溢出 */ }
// 或用 <stdint.h> 的溢出安全函数
```

**工具：** `-fsanitize=signed-integer-overflow`（Clang/GCC）、Coverity

---

#### 无符号下溢

**问题：** 无符号数减到负数会回绕到极大值。

```c
// ❌ 错误
unsigned int len = 10;
for (unsigned int i = len - 1; i >= 0; i--) { /* 死循环！unsigned 永远 >= 0 */ }

// ✅ 正确
for (int i = (int)len - 1; i >= 0; i--) { ... }
```

**工具：** `-fsanitize=unsigned-integer-overflow`、PC-lint

---

#### 整数提升陷阱

**问题：** `char`/`short` 运算前会被提升为 `int`，可能导致意外结果。

```c
// ❌ 错误
unsigned char a = 200;
unsigned char b = 200;
if (a + b < 500) { /* false！a+b 提升为 int = 400，< 500 为 true，但意图可能不同 */ }

// ✅ 正确
if ((unsigned int)a + b < 500u) { ... }
```

---

#### 有符号/无符号混合比较

**问题：** 有符号数隐式转为无符号，负数变成极大值。

```cpp
// ❌ 错误
int i = -1;
std::vector<int> v;
if (i < v.size()) { /* false！size() 是 size_t，-1 变成极大值 */ }

// ✅ 正确
if ((size_t)i < v.size()) { ... } // 显式转换
```

**工具：** `-Wsign-compare`、clang-tidy

---

#### 大端/小端字节序

**问题：** 跨平台网络通信或文件读写时，字节序不一致导致数据错乱。

```c
// ❌ 错误：直接写 int 到文件/网络
int32_t val = 0x12345678;
write(fd, &val, 4); // 在小端机器上写入 78 56 34 12

// ✅ 正确：用 hton/ntoh 或手动转换
int32_t net_val = htonl(val);
write(fd, &net_val, 4);
```

---

### 1.2 浮点数

#### IEEE 754 精度问题

**问题：** 0.1 和 0.2 无法精确表示，相加不等于 0.3。

```python
# ❌ 错误
print(0.1 + 0.2 == 0.3)  # False

# ✅ 正确
import math
print(math.isclose(0.1 + 0.2, 0.3))  # True

# ✅ 金额计算用整数（分）
price = 1034  # 10.34 元
```

---

#### 直接 == 比较

```java
// ❌ 错误
double a = 0.1 + 0.2;
if (a == 0.3) { ... }

// ✅ 正确
private static final double EPS = 1e-9;
if (Math.abs(a - 0.3) < EPS) { ... }
```

---

#### NaN 传播

**问题：** NaN 与任何值比较（包括自身）都返回 false。

```javascript
// ❌ 错误
let x = NaN;
if (x === x) { /* 永远 false */ }
if (x !== NaN) { /* 永远 true，无法检测 NaN */ }

// ✅ 正确
if (Number.isNaN(x)) { ... }  // JS
if (Double.isNaN(x)) { ... }  // Java
if (std::isnan(x)) { ... }    // C++
```

---

#### 精度损失（大数吃小数）

```python
# ❌ 错误
big = 1e16
small = 1.0
result = big + small - big  # 0.0，small 被丢弃

# ✅ 正确：用 Kahan 求和或 decimal 模块
from decimal import Decimal
result = Decimal('1e16') + Decimal('1.0') - Decimal('1e16')  # 1.0
```

---

### 1.3 字符串

#### 编码问题

```python
# ❌ 错误：默认编码不保证
with open('data.txt', 'r') as f:  # Windows 上可能是 GBK
    text = f.read()

# ✅ 正确：显式指定编码
with open('data.txt', 'r', encoding='utf-8') as f:
    text = f.read()
```

**工具：** `chardet`（检测编码）、`file -I`（Linux）

---

#### SQL 注入

```python
# ❌ 错误：字符串拼接
cursor.execute(f"SELECT * FROM users WHERE name = '{user_input}'")

# ✅ 正确：参数化查询
cursor.execute("SELECT * FROM users WHERE name = %s", (user_input,))
```

**工具：** SQLMap、OWASP ZAP

---

#### XSS 攻击

```html
<!-- ❌ 错误：直接插入用户输入 -->
<div>{{ user_input }}</div>

<!-- ✅ 正确：转义输出 -->
<div>{{ escapeHtml(user_input) }}</div>
```

**工具：** DOMPurify（前端）、CSP 头、`htmlspecialchars()`（PHP）

---

#### 路径遍历

```python
# ❌ 错误：直接拼接路径
with open(f'/uploads/{filename}') as f: ...

# ✅ 正确：验证路径
import os
safe = os.path.join('/uploads', os.path.basename(filename))
if not safe.startswith('/uploads/'): raise ValueError
```

---

### 1.4 内存

#### 缓冲区溢出

```c
// ❌ 错误
char buf[64];
gets(buf); // 无长度检查

// ✅ 正确
fgets(buf, sizeof(buf), stdin);
```

**工具：** AddressSanitizer（`-fsanitize=address`）、Valgrind、Stack Canaries

---

#### Use-after-free

```cpp
// ❌ 错误
delete ptr;
*ptr = 42; // UB：悬垂指针

// ✅ 正确
ptr = nullptr; // 置空后使用前检查
if (ptr) *ptr = 42;
```

**工具：** AddressSanitizer（`-fsanitize=address`）、Use-After-Free 检测模式

---

#### 内存泄漏检测

```cpp
// ❌ 错误：异常路径未释放
void* p = malloc(100);
if (error) return; // 泄漏！
free(p);

// ✅ 正确：用 RAII 或智能指针
std::unique_ptr<char[]> buf(new char[100]);
if (error) return; // 自动释放
```

**工具：** Valgrind `--leak-check=full`、LeakSanitizer（`-fsanitize=leak`）、Instruments（macOS）

---

### 1.5 并发

#### 数据竞争

```go
// ❌ 错误：两个 goroutine 同时写
var counter int
go func() { counter++ }()
go func() { counter++ }()

// ✅ 正确
var counter int64
var wg sync.WaitGroup
wg.Add(2)
go func() { atomic.AddInt64(&counter, 1); wg.Done() }()
go func() { atomic.AddInt64(&counter, 1); wg.Done() }()
wg.Wait()
```

**工具：** ThreadSanitizer（`-fsanitize=thread`）、Go Race Detector（`-race`）

---

#### 死锁

**问题：** 四个必要条件——互斥、持有并等待、不可抢占、循环等待。

```java
// ❌ 错误：嵌套锁且顺序不一致
// 线程1: lock(A) -> lock(B)
// 线程2: lock(B) -> lock(A)  → 死锁！

// ✅ 正确：统一加锁顺序
void transfer(Account from, Account to) {
    Account first = from.id < to.id ? from : to;
    Account second = from.id < to.id ? to : from;
    synchronized (first) {
        synchronized (second) { ... }
    }
}
```

**工具：** deadlock detection（JVM）、`jstack`、`pstack`

---

#### 条件变量虚假唤醒

```cpp
// ❌ 错误
std::unique_lock<std::mutex> lk(m);
cv.wait(lk); // 可能虚假唤醒

// ✅ 正确：带谓词
cv.wait(lk, [] { return ready; });
```

---

#### volatile 误解

```cpp
// ❌ 错误：volatile 不是线程安全的
volatile int counter = 0;
counter++; // 不是原子操作！

// ✅ 正确
std::atomic<int> counter{0};
counter.fetch_add(1, std::memory_order_relaxed);
```

---

### 1.6 时间与日期

#### 时区处理

```python
# ❌ 错误：依赖本地时区
from datetime import datetime
dt = datetime.now()  # 本地时间，部署环境可能不同

# ✅ 正确：始终用 UTC 存储，显示时转换
from datetime import datetime, timezone
dt = datetime.now(timezone.utc)
# 显示时：dt.astimezone(ZoneInfo("Asia/Shanghai"))
```

---

#### 单调时钟 vs 墙钟

```go
// ❌ 错误：用墙钟计算耗时（NTP 校时会跳变）
start := time.Now()
doWork()
elapsed := time.Since(start) // 可能为负！

// ✅ 正确
start := time.Now() // Go 的 Since 基于单调时钟，已自动处理
// C++: std::chrono::steady_clock
```

---

### 1.7 网络

#### TCP 粘包/拆包

```python
# ❌ 错误：假设一次 recv 收到完整消息
data = sock.recv(4096)

# ✅ 正确：定义消息协议（长度前缀 / 分隔符）
def read_msg(sock):
    length_bytes = recv_exact(sock, 4)
    length = struct.unpack('!I', length_bytes)[0]
    return recv_exact(sock, length)
```

---

#### 连接池泄漏

```python
# ❌ 错误：异常时未归还连接
conn = pool.get()
result = conn.query(...)  # 可能抛异常
pool.put(conn)  # 跳过了

# ✅ 正确
with pool.connection() as conn:
    result = conn.query(...)  # 自动归还
```

---

### 1.8 安全

#### 命令注入

```python
# ❌ 错误
os.system(f"ls {user_input}")

# ✅ 正确
subprocess.run(["ls", user_input])  # 参数列表，不经过 shell
```

**工具：** Bandit（Python）、Semgrep

---

#### 不安全的随机数

```python
# ❌ 错误：用 random 生成安全令牌
import random
token = ''.join(random.choices('abc...', k=32))

# ✅ 正确
import secrets
token = secrets.token_urlsafe(32)
```

---

## 二、语言特定陷阱

### 2.1 C

#### sizeof(数组) vs sizeof(指针)

```c
// ❌ 错误
void foo(int arr[]) {
    size_t n = sizeof(arr) / sizeof(arr[0]); // sizeof(int*) / sizeof(int) = 指针大小
}

// ✅ 正确：传入长度
void foo(int arr[], size_t n) { ... }
foo(arr, sizeof(arr) / sizeof(arr[0]));
```

---

#### 宏的副作用

```c
// ❌ 错误
#define SQUARE(x) ((x) * (x))
int a = SQUARE(i++); // i++ 执行两次！

// ✅ 正确
static inline int square(int x) { return x * x; }
// 或用 GCC statement expression: ({ typeof(x) _x = (x); _x * _x; })
```

---

### 2.2 C++

#### 虚析构缺失

```cpp
// ❌ 错误
class Base { public: ~Base() {} };
class Derived : public Base { int* data = new int[10]; ~Derived() { delete[] data; } };
Base* p = new Derived();
delete p; // 只调用 ~Base，Derived 的 data 泄漏！

// ✅ 正确
class Base { public: virtual ~Base() = default; };
```

**工具：** clang-tidy `modernize-use-override`、`-Wnon-virtual-dtor`

---

#### 迭代器失效

```cpp
// ❌ 错误：遍历时删除
std::vector<int> v = {1, 2, 3, 4, 5};
for (auto it = v.begin(); it != v.end(); ++it) {
    if (*it == 3) v.erase(it); // it 已失效！
}

// ✅ 正确
for (auto it = v.begin(); it != v.end(); ) {
    if (*it == 3) it = v.erase(it);
    else ++it;
}
```

---

#### 循环引用 shared_ptr

```cpp
// ❌ 错误：互相引用导致永不释放
struct Node { std::shared_ptr<Node> next; std::shared_ptr<Node> prev; };

// ✅ 正确：用 weak_ptr 打破循环
struct Node { std::shared_ptr<Node> next; std::weak_ptr<Node> prev; };
```

---

### 2.3 Rust

#### RefCell 运行时 panic

```rust
// ❌ 错误：运行时借用冲突
use std::cell::RefCell;
let data = RefCell::new(vec![1, 2, 3]);
let r1 = data.borrow();
let r2 = data.borrow_mut(); // panic! at runtime

// ✅ 正确：确保借用范围不重叠，或重构数据结构
```

---

#### async 死锁

```rust
// ❌ 错误：.await 持有 Mutex 跨越另一个 .await
let data = mutex.lock().await;
some_async_fn().await; // 持有锁期间让出执行权，可能死锁

// ✅ 正确：缩小锁的范围
let data = {
    let guard = mutex.lock().await;
    guard.clone()
};
some_async_fn().await; // 锁已释放
```

---

### 2.4 Python

#### 可变默认参数

```python
# ❌ 错误：默认列表在函数定义时创建，所有调用共享
def append_to(item, lst=[]):
    lst.append(item)
    return lst

# ✅ 正确
def append_to(item, lst=None):
    if lst is None:
        lst = []
    lst.append(item)
    return lst
```

---

#### 闭包变量绑定（late binding）

```python
# ❌ 错误：所有 lambda 引用同一个 i（最终值 3）
funcs = [lambda: i for i in range(4)]
print([f() for f in funcs])  # [3, 3, 3, 3]

# ✅ 正确：用默认参数捕获值
funcs = [lambda i=i: i for i in range(4)]
print([f() for f in funcs])  # [0, 1, 2, 3]
```

---

#### is vs ==

```python
# ❌ 错误：is 比较身份，不是值
a = [1, 2, 3]
b = [1, 2, 3]
print(a is b)   # False
print(a == b)   # True

# 整数缓存陷阱
a = 256; b = 256
print(a is b)   # True（-5~256 缓存）
a = 257; b = 257
print(a is b)   # False
```

---

### 2.5 JavaScript/TypeScript

#### this 绑定

```javascript
// ❌ 错误：回调中 this 丢失
class Timer {
    constructor() { this.seconds = 0; }
    start() {
        setInterval(function() { this.seconds++; }, 1000); // this 是 undefined
    }
}

// ✅ 正确
start() {
    setInterval(() => { this.seconds++; }, 1000); // 箭头函数继承 this
}
```

---

#### == vs ===

```javascript
// ❌ == 的隐式转换
"0" == false   // true
"" == false    // true
null == undefined // true
[] == false    // true

// ✅ 始终用 ===
"0" === false  // false
```

---

#### 事件监听器泄漏

```javascript
// ❌ 错误：每次渲染都添加新监听器
function render() {
    button.addEventListener('click', handler);
}

// ✅ 正确：先移除或用 AbortController
const controller = new AbortController();
button.addEventListener('click', handler, { signal: controller.signal });
// 清理时
controller.abort();
```

---

### 2.6 Java

#### Integer 缓存

```java
// ❌ 陷阱
Integer a = 127, b = 127;
System.out.println(a == b);  // true（缓存）

Integer c = 128, d = 128;
System.out.println(c == d);  // false！超出缓存范围

// ✅ 始终用 .equals()
System.out.println(c.equals(d));  // true
```

---

#### 泛型擦除

```java
// ❌ 错误：运行时无法区分 List<String> 和 List<Integer>
List<String> strings = new ArrayList<>();
List<Integer> ints = new ArrayList<>();
System.out.println(strings.getClass() == ints.getClass()); // true！

// ✅ 需要类型信息时传 Class<T>
public <T> T deserialize(String json, Class<T> clazz) {
    return gson.fromJson(json, clazz);
}
```

---

#### 资源泄漏

```java
// ❌ 错误：异常时流未关闭
InputStream is = new FileInputStream("file");
process(is);  // 异常则泄漏

// ✅ 正确
try (InputStream is = new FileInputStream("file")) {
    process(is);
} // 自动关闭
```

**工具：** SpotBugs、ErrorProne、 `-Xlint:resource`

---

## 三、系统设计陷阱

### N+1 查询问题

```sql
-- ❌ 错误：循环中查询
SELECT * FROM users;
-- 对每个 user: SELECT * FROM orders WHERE user_id = ?

-- ✅ 正确：JOIN 或 IN
SELECT u.*, o.* FROM users u LEFT JOIN orders o ON u.id = o.user_id;
-- 或预加载
SELECT * FROM orders WHERE user_id IN (SELECT id FROM users WHERE ...);
```

**工具：** Django Debug Toolbar、SQL EXPLAIN、慢查询日志

---

### 缓存穿透/击穿/雪崩

| 问题 | 场景 | 解决方案 |
|------|------|----------|
| **穿透** | 查不存在的 key，缓存不生效 | 布隆过滤器 / 缓存空值 |
| **击穿** | 热点 key 过期，大量请求打到 DB | 互斥锁 / 永不过期+异步刷新 |
| **雪崩** | 大量 key 同时过期 | 过期时间加随机偏移 |

---

### 消息重复消费

```
❌ 只处理不幂等 → 消费者收到重复消息导致数据重复
✅ 设计幂等消费：唯一消息ID + 去重表 / 数据库唯一约束 / Redis SETNX
```

---

### 分布式事务一致性

```
❌ 跨服务直接调用，部分失败导致不一致
✅ 方案选择：
   - 强一致：2PC / TCC（Seata）
   - 最终一致：Saga 模式 / 本地消息表 + MQ
   - 根据业务容忍度选择
```

---

### CAP 权衡

- **CP**（ZooKeeper）：牺牲可用性，保证一致性
- **AP**（Eureka/Cassandra）：牺牲一致性，保证可用性
- **BASE 理论**：Basically Available + Soft State + Eventually Consistent

---

## 四、代码审查清单

- [ ] **边界条件** — 空、零、最大值、最小值、负数、空集合
- [ ] **错误处理** — 所有路径是否处理异常/错误码
- [ ] **资源释放** — 文件、连接、内存、锁、fd 是否在所有路径释放
- [ ] **并发安全** — 共享数据是否有保护，锁顺序是否一致
- [ ] **输入验证** — 所有外部输入是否验证（长度、范围、格式、白名单）
- [ ] **安全** — 注入、XSS、CSRF、SSRF、权限检查、敏感数据不硬编码
- [ ] **性能** — N+1 查询、大循环内分配、不必要的拷贝、算法复杂度
- [ ] **可维护性** — 命名清晰、注释必要处、圈复杂度可控、无魔法数字
- [ ] **整数安全** — 溢出检查、符号转换、除零
- [ ] **浮点安全** — 不用 == 比较、金额用整数
- [ ] **时区** — 存储 UTC、显示转本地
- [ ] **日志** — 不打印敏感信息、关键操作有日志
