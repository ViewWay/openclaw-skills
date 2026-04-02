# Linux 内核与系统编程 Skill

> 资深工程师级别的 Linux 系统编程与内核知识库。每个子领域按「概念 → API/命令 → 代码示例 → 调优参数 → 最佳实践」组织。

---

# 第一部分：Linux 系统编程

## 1. 文件 I/O

### 概念

文件 I/O 是 Unix 一切皆文件哲学的核心。Linux 提供 POSIX 文件描述符（fd）机制：每个进程默认打开 fd 0（stdin）、1（stdout）、2（stderr），上限由 `ulimit -n` 控制（默认 1024，可调至百万级）。

**I/O 模式对比：**

| 模式 | 特点 | 适用场景 |
|------|------|----------|
| 缓冲 I/O（glibc） | 用户空间缓冲，减少系统调用 | 通用文件操作 |
| 直接 I/O（O_DIRECT） | 绕过页缓存，对齐约束 | 数据库、自缓存应用 |
| 零拷贝（sendfile） | 内核空间直接传输，无用户态拷贝 | 静态文件服务 |
| 内存映射（mmap） | 文件映射到虚拟地址空间 | 随机访问大文件、共享内存 |

### API

```c
// 基础系统调用
int open(const char *pathname, int flags, mode_t mode);
ssize_t read(int fd, void *buf, size_t count);
ssize_t write(int fd, const void *buf, size_t count);
int close(int fd);
off_t lseek(int fd, off_t offset, int whence);  // SEEK_SET/SEEK_CUR/SEEK_END

// 零拷贝
ssize_t sendfile(int out_fd, int in_fd, off_t *offset, size_t count);
ssize_t splice(int fd_in, loff_t *off_in, int fd_out, loff_t *off_out, size_t len, unsigned int flags);
ssize_t tee(int fd_in, int fd_out, size_t len, unsigned int flags);

// 内存映射
void *mmap(void *addr, size_t length, int prot, int flags, int fd, off_t offset);
int munmap(void *addr, size_t length);
int msync(void *addr, size_t length, int flags);  // MS_SYNC / MS_ASYNC

// 文件锁
int flock(int fd, int operation);          // LOCK_SH / LOCK_EX / LOCK_UN / LOCK_NB
int fcntl(int fd, int cmd, ...);           // F_SETLK / F_SETLKW / F_GETLK

// 文件监控
int inotify_init(void);
int inotify_add_watch(int fd, const char *pathname, uint32_t mask);
int inotify_rm_watch(int fd, int wd);
```

### 代码示例

#### epoll 事件循环（ET 模式 + 非阻塞）

```c
#define _GNU_SOURCE
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <fcntl.h>
#include <errno.h>
#include <sys/epoll.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <string.h>

#define MAX_EVENTS 1024
#define PORT 8080

static int set_nonblocking(int fd) {
    int flags = fcntl(fd, F_GETFL, 0);
    return fcntl(fd, F_SETFL, flags | O_NONBLOCK);
}

static void handle_read(int fd) {
    char buf[4096];
    for (;;) {
        ssize_t n = read(fd, buf, sizeof(buf));
        if (n > 0) {
            write(fd, buf, n);  // echo back
        } else if (n == 0) {
            close(fd);
            break;
        } else {
            if (errno == EAGAIN || errno == EWOULDBLOCK) break;
            perror("read");
            close(fd);
            break;
        }
    }
}

int main(void) {
    int listen_fd = socket(AF_INET, SOCK_STREAM | SOCK_NONBLOCK, 0);
    int opt = 1;
    setsockopt(listen_fd, SOL_SOCKET, SO_REUSEADDR, &opt, sizeof(opt));

    struct sockaddr_in addr = {
        .sin_family = AF_INET, .sin_port = htons(PORT), .sin_addr.s_addr = INADDR_ANY
    };
    bind(listen_fd, (struct sockaddr *)&addr, sizeof(addr));
    listen(listen_fd, SOMAXCONN);

    int epoll_fd = epoll_create1(EPOLL_CLOEXEC);
    struct epoll_event ev = { .events = EPOLLIN | EPOLLET, .data.fd = listen_fd };
    epoll_ctl(epoll_fd, EPOLL_CTL_ADD, listen_fd, &ev);

    struct epoll_event events[MAX_EVENTS];
    for (;;) {
        int nfds = epoll_wait(epoll_fd, events, MAX_EVENTS, -1);
        for (int i = 0; i < nfds; i++) {
            if (events[i].data.fd == listen_fd) {
                for (;;) {
                    int conn = accept4(listen_fd, NULL, NULL, SOCK_NONBLOCK);
                    if (conn == -1) {
                        if (errno == EAGAIN || errno == EWOULDBLOCK) break;
                        break;
                    }
                    struct epoll_event cev = {
                        .events = EPOLLIN | EPOLLET | EPOLLRDHUP,
                        .data.fd = conn
                    };
                    epoll_ctl(epoll_fd, EPOLL_CTL_ADD, conn, &cev);
                }
            } else {
                if (events[i].events & (EPOLLERR | EPOLLHUP | EPOLLRDHUP)) {
                    close(events[i].data.fd);
                } else if (events[i].events & EPOLLIN) {
                    handle_read(events[i].data.fd);
                }
            }
        }
    }
}
```

#### sendfile 零拷贝

```c
#include <sys/sendfile.h>
#include <sys/stat.h>
#include <fcntl.h>

void serve_file(int conn_fd, const char *path) {
    int fd = open(path, O_RDONLY);
    if (fd < 0) return;
    struct stat st;
    fstat(fd, &st);
    off_t offset = 0;
    while (offset < st.st_size) {
        ssize_t sent = sendfile(conn_fd, fd, &offset, st.st_size - offset);
        if (sent <= 0) break;
    }
    close(fd);
}
```

#### inotify 文件监控

```c
#include <sys/inotify.h>
#include <unistd.h>
#include <stdio.h>

#define BUF_SIZE (1024 * (sizeof(struct inotify_event) + 256))

int main(void) {
    int fd = inotify_init1(IN_NONBLOCK);
    inotify_add_watch(fd, "/tmp/watch",
        IN_CREATE | IN_DELETE | IN_MODIFY | IN_MOVED_FROM | IN_MOVED_TO);

    char buf[BUF_SIZE];
    ssize_t len = read(fd, buf, sizeof(buf));
    char *ptr = buf;
    while (ptr < buf + len) {
        struct inotify_event *e = (struct inotify_event *)ptr;
        if (e->mask & IN_CREATE)  printf("CREATE: %s\n", e->name);
        if (e->mask & IN_DELETE)  printf("DELETE: %s\n", e->name);
        if (e->mask & IN_MODIFY)  printf("MODIFY: %s\n", e->name);
        ptr += sizeof(struct inotify_event) + e->len;
    }
    close(fd);
}
```

### 调优参数

```bash
ulimit -n 1048576                       # fd 上限
echo 1048576 > /proc/sys/fs/file-max   # 系统级 fd 上限
echo 524288 > /proc/sys/fs/inotify/max_user_watches
echo 524288 > /proc/sys/fs/inotify/max_user_instances
blockdev --setra 4096 /dev/sda         # 预读 2MB
```

### 最佳实践

1. **ET 模式必须配合非阻塞 I/O**，否则 read/write 可能永久阻塞
2. **accept4 + SOCK_NONBLOCK** 避免竞态
3. **sendfile** 适用于静态文件服务（Nginx 默认策略）
4. **O_DIRECT 要求对齐**：内存/偏移/长度均为 512B 或 4KB 的倍数
5. **flock 是进程级锁**，**fcntl 是细粒度记录锁**
6. 高并发场景优先用 **io_uring** 替代 epoll

---

## 2. 进程管理

### 概念

**进程状态机：**

```
TASK_RUNNING → TASK_INTERRUPTIBLE / TASK_UNINTERRUPTIBLE → TASK_RUNNING
TASK_RUNNING → EXIT_ZOMBIE → EXIT_DEAD
```

- **孤儿进程**：父进程先退出 → init(1) 收养
- **僵尸进程**：子进程退出但父进程未 wait → task_struct 残留 → 解决：wait/SIGCHLD handler

### API

```c
pid_t fork(void);
pid_t vfork(void);  // 已过时
int execve(const char *pathname, char *const argv[], char *const envp[]);
pid_t wait(int *status);
pid_t waitpid(pid_t pid, int *status, int options);
void _exit(int status);

pid_t setsid(void);
int daemon(int nochdir, int noclose);
int clone(int (*fn)(void *), void *child_stack, int flags, void *arg);
int unshare(int flags);
int setns(int fd, int nstype);
int prctl(int option, ...);  // PR_SET_NAME / PR_SET_PDEATHSIG / PR_SET_CHILD_SUBREAPER
```

### 代码示例

#### fork + exec + SIGCHLD

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <sys/wait.h>
#include <signal.h>

static void sigchld_handler(int sig) {
    while (waitpid(-1, NULL, WNOHANG) > 0);  // 非阻塞回收所有僵尸
}

int main(void) {
    struct sigaction sa = { .sa_handler = sigchld_handler, .sa_flags = SA_RESTART | SA_NOCLDSTOP };
    sigemptyset(&sa.sa_mask);
    sigaction(SIGCHLD, &sa, NULL);

    pid_t pid = fork();
    if (pid == 0) {
        execlp("/bin/ls", "ls", "-la", NULL);
        _exit(127);
    }

    int status;
    waitpid(pid, &status, 0);
    if (WIFEXITED(status))
        printf("Exit code: %d\n", WEXITSTATUS(status));
    else if (WIFSIGNALED(status))
        printf("Killed by signal %d\n", WTERMSIG(status));
}
```

#### 守护进程创建

```c
#include <unistd.h>
#include <sys/stat.h>
#include <fcntl.h>
#include <signal.h>

int become_daemon(const char *pidfile) {
    pid_t pid = fork();
    if (pid < 0) return -1;
    if (pid > 0) _exit(0);            // 1. 父进程退出

    setsid();                          // 2. 新会话

    pid = fork();
    if (pid < 0) return -1;
    if (pid > 0) _exit(0);            // 3. 第二次 fork 防止获取控制终端

    umask(0);                          // 4. 清除 umask
    chdir("/");                        // 5. 切换工作目录

    for (int fd = sysconf(_SC_OPEN_MAX); fd >= 0; fd--) close(fd);  // 6. 关闭 fd

    int fd = open("/dev/null", O_RDWR);
    dup(fd); dup(fd);                  // 7. 重定向 0/1/2
    if (fd > 2) close(fd);

    signal(SIGHUP, SIG_IGN);           // 8. 忽略 SIGHUP

    if (pidfile) {
        fd = open(pidfile, O_WRONLY | O_CREAT | O_TRUNC, 0644);
        if (fd >= 0) { dprintf(fd, "%d\n", getpid()); close(fd); }
    }
    return 0;
}
```

#### /proc 文件系统

```bash
cat /proc/[pid]/status      # VmRSS/Threads/NSpid
cat /proc/[pid]/cmdline     # 命令行
cat /proc/[pid]/fd          # fd 符号链接
cat /proc/[pid]/maps        # 虚拟内存映射
cat /proc/[pid]/smaps       # 详细内存（PSS/Dirty）
cat /proc/[pid]/stack       # 内核栈
cat /proc/[pid]/wchan       # 等待的内核函数
cat /proc/[pid]/cgroup      # cgroup 归属
cat /proc/[pid]/ns/*        # 命名空间
cat /proc/meminfo /proc/cpuinfo /proc/interrupts
cat /proc/net/tcp /proc/net/udp
```

### 调优参数

```bash
echo 4194303 > /proc/sys/kernel/pid_max
echo "/tmp/core.%e.%p.%t" > /proc/sys/kernel/core_pattern
ulimit -c unlimited
```

### 最佳实践

1. **fork 前只调用 async-signal-safe 函数**
2. **SA_NOCLDSTOP** 只在子进程退出时通知
3. **PR_SET_CHILD_SUBREAPER** 让特定进程收养孤儿（容器常用）
4. systemd 的 **Type=notify** 或 **Type=simple** 比 **Type=forking** 更简单

---

## 3. 信号处理

### 概念

- **不可靠信号（1-31）：** 可能丢失，SIGKILL(9)/SIGSTOP(19) 不可捕获
- **可靠信号（34-64）：** 排队投递，携带 int 数据

### API

```c
int kill(pid_t pid, int sig);
int sigaction(int signum, const struct sigaction *act, struct sigaction *oldact);
int sigprocmask(int how, const sigset_t *set, sigset_t *oldset);
int timer_create(clockid_t clockid, struct sigevent *sevp, timer_t *timerid);
int timer_settime(timer_t timerid, int flags, const struct itimerspec *new, struct itimerspec *old);
int signalfd(int fd, const sigset_t *mask, int flags);
```

### 代码示例

#### sigaction + signalfd（推荐：统一到 epoll）

```c
#include <signal.h>
#include <sys/signalfd.h>
#include <sys/epoll.h>
#include <unistd.h>
#include <stdio.h>

int main(void) {
    sigset_t mask;
    sigemptyset(&mask);
    sigaddset(&mask, SIGINT);
    sigaddset(&mask, SIGTERM);
    sigprocmask(SIG_BLOCK, &mask, NULL);  // 必须 block

    int sfd = signalfd(-1, &mask, SFD_NONBLOCK);
    int efd = epoll_create1(EPOLL_CLOEXEC);
    struct epoll_event ev = { .events = EPOLLIN, .data.fd = sfd };
    epoll_ctl(efd, EPOLL_CTL_ADD, sfd, &ev);

    struct epoll_event events[1];
    for (;;) {
        int n = epoll_wait(efd, events, 1, -1);
        if (n > 0) {
            struct signalfd_siginfo si;
            if (read(sfd, &si, sizeof(si)) == sizeof(si)) {
                printf("Signal %d from PID %d\n", si.ssi_signo, si.ssi_pid);
                if (si.ssi_signo == SIGTERM) break;
            }
        }
    }
    close(sfd); close(efd);
}
```

### 最佳实践

1. **永远用 sigaction 代替 signal**
2. **SA_RESTART** 让被中断的系统调用自动重启
3. 信号处理函数中 **只调用 async-signal-safe 函数**（write/_exit/sigprocmask，绝对不能 printf/malloc）
4. **signalfd** 是将信号集成到事件循环的最佳方式
5. **volatile sig_atomic_t** 是唯一的信号安全共享变量类型

---

## 4. 进程间通信（IPC）

### 概念

| 机制 | 方向 | 数据拷贝 | 适用场景 |
|------|------|----------|----------|
| pipe | 单向、血缘 | 2 次 | 父子进程 |
| FIFO (mkfifo) | 单向、无血缘 | 2 次 | 无血缘进程 |
| 共享内存 | 双向 | 0 次（最快） | 大量数据 |
| 信号量 | 同步原语 | 0 次 | 配合共享内存 |
| Socket | 双向、跨主机 | 2 次 | 网络/本地 |
| eventfd | 事件通知 | 0 次 | 轻量级唤醒 |
| D-Bus | 高级 IPC | 多次 | 桌面/系统服务 |

### API

```c
int pipe(int pipefd[2]);
int pipe2(int pipefd[2], int flags);
int mkfifo(const char *pathname, mode_t mode);

// POSIX 共享内存（推荐）
int shm_open(const char *name, int oflag, mode_t mode);
int shm_unlink(const char *name);
// 搭配 mmap 使用

// POSIX 信号量
sem_t *sem_open(const char *name, int oflag, mode_t mode, unsigned int value);
int sem_wait(sem_t *sem);   // P（阻塞）
int sem_post(sem_t *sem);   // V
sem_t *sem_init(sem_t *sem, int pshared, unsigned int value);  // 匿名

// eventfd / timerfd
int eventfd(unsigned int initval, int flags);
int timerfd_create(int clockid, int flags);
int timerfd_settime(int fd, int flags, const struct itimerspec *new, struct itimerspec *old);
```

### 代码示例

#### 共享内存 + 信号量（生产者-消费者）

```c
#define SHM_NAME "/my_shm"
#define SEM_PROD "/sem_prod"
#define SEM_CONS "/sem_cons"
#define SHM_SIZE 4096

// Producer
int shm_fd = shm_open(SHM_NAME, O_CREAT | O_RDWR, 0666);
ftruncate(shm_fd, SHM_SIZE);
char *ptr = mmap(NULL, SHM_SIZE, PROT_READ | PROT_WRITE, MAP_SHARED, shm_fd, 0);
sem_t *prod = sem_open(SEM_PROD, O_CREAT, 0666, SHM_SIZE);  // 空槽
sem_t *cons = sem_open(SEM_CONS, O_CREAT, 0666, 0);          // 已用

for (int i = 0; i < 100; i++) {
    sem_wait(prod);
    snprintf(ptr, SHM_SIZE, "Message %d", i);
    sem_post(cons);
}

// Consumer（对称：sem_wait(cons) + read + sem_post(prod)）
```

#### eventfd 线程间唤醒

```c
int efd = eventfd(0, EFD_CLOEXEC | EFD_NONBLOCK);

// 等待方
uint64_t val;
read(efd, &val, sizeof(val));  // 阻塞

// 唤醒方
val = 1;
write(efd, &val, sizeof(val));
```

### 最佳实践

1. **优先用 POSIX IPC**（shm_open/sem_open）而非 System V
2. **共享内存必须配合同步原语**（信号量/mutex/futex）
3. **eventfd** 比 pipe 更高效用于事件通知（8 字节 vs 管道缓冲区）
4. **timerfd** 可将定时器集成到 epoll 事件循环
5. D-Bus 用于桌面 IPC；高性能场景用共享内存 + 自定义协议

---

## 5. 线程与同步

### 概念

Linux 线程（NPTL）本质是共享地址空间的进程（clone CLONE_VM | CLONE_FS | CLONE_FILES | CLONE_SIGHAND）。每个线程有独立栈、TLS、信号掩码。

### API

```c
int pthread_create(pthread_t *thread, const pthread_attr_t *attr,
                   void *(*start_routine)(void *), void *arg);
int pthread_join(pthread_t thread, void **retval);
int pthread_detach(pthread_t thread);
pthread_t pthread_self(void);

// 互斥锁
int pthread_mutex_init(pthread_mutex_t *mutex, const pthread_mutexattr_t *attr);
int pthread_mutex_lock(pthread_mutex_t *mutex);
int pthread_mutex_unlock(pthread_mutex_t *mutex);
int pthread_mutex_destroy(pthread_mutex_t *mutex);
// 属性：PTHREAD_MUTEX_NORMAL / ERRORCHECK / RECURSIVE

// 读写锁
int pthread_rwlock_init/rlock/wlock/unlock/destroy(pthread_rwlock_t *);

// 条件变量
int pthread_cond_init(pthread_cond_t *cond, const pthread_condattr_t *attr);
int pthread_cond_wait(pthread_cond_t *cond, pthread_mutex_t *mutex);
int pthread_cond_signal(pthread_cond_t *cond);   // 唤醒一个
int pthread_cond_broadcast(pthread_cond_t *cond); // 唤醒全部

// 自旋锁（用户态，慎用）
int pthread_spin_init(pthread_spinlock_t *lock, int pshared);
int pthread_spin_lock/unlock/destroy(pthread_spinlock_t *);

// 屏障
int pthread_barrier_init(pthread_barrier_t *barrier, const pthread_barrierattr_t *attr, unsigned count);
int pthread_barrier_wait(pthread_barrier_t *barrier);

// TLS
__thread int tls_var = 0;  // 编译器内置
int pthread_key_create(pthread_key_t *key, void (*destructor)(void *));
int pthread_setspecific(pthread_key_t key, const void *value);
void *pthread_getspecific(pthread_key_t key);
```

### 代码示例

#### 线程池

```c
#define _GNU_SOURCE
#include <pthread.h>
#include <stdlib.h>
#include <unistd.h>
#include <stdio.h>
#include <errno.h>
#include <string.h>

typedef struct task {
    void (*func)(void *);
    void *arg;
    struct task *next;
} task_t;

typedef struct {
    pthread_mutex_t lock;
    pthread_cond_t  has_task;
    pthread_cond_t  no_task;  // 所有任务完成时通知
    task_t         *head;
    task_t         *tail;
    int             count;     // 待处理任务数
    int             shutdown;
    pthread_t      *threads;
    int             num_threads;
} thread_pool_t;

static void *worker(void *arg) {
    thread_pool_t *pool = arg;
    for (;;) {
        pthread_mutex_lock(&pool->lock);
        while (!pool->head && !pool->shutdown)
            pthread_cond_wait(&pool->has_task, &pool->lock);

        if (pool->shutdown && !pool->head) {
            pthread_mutex_unlock(&pool->lock);
            break;
        }

        task_t *task = pool->head;
        pool->head = task->next;
        if (!pool->head) pool->tail = NULL;
        pool->count--;
        pthread_mutex_unlock(&pool->lock);

        task->func(task->arg);
        free(task);

        pthread_mutex_lock(&pool->lock);
        if (pool->count == 0)
            pthread_cond_signal(&pool->no_task);
        pthread_mutex_unlock(&pool->lock);
    }
    return NULL;
}

int pool_init(thread_pool_t *pool, int num_threads) {
    memset(pool, 0, sizeof(*pool));
    pthread_mutex_init(&pool->lock, NULL);
    pthread_cond_init(&pool->has_task, NULL);
    pthread_cond_init(&pool->no_task, NULL);
    pool->num_threads = num_threads;
    pool->threads = calloc(num_threads, sizeof(pthread_t));

    for (int i = 0; i < num_threads; i++)
        pthread_create(&pool->threads[i], NULL, worker, pool);
    return 0;
}

int pool_submit(thread_pool_t *pool, void (*func)(void *), void *arg) {
    task_t *task = malloc(sizeof(task_t));
    task->func = func;
    task->arg = arg;
    task->next = NULL;

    pthread_mutex_lock(&pool->lock);
    if (pool->tail) pool->tail->next = task;
    else pool->head = task;
    pool->tail = task;
    pool->count++;
    pthread_cond_signal(&pool->has_task);
    pthread_mutex_unlock(&pool->lock);
    return 0;
}

void pool_wait(thread_pool_t *pool) {
    pthread_mutex_lock(&pool->lock);
    while (pool->count > 0)
        pthread_cond_wait(&pool->no_task, &pool->lock);
    pthread_mutex_unlock(&pool->lock);
}

void pool_destroy(thread_pool_t *pool) {
    pthread_mutex_lock(&pool->lock);
    pool->shutdown = 1;
    pthread_cond_broadcast(&pool->has_task);
    pthread_mutex_unlock(&pool->lock);

    for (int i = 0; i < pool->num_threads; i++)
        pthread_join(pool->threads[i], NULL);

    free(pool->threads);
    pthread_mutex_destroy(&pool->lock);
    pthread_cond_destroy(&pool->has_task);
    pthread_cond_destroy(&pool->no_task);
}
```

### 最佳实践

1. **条件变量必须搭配 mutex** 使用，且 wait 前要持有锁
2. **自旋锁只在临界区极短（<2 倍上下文切换时间）时使用**，否则浪费 CPU
3. **优先用 pthread_mutex**（可能用 futex 实现，用户态快速路径）
4. **pthread_cond_broadcast** 比 signal + 检查更安全（避免惊群但避免丢失唤醒）
5. **__thread 比 pthread_key 更高效**（编译器直接生成 TLS 访问代码）
6. 线程数建议：**CPU 密集型 = 核心数，I/O 密集型 = 核心数 × (1 + W/C)**（W=等待时间，C=计算时间）

---

## 6. 网络 Socket 编程

### 概念

**I/O 多路复用对比：**

| 机制 | 复杂度 | 性能 | 最大 fd |
|------|--------|------|---------|
| select | O(n) | 低 | FD_SETSIZE (1024) |
| poll | O(n) | 低 | 无限制 |
| epoll | O(1) 就绪 | 高 | 无限制 |

**epoll 两种模式：**
- **LT（水平触发，默认）：** 缓冲区有数据就通知，直到读完
- **ET（边沿触发）：** 新数据到达时通知一次，必须一次读完（配合非阻塞）

### API

```c
int socket(int domain, int type, int protocol);
int bind(int sockfd, const struct sockaddr *addr, socklen_t addrlen);
int listen(int sockfd, int backlog);
int accept(int sockfd, struct sockaddr *addr, socklen_t *addrlen);
int connect(int sockfd, const struct sockaddr *addr, socklen_t addrlen);
ssize_t send(int sockfd, const void *buf, size_t len, int flags);
ssize_t recv(int sockfd, void *buf, size_t len, int flags);
ssize_t sendto(int sockfd, const void *buf, size_t len, int flags,
               const struct sockaddr *dest_addr, socklen_t addrlen);
ssize_t recvfrom(int sockfd, void *buf, size_t len, int flags,
                 struct sockaddr *src_addr, socklen_t *addrlen);
int setsockopt(int sockfd, int level, int optname, const void *optval, socklen_t optlen);
int getsockopt(int sockfd, int level, int optname, void *optval, socklen_t *optlen);
```

### 代码示例

#### TCP Server-Client 完整示例

```c
// server.c
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <unistd.h>
#include <stdio.h>
#include <string.h>

#define PORT 9000
#define BUFSIZE 4096

int main(void) {
    int sfd = socket(AF_INET, SOCK_STREAM, 0);
    int opt = 1;
    setsockopt(sfd, SOL_SOCKET, SO_REUSEADDR, &opt, sizeof(opt));

    struct sockaddr_in addr = {
        .sin_family = AF_INET, .sin_port = htons(PORT),
        .sin_addr.s_addr = INADDR_ANY
    };
    bind(sfd, (struct sockaddr *)&addr, sizeof(addr));
    listen(sfd, 128);

    for (;;) {
        struct sockaddr_in cli;
        socklen_t cli_len = sizeof(cli);
        int cfd = accept(sfd, (struct sockaddr *)&cli, &cli_len);
        char buf[BUFSIZE];
        ssize_t n = read(cfd, buf, sizeof(buf) - 1);
        if (n > 0) {
            buf[n] = '\0';
            printf("From %s:%d: %s", inet_ntoa(cli.sin_addr), ntohs(cli.sin_port), buf);
            write(cfd, buf, n);  // echo
        }
        close(cfd);
    }
}

// client.c
int main(void) {
    int fd = socket(AF_INET, SOCK_STREAM, 0);
    struct sockaddr_in addr = {
        .sin_family = AF_INET, .sin_port = htons(PORT),
        .sin_addr.s_addr = inet_addr("127.0.0.1")
    };
    connect(fd, (struct sockaddr *)&addr, sizeof(addr));

    const char *msg = "Hello, Server!\n";
    write(fd, msg, strlen(msg));

    char buf[1024];
    ssize_t n = read(fd, buf, sizeof(buf) - 1);
    if (n > 0) { buf[n] = '\0'; printf("Reply: %s", buf); }

    close(fd);
}
```

#### UDP 收发

```c
// server
int sfd = socket(AF_INET, SOCK_DGRAM, 0);
struct sockaddr_in addr = { .sin_family = AF_INET, .sin_port = htons(53), .sin_addr.s_addr = INADDR_ANY };
bind(sfd, (struct sockaddr *)&addr, sizeof(addr));

char buf[65535];
struct sockaddr_in from;
socklen_t fromlen = sizeof(from);
ssize_t n = recvfrom(sfd, buf, sizeof(buf), 0, (struct sockaddr *)&from, &fromlen);
sendto(sfd, buf, n, 0, (struct sockaddr *)&from, fromlen);  // echo

// client
struct sockaddr_in dest = { .sin_family = AF_INET, .sin_port = htons(53), .sin_addr.s_addr = inet_addr("8.8.8.8") };
sendto(sfd, msg, len, 0, (struct sockaddr *)&dest, sizeof(dest));
```

#### UNIX Domain Socket

```c
int sfd = socket(AF_UNIX, SOCK_STREAM, 0);
struct sockaddr_un addr = { .sun_family = AF_UNIX };
strncpy(addr.sun_path, "/tmp/my.sock", sizeof(addr.sun_path) - 1);
unlink("/tmp/my.sock");
bind(sfd, (struct sockaddr *)&addr, sizeof(addr));
listen(sfd, 128);

// 支持传递文件描述符（SCM_RIGHTS）
struct msghdr msg = {0};
struct iovec iov;
char buf[1];
iov.iov_base = buf; iov.iov_len = 1;
msg.msg_iov = &iov; msg.msg_iovlen = 1;

char cmsgbuf[CMSG_SPACE(sizeof(int))];
msg.msg_control = cmsgbuf;
msg.msg_controllen = sizeof(cmsgbuf);

struct cmsghdr *cmsg = CMSG_FIRSTHDR(&msg);
cmsg->cmsg_level = SOL_SOCKET;
cmsg->cmsg_type = SCM_RIGHTS;
cmsg->cmsg_len = CMSG_LEN(sizeof(int));
memcpy(CMSG_DATA(cmsg), &fd_to_send, sizeof(int));

sendmsg(sfd, &msg, 0);  // 发送 fd
```

### 调优参数

```bash
# Socket 选项
SO_REUSEADDR      # TIME_WAIT 状态复用地址
SO_REUSEPORT      # 多进程/线程绑定同一端口（负载均衡）
SO_REUSEPORT_LB   # 内核级负载均衡（5.7+）
TCP_NODELAY       # 禁用 Nagle 算法（低延迟场景）
TCP_CORK          # 聚合小包（write+write → 一个大包，类似 Nagle 但更可控）
SO_KEEPALIVE      # 保活探测
SO_RCVBUF/SO_SNDBUF  # 调整缓冲区

# 系统参数
net.core.somaxconn = 4096
net.ipv4.tcp_max_syn_backlog = 8192
net.ipv4.tcp_tw_reuse = 1
net.ipv4.tcp_fin_timeout = 15
```

### 最佳实践

1. **TCP_NODELAY** 对延迟敏感的场景（HTTP/2、gRPC）务必开启
2. **TCP_CORK** 适合 sendfile 后立即发 header 的场景（Nginx 模式）
3. **SO_REUSEPORT** 让多个 worker 进程各自 accept，避免 thundering herd
4. **非阻塞 I/O**：设置 O_NONBLOCK 后，connect/read/write 返回 -1 且 errno=EAGAIN/EWOULDBLOCK 表示未就绪
5. **TCP 三次握手时 backlog**：min(somaxconn, tcp_max_syn_backlog, 应用层 listen backlog)
6. **TIME_WAIT**：主动关闭方进入，持续 2MSL（60s），tcp_tw_reuse 仅用于客户端

---

## 7. 内存管理

### 概念

**虚拟内存布局（64 位）：**

```
高地址
┌─────────────┐
│  栈 (向下增长) │ 8MB 默认 (ulimit -s)
├─────────────┤
│    ↕ 空隙    │
├─────────────┤
│  mmap 映射区  │ 共享库、mmap、shm
├─────────────┤
│    堆 (向上)  │ brk/sbrk / malloc
├─────────────┤
│  BSS (未初始化) │
├─────────────┤
│  数据段 (已初始化)│
├─────────────┤
│  代码段 (只读)  │
低地址
```

**malloc 实现：**
- **glibc ptmalloc2**：基于 arena + bin 的设计，多线程用多个 arena 减少竞争
- **tcmalloc**（Google）：Thread-local cache，减少锁竞争
- **jemalloc**（Facebook/Redis）：size class 优化，碎片更少

### API

```c
void *brk(const void *addr);
tptrdiff_t increment);
// brk/sbrk 是堆增长的底层，malloc 内部使用

// overcommit 控制
// /proc/sys/vm/overcommit_memory:
//   0 - heuristic（默认，允许适度超分配）
//   1 - always（总是允许，可能 OOM）
//   2 - never（不允许超过 commit_ratio，swap+RAM）

// OOM 评分
// /proc/[pid]/oom_score       (0-1000，越高越容易被杀)
// /proc/[pid]/oom_score_adj   (-1000 到 1000，-1000 = 永不被杀)
```

### 代码示例

#### 查看/调整 overcommit

```bash
cat /proc/sys/vm/overcommit_memory
echo 2 > /proc/sys/vm/overcommit_memory   # 严格模式
cat /proc/self/smaps_rollup
pmap -x $(pidof myapp)
```

#### 内存碎片诊断

```c
#include <malloc.h>
struct mallinfo mi = mallinfo();
printf("Arena: %d, Uordblks: %d, Fordblks: %d\n",
       mi.arena, mi.ordblks, mi.uordblks, mi.fordblks);
```

### 调优参数

```bash
echo 20 > /proc/sys/vm/nr_hugepages
echo madvise > /sys/kernel/mm/transparent_hugepage/enabled
echo 10 > /proc/sys/vm/swappiness
echo -1000 > /proc/$(pidof myapp)/oom_score_adj
echo lz4 > /sys/module/zswap/parameters/compressor
echo "100M" > /sys/fs/cgroup/mygroup/memory.max
```

### 最佳实践

1. **大块内存分配用 mmap**（>128KB 默认阈值 MMAP_THRESHOLD）
2. **数据库/大文件用 Huge Pages** 减少 TLB miss
3. **生产环境 THP=madvise** 而非 always
4. **swappiness=10** 适合大多数服务器
5. **zswap > zram**

---

# 第二部分：Linux 内核

## 8. 内核架构

### 概念

Linux 采用**宏内核（Monolithic Kernel）**：所有内核服务运行在同一个地址空间。

**虚拟文件系统：** `/proc`（procfs）、`/sys`（sysfs）、`/dev`（devtmpfs）

### 命令

```bash
insmod module.ko && rmmod module && modprobe module
modinfo module && lsmod
make menuconfig && make -j$(nproc) && make modules_install && make install
dracut --force
```

### 代码示例

#### 最简内核模块

```c
#include <linux/module.h>
#include <linux/init.h>

static int __init hello_init(void) {
    pr_info("Hello, kernel!\n");
    return 0;
}
static void __exit hello_exit(void) {
    pr_info("Goodbye, kernel!\n");
}
module_init(hello_init);
module_exit(hello_exit);
MODULE_LICENSE("GPL");
```

```makefile
obj-m += hello.o
KDIR := /lib/modules/$(shell uname -r)/build
all:
	make -C $(KDIR) M=$(PWD) modules
clean:
	make -C $(KDIR) M=$(PWD) clean
```

#### 内核启动流程

```
BIOS → POST → Bootloader (GRUB) → 内核解压 → start_kernel()
  → setup_arch() → mm_init() → sched_init() → init_IRQ()
  → time_init() → console_init() → rest_init()
    → kernel_init → /sbin/init (PID 1) → systemd
```

### 最佳实践

1. **生产环境内核不要随意升级**
2. **内核模块签名**（CONFIG_MODULE_SIG）防止恶意模块
3. **initramfs** 包含启动所需驱动

---

## 9. 内核同步

### 概念

| 原语 | 能否睡眠 | 适用场景 |
|------|----------|----------|
| atomic | 否 | 计数器、标志位 |
| spinlock | 否 | 短临界区、中断上下文 |
| mutex | 能 | 长临界区、进程上下文 |
| rw_semaphore | 能 | 读多写少 |
| RCU | 否（读者） | 读多写极少 |
| completion | 能 | 等待事件完成 |

### API

```c
// 自旋锁
spin_lock(&lock);
spin_lock_irq(&lock);          // 禁用本地中断 + 加锁
spin_lock_irqsave(&lock, flags);
spin_unlock_irqrestore(&lock, flags);

// 互斥锁
mutex_init(&lock);
mutex_lock(&lock);             // 可睡眠！
mutex_unlock(&lock);

// RCU
rcu_read_lock();
p = rcu_dereference(gptr);
rcu_read_unlock();
rcu_assign_pointer(gptr, new_ptr);
synchronize_rcu();
kfree_rcu(old_ptr, rcu_head);

// 原子操作
atomic_t count = ATOMIC_INIT(0);
atomic_inc(&count);
atomic_cmpxchg(&count, old, new);  // CAS

// 完成量
init_completion(&comp);
wait_for_completion(&comp);
complete(&comp);

// 内存屏障
smp_wmb(); smp_rmb(); smp_mb();
```

### 代码示例

```c
// RCU 链表遍历
void read_entries(void) {
    struct my_entry *entry;
    rcu_read_lock();
    list_for_each_entry_rcu(entry, &my_list, list)
        printk("%d\n", entry->data);
    rcu_read_unlock();
}

void remove_entry(struct my_entry *target) {
    list_del_rcu(&target->list);
    kfree_rcu(target, rcu);
}
```

### 最佳实践

1. **中断上下文绝对不能睡眠**
2. **RCU 读者零开销**，写者开销大
3. **mutex 比 spinlock 更安全**，进程上下文优先用 mutex
4. **lockdep**（CONFIG_LOCKDEP）运行时检测死锁

---

## 10. 中断与软中断

### 概念

**上半部（hardirq）：** 快速响应，不能睡眠
**下半部（softirq / tasklet / workqueue）：** 延迟处理

### API

```c
int request_irq(unsigned int irq, irq_handler_t handler,
                unsigned long flags, const char *name, void *dev);
void free_irq(unsigned int irq, void *dev);
void tasklet_init(struct tasklet_struct *t, void (*func)(unsigned long), unsigned long data);
void tasklet_schedule(struct tasklet_struct *t);
INIT_WORK(&work, work_handler);
schedule_work(&work);
flush_work(&work);
cancel_work_sync(&work);
```

### 最佳实践

1. **上半部 <100μs**
2. **需要睡眠用 workqueue**
3. 高网络吞吐用 **RPS/RFS** 分散软中断

---

## 11. 设备驱动

### 概念

字符设备（cdev）/ 块设备 / 网络设备。platform_device + platform_driver 通过名字匹配绑定。

### 代码示例

#### miscdevice 驱动

```c
#include <linux/module.h>
#include <linux/miscdevice.h>
#include <linux/uaccess.h>

static char msg[256] = "Hello from kernel!\n";
static size_t msg_len = 20;

static ssize_t dev_read(struct file *f, char __user *buf, size_t n, loff_t *off) {
    if (*off >= msg_len) return 0;
    if (n > msg_len - *off) n = msg_len - *off;
    if (copy_to_user(buf, msg + *off, n)) return -EFAULT;
    *off += n; return n;
}

static ssize_t dev_write(struct file *f, const char __user *buf, size_t n, loff_t *off) {
    if (n >= sizeof(msg)) n = sizeof(msg) - 1;
    if (copy_from_user(msg, buf, n)) return -EFAULT;
    msg[n] = '\0'; msg_len = n; return n;
}

static const struct file_operations dev_fops = {
    .owner = THIS_MODULE, .read = dev_read, .write = dev_write,
};
static struct miscdevice my_misc = {
    .minor = MISC_DYNAMIC_MINOR, .name = "mydev", .fops = &dev_fops,
};
static int __init dev_init(void) { return misc_register(&my_misc); }
static void __exit dev_exit(void) { misc_deregister(&my_misc); }
module_init(dev_init); module_exit(dev_exit);
MODULE_LICENSE("GPL");
```

#### ioctl 接口

```c
#define MYDEV_IOC_MAGIC 'M'
#define MYDEV_SET_MSG _IOW(MYDEV_IOC_MAGIC, 1, char[256])
#define MYDEV_GET_MSG _IOR(MYDEV_IOC_MAGIC, 2, char[256])
#define MYDEV_RESET   _IO(MYDEV_IOC_MAGIC, 3)

static long dev_ioctl(struct file *f, unsigned int cmd, unsigned long arg) {
    switch (cmd) {
    case MYDEV_SET_MSG:
        if (copy_from_user(msg, (char __user *)arg, 256)) return -EFAULT; break;
    case MYDEV_GET_MSG:
        if (copy_to_user((char __user *)arg, msg, 256)) return -EFAULT; break;
    case MYDEV_RESET: memset(msg, 0, sizeof(msg)); break;
    default: return -ENOTTY;
    }
    return 0;
}
```

#### 设备树（DTS）

```dts
/dts-v1/;
/ {
    compatible = "vendor,board";
    my_device@12340000 {
        compatible = "vendor,my-device";
        reg = <0x12340000 0x1000>;
        interrupts = <0 23 4>;
        status = "okay";
    };
};
```

### 最佳实践

1. **优先用 miscdevice** 简化注册
2. **ioctl 用 _IO/_IOR/_IOW 宏**
3. **copy_to_user/from_user 必须检查返回值**
4. **sysfs 属性**用 DEVICE_ATTR 宏

---

## 12. 内核调试

```bash
echo 8 > /proc/sys/kernel/printk
dmesg -T -w

cd /sys/kernel/debug/tracing
echo function > current_tracer
echo 1 > tracing_on && cat trace

perf stat ./myapp
perf record -g -p <pid> && perf report
perf flamegraph

execsnoop-bpfcc opensnoop-bpfcc biosnoop-bpfcc tcplife-bpfcc

echo 'p:myprobe do_sys_open filename=+0(%dx):string' > kprobe_events
echo 1 > events/kprobes/myprobe/enable && cat trace_pipe

kdumpctl status
crash /usr/lib/debug/lib/modules/$(uname -r)/vmlinux /var/crash/vmcore
```

**eBPF 是内核观测的未来。kdump 分析内核 panic，生产环境务必配置。**

---

## 13. eBPF

### 概念

eBPF 允许安全地在内核中运行自定义代码。工作流：C → BPF 字节码 → 验证器 → JIT → hook 点执行。

### 工具链

```bash
# BCC
execsnoop-bpfcc opensnoop-bpfcc biosnoop-bpfcc tcplife-bpfcc profile-bpfcc
# libbpf + CO-RE（生产推荐）
# bpftrace（快速原型）
bpftrace -e 'tracepoint:syscalls:sys_enter_openat { printf("%s %s\n", comm, str(args->filename)); }'
# XDP
ip link set dev eth0 xdp obj xdp_prog.o sec xdp
```

### XDP 示例

```c
#include <linux/bpf.h>
#include <bpf/bpf_helpers.h>

SEC("xdp")
int xdp_drop_prog(struct xdp_md *ctx) {
    void *data = (void *)(long)ctx->data;
    void *data_end = (void *)(long)ctx->data_end;
    struct ethhdr *eth = data;
    if ((void *)(eth + 1) > data_end) return XDP_PASS;
    if (eth->h_dest[0] == 0x00 && eth->h_dest[1] == 0x11) return XDP_DROP;
    return XDP_PASS;
}
char LICENSE[] SEC("license") = "GPL";
```

**开发用 BCC/bpftrace，生产用 libbpf + CO-RE。XDP 可达 100Gbps+。**

---

# 第三部分：Linux 网络

## 14. 网络栈

### 收发路径

```
收: NIC → IRQ → NAPI → netif_receive_skb → IP → TCP → socket recv queue → read()
发: write() → send buffer → TCP → IP → qdisc → NIC → DMA
```

### 调优参数

```bash
net.ipv4.tcp_tw_reuse = 1
net.ipv4.tcp_max_syn_backlog = 8192
net.core.somaxconn = 4096
net.ipv4.tcp_fin_timeout = 15
net.core.default_qdisc = fq
net.ipv4.tcp_congestion_control = bbr
net.core.rmem_max = 16777216
net.core.wmem_max = 16777216
net.ipv4.tcp_rmem = 4096 87380 16777216
net.ipv4.tcp_wmem = 4096 65536 16777216
```

**BBR + fq 是最佳 TCP 拥塞控制。nftables 替代 iptables。**

---

## 15. 网络配置

```bash
ip link add name br0 type bridge && ip link set eth0 master br0
ip link add link eth0 name eth0.100 type vlan id 100
ip link add bond0 type bond mode 802.3ad miimon 100
ip link add vrf_blue type vrf table 100
ip rule add from 10.0.0.0/24 table 100
ip route add table 100 default via 192.168.1.1
```

---

## 16. 容器网络

| 插件 | 模式 | 性能 |
|------|------|------|
| bridge | 二层 | 中 |
| Flannel | VXLAN | 中 |
| Calico | BGP | 高 |
| Cilium | eBPF | 最高 |

```bash
ip netns add c1
ip link add veth0 type veth peer name veth1
ip link set veth1 netns c1
ip link set veth0 master br0
ip netns exec c1 ip addr add 172.17.0.2/24 dev veth1
```

**Cilium + eBPF 是 K8s 网络最佳选择。**

---

# 第四部分：Linux 安全

## 17. 权限模型

```
DAC (UID/GID + rwx + SUID/SGID)
  → capabilities (细粒度特权)
    → MAC (SELinux / AppArmor)
      → seccomp (限制系统调用)
        → namespaces (隔离)
```

```bash
setcap cap_net_bind_service=+ep /app
getenforce && audit2allow -a -M mypolicy
docker run --security-opt seccomp=profile.json
unshare --net /bin/bash
nsenter -t <pid> -n -p
```

#### seccomp 示例

```c
#include <linux/seccomp.h>
#include <linux/filter.h>
#include <sys/prctl.h>

struct sock_filter filter[] = {
    BPF_STMT(BPF_LD | BPF_W | BPF_ABS, offsetof(struct seccomp_data, nr)),
    BPF_JUMP(BPF_JMP | BPF_JEQ, __NR_write, 0, 1),
    BPF_STMT(BPF_RET | BPF_K, SECCOMP_RET_ALLOW),
    BPF_STMT(BPF_RET | BPF_K, SECCOMP_RET_KILL_PROCESS),
};
struct sock_fprog prog = { .len = 4, .filter = filter };
prctl(PR_SET_NO_NEW_PRIVS, 1, 0, 0, 0);
prctl(PR_SET_SECCOMP, SECCOMP_MODE_FILTER, &prog);
```

**容器安全基线：非 root + readonly + drop ALL caps + seccomp**

---

## 18. 安全工具

```bash
auditctl -w /etc/passwd -p wa -k passwd_changes
ausearch -k passwd_changes
fail2ban-client status sshd
```

---

# 第五部分：Linux 性能调优

## 19. CPU 调优

```bash
taskset -c 0,1 ./myapp
chrt -f -p 99 <pid>
numactl --cpunodebind=0 --membind=0 ./myapp
echo performance > /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor
perf stat -e cycles,instructions,cache-misses ./myapp
perf record -g -p <pid> && perf report
```

**CPU 隔离：** 启动参数 `isolcpus=2,3 nohz_full=2,3 rcu_nocbs=2,3`

---

## 20. 内存调优

```bash
echo 20 > /proc/sys/vm/nr_hugepages
echo madvise > /sys/kernel/mm/transparent_hugepage/enabled
echo 10 > /proc/sys/vm/swappiness
echo -1000 > /proc/$(pidof myapp)/oom_score_adj
echo lz4 > /sys/module/zswap/parameters/compressor
```

---

## 21. I/O 调优

```bash
cat /sys/block/sda/queue/scheduler
echo mq-deadline > /sys/block/sda/queue/scheduler
echo 4096 > /sys/block/sda/queue/nr_requests
iostat -x 1
fio --name=randread --ioengine=libaio --iodepth=32 --rw=randread --bs=4k --direct=1 --size=1G --numjobs=4 --runtime=30
# io_uring（Linux 5.1+）是最强异步 I/O
```

**生产环境推荐 mq-deadline 或 none（SSD）。**

---

## 22. 网络调优

```bash
net.ipv4.tcp_tw_reuse = 1
net.ipv4.tcp_max_syn_backlog = 8192
net.core.somaxconn = 4096
net.ipv4.tcp_slow_start_after_idle = 0
net.core.default_qdisc = fq
net.ipv4.tcp_congestion_control = bbr
net.netfilter.nf_conntrack_max = 131072
net.netfilter.nf_conntrack_tcp_timeout_established = 7200
```

---

## 23. 系统调优

```bash
sysctl -a
sysctl -w kernel.pid_max=4194303

systemctl edit myservice
systemctl daemon-reload

mkdir /sys/fs/cgroup/myapp
echo "100M" > /sys/fs/cgroup/myapp/memory.max
echo "500000" > /sys/fs/cgroup/myapp/cpu.max
echo $$ > /sys/fs/cgroup/myapp/cgroup.procs

journalctl -u myservice -f
journalctl --vacuum-size=100M
```

**sysctl 改动写入 /etc/sysctl.d/*.conf 持久化。cgroups v2 统一资源控制。**

---

# 附录：面试高频问题速查

1. **epoll LT vs ET：** LT 缓冲区有数据就通知，ET 只在新数据到达时通知一次
2. **fork COW：** 物理页在写入时才复制
3. **进程 vs 线程：** 线程共享地址空间。Linux 线程本质是 clone(CLONE_VM) 的进程
4. **僵尸进程：** 子进程退出但父进程未 wait → wait/SIGCHLD 回收
5. **select vs epoll：** select O(n) + 1024 fd 限制，epoll O(1) + 无限制
6. **mmap vs read：** mmap 零拷贝映射，read 需要内核→用户态拷贝
7. **RCU：** 读无锁，写先复制再替换指针，等 grace period 后释放旧数据
8. **TCP 三次握手/四次挥手：** SYN→SYN-ACK→ACK；FIN→ACK→FIN→ACK
9. **Nagle 算法：** 小包聚合，TCP_NODELAY 禁用
10. **OOM Killer：** 根据 oom_score 选择进程杀死，oom_score_adj 可调整权重
