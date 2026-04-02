# 并发编程 (Concurrency Programming)

> 来源：cpp-programming/SKILL.md 的并发部分 + cppreference/SKILL.md 的 thread/mutex/atomic/future 部分 + c-cpp-tutorial/SKILL.md 的并发部分
> 覆盖范围：C++ 线程、互斥量、条件变量、原子操作、异步、C11 线程、线程池、无锁编程

---

## 1. C++ 线程基础

### std::thread

```cpp
#include <thread>

void worker(int id) {
    for (int i = 0; i < 3; ++i)
        std::cout << "Thread " << id << ": " << i << "\n";
}

int main() {
    std::thread t1(worker, 1);
    std::thread t2(worker, 2);
    t1.join();
    t2.join();
}
```

### std::jthread (C++20)

```cpp
std::jthread t([](std::stop_token st) {
    while (!st.stop_requested()) {
        // 工作...
    }
});  // 析构时自动 join
t.request_stop();  // 协作式取消
```

### this_thread 命名空间

```cpp
this_thread::get_id();                    // 当前线程 ID
this_thread::yield();                     // 让出时间片
this_thread::sleep_for(100ms);           // 休眠一段时间
this_thread::sleep_until(time_point);    // 休眠到时间点
thread::hardware_concurrency();           // 硬件线程数
```

---

## 2. 互斥量

### 互斥量类型

| 类型 | 说明 |
|------|------|
| `std::mutex` | 基本互斥量 |
| `std::recursive_mutex` | 可递归锁定 |
| `std::timed_mutex` | 支持超时锁定 |
| `std::shared_mutex` | 读写锁（C++17） |

### RAII 锁

```cpp
std::mutex mtx;

// lock_guard：简单作用域锁
{
    std::lock_guard<std::mutex> lock(mtx);
    // 临界区
}  // 自动解锁

// unique_lock：灵活锁（支持延迟锁定、手动解锁、移动）
std::unique_lock<std::mutex> lk(mtx);
lk.unlock();
lk.lock();

// scoped_lock：多锁同时锁定，防死锁 (C++17)
std::scoped_lock lk(m1, m2);  // 同时锁定，防死锁

// shared_lock：共享（读）锁 (C++14)
std::shared_lock<std::shared_mutex> lk(rw_mtx);
```

### call_once

```cpp
std::once_flag flag;
std::call_once(flag, []{ /* 只执行一次 */ });
```

---

## 3. 条件变量

```cpp
std::mutex queue_mtx;
std::condition_variable cv;
std::queue<int> q;
bool done = false;

// 生产者
void producer() {
    for (int i = 0; i < 10; ++i) {
        {
            std::lock_guard<std::mutex> lock(queue_mtx);
            q.push(i);
        }
        cv.notify_one();
    }
    done = true;
    cv.notify_all();
}

// 消费者
void consumer() {
    while (true) {
        std::unique_lock<std::mutex> lock(queue_mtx);
        cv.wait(lock, [] { return !q.empty() || done; });  // 防止虚假唤醒
        while (!q.empty()) {
            int val = q.front();
            q.pop();
            lock.unlock();
            // 处理 val...
            lock.lock();
        }
        if (done && q.empty()) break;
    }
}
```

### 条件变量操作

| 操作 | 说明 |
|------|------|
| `cv.wait(lk)` | 释放锁并等待通知 |
| `cv.wait(lk, pred)` | 带谓词等待（**推荐**） |
| `cv.wait_for(lk, dur)` | 超时等待 |
| `cv.notify_one()` | 唤醒一个等待线程 |
| `cv.notify_all()` | 唤醒所有等待线程 |

---

## 4. 异步 (Future/Promise)

```cpp
#include <future>

// async — 异步执行
auto future = std::async(std::launch::async, [](int a, int b) {
    return a + b;
}, 10, 20);
int result = future.get();  // 阻塞等待结果

// promise/future — 手动控制
std::promise<int> prom;
std::future<int> fut = prom.get_future();
std::thread t([&prom]() {
    // 异步计算...
    prom.set_value(42);
});
int val = fut.get();  // 等待结果
t.join();

// packaged_task
std::packaged_task<int(int, int)> task([](int a, int b) { return a + b; });
std::future<int> f = task.get_future();
std::thread t2(std::move(task), 10, 20);
t2.join();
int r = f.get();
```

### future 操作

| 操作 | 说明 |
|------|------|
| `fut.get()` | 获取结果（阻塞） |
| `fut.wait()` | 等待就绪 |
| `fut.wait_for(dur)` | 超时等待，返回 `future_status` |
| `fut.wait_until(tp)` | 定时等待 |
| `prom.set_value(val)` | 设置结果 |
| `prom.set_exception(ex)` | 设置异常 |

---

## 5. 原子操作

### 基本用法

```cpp
#include <atomic>

std::atomic<int> counter{0};
counter++;                                    // 原子自增
counter.fetch_add(1);                         // 原子加
counter.load(std::memory_order_relaxed);      // 原子读取
counter.store(42, std::memory_order_release); // 原子写入
counter.exchange(42);                         // 原子交换
```

### CAS (Compare-And-Swap)

```cpp
std::atomic<int> value{0};
int expected = 0;
bool success = value.compare_exchange_weak(expected, 1);
// 如果 value == expected，设为 1，返回 true
// 否则 expected 被更新为当前值，返回 false
```

### 原子 fetch 操作

```cpp
a.fetch_add(delta, order);   // 原子加
a.fetch_sub(delta, order);   // 原子减
a.fetch_and(val, order);     // 原子与
a.fetch_or(val, order);      // 原子或
a.fetch_xor(val, order);     // 原子异或
```

### 内存序

| 内存序 | 说明 | 使用场景 |
|--------|------|----------|
| `memory_order_relaxed` | 无序，仅保证原子性 | 简单计数器 |
| `memory_order_acquire` | 获取语义（读） | 读取共享数据前 |
| `memory_order_release` | 释放语义（写） | 写入共享数据后 |
| `memory_order_acq_rel` | 获取+释放 | 读-改-写操作 |
| `memory_order_seq_cst` | 顺序一致性（默认） | 需要全局顺序时 |

### 自旋锁

```cpp
std::atomic_flag lock = ATOMIC_FLAG_INIT;
void critical_section() {
    while (lock.test_and_set(memory_order_acquire)) {}
    // 临界区...
    lock.clear(memory_order_release);
}
```

### atomic_flag / atomic<bool> / atomic<T*>

```cpp
std::atomic<bool> ready{false};
std::atomic<int*> ptr{nullptr};
```

---

## 6. false sharing 避免

```cpp
// ❌ 两个原子变量在同一缓存行
struct Bad {
    std::atomic<int> a;
    std::atomic<int> b;
};

// ✅ padding 分离
struct Good {
    alignas(64) std::atomic<int> a;
    alignas(64) std::atomic<int> b;
};
```

---

## 7. 线程池模式

```cpp
class ThreadPool {
    std::vector<std::thread> workers;
    std::queue<std::function<void()>> tasks;
    std::mutex queue_mutex;
    std::condition_variable cv;
    bool stop;
public:
    ThreadPool(size_t threads);
    template<class F> void enqueue(F&& f);
    ~ThreadPool();  // join 所有线程
};

// 使用
ThreadPool pool(4);
pool.enqueue([]{ /* 任务 1 */ });
pool.enqueue([]{ /* 任务 2 */ });
```

---

## 8. C11 线程（可选，实现不全）

```c
#include <threads.h>

thrd_t t;
thrd_create(&t, func, arg);
thrd_join(t, &result);

mtx_t mtx;
mtx_init(&mtx, mtx_plain);
mtx_lock(&mtx);
mtx_unlock(&mtx);
mtx_destroy(&mtx);

cnd_t cv;
cnd_wait(&cv, &mtx);
cnd_signal(&cv);

atomic_int counter;
atomic_fetch_add(&counter, 1);
```

> ⚠️ C11 `threads.h` 在很多编译器上不完整实现，C++ 标准库线程更可靠。

---

## 9. 并发最佳实践

### 死锁避免
- 始终以相同顺序获取多个锁
- 使用 `std::scoped_lock`（C++17）同时锁定
- 使用 `std::lock` + `std::lock_guard` 的 `adopt_lock`

### 临界区优化
- 尽量缩小临界区范围
- 批量处理减少锁竞争
- 考虑用 `std::atomic` 替代 mutex

### 选择指南

| 场景 | 推荐 |
|------|------|
| 简单计数器 | `std::atomic` |
| 复杂临界区 | `std::mutex` + `lock_guard` |
| 多锁 | `std::scoped_lock` |
| 读多写少 | `std::shared_mutex` + `shared_lock` |
| 异步结果 | `std::async` / `std::future` |
| 生产者-消费者 | `std::mutex` + `std::condition_variable` |
| 停止线程 | `std::jthread` + `stop_token`（C++20） |

### 检测工具

```bash
# ThreadSanitizer
gcc -fsanitize=thread -g prog.cpp
clang -fsanitize=thread -g prog.cpp

# Helgrind (Valgrind)
valgrind --tool=helgrind ./prog
```
