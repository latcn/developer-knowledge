# Python 高级编程实践指南

> 基于 `nanobot v0.3.0` 源码提炼，涵盖类型系统、异步编程、设计模式与工程化最佳实践

---

## 前言

本指南面向有 Python 基础、希望进阶的开发者。所有知识点均从真实项目代码中提取，兼顾**原理深度**与**工程实用**。建议按顺序学习，并手写每个示例。

---

## 第一章：类型系统 —— 让代码自文档化

### 1.1 内置泛型（PEP 585）

Python 3.9+ 允许直接使用 `list`、`dict`、`set` 等内置类型作为泛型容器。

```python
# 旧写法（需从 typing 导入）
from typing import List, Dict, Set, Tuple
names: List[str] = []
config: Dict[str, int] = {}

# 新写法（推荐）
names: list[str] = []
config: dict[str, int] = {}
active_tasks: dict[str, set[asyncio.Task[Any]]] = {}
```

**原理**：`list` 实现了 `__class_getitem__`，使其支持下标操作，而不需继承 `Generic`。

**最佳实践**：
- 所有容器类型都使用内置泛型，无需 `typing.List` 等（除非需兼容 Python 3.8）。
- 嵌套泛型要完整写出，如 `dict[str, list[tuple[int, str]]]`。

### 1.2 联合类型（PEP 604）

用 `|` 替代 `Union` 和 `Optional`。

```python
# 旧写法
from typing import Optional, Union
def foo(a: Union[int, str]) -> Optional[str]: ...

# 新写法（Python 3.10+）
def foo(a: int | str) -> str | None: ...
```

**注意**：`isinstance(x, int | str)` 在 3.10+ 也合法。

### 1.3 字面量类型（PEP 586）

`Literal` 限制值只能为指定的字面量。

```python
from typing import Literal

def set_log_level(level: Literal["DEBUG", "INFO", "ERROR"]) -> None:
    print(f"Level: {level}")

set_log_level("INFO")   # OK
set_log_level("WARN")   # mypy 报错
```

**适用场景**：参数仅限少数固定字符串，且不需要运行时可遍历集合时，比 `Enum` 轻量。

### 1.4 TypedDict —— 给字典定义结构

```python
from typing import TypedDict

class UserPayload(TypedDict):
    id: int
    name: str
    email: str | None

def process_user(data: UserPayload) -> None:
    print(data["name"])  # IDE 补全

# 运行时仍然是普通 dict
```

**易错点**：`TypedDict` 不强制键存在，`total=False` 让所有键变为可选；推荐用 `NotRequired`（Python 3.11+）更精细控制。

### 1.5 TypeGuard —— 自定义类型收窄

```python
from typing import TypeGuard

def is_str_list(obj: object) -> TypeGuard[list[str]]:
    return isinstance(obj, list) and all(isinstance(x, str) for x in obj)

def process(data: object) -> None:
    if is_str_list(data):
        # 此处 data 被收窄为 list[str]
        print(data[0].upper())
    else:
        print("Not a list of strings")
```

**核心**：`TypeGuard` 不改变返回值类型（永远是 `bool`），只告诉类型检查器“当返回 True 时，参数类型为 T”。

---

## 第二章：数据类（dataclass）

### 2.1 基本用法与可变默认值陷阱

```python
from dataclasses import dataclass, field
from datetime import datetime

@dataclass
class InboundMessage:
    channel: str
    sender_id: str
    content: str
    timestamp: datetime = field(default_factory=datetime.now)
    media: list[str] = field(default_factory=list)
    metadata: dict[str, any] = field(default_factory=dict)
```

- **`default_factory`**：每次实例化调用工厂函数创建新对象，避免共享可变对象。
- **`datetime.now`** 不加括号，表示传入函数本身。

**错误写法**：`timestamp: datetime = datetime.now()` 会使得所有实例共享同一时刻。

### 2.2 不可变数据类（frozen=True）

```python
@dataclass(frozen=True, slots=True)
class LLMRuntime:
    model: str
    context_window: int
    temperature: float = 0.7
```

- `frozen=True`：实例不可修改，`__setattr__` 抛出异常，适合作为配置快照。
- `slots=True`（Python 3.10+）：去掉 `__dict__`，省内存、加速属性访问。

### 2.3 `__post_init__` —— 后处理与校验

```python
@dataclass
class Session:
    messages: list[dict]
    last_consolidated: int = 0

    def __post_init__(self) -> None:
        # 修复类型：bool 是 int 的子类，先排除
        if isinstance(self.last_consolidated, bool):
            self.last_consolidated = 0
        elif not isinstance(self.last_consolidated, int):
            self.last_consolidated = 0
        elif not 0 <= self.last_consolidated <= len(self.messages):
            self.last_consolidated = 0
```

**注意**：`bool` 是 `int` 的子类，单独检查避免 `True` 被当作索引。

### 2.4 修改不可变对象：`dataclasses.replace`

```python
from dataclasses import replace

new_runtime = replace(old_runtime, temperature=0.9)
```

返回新实例，旧对象不变，适合不可变数据更新。

---

## 第三章：抽象基类（ABC）与协议（Protocol）

### 3.1 ABC —— 名义子类型

```python
from abc import ABC, abstractmethod

class BaseChannel(ABC):
    @abstractmethod
    async def start(self) -> None: ...

    @abstractmethod
    async def stop(self) -> None: ...

    def is_allowed(self, sender: str) -> bool:  # 具体方法
        return True
```

- 子类必须实现所有 `@abstractmethod`，否则无法实例化。
- 基类可含具体方法，构成模板方法。

### 3.2 Protocol —— 结构子类型（鸭子类型）

```python
from typing import Protocol, runtime_checkable

@runtime_checkable
class ContextAware(Protocol):
    def set_context(self, ctx: object) -> None: ...

# 任何有 set_context 方法的类，无需继承，即满足 ContextAware
class Tool:
    def set_context(self, ctx: object) -> None: ...

def setup(tool: ContextAware) -> None:
    tool.set_context(...)

setup(Tool())  # OK
```

- `Protocol` 只存在于类型检查期，不强制继承。
- `@runtime_checkable` 允许 `isinstance(obj, ContextAware)` 运行时检查（仅检查方法存在，不检查签名）。

---

## 第四章：异步编程核心

### 4.1 事件循环与协程

```python
import asyncio

async def say_after(delay, word):
    await asyncio.sleep(delay)
    print(word)

async def main():
    await asyncio.gather(
        say_after(1, "hello"),
        say_after(2, "world")
    )

asyncio.run(main())
```

- `await` 挂起协程，让出控制权给事件循环。
- **永远不要**在协程中使用 `time.sleep()`，必须用 `asyncio.sleep()`。

### 4.2 任务管理三兄弟

| 函数 | 作用 | 返回 |
|------|------|------|
| `asyncio.create_task(coro)` | 调度协程并发执行 | `Task` 对象 |
| `asyncio.gather(*tasks)` | 等待所有任务完成 | 结果列表 |
| `asyncio.wait_for(coro, timeout)` | 限时等待，超时取消 | 结果或超时异常 |

**示例**：

```python
task = asyncio.create_task(long_operation())
# 后台运行，不等

try:
    result = await asyncio.wait_for(fetch_data(), timeout=5.0)
except asyncio.TimeoutError:
    print("Timeout")
```

**注意**：`wait_for` 超时后会取消任务，任务内需处理 `CancelledError`。

### 4.3 并发控制：Lock 与 Semaphore

```python
lock = asyncio.Lock()
sem = asyncio.Semaphore(3)  # 最多 3 个并发

async with lock:
    # 临界区，同一时刻仅一个协程进入
    await ...

async with sem:
    # 限制并发数
    await ...
```

**每会话串行 + 全局并发门**（项目模式）：

```python
# 同一会话排队（会话锁），不同会话并行（全局 Semaphore）
async with session_lock, concurrency_gate:
    await process_message()
```

### 4.4 生产者-消费者：`asyncio.Queue`

```python
queue: asyncio.Queue[int] = asyncio.Queue(maxsize=10)

async def producer():
    for i in range(20):
        await queue.put(i)   # 队列满时挂起

async def consumer():
    while True:
        item = await queue.get()   # 队列空时挂起
        process(item)
        queue.task_done()
```

- `maxsize` 控制背压，防止内存溢出。
- `put_nowait` 和 `get_nowait` 用于非阻塞，需捕获 `asyncio.QueueFull` / `asyncio.QueueEmpty`。

### 4.5 事件驱动关停：`asyncio.Event`

```python
shutdown_event = asyncio.Event()

def signal_handler():
    shutdown_event.set()

async def main():
    loop = asyncio.get_running_loop()
    loop.add_signal_handler(signal.SIGINT, signal_handler)

    worker = asyncio.create_task(work())
    shutdown_task = asyncio.create_task(shutdown_event.wait())

    done, pending = await asyncio.wait(
        {worker, shutdown_task},
        return_when=asyncio.FIRST_COMPLETED
    )
    if shutdown_task in done:
        worker.cancel()
        await asyncio.gather(*pending, return_exceptions=True)
```

- `Event` 的 `wait()` 挂起直到 `set()` 被调用。
- 取消任务后必须 `await` 等待其真正结束（清理资源）。

### 4.6 异步上下文管理器：`AsyncExitStack`

用于动态管理多个异步资源（数量运行时决定）。

```python
from contextlib import AsyncExitStack

async with AsyncExitStack() as stack:
    conns = []
    for url in urls:
        conn = await stack.enter_async_context(connect(url))
        conns.append(conn)
    # 所有连接在此保持打开，退出栈时自动关闭（LIFO）
```

### 4.7 超时与取消的正确姿势

协程内必须响应取消：

```python
async def worker():
    try:
        while True:
            await asyncio.sleep(1)
    except asyncio.CancelledError:
        print("Cleaning up...")
        # 释放资源
        raise  # 必须重新抛出，标记任务已取消
```

**检查取消状态**（3.11+）：`asyncio.current_task().cancelling()` 返回取消请求深度。

### 4.8 线程桥接：`asyncio.to_thread`

同步阻塞函数（如 `requests.get`、`imaplib`）必须扔线程池，避免卡死事件循环。

```python
data = await asyncio.to_thread(requests.get, "https://api.example.com")
```

**注意**：线程内无法访问 `ContextVar`（任务局部变量），需显式传参。

---

## 第五章：设计模式实战

### 5.1 注册表模式（插件发现与加载）

通道、工具、Provider 均采用“发现（元数据扫描）与加载（惰性 import）”分离。

**通道 manifest**：

```python
# nanobot/channels/telegram/manifest.py
from nanobot.channels.plugin import ChannelPlugin

PLUGIN = ChannelPlugin(
    name="telegram",
    runtime="nanobot.channels.telegram.runtime:TelegramChannel",
    dependencies=["python-telegram-bot>=22.0"],
)
```

**加载器**：

```python
def load_channel_class(plugin: ChannelPlugin) -> type[BaseChannel]:
    module_name, _, attr = plugin.runtime.partition(":")
    module = importlib.import_module(module_name)
    cls = getattr(module, attr)
    if not issubclass(cls, BaseChannel):
        raise TypeError(...)
    return cls
```

**优点**：新增通道只需新建包和 manifest，核心代码零改动。

### 5.2 模板方法 —— 统一流水线

```python
class BaseChannel(ABC):
    async def _handle_message(self, raw: dict) -> None:
        if not self.is_allowed(raw["sender"]):
            return
        msg = self._build_inbound(raw)
        await self.bus.publish_inbound(msg)

    @abstractmethod
    def _build_inbound(self, raw: dict) -> InboundMessage: ...
```

子类只需实现 `_build_inbound`，鉴权和发布由基类固定。

### 5.3 适配器模式 —— 包装外部接口

```python
class MCPToolWrapper(Tool):
    def __init__(self, session, tool_def):
        self._session = session
        self._tool_def = tool_def

    async def execute(self, **kwargs) -> str:
        return await self._session.call_tool(self._tool_def.name, kwargs)
```

将 MCP 工具适配为内部 `Tool`，核心 `AgentLoop` 不感知 MCP 协议。

---

## 第六章：数据持久化与安全

### 6.1 JSONL —— 追加日志格式

```python
# 写
with open(path, "a", encoding="utf-8") as f:
    f.write(json.dumps(obj, ensure_ascii=False) + "\n")

# 读（容错）
for line in open(path, encoding="utf-8"):
    try:
        obj = json.loads(line)
    except json.JSONDecodeError:
        continue   # 坏行跳过
```

**优点**：每行独立，追加高效，损坏不影响其他行。

### 6.2 原子写入（防半写）

```python
import os, secrets
from pathlib import Path

def atomic_write(path: Path, data: bytes) -> None:
    tmp = path.with_name(f".{path.name}.{secrets.token_hex(8)}.tmp")
    with open(tmp, "xb") as f:   # "x" 排他创建
        f.write(data)
        f.flush()
        os.fsync(f.fileno())
    os.replace(tmp, path)        # 原子重命名
    # 可选：fsync 目录
```

- `os.replace` 在 POSIX 下是原子操作（同一文件系统内）。
- 临时文件与目标同目录，避免跨文件系统失败。

### 6.3 路径安全（防止穿越）

```python
def secure_path(user_input: str, base: Path) -> Path:
    resolved = (base / user_input).resolve()
    if not resolved.is_relative_to(base):
        raise ValueError("Path traversal detected")
    return resolved
```

- 必须先 `resolve()` 解析 `..` 和符号链接。
- 校验前缀必须用 `is_relative_to`（Python 3.9+）或 `commonpath` 比较。

### 6.4 SSRF 防护

- 解析 URL 提取 hostname。
- DNS 解析出所有 IP。
- 检查每个 IP 是否在私有网络、环回、链路本地、云元数据网段中。
- 对重定向进行迭代校验，限制跳转次数。

---

## 第七章：工程化与测试

### 7.1 `pyproject.toml` 配置要点

```toml
[project]
name = "myapp"
version = "0.1.0"
requires-python = ">=3.11"
dependencies = ["aiohttp>=3.9.0,<4.0.0"]

[project.optional-dependencies]
dev = ["pytest>=8.0.0", "ruff>=0.4.0", "basedpyright>=1.20.0"]

[project.scripts]
mycli = "myapp.cli:main"

[tool.ruff.lint]
select = ["E", "F", "I", "W", "B"]
ignore = ["E501"]

[tool.basedpyright]
include = ["myapp"]
typeCheckingMode = "strict"

[tool.pytest.ini_options]
asyncio_mode = "auto"
testpaths = ["tests"]
addopts = "--cov=myapp"

[tool.coverage.report]
fail_under = 80
```

### 7.2 可测试性设计 —— 依赖注入

将文件、网络、配置等外部依赖通过构造函数注入，测试时用 Mock 替换。

```python
class GatewayApp:
    def __init__(self, config_loader, http_client, runtime_factory):
        self._loader = config_loader
        self._client = http_client
        self._factory = runtime_factory

# 测试中
def test_gateway():
    mock_loader = lambda: {"key": "value"}
    app = GatewayApp(mock_loader, MockClient(), MockFactory())
    # ...
```

### 7.3 异步测试（pytest-asyncio）

```python
import pytest

@pytest.mark.asyncio
async def test_async_function():
    result = await my_async_func()
    assert result == expected
```

需在 `pytest.ini` 设置 `asyncio_mode = auto`，自动识别 `async` 测试函数。

### 7.4 覆盖率门槛

`fail_under = 80` 确保低于 80% 时 CI 失败，推动测试覆盖。

---

## 第八章：常见反模式与教训

### 8.1 可变默认参数

```python
# ❌ 错误
def append_to(item, target=[]):
    target.append(item)
    return target

# ✅ 正确
def append_to(item, target=None):
    if target is None:
        target = []
    target.append(item)
    return target
```

### 8.2 在 asyncio 中使用同步阻塞

```python
# ❌ 卡死事件循环
data = requests.get("https://...").json()

# ✅ 使用线程池
data = await asyncio.to_thread(requests.get, "https://...").json()
```

### 8.3 忽略 `CancelledError`

```python
try:
    await long_task()
except Exception:
    pass   # 会吞掉 CancelledError，导致任务无法正确取消
```

正确做法是捕获 `asyncio.CancelledError` 并重新抛出或处理。

### 8.4 单线程不等于无竞态

`await` 点是协程切换点，共享状态的“读-改-写”需加锁。

```python
# 竞态代码
async def update():
    val = shared["count"]
    await asyncio.sleep(0)   # 切换点
    shared["count"] = val + 1

# 正确：加锁
async with lock:
    val = shared["count"]
    await asyncio.sleep(0)
    shared["count"] = val + 1
```

---

## 第九章：综合练习 —— 构建简易消息队列

**要求**：
- 支持生产者放入消息，消费者取出，可设置并发数。
- 支持优雅关停（收到信号后不再接收新消息，处理完已有消息后退出）。
- 使用类型注解，并通过 mypy strict 模式。

**参考实现**：

```python
import asyncio
import signal
from typing import Any, Callable

class AsyncQueue:
    def __init__(self, maxsize: int = 0):
        self._queue = asyncio.Queue(maxsize)
        self._closed = False
        self._shutdown = asyncio.Event()

    async def put(self, item: Any) -> None:
        if self._closed:
            raise RuntimeError("Queue closed")
        await self._queue.put(item)

    async def get(self) -> Any:
        return await self._queue.get()

    def close(self) -> None:
        self._closed = True
        self._shutdown.set()

    async def wait_drain(self) -> None:
        await self._queue.join()

async def worker(queue: AsyncQueue, consumer: Callable) -> None:
    while True:
        try:
            item = await asyncio.wait_for(queue.get(), timeout=1.0)
        except asyncio.TimeoutError:
            if queue._closed and queue._queue.empty():
                break
            continue
        await consumer(item)
        queue._queue.task_done()

async def main():
    queue = AsyncQueue(maxsize=10)
    workers = [asyncio.create_task(worker(queue, lambda x: asyncio.sleep(0.1))) for _ in range(3)]

    loop = asyncio.get_running_loop()
    loop.add_signal_handler(signal.SIGINT, queue.close)
    loop.add_signal_handler(signal.SIGTERM, queue.close)

    # 模拟生产者
    for i in range(20):
        await queue.put(i)
        await asyncio.sleep(0.2)
    queue.close()

    await queue.wait_drain()
    await asyncio.gather(*workers, return_exceptions=True)

asyncio.run(main())
```

---

## 结语

本指南涵盖 Python 中高级编程的主要支柱。建议在掌握理论后，深入阅读 `nanobot` 源码，尤其是：
- `agent/loop.py`（异步调度与状态管理）
- `bus/runtime_events.py`（观察者模式）
- `config/schema.py`（Pydantic 配置）
- `security/network.py`（SSRF 防护）

持续实践、阅读、反思，是成为 Python 专家的必经之路。祝你学习愉快！