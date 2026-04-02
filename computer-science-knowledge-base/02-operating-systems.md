# Operating Systems — Graduate-Level Reference

> 结构：概念 → 原理 → 代码/伪代码 → 复杂度 → 应用

---

## 1. 进程与线程

### 1.1 进程状态机

```
         ┌─── fork() ───┐
         │              ▼
       [新建] ──→ [就绪] ←── 时间片用完 / 抢占
                    │              ↑
              调度器选中           │
                    ▼              │
                  [运行] ──→ [阻塞]（等待 I/O / 资源）
                    │              │
               exit()       I/O 完成 / 资源可用
                    ▼              │
                 [终止] ───────────┘
```

### 1.2 进程控制

```c
#include <stdio.h>
#include <unistd.h>
#include <sys/wait.h>

int main() {
    pid_t pid = fork();  // 创建子进程，返回两次
    if (pid == 0) {
        // 子进程
        execlp("/bin/ls", "ls", "-l", NULL);  // 替换进程映像
        perror("execlp");
    } else if (pid > 0) {
        // 父进程
        int status;
        waitpid(pid, &status, 0);  // 等待子进程终止
        printf("Child exited: %d\n", WEXITSTATUS(status));
    }
    return 0;
}
```

**fork() 原理**：写时复制 (COW) — 子进程共享父进程页表，标记为只读；任一方写入时触发页错误，内核复制该页。

### 1.3 线程模型

| 模型 | 映射 | 优点 | 缺点 |
|------|------|------|------|
| 1:1 | 一个用户线程对应一个内核线程 | 真正并行 | 线程创建开销大 |
| N:1 | 多用户线程映射到一个内核线程 | 轻量 | 无法利用多核，阻塞全阻塞 |
| M:N | 多用户线程映射到多内核线程 | 兼顾两者 | 实现复杂（Go goroutine） |

### 1.4 进程间通信 (IPC)

```python
# 共享内存（Python multiprocessing）
from multiprocessing import Process, Value, Array, Queue

def worker(val, arr, q):
    val.value += 1
    arr[0] = -arr[0]
    q.put('done')

if __name__ == '__main__':
    v = Value('i', 0)           # 共享整数
    a = Array('d', [1.0, 2.0])  # 共享数组
    q = Queue()                  # 消息队列
    p = Process(target=worker, args=(v, a, q))
    p.start()
    p.join()
```

| IPC 方式 | 特点 |
|---------|------|
| 管道 (Pipe) | 半双工，父子进程间 |
| 命名管道 (FIFO) | 文件系统可见，无亲缘关系可用 |
| 消息队列 | 内核维护消息链表 |
| 共享内存 | 最快 IPC，需同步 |
| 信号量 | 计数器，用于同步 |
| Socket | 跨机器通信 |

### 1.5 调度算法

| 算法 | 特点 | 适用场景 |
|------|------|---------|
| FCFS | 简单，非抢占 | 批处理 |
| SJF | 最短优先，平均等待最小 | 批处理（需预知执行时间） |
| RR (时间片轮转) | 公平，响应好 | 交互式系统 |
| 优先级调度 | 可抢占/非抢占 | 实时系统 |
| CFS (完全公平调度) | 虚拟运行时间红黑树 | Linux 默认 |
| MLFQ (多级反馈队列) | 动态优先级调整 | 通用系统 |

**CFS 原理**：
- 每个任务维护 `vruntime`（虚拟运行时间 = 实际时间 × 优先级权重比）
- 选择 vruntime 最小的任务运行（红黑树 O(log n) 调度）
- 权重由 nice 值决定：nice 0 权重 1024，nice -20 权重 8861

**MLFQ 规则**：
1. 优先级高的先运行
2. 同优先级 RR
3. 新进程进最高优先级
4. 用完时间片降一级
5. 定期提升所有进程到最高优先级（防止饥饿）

### 1.6 死锁

**四个必要条件**：互斥、占有并等待、非抢占、循环等待。

**银行家算法**：
```
Available[j] = Available[j] - Request[i][j]
Allocation[i][j] = Allocation[i][j] + Request[i][j]
Need[i][j] = Need[i][j] - Request[i][j]
// 然后执行安全性算法：寻找 Finish[i]=false 且 Need[i] ≤ Work 的进程
```

```python
def bankers_check(available, max_need, allocation):
    n = len(max_need)
    need = [[max_need[i][j] - allocation[i][j] for j in range(len(available))]
            for i in range(n)]
    work = available[:]
    finish = [False] * n
    safe_seq = []
    for _ in range(n):
        found = False
        for i in range(n):
            if not finish[i] and all(need[i][j] <= work[j] for j in range(len(work))):
                work = [work[j] + allocation[i][j] for j in range(len(work))]
                finish[i] = True
                safe_seq.append(i)
                found = True
                break
        if not found:
            return None  # 不安全
    return safe_seq
```

---

## 2. 内存管理

### 2.1 虚拟地址空间

```
高地址 ┌─────────────┐
       │   内核空间    │ (用户不可直接访问)
       ├─────────────┤
       │   栈 ↓       │ (向下增长)
       │   ...        │
       ├─────────────┤
       │   共享库      │ (mmap)
       ├─────────────┤
       │   ...        │
       │   堆 ↑       │ (向上增长，brk/sbrk)
       ├─────────────┤
       │   BSS 段     │ (未初始化全局变量)
       │   Data 段    │ (已初始化全局变量)
       │   Text 段    │ (代码，只读)
低地址 └─────────────┘
```

### 2.2 分页机制

**虚拟地址 → 物理地址**：
```
虚拟地址 = [页号 (VPN)] [页内偏移 (offset)]
         ↓
页表查找 → 物理页框号 (PFN)
         ↓
物理地址 = PFN × 页大小 + offset
```

**多级页表**：减少页表占用。x86-64 使用 4 级页表（PML4 → PDPT → PD → PT）。

**TLB (Translation Lookaside Buffer)**：页表缓存，命中率通常 > 99%。

```
有效访存时间 = TLB 命中率 × (TLB 访问时间 + 内存访问时间)
            + (1 - TLB 命中率) × (TLB 访问时间 + 页表查找 + 内存访问时间)
```

**倒置页表**：按物理页框索引而非虚拟页号，节省空间但查找慢（需哈希）。

### 2.3 页面置换算法

| 算法 | 原理 | 性能 |
|------|------|------|
| OPT | 替换未来最久不使用的 | 理论最优（需预知） |
| FIFO | 替换最早进入的 | 简单，可能 Belady 异常 |
| LRU | 替换最近最少使用的 | 近似最优，开销大 |
| Clock | 近似 LRU，循环引用位 | 实用 |
| LFU | 替换访问频率最低的 | 适合局部性弱的场景 |

**Belady 异常**：FIFO 中增加帧数反而增加缺页率。

**LRU 实现**：栈方法 O(n)，时钟方法 O(1) 均摊。

```c
// Clock 算法伪代码
while (true) {
    if (reference_bit[clock_hand] == 0) {
        replace(clock_hand);
        advance_clock_hand();
        break;
    }
    reference_bit[clock_hand] = 0;
    advance_clock_hand();
}
```

### 2.4 内存分配算法

| 算法 | 原理 | 碎片 |
|------|------|------|
| 首次适配 | 第一个够大的空闲块 | 外部碎片多 |
| 最佳适配 | 最小够大的空闲块 | 小碎片多 |
| 伙伴系统 | 2 的幂次分割/合并 | 内部碎片 25% |
| Slab | 预分配固定大小对象缓存 | 极少碎片 |

**伙伴系统**：

```
分配 size=65:
  顺序查找：128 → 64 不够 → 128
  分割：128 → 64(A) + 64(B)，64(A) → 32 + 32，32 → 16 + 16
  分配 16 给请求
合并：释放时检查伙伴是否空闲，是则合并为上一级
```

**Slab 分配器**（Linux）：
- 为常用对象类型（如 `task_struct`、`inode`）预分配内存池
- 三级结构：cache → slab → object
- 优势：零碎片、快速分配（无需搜索）、缓存友好

### 2.5 写时复制 (COW)

```
1. fork() 后父子共享相同物理页，页表标记为只读
2. 任一方写入 → 触发页保护异常
3. 内核复制该页，修改页表指向新页，恢复可写
4. 优势：fork() O(1)，只有实际修改的页才复制
```

### 2.6 内存映射 (mmap)

```c
#include <sys/mman.h>

// 文件映射
int fd = open("file.txt", O_RDWR);
char *p = mmap(NULL, length, PROT_READ|PROT_WRITE, MAP_SHARED, fd, 0);
p[0] = 'H';  // 直接修改文件内容
munmap(p, length);

// 匿名映射（共享内存）
char *shm = mmap(NULL, 4096, PROT_READ|PROT_WRITE, MAP_SHARED|MAP_ANONYMOUS, -1, 0);
```

---

## 3. 文件系统

### 3.1 磁盘结构

```
超级块 (Superblock): 文件系统元信息（块大小、总块数、空闲块数、inode 数）
inode: 文件元数据（权限、大小、时间戳、数据块指针）
目录项 (Dentry): 文件名 → inode 号映射
数据块: 实际文件内容
```

**ext4 inode 结构**：
- 12 个直接块指针
- 1 个一级间接块
- 1 个二级间接块
- 1 个三级间接块
- 最大支持 4TB 单文件（4K 块）

### 3.2 日志文件系统

**原理**：写入前先记录日志（write-ahead logging），崩溃后可恢复一致性。

**ext4 模式**：
- `journal`：数据和元数据都记日志（最安全，最慢）
- `ordered`：元数据记日志，数据先写（默认）
- `writeback`：仅元数据记日志（最快）

**ZFS 特性**：COW、RAID-Z、快照、校验和、去重。

**Btrfs 特性**：COW、子卷、快照、透明压缩、RAID。

### 3.3 VFS (Virtual File System)

```
         用户空间
            │
      ┌─────┴─────┐
      │   VFS 层   │  统一接口：open/read/write/close/stat
      └──┬──┬──┬──┘
         │  │  │
      ┌──┘  │  └──┐
      │     │     │
    ext4  tmpfs  NFS    ← 具体文件系统实现
```

### 3.4 RAID

| 级别 | 原理 | 最少盘 | 容错 | 可用容量 |
|------|------|--------|------|---------|
| RAID 0 | 条带 | 2 | 无 | 100% |
| RAID 1 | 镜像 | 2 | 1 盘 | 50% |
| RAID 5 | 条带+分布式校验 | 3 | 1 盘 | (n-1)/n |
| RAID 6 | 双校验 | 4 | 2 盘 | (n-2)/n |
| RAID 10 | 镜像+条带 | 4 | 每组 1 盘 | 50% |

---

## 4. I/O 系统

### 4.1 I/O 模型

| 模型 | 描述 | 阻塞 |
|------|------|------|
| 阻塞 I/O | 等待操作完成 | ✅ |
| 非阻塞 I/O | 立即返回，轮询检查 | ❌ |
| I/O 多路复用 | select/poll/epoll 监听多个 fd | 部分阻塞 |
| 信号驱动 I/O | 内核发 SIGIO 通知 | ❌ |
| 异步 I/O (AIO) | 内核完成后通知 | ❌ |

```
阻塞 I/O:     ┃等待数据┃拷贝数据┃        ┃
非阻塞 I/O:   ┃EAGAIN┃EAGAIN┃┃数据就绪┃拷贝┃
I/O 多路复用: ┃select等待┃     ┃拷贝┃
异步 I/O:     ┃发起┃               ┃ ← 完成
```

### 4.2 I/O 多路复用

**select vs poll vs epoll**：

| 特性 | select | poll | epoll |
|------|--------|------|-------|
| 最大 fd 数 | 1024 (FD_SETSIZE) | 无限制 | 无限制 |
| 内核→用户拷贝 | 每次全量 | 每次全量 | 仅就绪事件 |
| 复杂度 | O(n) | O(n) | O(1)（就绪事件数） |
| 触发模式 | LT | LT | LT + ET |

```c
// epoll 示例
int epfd = epoll_create1(0);
struct epoll_event ev = {.events = EPOLLIN, .data.fd = listen_fd};
epoll_ctl(epfd, EPOLL_CTL_ADD, listen_fd, &ev);

while (1) {
    struct epoll_event events[MAX_EVENTS];
    int n = epoll_wait(epfd, events, MAX_EVENTS, -1);
    for (int i = 0; i < n; i++) {
        if (events[i].data.fd == listen_fd) {
            int client_fd = accept(listen_fd, NULL, NULL);
            ev.events = EPOLLIN | EPOLLET;  // 边缘触发
            ev.data.fd = client_fd;
            epoll_ctl(epfd, EPOLL_CTL_ADD, client_fd, &ev);
        } else {
            handle_client(events[i].data.fd);
        }
    }
}
```

**LT (水平触发)**：fd 就绪时持续通知直到处理完。**ET (边缘触发)**：fd 从未就绪变为就绪时通知一次，需一次性读完所有数据。

**kqueue (BSD/macOS)**：类似 epoll，支持更多事件类型（文件、进程、信号）。

### 4.3 零拷贝

```
传统 read + write:
  磁盘 → 内核缓冲区 → 用户缓冲区 → Socket 缓冲区 → 网卡
  (4 次拷贝, 4 次上下文切换)

sendfile:
  磁盘 → 内核缓冲区 → Socket 缓冲区 → 网卡
  (2 次拷贝, 2 次上下文切换)

mmap + write:
  文件映射到用户空间，直接操作页缓存
  (减少 1 次拷贝)
```

---

## 5. 同步原语

### 5.1 互斥锁与自旋锁

```c
// 互斥锁：争用时线程睡眠
pthread_mutex_t mutex = PTHREAD_MUTEX_INITIALIZER;
pthread_mutex_lock(&mutex);
// critical section
pthread_mutex_unlock(&mutex);

// 自旋锁：争用时忙等待（适合短临界区、多核）
pthread_spinlock_t spin;
pthread_spin_lock(&spin);
// critical section
pthread_spin_unlock(&spin);
```

### 5.2 条件变量

```c
pthread_mutex_t mutex = PTHREAD_MUTEX_INITIALIZER;
pthread_cond_t cond = PTHREAD_COND_INITIALIZER;
int ready = 0;

// 生产者
pthread_mutex_lock(&mutex);
ready = 1;
pthread_cond_signal(&cond);
pthread_mutex_unlock(&mutex);

// 消费者
pthread_mutex_lock(&mutex);
while (!ready)  // 必须用 while，防止虚假唤醒
    pthread_cond_wait(&cond, &mutex);
// 消费
pthread_mutex_unlock(&mutex);
```

### 5.3 读写锁

```c
pthread_rwlock_t rwl = PTHREAD_RWLOCK_INITIALIZER;

// 读操作（多读者可并发）
pthread_rwlock_rdlock(&rwl);
// read
pthread_rwlock_unlock(&rwl);

// 写操作（独占）
pthread_rwlock_wrlock(&rwl);
// write
pthread_rwlock_unlock(&rwl);
```

### 5.4 原子操作

```c
#include <stdatomic.h>

atomic_int counter = ATOMIC_VAR_INIT(0);

// CAS (Compare-And-Swap)
int expected = 0;
atomic_compare_exchange_strong(&counter, &expected, 1);

// FAA (Fetch-And-Add)
atomic_fetch_add(&counter, 1);

// 无锁栈（Treiber Stack）
struct Node { int val; struct Node *next; };
_Atomic(struct Node *) top = ATOMIC_VAR_INIT(NULL);

void push(int val) {
    struct Node *n = malloc(sizeof(struct Node));
    n->val = val;
    do { n->next = atomic_load(&top); }
    while (!atomic_compare_exchange_weak(&top, &n->next, n));
}
```

### 5.5 信号量

```c
#include <semaphore.h>

sem_t sem;
sem_init(&sem, 0, 3);  // 初始值 3（允许 3 个并发）

sem_wait(&sem);   // P 操作：值 -1，若 < 0 则阻塞
// critical section
sem_post(&sem);   // V 操作：值 +1，唤醒等待者
```

**生产者-消费者问题**：
```
empty = Semaphore(N)  // 空缓冲区数量
full = Semaphore(0)   // 满缓冲区数量
mutex = Semaphore(1)  // 互斥

Producer:                Consumer:
  P(empty)                P(full)
  P(mutex)                P(mutex)
  put(item)               get(item)
  V(mutex)                V(mutex)
  V(full)                 V(empty)
```

### 5.6 RCU (Read-Copy-Update)

**原理**：读者无锁访问旧数据；写者创建副本修改，然后原子切换指针；等待所有旧读者退出后释放旧数据。

**应用**：Linux 内核网络路由表、`task_struct` 管理。

**优势**：读者零开销（无原子操作），适合读多写少场景。

### 5.7 经典同步问题

**读者-写者问题**：
```c
// 读者优先
int readers = 0;
Semaphore mutex = 1, wrt = 1;

// Reader
P(mutex); readers++;
if (readers == 1) P(wrt);  // 第一个读者阻止写者
V(mutex);
// reading...
P(mutex); readers--;
if (readers == 0) V(wrt);  // 最后一个读者允许写者
V(mutex);

// Writer
P(wrt);
// writing...
V(wrt);
```

**哲学家就餐问题**：
```python
# 方案：奇数先拿左叉，偶数先拿右叉（破坏循环等待）
forks = [Semaphore(1) for _ in range(5)]

def philosopher(i):
    first, second = (i, (i+1)%5) if i % 2 == 1 else ((i+1)%5, i)
    while True:
        think()
        P(forks[first])
        P(forks[second])
        eat()
        V(forks[second])
        V(forks[first])
```

---

## 6. 虚拟化与容器

### 6.1 Hypervisor

| 类型 | 示例 | 原理 |
|------|------|------|
| Type 1 (裸金属) | VMware ESXi, Xen, Hyper-V | 直接运行在硬件上 |
| Type 2 (托管) | VMware Workstation, VirtualBox, QEMU | 运行在宿主 OS 上 |

**硬件辅助虚拟化**：Intel VT-x / AMD-V 提供特权级隔离（Root / Non-Root 模式）。

### 6.2 容器技术

**Docker 原理**：

```
Docker Container = chroot + namespace + cgroup + union filesystem
```

**Namespace 隔离**：

| Namespace | 隔离内容 |
|-----------|---------|
| PID | 进程 ID |
| NET | 网络栈 |
| MNT | 文件系统挂载点 |
| UTS | 主机名 |
| IPC | System V IPC, POSIX 消息队列 |
| USER | 用户/组 ID |
| CGROUP | Cgroup 根目录视图 |

**cgroup 资源限制**：
```bash
# CPU 限制：最多使用 1.5 个核心
docker run --cpus=1.5 nginx

# 内存限制
docker run -m 512m nginx

# cgroup v2 接口
echo 512M > /sys/fs/cgroup/mycontainer/memory.max
echo "1000000 1000000" > /sys/fs/cgroup/mycontainer/cpu.max  # quota period
```

**UnionFS (OverlayFS)**：
```
容器层（可写）
    ↓
镜像层（只读，叠加）
    ↓
基础层（只读）
```

### 6.3 seccomp 与 capabilities

**seccomp**：限制系统调用。
```c
// 只允许 read, write, exit, sigreturn
struct sock_filter filter[] = {
    BPF_STMT(BPF_LD | BPF_W | BPF_ABS, offsetof(struct seccomp_data, nr)),
    BPF_JUMP(BPF_JMP | BPF_JEQ | BPF_K, __NR_read, 0, 3),
    BPF_JUMP(BPF_JMP | BPF_JEQ | BPF_K, __NR_write, 0, 2),
    BPF_JUMP(BPF_JMP | BPF_JEQ | BPF_K, __NR_exit, 0, 1),
    BPF_STMT(BPF_RET | BPF_K, SECCOMP_RET_ALLOW),
    BPF_STMT(BPF_RET | BPF_K, SECCOMP_RET_KILL),
};
```

**capabilities**：细粒度权限，替代"全有或全无"的 root。

---

## 7. 内核

### 7.1 中断与异常

| 类型 | 来源 | 同步/异步 | 示例 |
|------|------|----------|------|
| 中断 (IRQ) | 外部硬件 | 异步 | 键盘、网卡、定时器 |
| 异常 (Exception) | CPU 执行指令 | 同步 | 缺页、除零、断点 |
| 系统调用 (syscall) | 程序主动 | 同步 | read(), write() |

**中断处理流程**：
```
1. 硬件发送中断信号
2. CPU 保存上下文（寄存器压栈）
3. 查找中断向量表 → 跳转到 ISR (中断服务程序)
4. 顶部半 (top half)：紧急操作，关中断
5. 底部半 (bottom half)：可延迟操作（tasklet, softirq, workqueue），开中断
6. 恢复上下文，返回
```

### 7.2 系统调用

```
用户空间调用 read(fd, buf, count):
  1. 将参数放入寄存器 (rdi, rsi, rdx)
  2. 将系统调用号放入 rax (0 for read)
  3. 执行 syscall 指令
  4. CPU 切换到内核态 (Ring 0)
  5. 保存用户态寄存器
  6. 查找 sys_call_table[rax]
  7. 执行 sys_read()
  8. 返回值放入 rax
  9. sysret 返回用户态
```

### 7.3 eBPF

**原理**：在内核安全地运行用户定义的沙盒程序，无需修改内核源码或加载模块。

```c
// eBPF 程序示例：统计网络包
SEC("xdp")
int count_packets(struct xdp_md *ctx) {
    counter++;
    return XDP_PASS;
}
```

**应用**：
- 网络过滤（Cilium、Katran）
- 性能分析（bcc 工具集）
- 安全监控（Falco、Tetragon）
- 可观测性（Pixie）

**安全保证**：验证器检查程序必定终止（无无限循环）、内存访问安全。

---

## 附录：关键公式速查

**虚拟地址翻译**：
```
物理地址 = 页表[VPN] × PAGESIZE + offset
```

**TLB 有效访存时间**：
```
EAT = h × (t_TLB + t_mem) + (1-h) × (t_TLB + n × t_mem + t_mem)
其中 h = TLB 命中率, n = 页表级数
```

**磁盘访问时间**：
```
T_access = T_seek + T_rotation + T_transfer
T_rotation = 1/(2 × RPM/60)
```

**CFS vruntime 增长**：
```
Δvruntime = Δruntime × (NICE_0_LOAD / weight)
```
