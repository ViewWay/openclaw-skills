# Software Engineering

## 概念→原理→代码→应用 结构化参考

---

## 1. 软件设计原则

### SOLID

| 原则 | 含义 | 违反示例 | 修正 |
|------|------|---------|------|
| **S** — 单一职责 | 一个类只一个变化理由 | `User` 类同时负责持久化和邮件通知 | 拆分 `UserRepository` + `EmailService` |
| **O** — 开闭原则 | 对扩展开放，对修改关闭 | `switch(type)` 硬编码新增类型 | 策略模式 / 工厂 + 接口 |
| **L** — 里氏替换 | 子类可替换父类而不破坏程序 | 子类覆写抛出 `NotImplementedException` | 重新设计继承层次或用组合 |
| **I** — 接口隔离 | 客户端不应依赖它不用的接口 | 胖接口 `IMachine{Print();Scan();Fax()}` | 拆为 `IPrinter`, `IScanner`, `IFax` |
| **D** — 依赖倒置 | 高层模块不依赖低层模块，都依赖抽象 | 业务层直接 `new MySQLDatabase()` | 注入 `IDatabase` 接口 |

**原理**：高内聚低耦合。每个原则都在降低变更的传播半径。

```python
# DIP 示例：依赖注入
from abc import ABC, abstractmethod

class NotificationSender(ABC):
    @abstractmethod
    def send(self, message: str) -> None: ...

class EmailSender(NotificationSender):
    def send(self, message: str) -> None:
        print(f"Email: {message}")

class NotificationService:
    def __init__(self, sender: NotificationSender):  # 依赖抽象
        self._sender = sender

    def notify(self, msg: str) -> None:
        self._sender.send(msg)
```

### DRY / KISS / YAGNI

- **DRY**（Don't Repeat Yourself）：每一份知识应有单一、明确、权威的表示。违反→抽象为函数/类/模块。
- **KISS**（Keep It Simple, Stupid）：简单方案优于聪明方案。复杂度是敌人。
- **YAGNI**（You Aren't Gonna Need It）：不为未来不确定的需求提前设计。

### 组合优于继承

**原理**：继承是编译期静态绑定，组合是运行期动态绑定。继承层次深→脆弱基类问题。

```python
# 继承方式（脆弱）
class FlyableAnimal(Animal):
    def fly(self): ...
class Bat(Mammal, FlyableAnimal): ...  # 菱形继承问题

# 组合方式（灵活）
class FlyBehavior(ABC):
    @abstractmethod
    def fly(self): ...

class Animal:
    def __init__(self, fly_behavior: FlyBehavior | None = None):
        self._fly = fly_behavior
    def try_fly(self):
        if self._fly: self._fly.fly()
```

### 关注点分离

将系统划分为重叠最小的功能区块。例：MVC 中 Model/View/Controller 各司其职。

---

## 2. 设计模式

### 创建型

**单例（Singleton）**
- 原理：保证一个类仅一个实例，全局访问点
- 注意：线程安全、反序列化破坏、测试困难

```python
from threading import Lock

class Singleton:
    _instance = None
    _lock = Lock()

    def __new__(cls):
        with cls._lock:
            if cls._instance is None:
                cls._instance = super().__new__(cls)
        return cls._instance
```

**工厂方法（Factory Method）**
- 原理：定义创建对象的接口，让子类决定实例化哪个类

```python
from abc import ABC, abstractmethod

class Transport(ABC):
    @abstractmethod
    def deliver(self) -> str: ...

class Truck(Transport):
    def deliver(self) -> str: return "Truck delivery"

class Ship(Transport):
    def deliver(self) -> str: return "Ship delivery"

class Logistics(ABC):
    @abstractmethod
    def create_transport(self) -> Transport: ...

    def plan_delivery(self) -> str:
        t = self.create_transport()
        return f"Planning: {t.deliver()}"

class RoadLogistics(Logistics):
    def create_transport(self) -> Transport: return Truck()
```

**抽象工厂**：工厂的工厂，创建一族相关对象。
**建造者**：分步构建复杂对象，链式调用。适用于构造参数多的场景。
**原型**：通过克隆已有对象创建新对象，避免昂贵的初始化。

### 结构型

| 模式 | 意图 | 典型场景 |
|------|------|---------|
| 适配器 | 接口转换兼容 | 第三方库集成 |
| 桥接 | 分离抽象与实现 | 跨平台 UI 渲染 |
| 组合 | 树形结构统一处理 | 文件系统、UI 组件 |
| 装饰器 | 动态添加职责 | I/O 流包装、中间件 |
| 外观 | 简化子系统接口 | SDK 封装 |
| 享元 | 共享细粒度对象 | 文字编辑器字符对象 |
| 代理 | 控制对象访问 | 延迟加载、权限控制 |

**装饰器示例**：

```python
from functools import wraps

def retry(max_attempts=3, exceptions=(Exception,)):
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            for attempt in range(1, max_attempts + 1):
                try:
                    return func(*args, **kwargs)
                except exceptions as e:
                    if attempt == max_attempts:
                        raise
                    print(f"Attempt {attempt} failed: {e}, retrying...")
        return wrapper
    return decorator

@retry(max_attempts=3, exceptions=(ConnectionError,))
def fetch_data(url):
    ...
```

### 行为型

**策略模式**：运行时切换算法族。**观察者**：一对多依赖通知。**责任链**：请求沿链传递直到被处理。**命令**：将请求封装为对象（支持撤销/队列）。**状态**：对象行为随状态改变。**模板方法**：父类定义算法骨架，子类实现步骤。**中介者**：集中管理对象间交互。**迭代器**：顺序访问集合元素。**访问者**：在不改变类的前提下增加操作。

**观察者示例**：

```python
from collections.abc import Callable

class EventBus:
    def __init__(self):
        self._subscribers: dict[str, list[Callable]] = {}

    def subscribe(self, event: str, handler: Callable):
        self._subscribers.setdefault(event, []).append(handler)

    def publish(self, event: str, data=None):
        for handler in self._subscribers.get(event, []):
            handler(data)

bus = EventBus()
bus.subscribe("user.created", lambda d: print(f"Welcome {d}"))
bus.publish("user.created", "Alice")
```

### 并发模式

- **生产者-消费者**：缓冲区解耦生产与消费速率
- **读者-写者**：读共享、写互斥
- **Future/Promise**：异步计算的占位符
- **Active Object**：将方法调用与执行解耦，自带消息队列

### 架构模式

| 模式 | 核心思想 | 适用场景 |
|------|---------|---------|
| MVC | Model-View-Controller 分离 | Web 应用 |
| MVP | View 通过 Presenter 与 Model 交互 | Android 早期 |
| MVVM | View 与 ViewModel 双向绑定 | WPF/Vue/React |
| Clean | 依赖规则：外层依赖内层 | 大型系统 |
| Hexagonal | 核心逻辑不依赖任何外部技术 | 可移植系统 |
| CQRS | 读写模型分离 | 高读写比系统 |
| Event Sourcing | 以事件序列为数据源 | 审计、回溯 |

---

## 3. 软件架构

### 分层架构

```
表现层 (Presentation) → 应用层 (Application) → 领域层 (Domain) → 基础设施层 (Infrastructure)
```
依赖方向：外→内。领域层零外部依赖。

### 微服务

**优势**：独立部署、技术栈异构、故障隔离、团队自治。
**挑战**：分布式事务、服务发现、网络延迟、数据一致性、运维复杂度。

**服务间通信**：
- 同步：REST/gRPC
- 异步：消息队列（Kafka/RabbitMQ）、事件总线

**数据一致性**：Saga 模式（编排/协调）、最终一致性。

### Serverless

- FaaS（函数即服务）：AWS Lambda / Cloudflare Workers
- 事件驱动、按调用计费、自动扩缩容
- 冷启动延迟、执行时间限制、本地调试困难

### API 设计

**RESTful**：资源 + HTTP 动词 + HATEOAS。适合 CRUD 场景。
**GraphQL**：客户端指定所需数据，单端点。适合复杂查询、多端。
**gRPC**：Protocol Buffers + HTTP/2，强类型 + 流式。适合微服务内部通信。

```protobuf
// gRPC 示例
service OrderService {
  rpc GetOrder(OrderRequest) returns (OrderResponse);
  rpc StreamOrders(StreamRequest) returns (stream OrderResponse);
}
```

### 领域驱动设计（DDD）

- **战略**：限界上下文（Bounded Context）、统一语言（Ubiquitous Language）、上下文映射
- **战术**：实体（Entity）、值对象（Value Object）、聚合（Aggregate）、领域事件、仓储（Repository）、领域服务

### 架构决策记录（ADR）

```markdown
# ADR-001: 选择 PostgreSQL 作为主数据库

## 状态：已接受
## 语境：需要关系型数据库支持复杂查询和事务
## 决策：选择 PostgreSQL 而非 MySQL
## 理由：JSONB 支持、扩展性、PostGIS、社区活跃
## 后果：团队需学习 PostgreSQL 特有功能；部分 ORM 需适配
```

---

## 4. 测试

### 测试金字塔

```
        /  E2E  \          ← 少量，慢，高成本
       / 集成测试  \        ← 适量
      /  单元测试    \      ← 大量，快，低成本
```

### TDD 流程

Red（写失败测试）→ Green（最少代码通过）→ Refactor（重构）

```python
# TDD 示例
import pytest

def fibonacci(n: int) -> int:
    if n <= 1: return n
    return fibonacci(n - 1) + fibonacci(n - 2)

def test_fibonacci():
    assert fibonacci(0) == 0
    assert fibonacci(1) == 1
    assert fibonacci(10) == 55
    with pytest.raises(RecursionError):
        fibonacci(-1)  # 或自定义异常
```

### BDD

用自然语言描述行为，映射到测试。Gherkin 语法：

```gherkin
Feature: 用户登录
  Scenario: 正确凭证登录
    Given 用户在登录页面
    When 输入正确的用户名和密码
    Then 应跳转到首页
```

### Mock / Stub / Spy

```python
from unittest.mock import Mock, patch, call

# Mock：验证交互
notifier = Mock()
service = UserService(notifier)
service.register("Alice")
notifier.send_welcome.assert_called_once_with("Alice")

# Stub：返回预设值
with patch('requests.get') as mock_get:
    mock_get.return_value.json.return_value = {"status": "ok"}
    result = fetch_status()

# Spy：记录调用但不改变行为
spy_list = Spy(wrapping=real_object)
```

### 测试覆盖率

- 语句覆盖率 / 分支覆盖率 / 路径覆盖率
- 工具：`coverage.py`（Python）、`istanbul/nyc`（JS）、`JaCoCo`（Java）
- 原则：覆盖率是手段不是目的，80%+ 通常足够

### 模糊测试（Fuzzing）

随机/半随机输入发现崩溃和安全漏洞。工具：AFL、libFuzzer、Go-fuzz。

### 属性测试（Property-based Testing）

描述属性而非具体例子，框架自动生成测试数据。

```python
from hypothesis import given, strategies as st

@given(st.lists(st.integers()))
def test_sort_idempotent(lst):
    result = sorted(lst)
    assert sorted(result) == result  # 排序幂等性
    assert len(result) == len(lst)   # 长度不变
```

---

## 5. CI/CD

### 流水线

```
代码提交 → 构建 → 单元测试 → 集成测试 → 安全扫描 → 制品构建 → 部署（Staging）→ 验收 → 生产
```

### GitHub Actions 示例

```yaml
name: CI
on: [push, pull_request]
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with: { python-version: '3.12' }
      - run: pip install -r requirements.txt
      - run: pytest --cov=src --cov-fail-under=80
      - run: ruff check .
```

### 容器化

```dockerfile
# 多阶段构建
FROM python:3.12-slim AS builder
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

FROM python:3.12-slim
COPY --from=builder /usr/local/lib/python3.12/site-packages /usr/local/lib/python3.12/site-packages
COPY . .
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

### Kubernetes 核心概念

Pod → Deployment → Service → Ingress → ConfigMap/Secret

### 部署策略

| 策略 | 原理 | 回滚 |
|------|------|------|
| 蓝绿 | 两套环境切换 DNS/负载均衡 | 切回旧环境 |
| 金丝雀 | 小流量→新版本，逐步扩大 | 停止引流 |
| 滚动更新 | 逐个替换实例 | `kubectl rollout undo` |

### 特性开关（Feature Flags）

```python
flags = {
    "new_checkout": True,
    "dark_mode": lambda user: user.beta_tester,
}

if flags["new_checkout"]:
    render_new_checkout()
```

---

## 6. 代码质量

### 代码审查要点

1. 正确性：逻辑是否正确？
2. 可读性：6 个月后还能看懂吗？
3. 设计：是否遵循 SOLID？
4. 测试：是否有充分测试？
5. 性能：是否有明显瓶颈？
6. 安全：是否有注入/泄露风险？

### 静态分析

- Python: `ruff`, `mypy`, `pylint`
- JS/TS: `eslint`, `prettier`, `tsc --noEmit`
- Java: `SonarQube`, `SpotBugs`, `Checkstyle`
- 通用: `SonarQube`（技术债务量化、代码异味检测）

### 重构手法

- **提取方法**：将代码片段抽取为独立方法
- **内联方法**：方法体简单时直接展开
- **重命名**：改善命名表达力
- **移动方法/字段**：调整归属
- **提取类**：一个类职责过多时拆分
- **以多态替代条件**：消除 `switch/type` 检查

### 代码度量

- **圈复杂度**（Cyclomatic Complexity）：独立路径数，>10 需重构
- **耦合度**：模块间依赖程度
- **内聚度**：模块内部元素关联程度（LCOM）

---

## 7. 版本控制

### Git Flow vs Trunk-based

| | Git Flow | Trunk-based |
|--|---------|-------------|
| 主分支 | main + develop + feature/* + release/* + hotfix/* | main + 短命 feature branch |
| 适用 | 发布周期长 | 持续部署 |
| 合并 | release 分支合回 main + develop | PR 直合 main |
| 复杂度 | 高 | 低 |

### 常用操作

```bash
# 交互式变基，整理提交历史
git rebase -i HEAD~5

# Cherry-pick 特定提交
git cherry-pick abc123

# 子模块
git submodule add https://github.com/org/lib.git vendor/lib
git submodule update --init --recursive

# 子树（替代 submodule）
git subtree add --prefix=vendor/lib https://github.com/org/lib.git main --squash
```

---

## 8. 项目管理

### 敏捷框架

**Scrum**：Sprint（1-4周）→ Sprint Planning → Daily Standup → Sprint Review → Retrospective
**Kanban**：可视化看板，WIP 限制，持续流

### 估算

- 故事点（Story Points）：斐波那契数列（1, 2, 3, 5, 8, 13）
- Planning Poker：团队共识式估算
- 速度（Velocity）：每 Sprint 完成的故事点

### 需求分析

- 用户故事（User Story）：作为[角色]，我想要[功能]，以便[价值]
- 验收标准（Acceptance Criteria）：Given-When-Then
- INVEST 原则：Independent, Negotiable, Valuable, Estimable, Small, Testable
