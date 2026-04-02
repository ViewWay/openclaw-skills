# Embedded OS 全家桶 Skill

> 嵌入式操作系统开发参考手册，覆盖裸机编程到主流 RTOS/物联网 OS，面向开发者实用级别。

---

## 目录

1. [裸机编程（Non-OS）](#1-裸机编程non-os)
2. [FreeRTOS（重点）](#2-freertos重点)
3. [Zephyr RTOS](#3-zephyr-rtos)
4. [RT-Thread](#4-rt-thread)
5. [μC/OS-II/III](#5-ucos-iiiii)
6. [VxWorks](#6-vxworks)
7. [Contiki / TinyOS](#7-contiki--tinyos)
8. [mbed OS](#8-mbed-os)
9. [RIOT OS](#9-riot-os)
10. [LwIP（轻量级 TCP/IP 协议栈）](#10-lwip轻量级-tcpip-协议栈)
11. [RTOS 对比表](#11-rtos-对比表)

---

## 1. 裸机编程（Non-OS）

### 1.1 超级循环（Super Loop / Bare-Metal）

#### 前台/后台架构

```
┌─────────────────────────────┐
│  初始化（Init）              │
├─────────────────────────────┤
│  while (1) {                │  ← 后台循环
│    task1();                 │
│    task2();                 │
│    // 无优先级，顺序执行      │
│  }                          │
├─────────────────────────────┤
│  ISR → 设置标志位/写入缓冲   │  ← 前台（中断）
└─────────────────────────────┘
```

**缺点：** 任务间无优先级、长任务阻塞其他任务、难以管理复杂时序。

#### 状态机设计（FSM）

```c
// 简单有限状态机
typedef enum { STATE_IDLE, STATE_RUN, STATE_ERROR } State_t;

void task_handler(Event_t event) {
    static State_t state = STATE_IDLE;
    switch (state) {
        case STATE_IDLE:
            if (event == EVT_START) { start_action(); state = STATE_RUN; }
            break;
        case STATE_RUN:
            if (event == EVT_DONE)   { state = STATE_IDLE; }
            if (event == EVT_ERROR)  { state = STATE_ERROR; }
            break;
        case STATE_ERROR:
            if (event == EVT_RESET)  { recover(); state = STATE_IDLE; }
            break;
    }
}
```

**层次状态机（HSM）：** 用函数指针表或状态嵌套实现层级转换，适合复杂协议解析。

#### 中断驱动架构

```c
volatile uint8_t flag = 0;

void EXTI_IRQHandler(void) {
    flag = 1;              // ISR 只做最少工作
    __DSB();               // 数据同步屏障
}

int main(void) {
    for (;;) {
        if (flag) {
            flag = 0;
            process_data(); // 在主循环处理
        }
        sleep_mode();       // 空闲时进入低功耗
    }
}
```

#### 协程式设计（Protothreads）

```c
// 简易协程宏（基于 switch-case Duff's Device）
#define PT_BEGIN(pt)   switch((pt)->lc) { case 0:
#define PT_END(pt)     } (pt)->lc = 0; return 0;
#define PT_WAIT_UNTIL(pt, cond)  \
    do { (pt)->lc = __LINE__; case __LINE__: \
         if (!(cond)) return 1; } while(0)

typedef struct { int lc; } pt_t;

int task1(pt_t *pt) {
    PT_BEGIN(pt);
    PT_WAIT_UNTIL(pt, data_ready());
    process();
    PT_END(pt);
}
```

### 1.2 中断管理

#### NVIC 配置（ARM Cortex-M）

```c
// 优先级分组：4 位抢占优先级，0 位子优先级
HAL_NVIC_SetPriorityGrouping(NVIC_PRIORITYGROUP_4);

// 设置中断优先级（数值越小优先级越高）
HAL_NVIC_SetPriority(EXTI0_IRQn, 5, 0);  // 抢占=5, 子=0
HAL_NVIC_EnableIRQ(EXTI0_IRQn);
```

**优先级分组对照表：**

| 分组 | 抢占位 | 子优先级位 | 优先级数 |
|------|--------|-----------|---------|
| NVIC_PRIORITYGROUP_0 | 0 | 4 | 16 子优先级 |
| NVIC_PRIORITYGROUP_2 | 2 | 2 | 4×4 |
| NVIC_PRIORITYGROUP_4 | 4 | 0 | 16 抢占级 |

#### 中断延迟三要素

```
请求 → [中断延迟] → ISR入口 → [中断处理] → [中断恢复] → 返回
         ↑                      ↑                ↑
       硬件响应+            实际处理逻辑      上下文恢复
       挂起中断排队          不得过长          寄存器出栈
```

- **响应延迟：** Cortex-M3/M4/M7 通常 12 周期（尾链 6 周期）
- **处理时间：** 保持短（< 1% CPU 负载），耗时操作交给主循环
- **恢复延迟：** 12 周期

#### 中断嵌套

高抢占优先级中断可以打断低优先级中断。子优先级只决定同抢占级下的排队顺序。

#### volatile 关键字

```c
volatile uint32_t sensor_data;  // ISR 写，主循环读
volatile bool flag;

// 必须用 volatile 的场景：
// 1. ISR 和主循环共享变量
// 2. 内存映射 I/O 寄存器
// 3. 多线程共享（无 OS 下的编译器优化屏障）
```

#### 临界区（裸机）

```c
// 方法1：开关中断（最简单）
uint32_t primask = __get_PRIMASK();
__disable_irq();
// 临界区操作
__set_PRIMASK(primask);

// 方法2：Cortex-M 基址屏蔽（只禁指定优先级以下的中断）
__set_BASEPRI(0x20);  // 屏蔽优先级 >= 0x20 的中断
// 临界区操作
__set_BASEPRI(0);     // 恢复所有中断
```

### 1.3 定时与延时

#### SysTick 滴答定时器

```c
volatile uint32_t system_ticks = 0;

void SysTick_Handler(void) {
    system_ticks++;
}

// 初始化 1ms 滴答
SysTick_Config(SystemCoreClock / 1000);
```

#### 非阻塞延时（时间戳差值法）⭐

```c
// 推荐方式：利用溢出回绕的数学特性
#define DELAY_MS(ms) ((uint32_t)((ms) * (SystemCoreClock / 1000 / SysTick->LOAD)))

bool timeout_reached(uint32_t start, uint32_t delay_ticks) {
    return (system_ticks - start) >= delay_ticks;
}

// 使用
void loop(void) {
    static uint32_t last_tick = 0;
    if (timeout_reached(last_tick, 100)) {  // 100 ticks 延时
        last_tick = system_ticks;
        do_periodic_task();
    }
}
```

**为什么差值法安全：** `uint32_t` 减法在溢出时仍正确（`0 - 0xFFFFFFF0 = 0x10`）。

#### 软件延时 vs 硬件延时

| 方式 | 优点 | 缺点 |
|------|------|------|
| 软件延时 `for(i=0;i<N;i++)` | 无需外设 | 不精确、阻塞 CPU、受编译优化影响 |
| 硬件延时（SysTick/Timer） | 精确、可释放 CPU | 占用硬件资源 |
| 非阻塞延时（时间戳） | 非阻塞、精确 | 需要全局时基 |

---

## 2. FreeRTOS（重点）

### 2.1 核心概念

#### 任务状态机

```
          ┌──────────┐
   空闲→  │  就绪     │ ──调度──→ 运行
          │ (Ready)  │ ←──抢占──  │(Running)│
          └──────────┘            └────┬────┘
               ↑                      │
          超时/事件               阻塞/挂起
               │                      ↓
          ┌────┴────┐            ┌──────────┐
          │  阻塞    │            │  挂起     │
          │(Blocked)│            │(Suspended)│
          └─────────┘            └──────────┘
```

#### 任务创建

```c
// 动态创建（使用 FreeRTOS 堆）
TaskHandle_t task1_handle;
xTaskCreate(
    vTask1,           // 任务函数
    "Task1",          // 名称（调试用）
    256,              // 栈大小（单位：word，即 4 bytes）
    NULL,             // 参数
    configMAX_PRIORITIES - 2,  // 优先级
    &task1_handle     // 句柄
);

// 静态创建（推荐嵌入式，无碎片）
static StackType_t task1_stack[256];
static StaticTask_t task1_tcb;
task1_handle = xTaskCreateStatic(
    vTask1, "Task1", 256, NULL,
    configMAX_PRIORITIES - 2,
    task1_stack, &task1_tcb
);

void vTask1(void *pvParam) {
    for (;;) {
        // 任务逻辑
        vTaskDelay(pdMS_TO_TICKS(100));  // 释放 CPU 100ms
    }
    // vTaskDelete(NULL);  // 自删除（如需要）
}
```

#### 任务通知（轻量 IPC）

```c
// 发送端（ISR 中）
void UART_IRQHandler(void) {
    BaseType_t xHigherPriorityTaskWoken = pdFALSE;
    vTaskNotifyGiveFromISR(rx_task_handle, &xHigherPriorityTaskWoken);
    portYIELD_FROM_ISR(xHigherPriorityTaskWoken);
}

// 接收端
void rx_task(void *arg) {
    for (;;) {
        ulTaskNotifyTake(pdTRUE, portMAX_DELAY);  // 等待通知，计数清零
        process_uart_data();
    }
}
```

#### Idle 任务与 Hook

```c
// configUSE_IDLE_HOOK = 1
void vApplicationIdleHook(void) {
    // 空闲时进入低功耗
    __WFI();  // Wait For Interrupt
}
```

#### 栈溢出检测

```c
// FreeRTOSConfig.h
#define configCHECK_FOR_STACK_OVERFLOW 2  // 方法2（更彻底）

// Hook 函数
void vApplicationStackOverflowHook(TaskHandle_t xTask, char *pcTaskName) {
    printf("STACK OVERFLOW: %s\n", pcTaskName);
    for (;;) {}  // 死循环，方便调试器捕获
}
```

### 2.2 任务间通信

#### 队列（Queue）

```c
QueueHandle_t msg_queue;

// 创建（深度5，每个元素 sizeof(msg_t)）
msg_queue = xQueueCreate(5, sizeof(msg_t));

// 发送（任务中）
msg_t msg = {.id = 1, .data = 42};
xQueueSend(msg_queue, &msg, pdMS_TO_TICKS(100));

// 发送（ISR 中）
xQueueSendFromISR(msg_queue, &msg, &xHigherPriorityTaskWoken);

// 接收
msg_t received;
if (xQueueReceive(msg_queue, &received, portMAX_DELAY) == pdTRUE) {
    handle(received);
}
```

#### 信号量（Semaphore）

```c
// 二值信号量（同步）
SemaphoreHandle_t sem = xSemaphoreCreateBinary();
xSemaphoreGive(sem);                    // 发送
xSemaphoreTake(sem, portMAX_DELAY);    // 等待

// 计数信号量（资源管理，如缓冲区槽位）
SemaphoreHandle_t counting_sem = xSemaphoreCreateCounting(10, 0);
```

#### 互斥量（Mutex）

```c
SemaphoreHandle_t mutex = xSemaphoreCreateMutex();

xSemaphoreTake(mutex, portMAX_DELAY);
shared_resource++;
xSemaphoreGive(mutex);

// 递归互斥量（同任务可重复获取）
SemaphoreHandle_t rmutex = xSemaphoreCreateRecursiveMutex();
xSemaphoreTakeRecursive(rmutex, portMAX_DELAY);
  xSemaphoreTakeRecursive(rmutex, portMAX_DELAY);  // OK，不会死锁
  // ...
  xSemaphoreGiveRecursive(rmutex);
xSemaphoreGiveRecursive(rmutex);
```

**优先级继承：** 互斥量自动解决优先级反转问题。信号量没有此特性。

#### 事件组（Event Group）

```c
EventGroupHandle_t events = xEventGroupCreate();

#define EVT_WIFI_CONNECTED   (1 << 0)
#define EVT_TIME_SYNCED      (1 << 1)
#define EVT_DATA_READY       (1 << 2)

// 设置事件位
xEventGroupSetBits(events, EVT_WIFI_CONNECTED);

// 等待多个事件（所有位都置位）
EventBits_t bits = xEventGroupWaitBits(
    events,
    EVT_WIFI_CONNECTED | EVT_TIME_SYNCED,
    pdTRUE,    // 退出时清除位
    pdTRUE,    // 等待所有位
    portMAX_DELAY
);
```

#### 流缓冲 / 消息缓冲

```c
// 流缓冲：连续字节流（如 UART DMA 接收）
StreamBufferHandle_t stream = xStreamBufferCreate(1024, 1);
xStreamBufferSend(stream, data, len, pdMS_TO_TICKS(100));
xStreamBufferReceive(stream, buf, sizeof(buf), portMAX_DELAY);

// 消息缓冲：有界离散消息（消息长度前缀自动处理）
MessageBufferHandle_t msgbuf = xMessageBufferCreate(512);
xMessageBufferSend(msgbuf, &msg, sizeof(msg), 0);
xMessageBufferReceive(msgbuf, &rx_msg, sizeof(rx_msg), portMAX_DELAY);
```

**选择指南：**

| 通信方式 | 适用场景 | 开销 |
|---------|---------|------|
| 任务通知 | 单生产者→单消费者，轻量同步 | 最低 |
| 队列 | 结构化数据传递 | 低 |
| 流缓冲 | 连续字节流（UART/SPI） | 低 |
| 信号量 | 事件同步、资源计数 | 低 |
| 互斥量 | 共享资源保护 | 中（优先级继承） |
| 事件组 | 多事件同步 | 中 |

### 2.3 同步原语

```c
// 临界区（关中断，极短临界区用）
taskENTER_CRITICAL();
shared_var++;
taskEXIT_CRITICAL();

// 调度器锁（允许中断，禁止任务切换）
vTaskSuspendAll();
// 多个共享操作
xTaskResumeAll();

// ⚠️ 区别：
// taskENTER_CRITICAL  → 关中断，极短（< 几十 us）
// vTaskSuspendAll     → 不关中断，较长临界区（可被 ISR 打断）
```

### 2.4 定时器

```c
// 创建软件定时器
TimerHandle_t timer = xTimerCreate(
    "blink",                     // 名称
    pdMS_TO_TICKS(500),          // 周期 500ms
    pdTRUE,                      // pdTRUE=自动重载（周期），pdFALSE=单次
    NULL,                        // 参数
    vTimerCallback               // 回调
);

xTimerStart(timer, 0);

void vTimerCallback(TimerHandle_t xTimer) {
    // ⚠️ 在定时器守护任务上下文执行，不能阻塞！
    LED_TOGGLE();
}
```

#### Tickless Idle（低功耗）

```c
// FreeRTOSConfig.h
#define configUSE_TICKLESS_IDLE  2

// 需要实现：
// - preSleepProcessing()：设置唤醒源（RTC/LPTIMER），停止 SysTick
// - postSleepProcessing()：恢复 SysTick，补偿睡过的 tick 数
// 参考 FreeRTOS/Demo/Corridor 中 LowPower_TickManagement
```

### 2.5 内存管理

| 方案 | 分配 | 释放 | 碎片 | 适用场景 |
|------|------|------|------|---------|
| heap_1 | ✅ | ❌ | 无 | 只分配不释放（静态系统） |
| heap_2 | ✅ | ✅ | 有 | 不推荐（首次适应，碎片多） |
| heap_3 | ✅ | ✅ | 有 | 包装标准 malloc（有锁开销） |
| heap_4 | ✅ | ✅ | 低 | **推荐**（合并相邻空闲块） |
| heap_5 | ✅ | ✅ | 低 | 非连续内存区域 |

```c
// 静态分配（推荐）
static StackType_t task_stack[256];
static StaticTask_t task_tcb;

// heap_4 使用
void *ptr = pvPortMalloc(1024);
vPortFree(ptr);
```

### 2.6 调试

```c
// 运行时任务列表（需 configUSE_TRACE_FACILITY=1, configUSE_STATS_FORMATTING_FUNCTIONS=1）
char buf[512];
vTaskList(buf);              // 任务状态表
vTaskGetRunTimeStats(buf);   // CPU 占用率

// 配置断言
#define configASSERT(x)  if(!(x)) { taskDISABLE_INTERRUPTS(); for(;;); }
```

**工具链：**
- **Tracealyzer**（Percepio）：任务调度可视化、时间线分析
- **SystemView**（SEGGER）：实时记录，与 J-Link 集成
- **FreeRTOS+CLI**：命令行调试接口

### 2.7 Rust 绑定

```rust
use freertos_rust::{Task, Duration, CurrentTask};

fn task_fn() {
    loop {
        // 安全的 Rust API，内部封装 FreeRTOS C 调用
        CurrentTask::delay(Duration::ms(100));
    }
}

// 静态分配，编译期确定栈大小
Task::new()
    .name("rust-task")
    .stack_size(512)
    .priority(freertos_rust::TaskPriority(5))
    .start(task_fn)
    .unwrap();

// unsafe 边界：仅在与 C FFI 交互时
unsafe {
    // 直接调用 FreeRTOS C API
    freertos_rs_xTaskNotifyGive(handle);
}

// Send/Sync 保证：FreeRTOS 任务间通信已由 OS 保证线程安全
// 但裸指针共享仍需 unsafe + 临界区保护
```

---

## 3. Zephyr RTOS

### 概述

Apache 2.0 开源 RTOS，Nordic 和 Intel 主导开发。面向物联网和嵌入式 Linux 替代场景。支持 600+ 开发板。

### 核心 API

#### 线程

```c
// Kconfig 启用
# CONFIG_THREADS=y
# CONFIG_NUM_PREEMPT_PRIORITIES=32

// 定义线程栈和 TCB
#define STACK_SIZE 1024
struct k_thread my_thread;
K_THREAD_STACK_DEFINE(my_stack, STACK_SIZE);

// 创建线程
k_thread_create(&my_thread, my_stack, STACK_SIZE,
                my_thread_func, NULL, NULL, NULL,
                K_PRIO_PREEMPT(10), 0, K_NO_WAIT);

// 线程函数
void my_thread_func(void *arg1, void *arg2, void *arg3) {
    while (1) {
        k_msleep(100);
        // ...
    }
}

// 协作式线程（需主动让出 CPU）
k_thread_create(&my_thread, my_stack, STACK_SIZE,
                my_thread_func, NULL, NULL, NULL,
                K_PRIO_COOP(5), 0, K_NO_WAIT);
```

#### 同步原语

```c
// 信号量
struct k_sem my_sem;
k_sem_init(&my_sem, 0, 1);
k_sem_give(&my_sem);
k_sem_take(&my_sem, K_FOREVER);

// 互斥量（支持优先级继承）
struct k_mutex my_mutex;
k_mutex_init(&my_mutex);
k_mutex_lock(&my_mutex, K_FOREVER);
k_mutex_unlock(&my_mutex);

// 条件变量
struct k_condvar cv;
k_condvar_init(&cv);
k_condvar_wait(&cv, &my_mutex, K_FOREVER);
k_condvar_signal(&cv);
```

#### 数据传递

```c
// 消息队列
struct k_msgq my_msgq;
K_MSGQ_DEFINE(my_msgq, sizeof(struct sensor_data), 16, 4);
struct sensor_data data = {.temp = 25};
k_msgq_put(&my_msgq, &data, K_NO_WAIT);
k_msgq_get(&my_msgq, &data, K_FOREVER);

// FIFO
struct k_fifo my_fifo;
k_fifo_init(&my_fifo);
struct k_fifo node;
k_fifo_put(&my_fifo, &node);
k_fifo_get(&my_fifo, K_FOREVER);

// 工作队列（延迟/周期工作）
struct k_work my_work;
void work_handler(struct k_work *work) { /* ... */ }
k_work_init(&my_work, work_handler);
k_work_submit(&my_work);  // 提交到系统工作队列

// 延迟工作
struct k_work_delayable dwork;
k_work_init_delayable(&dwork, work_handler);
k_work_schedule(&dwork, K_MSEC(500));
```

### 设备模型

```c
// 设备树定义（.dts）
// uart0: serial@40002000 {
//     compatible = "nordic,nrf-uart";
//     status = "okay";
// };

// C 代码中使用
const struct device *uart_dev = DEVICE_DT_GET(DT_NODELABEL(uart0));
if (!device_is_ready(uart_dev)) { /* error */ }

// 自定义驱动
static const struct uart_driver_api my_uart_api = {
    .poll_in  = my_uart_poll_in,
    .poll_out = my_uart_poll_out,
};

#define MY_UART_INIT(inst)                           \
    static struct my_uart_data data_##inst;          \
    static const struct my_uart_config config_##inst;\
    DEVICE_DT_INST_DEFINE(inst, my_uart_init,        \
        NULL, &data_##inst, &config_##inst,          \
        POST_KERNEL, CONFIG_UART_INIT_PRIORITY,      \
        &my_uart_api);

DT_INST_FOREACH_STATUS_OKAY(MY_UART_INIT)
```

### 网络栈

```c
// Zephyr 原生网络栈
#include <zephyr/net/net_if.h>
#include <zephyr/net/net_context.h>

// 创建 UDP socket
struct net_context *ctx;
net_context_get(AF_INET, SOCK_DGRAM, IPPROTO_UDP, &ctx);

// Socket API（兼容 POSIX）
#include <zephyr/posix/posix_syssocket.h>
int sock = socket(AF_INET, SOCK_DGRAM, 0);
```

### 特性总结

| 特性 | 说明 |
|------|------|
| Kconfig | 统一配置系统，编译时裁剪 |
| Device Tree | 硬件描述与驱动解耦 |
| 调度 | 抢占式/协作式混合 |
| 内存 slab | 固定大小对象快速分配 |
| TF-M | Trusted Firmware-M，安全启动 + PSA Crypto |
| OpenThread | 6LoWPAN/Thread 无线 mesh 网络 |
| 蓝牙 | 内置 BLE 5.0 全协议栈 |

### 适用场景

Nordic nRF52 系列、Intel Apollo Lake、物联网网关、需要安全认证的消费电子。

---

## 4. RT-Thread

### 概述

国产开源 RTOS（Apache 2.0），组件生态最丰富，国内社区活跃。RT-Thread Studio IDE 提供一站式开发体验。

### 核心 API

```c
#include <rtthread.h>

// 线程（动态创建）
rt_thread_t tid = rt_thread_create(
    "led",
    led_entry,           // 入口函数 void led_entry(void *param)
    RT_NULL,
    512,                 // 栈大小
    10,                  // 优先级
    20                   // 时间片
);
rt_thread_startup(tid);

// 静态创建
ALIGN(RT_ALIGN_SIZE)
static rt_uint8_t led_stack[512];
static struct rt_thread led_thread;
rt_thread_init(&led_thread, "led", led_entry, RT_NULL,
               led_stack, sizeof(led_stack), 10, 20);
rt_thread_startup(&led_thread);
```

#### IPC 机制

```c
// 信号量
rt_sem_t sem = rt_sem_create("sem", 0, RT_IPC_FLAG_FIFO);
rt_sem_release(sem);
rt_sem_take(sem, RT_WAITING_FOREVER);

// 互斥量（支持优先级继承）
rt_mutex_t mutex = rt_mutex_create("mutex", RT_IPC_FLAG_PRIO);
rt_mutex_take(mutex, RT_WAITING_FOREVER);
rt_mutex_release(mutex);

// 事件集
rt_event_t event = rt_event_create("event", RT_IPC_FLAG_FIFO);
rt_event_send(event, (1 << 0) | (1 << 1));
rt_uint32_t recv;
rt_event_recv(event, (1 << 0), RT_EVENT_FLAG_AND | RT_EVENT_FLAG_CLEAR,
              RT_WAITING_FOREVER, &recv);

// 邮箱（4 字节消息，适合指针传递）
rt_mailbox_t mb = rt_mailbox_create("mb", 8);
rt_mailbox_send(mb, (rt_ubase_t)msg_ptr);
rt_mailbox_recv(mb, &ptr, RT_WAITING_FOREVER);

// 消息队列
rt_mq_t mq = rt_mq_create("mq", sizeof(struct msg), 16, RT_IPC_FLAG_FIFO);
rt_mq_send(mq, &msg, sizeof(msg));
rt_mq_recv(mq, &msg, sizeof(msg), RT_WAITING_FOREVER);
```

### FinSH 控制台

```c
// 自动注册命令
#include <rtthread.h>

void hello(int argc, char **argv) {
    rt_kprintf("Hello RT-Thread! arg=%s\n", argc > 1 ? argv[1] : "none");
}
MSH_CMD_EXPORT(hello, say hello - usage: hello [name]);

// FinSH 内置命令
// list_thread    - 列出所有线程
// list_sem       - 列出信号量
// free           - 显示堆内存
// ps             - 进程列表
```

### 设备驱动框架

```c
// 注册 I/O 设备
rt_device_t dev = rt_device_create(RT_Device_Class_Char, 0);
rt_device_register(dev, "uart1", RT_DEVICE_FLAG_RDWR | RT_DEVICE_FLAG_STREAM);

// 打开/读写
rt_device_open(dev, RT_DEVICE_OFLAG_RDWR);
rt_device_write(dev, 0, data, len);
rt_device_read(dev, 0, buf, sizeof(buf));
rt_device_close(dev);
```

### 特性总结

| 特性 | 说明 |
|------|------|
| FinSH | 交互式命令行，支持 C 表达式执行 |
| DFS | 设备文件系统（FAT/LittleFS/RomFS） |
| SAL | 套接字抽象层（AT/Socket 统一接口） |
| 组件 | 网络/文件系统/USB/GuiLite/ulog 等 200+ 软件包 |
| ENV 工具 | 图形化配置 + 包管理 |

### 适用场景

国产芯片平台、需要快速原型开发、物联网终端、教学与科研。

---

## 5. μC/OS-II/III

### 概述

商业 RTOS，Micrium 开发（现归属 Silicon Labs）。以确定性、可裁剪著称，广泛用于工业和汽车领域。μC/OS-III 是 μC/OS-II 的升级版，增加了时间片轮转、信号量删除保护等特性。

### 核心 API

```c
#include "os.h"

// 任务创建（μC/OS-III）
OS_TCB  my_task_tcb;
CPU_STK my_task_stk[256];

void MyTask(void *p_arg) {
    OS_ERR err;
    while (DEF_ON) {
        // 任务逻辑
        OSTimeDlyHMSM(0, 0, 0, 100, OS_OPT_TIME_DLY, &err);
    }
}

void main(void) {
    OS_ERR err;
    OSInit(&err);

    OSTaskCreate(&my_task_tcb,
                 "My Task",
                 MyTask,
                 NULL,
                 10,                     // 优先级
                 &my_task_stk[0],
                 256 / 10,               // 限制区（栈溢出检测）
                 256,
                 0,
                 0,
                 NULL,
                 OS_OPT_TASK_STK_CHK | OS_OPT_TASK_STK_CLR,
                 &err
    );

    OSStart(&err);
}
```

#### 同步与通信

```c
// 信号量
OS_SEM my_sem;
OSSemCreate(&my_sem, "Sem", 0, &err);
OSSemPost(&my_sem, OS_OPT_POST_ALL, &err);
OSSemPend(&my_sem, 0, OS_OPT_PEND_BLOCKING, NULL, &err);

// 互斥量（优先级继承）
OS_MUTEX my_mutex;
OSMutexCreate(&my_mutex, "Mutex", &err);
OSMutexPend(&my_mutex, 0, OS_OPT_PEND_BLOCKING, NULL, &err);
OSMutexPost(&my_mutex, OS_OPT_POST_NONE, &err);

// 消息队列
OS_Q my_q;
OSQCreate(&my_q, "Queue", 16, &err);
OSQPost(&my_q, (void *)msg_ptr, sizeof(*msg_ptr), OS_OPT_POST_FIFO, &err);
OSQPend(&my_q, 0, OS_OPT_PEND_BLOCKING, &msg_size, NULL, &err);

// 事件标志组
OS_FLAG_GRP my_flags;
OSFlagCreate(&my_flags, "Flags", 0, &err);
OSFlagPost(&my_flags, 0x03, OS_OPT_POST_FLAG_SET, &err);
OSFlags = OSFlagPend(&my_flags, 0x03, 0, OS_OPT_PEND_FLAG_SET_ALL, &ts, &err);
```

### 特性总结

| 特性 | μC/OS-II | μC/OS-III |
|------|----------|-----------|
| 最大任务数 | 64（可配 255） | 无限 |
| 时间片轮转 | ❌ | ✅ |
| 优先级继承 | ❌（互斥量无） | ✅ |
| 信号量删除保护 | ❌ | ✅ |
| 调度算法 | 位图就绪表 | 位图 + 就绪列表 |
| 代码质量认证 | DO-178C | DO-178C Level A |

### 适用场景

安全认证要求高的航空航天、医疗设备、汽车电子（Silicon Labs EFM32/EFR32 平台）。

---

## 6. VxWorks

### 概述

Wind River 商业 RTOS，业界标杆，广泛用于航空航天（火星探测器）、工业控制、汽车（AUTOSAR）、网络设备。VxWorks 653 通过 ARINC 653 DO-178C Level A 航空最高安全认证。

### 核心 API

```c
#include <taskLib.h>
#include <semLib.h>
#include <msgQLib.h>

// 任务
TASK_ID tid = taskSpawn("myTask", 100, 0, 8192, myTaskFunc,
                         0, 0, 0, 0, 0, 0, 0, 0, 0, 0);

// 信号量
SEM_ID sem = semBCreate(SEM_Q_PRIORITY, SEM_EMPTY);
semGive(sem);
semTake(sem, WAIT_FOREVER);

// 消息队列
MSG_Q_ID mq = msgQCreate(16, sizeof(Msg), MSG_Q_PRIORITY);
msgQSend(mq, (char *)&msg, sizeof(msg), WAIT_FOREVER, MSG_PRI_NORMAL);
msgQReceive(mq, (char *)&msg, sizeof(msg), WAIT_FOREVER);
```

### 特性总结

| 特性 | 说明 |
|------|------|
| 调度 | 256 级优先级，FIFO/Round-Round |
| 内存保护 | MMU 支持，任务间隔离 |
| VxWorks 653 | ARINC 653 时间/空间分区 |
| 网络 | 完整 IPv4/IPv6/TCP/UDP 栈 |
| 文件系统 | HRFS/DOSFS/TFFS/NFS |
| Wind River Linux | VxWorks + Linux 混合部署 |
| CERT | DO-178C, IEC 61508, ISO 26262 |

### 适用场景

航空电子、航天探测器、工业控制器、汽车 ECU、5G 基站、核心网络设备。

---

## 7. Contiki / TinyOS

### 7.1 Contiki

物联网 OS，以极低内存占用和 Protothreads 协程模型著称。

```c
// Protothreads 示例
#include "contiki.h"
#include <stdio.h>

PROCESS(sensor_process, "Sensor Process");
AUTOSTART_PROCESSES(&sensor_process);

PROCESS_THREAD(sensor_process, ev, data) {
    PROCESS_BEGIN();
    while (1) {
        PROCESS_WAIT_EVENT_UNTIL(ev == sensors_event);
        printf("Sensor: %d\n", *(int *)data);
    }
    PROCESS_END();
}
```

**特性：**
- Protothreads：无栈协程，极低 RAM（每个协程仅几十字节）
- uIP/TCP-IP：最小 TCP/IP 栈（< 10KB ROM）
- 6LoWPAN + RPL：IPv6 低功耗 mesh 网络
- Coffee 文件系统：日志结构 Flash 文件系统
- 平台：Z1, TelosB, CC2538, OpenMote

### 7.2 TinyOS

传感器网络 OS，使用 nesC 组件化语言。

```nesC
// nesC 组件示例
configuration BlinkAppC {}
implementation {
    components MainC, BlinkC, LedsC;
    components new TimerMilliC() as Timer;

    MainC.Boot <- BlinkC.Boot;
    BlinkC.Timer -> Timer;
    BlinkC.Leds -> LedsC;
}
```

**特性：**
- nesC 语言：事件驱动组件模型，编译时静态分析
- Active Message：传感器网络通信抽象
- 平台：TelosB, MicaZ, IRIS
- TinyOS 2.x：支持多平台，TOSSIM 仿真器

### 适用场景

无线传感器网络（WSN）、低功耗环境监测、智能家居传感节点、学术研究。

---

## 8. mbed OS

### 概述

ARM 官方物联网 OS，提供 C++ API，基于 CMSIS-RTOS2 内核（Cortex-M 优化）。已被 ARM 宣布停止维护（2026），但代码和生态仍可使用。

### 核心 API

```cpp
#include "mbed.h"

// 线程
Thread led_thread(osPriorityNormal, 1024);

void led_func() {
    while (true) {
        led = !led;
        ThisThread::sleep_for(500ms);
    }
}

// 互斥量
Mutex stdio_mutex;
{
    LockGuard<Mutex> lock(stdio_mutex);
    printf("Protected output\n");
}

// 队列
Mail<Msg, 16> mail_box;
Msg *msg = mail_box.alloc();
msg->value = 42;
mail_box.put(msg);
osEvent evt = mail_box.get();
Msg *rx = (Msg *)evt.value.p;

// 定时器
Ticker ticker;
ticker.attach(callback(&led_func), 500ms);

// BLE
BLE &ble = BLE::Instance();
ble.init(bleInitComplete);
```

### 特性总结

| 特性 | 说明 |
|------|------|
| API | 现代 C++，易上手 |
| 内核 | CMSIS-RTOS2（RTX5） |
| BLE | 内置蓝牙协议栈 |
| TLS | mbed TLS（加密库） |
| 存储 | BlockDevice 抽象（Flash/SPIF/QSPIF） |
| 云 | mbed Cloud 设备管理 |
| OTA | 固件空中升级 |

### 适用场景

快速原型开发、Cortex-M 设备、BLE 外设、教学。⚠️ 注意：ARM 已停止维护，新项目建议迁移至 Zephyr。

---

## 9. RIOT OS

### 概述

开源物联网 OS，强调标准和可移植性（C11/POSIX/SUIT/CoAP/LwM2M）。

### 核心 API

```c
#include <thread.h>
#include <xtimer.h>
#include <msg.h>

// 线程
char stack[THREAD_STACKSIZE_DEFAULT];
kernel_pid_t pid = thread_create(stack, sizeof(stack),
                                  THREAD_PRIORITY_MAIN - 1,
                                  THREAD_CREATE_STACKTEST,
                                  my_thread_func, NULL, "my_thread");

// 定时器
xtimer_t timer;
xtimer_set(&timer, 1000000);  // 1 秒（微秒单位）

// 消息传递
msg_t msg;
msg_send(&msg, pid);
msg_receive(&msg);

// POSIX 兼容
#include <pthread.h>
#include <semaphore.h>
sem_t sem;
sem_init(&sem, 0, 0);
sem_wait(&sem);
sem_post(&sem);
```

### 特性总结

| 特性 | 说明 |
|------|------|
| 标准 | C11, POSIX 子集, SUIT（固件更新） |
| 网络 | GNRC 网络栈，原生 IPv6, 6LoWPAN, CoAP, LwM2M |
| 平台 | nRF52, STM32, iotlab-m3, ESP32, RISC-V |
| Shell | 内置交互式 shell |
| 调试 | 串口, GDB, OpenOCD |

### 适用场景

物联网研究、学术项目、需要 POSIX 兼容的嵌入式设备、低功耗无线网络。

---

## 10. LwIP（轻量级 TCP/IP 协议栈）

### 概述

Lightweight IP，专为嵌入式设计，RAM < 40KB，ROM < 30KB 即可运行完整 TCP/IP。

### 三种 API 层次

```
┌─────────────────────────┐
│  Socket API             │  ← 最易用，需 RTOS + sys_arch.c
│  (类似 POSIX socket)    │
├─────────────────────────┤
│  Netconn API            │  ← 中间层，基于 RTOS 信号量/邮箱
│  (sequential API)       │
├─────────────────────────┤
│  RAW/callback API       │  ← 无需 OS，回调驱动，最轻量
│  (event-driven)         │
└─────────────────────────┘
```

### RAW API 示例（无 OS）

```c
#include "lwip/tcp.h"

static err_t tcp_recv_cb(void *arg, struct tcp_pcb *tpcb, struct pbuf *p, err_t err) {
    if (p != NULL) {
        tcp_recved(tpcb, p->tot_len);  // 通知已处理
        // 处理数据
        pbuf_free(p);
    } else {
        tcp_close(tpcb);  // 连接关闭
    }
    return ERR_OK;
}

static err_t tcp_accept_cb(void *arg, struct tcp_pcb *newpcb, err_t err) {
    tcp_recv(newpcb, tcp_recv_cb);
    return ERR_OK;
}

void tcp_server_init(void) {
    struct tcp_pcb *pcb = tcp_new();
    tcp_bind(pcb, IP_ADDR_ANY, 80);
    pcb = tcp_listen(pcb);
    tcp_accept(pcb, tcp_accept_cb);
}
```

### Netconn API 示例（需 RTOS）

```c
#include "lwip/sockets.h"  // Socket API

int sock = socket(AF_INET, SOCK_STREAM, 0);
struct sockaddr_in addr = {
    .sin_family = AF_INET,
    .sin_port = htons(80),
    .sin_addr.s_addr = INADDR_ANY
};
bind(sock, (struct sockaddr *)&addr, sizeof(addr));
listen(sock, 5);

while (1) {
    int client = accept(sock, NULL, NULL);
    char buf[256];
    int n = recv(client, buf, sizeof(buf), 0);
    // 处理请求
    close(client);
}
```

### 移植要点

```c
// 网卡驱动接口（netif）
static err_t low_level_init(struct netif *netif) {
    netif->hwaddr_len = ETHARP_HWADDR_LEN;
    netif->hwaddr[0] = 0x02;  // MAC 地址
    netif->mtu = 1500;
    netif->flags = NETIF_FLAG_BROADCAST | NETIF_FLAG_ETHARP;
    return ERR_OK;
}

static err_t low_level_output(struct netif *netif, struct pbuf *p) {
    struct pbuf *q;
    for (q = p; q != NULL; q = q->next) {
        ETH_SendData(q->payload, q->len);  // HAL 发送
    }
    return ERR_OK;
}

// ETH 接收中断中调用
void ETH_IRQHandler(void) {
    struct pbuf *p = pbuf_alloc(PBUF_RAW, rx_len, PBUF_POOL);
    pbuf_take(p, rx_buffer, rx_len);
    netif->input(p, netif);  // 投递到 lwIP
}
```

### 与 FreeRTOS 集成（sys_arch.c）

```c
// sys_arch.c 需实现以下接口：
// sys_sem_new / sys_sem_free / sys_sem_signal / sys_arch_sem_wait
// sys_mutex_new / sys_mutex_free / sys_mutex_lock / sys_mutex_unlock
// sys_mbox_new / sys_mbox_free / sys_mbox_post / sys_arch_mbox_fetch
// sys_new_thread / sys_arch_timeouts

// FreeRTOS 到 lwIP 的桥接示例
sys_sem_t sys_sem_new(u8_t count) {
    SemaphoreHandle_t sem;
    if (count > 0)
        sem = xSemaphoreCreateCounting(255, count);
    else
        sem = xSemaphoreCreateBinary();
    return (sys_sem_t)sem;
}
```

### 内存管理

```c
// 内存池模式（推荐，确定性强）
#define MEM_LIBC_MALLOC  0     // 使用 lwIP 内置内存池
#define MEMP_NUM_TCP_PCB 5    // TCP 控制块数量
#define PBUF_POOL_SIZE   16   // pbuf 池大小

// 堆模式（灵活但有碎片）
#define MEM_LIBC_MALLOC  1     // 使用标准 malloc
// 或
#define MEM_LIBC_MALLOC  0     // 使用 lwIP 内置堆
#define MEM_SIZE         (8 * 1024)  // 8KB 堆
```

---

## 11. RTOS 对比表

| 指标 | FreeRTOS | Zephyr | RT-Thread | μC/OS-III | VxWorks | Contiki | mbed OS | RIOT |
|------|----------|--------|-----------|-----------|---------|---------|---------|------|
| **许可证** | MIT | Apache 2.0 | Apache 2.0 | 商业（有免费版） | 商业 | BSD | Apache 2.0 | LGPL |
| **最大任务数** | 无限 | 无限 | 无限 | 无限 | 无限 | 协程 | 无限 | 无限 |
| **最小 RAM** | ~4 KB | ~10 KB | ~3 KB | ~2 KB | ~20 KB | ~1 KB | ~16 KB | ~4 KB |
| **最小 ROM** | ~8 KB | ~20 KB | ~6 KB | ~6 KB | ~100 KB | ~10 KB | ~40 KB | ~12 KB |
| **调度** | 抢占+时间片 | 抢占+协作 | 抢占+时间片 | 抢占+时间片 | 抢占+FIFO | 协程 | 抢占+时间片 | 抢占+时间片 |
| **网络栈** | LwIP（需集成） | 原生+LwIP | SAL+LwIP | uC/TCP-IP | 完整栈 | uIP/6LoWPAN | 原生 | GNRC |
| **文件系统** | 无（需 FatFS） | FatFS/LittleFS | DFS | 无 | HRFS/DOSFS | Coffee | BlockDevice | VFS |
| **USB** | 无 | ✅ | ✅ | ✅ | ✅ | ❌ | ✅ | ✅ |
| **BLE** | 无 | ✅ | ✅ | ❌ | ❌ | ❌ | ✅ | ✅ |
| **安全认证** | 无 | TF-M | 无 | DO-178C A | DO-178C A / IEC 61508 | 无 | 无 | SUIT |
| **活跃社区** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐（国内） | ⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐ | ⭐⭐ | ⭐⭐⭐ |
| **语言** | C (Rust绑定) | C | C/C++ | C | C/C++ | C (nesC) | C++ | C |
| **POSIX** | ❌ | 部分 | ❌ | ❌ | ✅ | ❌ | 部分 | 部分 |

### 选型建议

```
资源极受限（< 10KB RAM）
  → Contiki / 裸机 FSM

通用嵌入式（Cortex-M, 10-100KB RAM）
  → FreeRTOS（最成熟、生态最大、学习资源多）
  → RT-Thread（国内首选、组件丰富）

需要设备驱动标准化
  → Zephyr（Device Tree + Kconfig、Nordic 生态）

需要安全认证（航空/医疗/汽车）
  → μC/OS-III（DO-178C Level A）
  → VxWorks 653（ARINC 653、行业标杆）

物联网 + 无线
  → Zephyr（BLE + Thread + WiFi）
  → RIOT（GNRC、标准化）
  → Contiki（6LoWPAN、WSN）

快速原型 / 教学
  → mbed OS（C++ 友好）⚠️ 已停止维护
  → RT-Thread（Studio IDE、软件包生态）
```

---

## 附录：通用设计模式

### 中断到任务的经典模式

```c
// 1. ISR 只做最少工作
void UART_IRQHandler(void) {
    BaseType_t woken = pdFALSE;
    BaseType_t higher_prio_woken = pdFALSE;

    // 从 FIFO 读数据
    while (UART->RXNE) {
        char c = UART->DR;
        xStreamBufferSendFromISR(stream_buf, &c, 1, &higher_prio_woken);
    }

    portYIELD_FROM_ISR(higher_prio_woken);
}

// 2. 任务处理数据
void uart_task(void *arg) {
    uint8_t buf[64];
    for (;;) {
        size_t len = xStreamBufferReceive(stream_buf, buf, sizeof(buf), portMAX_DELAY);
        process(buf, len);
    }
}
```

### 低功耗设计要点

1. **Tickless Idle**：FreeRTOS/Zephyr 原生支持，空闲时停 SysTick
2. **外设按需开关**：不用时关 ADC/DMA/UART 时钟
3. **中断唤醒**：配置 GPIO EXTI 或 RTC 作为唤醒源
4. **DMA 替代轮询**：用 DMA 传输后中断唤醒，CPU 保持睡眠
5. **时钟降频**：非计算密集期降低主频

### RTOS 通用调试技巧

```
1. 栈溢出 → 开启栈溢出检测 + Idle Hook 中监控栈水位
2. 死锁 → 记录 mutex 获取顺序，启用优先级继承
3. 优先级反转 → 使用互斥量（非信号量），启用优先级继承
4. 中断延迟 → 量化每个 ISR 执行时间，确保 < 100us
5. 内存泄漏 → 静态分配 + heap_4，定期检查 uxTaskGetStackHighWaterMark
6. 时序抖动 → 使用硬件定时器 + DMA，减少软件参与
```

---

*Last updated: 2026-04-02*
