# 🐈 基于 nanobot 项目的 Python 学习指南

> 本指南以 nanobot 项目（`F:\ai-agent\nanobot\nanobot`）的真实代码为教材，逐知识点讲解
> **"代码长什么样 → 为什么这么写 → 最佳实践是什么"**。
> 阅读时建议同时打开对应源码文件，边看边读。

---

## 目录

- [第一部分 项目速览与学习地图](#第一部分-项目速览与学习地图)
- [第二部分 语言基础](#第二部分-语言基础)
  - [1. 类型注解体系（PEP 484 / 585 / 604）](#1-类型注解体系)
  - [1.5 泛型与类型参数（Generics）](#15-泛型与类型参数generics--queueinboundmessage-是什么)
  - [2. dataclass 数据类](#2-dataclass-数据类)
  - [3. ABC 抽象基类与多态](#3-abc-抽象基类与多态)
  - [4. property 属性](#4-property-属性)
  - [5. 异常处理](#5-异常处理)
  - [6. f-string 与字符串处理](#6-f-string-与字符串处理)
  - [7. Pathlib 文件系统操作](#7-pathlib-文件系统操作)
  - [8. 模块与包、`__main__.py`](#8-模块与包)
  - [9. 函数进阶：默认参数与可变对象陷阱](#9-函数进阶)
  - [10. global 全局变量](#10-global-全局变量)
- [第三部分 面向对象与设计模式](#第三部分-面向对象与设计模式)
- [第四部分 异步编程 asyncio（重点）](#第四部分-异步编程-asyncio重点)
- [第五部分 数据、序列化与配置](#第五部分-数据序列化与配置)
- [第六部分 系统编程、正则与安全](#第六部分-系统编程正则与安全)
- [第七部分 第三方生态实战](#第七部分-第三方生态实战)
- [第八部分 工程化与测试](#第八部分-工程化与测试)
- [第九部分 实战教训与反模式](#第九部分-实战教训与反模式)
- [第十部分 学习路径建议](#第十部分-学习路径建议)

---

## 第一部分 项目速览与学习地图

nanobot 是一个约 4,000 行的异步 AI 聊天机器人框架，**几乎涵盖了 Python 中级到进阶的全部知识点**：

| 技术领域 | 涉及模块 |
|---|---|
| 类型注解、dataclass、ABC | `bus/events.py`、`providers/base.py`、`channels/base.py` |
| asyncio 全家族 | `bus/queue.py`、`agent/loop.py`、`heartbeat/service.py`、`cron/service.py` |
| 面向对象与设计模式 | `providers/registry.py`、`agent/tools/registry.py`、`channels/*` |
| 数据序列化 | `session/manager.py`（JSONL）、`cron/service.py`（JSON）、`config/loader.py` |
| Pydantic 配置 | `config/schema.py` |
| 正则、subprocess、时间 | `agent/tools/shell.py`、`channels/telegram.py`、`cron/service.py` |
| 安全编程 | `agent/tools/shell.py`、`agent/tools/web.py`、`channels/base.py` |
| 第三方生态 | typer、rich、loguru、httpx、prompt_toolkit、litellm |
| 工程化 | `pyproject.toml`、惰性导入、测试 |

**建议学习顺序**：第二部分（语言基础）→ 第三部分（OOP/模式）→ 第四部分（asyncio，本项目最值得学）→ 第五~八部分 → 第九部分（吸取教训）→ 第十部分（路线图）。

---

## 第二部分 语言基础

### 1. 类型注解体系

Python 从 3.5 起逐步支持类型注解，本项目用了三套现代语法：

**① PEP 585 —— 内置泛型（Python 3.9+）**

```python
# nanobot/bus/queue.py:22
self._outbound_subscribers: dict[str, list[Callable[[OutboundMessage], Awaitable[None]]]] = {}
```

> 解析：旧写法是 `Dict[str, List[...]]`（来自 `typing` 模块），3.9 起可以直接用 `dict[str, ...]`、`list[str]`、`tuple[int, ...]`。更简洁，且 `typing.Dict/List` 已废弃。

**② PEP 604 —— 联合类型 `|`（Python 3.10+）**

```python
# nanobot/channels/base.py:91
media: list[str] | None = None
# nanobot/agent/loop.py:150
def _strip_think(text: str | None) -> str | None:
```

> 解析：`str | None` 等价于旧的 `Optional[str]`；`A | B` 等价于 `Union[A, B]`。注意 `|` 语法需要 Python 3.10+（项目要求 ≥3.11）。

**③ 函数类型注解**

```python
# nanobot/bus/queue.py:41
def subscribe_outbound(
    self,
    channel: str,
    callback: Callable[[OutboundMessage], Awaitable[None]]
) -> None:
# nanobot/cron/service.py:54
on_job: Callable[[CronJob], Coroutine[Any, Any, str | None]] | None = None
```

> 解析：
> - `Callable[[参数类型], 返回类型]` 描述"可调用对象"（函数/方法/lambda/类实例）。
> - `Awaitable[None]` 表示"可 await 的对象"（协程、Task、Future）。
> - `Coroutine[Any, Any, T]` 更精确地描述"协程，返回 T"。
> - 用类型标注把"回调函数"这个约定写进签名，比注释可靠得多。

**④ `Any` 与泛型参数**

```python
# nanobot/bus/events.py:18
metadata: dict[str, Any] = field(default_factory=dict)
# nanobot/agent/tools/base.py:43
async def execute(self, **kwargs: Any) -> str:
```

> 解析：`Any` 表示"任意类型"，是类型检查器的逃生舱。工具参数天然是动态的（LLM 传来的 JSON），所以用 `Any` 合理；但**能用具体类型就不用 Any**。

**最佳实践**
- 所有函数签名都写类型注解（本项目 100% 覆盖），让 IDE 有补全、让 mypy/pyright 能检查。
- 用 `| None` 表达"可空"，用 `list[str]` 而非 `List[str]`。
- 回调签名写全：`Callable[[InboundMessage], Awaitable[None]]` 比裸 `callback` 强一百倍。
- 进阶：给返回值也标注，尤其是 `-> tuple[str | None, list[str]]`（`agent/loop.py:171`）。

---

### 1.5 泛型与类型参数（Generics）—— `Queue[InboundMessage]` 是什么

**① Python 泛型是"两个世界"**：与 Java/C# 不同，Python 的类型参数**运行时完全不检查**，只是给静态检查器（mypy/pyright）和 IDE 看的元数据。写错类型时程序照常运行，只有跑 mypy 才报错。

```python
# nanobot/bus/queue.py:20 —— 泛型用法：Queue[InboundMessage] = "装 InboundMessage 的队列"
self.inbound: asyncio.Queue[InboundMessage] = asyncio.Queue()
```

**解析**
- `InboundMessage` 是自定义 dataclass，任何类都能当类型参数，**不需要它本身是泛型**。
- `asyncio.Queue` 内部定义为 `class Queue(Generic[_T])`，所以能用 `[X]` 下标生成"参数化别名"（运行时是 `types.GenericAlias` 对象，仅作注解、随即丢弃）。
- 真正的好处：mypy 会推断 `queue.get()` 返回 `InboundMessage`，补全、错误提示全部生效；但往队列塞字符串**运行时不会报错**——想要运行时校验得自己写（如自定义 `TypedQueue` 子类做 `isinstance` 检查）。

**② 自定义泛型类（TypeVar + Generic）**

```python
from typing import Generic, TypeVar

T = TypeVar("T", bound=BaseChannel)   # 约束：T 必须是 BaseChannel 或其子类

class ChannelPool(Generic[T]):
    def __init__(self) -> None:
        self._items: list[T] = []
    def add(self, item: T) -> None:
        self._items.append(item)
    def get(self) -> T:
        return self._items.pop(0)

pool: ChannelPool[TelegramChannel] = ChannelPool()   # 与 Queue[X] 同款写法
```

**③ 项目里的嵌套泛型（从外往里读）**

```python
# nanobot/bus/queue.py:22 —— 键为 str、值为"回调函数列表"的字典
self._outbound_subscribers: dict[str, list[Callable[[OutboundMessage], Awaitable[None]]]] = {}

# nanobot/providers/registry.py:39 —— 元素为 (str, str) 二元组的元组
env_extras: tuple[tuple[str, str], ...] = ()
```

**④ 两个常见坑**
- `typing.List[X]`/`typing.Dict[X, Y]` 在 3.9+ 已被 `list[X]`/`dict[X, Y]` 取代（PEP 585），新代码一律用后者。
- 普通类不能下标：`SomeClass[int]` 会抛 `TypeError: type object is not subscriptable`，除非它继承 `Generic` 或实现了 `__class_getitem__`。

---

### 2. dataclass 数据类
dataclass 是"数据容器"的标准答案：自动生成 `__init__`、`__repr__`、`__eq__`。

```python
# nanobot/bus/events.py:8
@dataclass
class InboundMessage:
    channel: str                      # telegram, discord, slack, whatsapp
    sender_id: str
    chat_id: str
    content: str
    timestamp: datetime = field(default_factory=datetime.now)
    media: list[str] = field(default_factory=list)
    metadata: dict[str, Any] = field(default_factory=dict)

    @property
    def session_key(self) -> str:
        return f"{self.channel}:{self.chat_id}"
```

**解析**
- `field(default_factory=datetime.now)`：**可变默认值的正确写法**。直接写 `= []` 或 `= datetime.now()` 是经典陷阱（见下文第 9 节）。
- `default_factory=datetime.now` 注意**不写括号**——传的是"工厂函数"，每次实例化时调用它生成新值；写 `datetime.now()` 则所有实例共享同一个创建时间。
- 无 `order=True` 时 dataclass 不可比较排序；需要排序加 `@dataclass(order=True)`。

**更多 dataclass 用法**

```python
# nanobot/cron/types.py:8
@dataclass
class CronSchedule:
    kind: Literal["at", "every", "cron"]
    at_ms: int | None = None
    ...
```

```python
# nanobot/providers/registry.py:20
@dataclass(frozen=True)
class ProviderSpec:
    """frozen=True → 实例不可变，安全作为全局常量表"""
```

```python
# nanobot/session/manager.py:15
@dataclass
class Session:
    key: str
    messages: list[dict[str, Any]] = field(default_factory=list)
    last_consolidated: int = 0
```

**最佳实践**
- 只装数据的类用 `@dataclass`，别手写 `__init__`。
- 所有 `list`/`dict`/`set` 默认值必须 `field(default_factory=...)`。
- 不可变配置/常量表用 `@dataclass(frozen=True)`（如 `ProviderSpec`）。
- 需要给 dataclass 增加"派生字段"时用 `@property`（如 `session_key`），保持数据单一来源。

---

### 3. ABC 抽象基类与多态

抽象基类（Abstract Base Class）定义"接口契约"：子类必须实现哪些方法。

```python
# nanobot/channels/base.py:12
class BaseChannel(ABC):
    name: str = "base"

    def __init__(self, config: Any, bus: MessageBus):
        self.config = config
        self.bus = bus
        self._running = False

    @abstractmethod
    async def start(self) -> None: ...

    @abstractmethod
    async def stop(self) -> None: ...

    @abstractmethod
    async def send(self, msg: OutboundMessage) -> None: ...

    def is_allowed(self, sender_id: str) -> bool:      # 公共逻辑，非抽象
        ...

    async def _handle_message(self, ...) -> None:       # 公共逻辑，非抽象
        ...
```

**解析**
- `class X(ABC)` + `@abstractmethod` 标注的方法：**子类不实现就无法实例化**（实例化时报 `TypeError`）。
- 抽象方法可以是**异步方法**（`async def`），Python 天然支持。
- 抽象类里可以有**具体方法**（`is_allowed`、`_handle_message`）——这是"模板方法"模式的基础：公共流程写在基类，变化点留给子类。
- 本项目有 3 个抽象基类，对应 3 个扩展点：
  - `BaseChannel`（`channels/base.py`）→ 9 个通道子类
  - `LLMProvider`（`providers/base.py`）→ LiteLLM/Custom/Codex 3 个实现
  - `Tool`（`agent/tools/base.py`）→ 8+ 个工具

**抽象属性写法**（Tool 类）：

```python
# nanobot/agent/tools/base.py:24
@property
@abstractmethod
def name(self) -> str: ...

@property
@abstractmethod
def parameters(self) -> dict[str, Any]: ...
```

> 注意装饰器顺序：`@property` 在上、`@abstractmethod` 在下。

**最佳实践**
- 用 ABC 定义"插件接口"，让核心代码只依赖抽象、不依赖具体实现（依赖倒置）。
- 基类里放公共逻辑（模板方法），子类只写差异。
- 用 `isinstance(obj, BaseChannel)` 做类型守卫（`agent/loop.py:139`）。

---

### 4. property 属性

`@property` 把方法伪装成属性，实现"计算字段"或"只读接口"。

```python
# nanobot/bus/events.py:20
@property
def session_key(self) -> str:
    """Unique key for session identification."""
    return f"{self.channel}:{self.chat_id}"
```

```python
# nanobot/bus/queue.py:73
@property
def inbound_size(self) -> int:
    """Number of pending inbound messages."""
    return self.inbound.qsize()
```

**解析**
- 调用方写 `msg.session_key`（无括号），语义上是"这个对象的属性"。
- 好处：**未来可以改成真实存储字段而不破坏调用方**（接口稳定）。
- 配合 `@x.setter` 可实现带校验的赋值，本项目未用到。

**最佳实践**
- 派生值（由其他字段计算而来）用 property。
- 对外暴露"只读状态"用 property（如 `inbound_size`、`is_running`）。
- 别滥用：property 里做重计算或 IO 会让人误以为"访问属性很便宜"。

---

### 5. 异常处理

本项目展示了多层级的异常处理哲学：**外层兜底 + 内层细化**。

**① 精确捕获**

```python
# nanobot/agent/tools/shell.py:82
try:
    stdout, stderr = await asyncio.wait_for(process.communicate(), timeout=self.timeout)
except asyncio.TimeoutError:
    process.kill()
    return f"Error: Command timed out after {self.timeout} seconds"
```

**② 兜底捕获 + 记录（但不崩溃）**

```python
# nanobot/channels/manager.py:144
try:
    await channel.start()
except Exception as e:
    logger.error(f"Failed to start channel {name}: {e}")
```

**③ 异常转结果（本项目特色：错误不抛出，转为给 LLM 看的内容）**

```python
# nanobot/agent/tools/registry.py:61
try:
    return await tool.execute(**params)
except Exception as e:
    return f"Error executing {name}: {str(e)}"
```

**④ finally 清理**

```python
# nanobot/cli/commands.py:429
finally:
    await agent.close_mcp()
    heartbeat.stop()
    cron.stop()
    agent.stop()
    await channels.stop_all()
```

**⑤ 特殊异常：asyncio.CancelledError 与 BaseExceptionGroup**

```python
# nanobot/heartbeat/service.py:97
except asyncio.CancelledError:
    break

# nanobot/agent/loop.py:263
except (RuntimeError, BaseExceptionGroup):
    pass  # MCP SDK cancel scope cleanup is noisy but harmless
```

> 解析：
> - `CancelledError` 在 Python 3.8 起继承自 `BaseException` 而非 `Exception`，所以**捕获它必须显式写**，且捕获后要么重新抛出要么正确处理。
> - `BaseExceptionGroup` 是 3.11 的异常组（`except*` 配套），MCP 关闭时可能抛出异常组。

**最佳实践**
- 能精确捕获就精确捕获（`TimeoutError`、`KeyError`），不要一上来 `except Exception`。
- `except Exception` 兜底时**必须记录日志**（loguru 的 `logger.error(f"...: {e}")`），否则错误被吞掉无处排查。
- 异步取消（CancelledError）要单独处理。
- 别把异常吞掉当没发生——要么记录，要么转成返回值（本项目习惯），要么重新抛出。

---

### 6. f-string 与字符串处理

**① f-string 插值（Python 3.6+）**

```python
# nanobot/channels/base.py:108
logger.warning(
    f"Access denied for sender {sender_id} on channel {self.name}. "
    f"Add them to allowFrom list in config to grant access."
)
```

**② 三元表达式 + walrus（海象运算符 `:=`，3.8+）**

```python
# nanobot/agent/loop.py:160
def _fmt(tc):
    val = next(iter(tc.arguments.values()), None) if tc.arguments else None
    if not isinstance(val, str):
        return tc.name
    return f'{tc.name}("{val[:40]}…")' if len(val) > 40 else f'{tc.name}("{val}")'
```

```python
# nanobot/agent/skills.py:123
desc = escape_xml(self._get_skill_description(s["name"]))
```

**③ 字符串切片与拼接**

```python
# nanobot/agent/loop.py:294
preview = msg.content[:80] + "..." if len(msg.content) > 80 else msg.content
```

**④ 多行字符串（用于 prompt 模板）**

```python
# nanobot/agent/loop.py:442
prompt = f"""You are a memory consolidation agent. Process this conversation and return a JSON object with exactly two keys:

1. "history_entry": A paragraph (2-5 sentences) ...
"""
```

**最佳实践**
- 字符串拼接优先 f-string；`+` 拼接只用于短字符串；大量拼接用 `"".join(...)`。
- 三引号字符串天然适合 prompt/SQL/模板，注意 `"""` 内不要出现未转义的 `"""`。
- 用 `len(s) > n` 判断长度再切片，避免切片结果为空串的边界问题。
- prompt 模板中嵌入变量用 f-string，但**注意 f-string 里的大括号**（如 JSON 示例需要 `{{}}` 转义）。

---

### 7. Pathlib 文件系统操作

本项目**全程用 `pathlib.Path`，没有一处 `os.path.join`**——这是现代 Python 的标准做法。

**① 路径拼接与展开**

```python
# nanobot/config/schema.py:282
def workspace_path(self) -> Path:
    return Path(self.agents.defaults.workspace).expanduser()
```

**② 目录创建（幂等）**

```python
# nanobot/utils/helpers.py:8
def ensure_dir(path: Path) -> Path:
    path.mkdir(parents=True, exist_ok=True)
    return path
```

> `mkdir(parents=True, exist_ok=True)` 是"创建目录链 + 已存在不报错"的标准写法。

**③ 读写文件**

```python
# nanobot/agent/memory.py:22
def write_long_term(self, content: str) -> None:
    self.memory_file.write_text(content, encoding="utf-8")
```

**④ 遍历目录**

```python
# nanobot/agent/skills.py:40
for skill_dir in self.workspace_skills.iterdir():
    if skill_dir.is_dir():
        ...
```

```python
# nanobot/session/manager.py:181
for path in self.sessions_dir.glob("*.jsonl"):
```

**⑤ 解析与安全校验（resolve）**

```python
# nanobot/agent/tools/filesystem.py:9
def _resolve_path(path: str, allowed_dir: Path | None = None) -> Path:
    resolved = Path(path).expanduser().resolve()
    if allowed_dir and not str(resolved).startswith(str(allowed_dir.resolve())):
        raise PermissionError(f"Path {path} is outside allowed directory {allowed_dir}")
    return resolved
```

> `resolve()` 会把相对路径、`..`、符号链接解析成绝对规范路径——这是**做路径安全校验的前提**（不 resolve 的话 `a/../../etc/passwd` 能绕过前缀检查）。

**最佳实践**
- 一律 `from pathlib import Path`，拒绝 `os.path` 拼接。
- 文件读写显式 `encoding="utf-8"`（Windows 默认编码是 gbk，不写会乱码）。
- 安全路径判断用 `resolve()` 后的规范化结果比较。
- 追加写用 `open(path, "a", encoding="utf-8")`（`agent/memory.py:25`）。

---

### 8. 模块与包

**① 包结构（`__init__.py` 双职责）**

```python
# nanobot/bus/__init__.py —— 把子模块"提升"到包顶层，方便调用方
from nanobot.bus.events import InboundMessage, OutboundMessage
from nanobot.bus.queue import MessageBus
```

> 这样调用方写 `from nanobot.bus import MessageBus` 而不是 `from nanobot.bus.queue import MessageBus`——`__init__.py` 作为**包的公共门面（facade）**。

```python
# nanobot/bus/queue.py —— 而 queue.py 内部自己 import 自己的兄弟模块
from nanobot.bus.events import InboundMessage, OutboundMessage
```

**② 入口文件 `__main__.py`**

```python
# nanobot/__main__.py
from nanobot.cli.commands import app

if __name__ == "__main__":
    app()
```

> 效果：`python -m nanobot` 即可运行；配合 `pyproject.toml` 的 `[project.scripts] nanobot = "nanobot.cli.commands:app"`，安装后直接有 `nanobot` 命令。

**③ `if __name__ == "__main__"` 的意义**：文件被**直接运行**时才执行，被 import 时不执行——让模块既可被复用又可被单独运行。

**④ 惰性导入（函数内 import）——本项目的重要技巧**

```python
# nanobot/channels/manager.py:40
if self.config.channels.telegram.enabled:
    try:
        from nanobot.channels.telegram import TelegramChannel
        self.channels["telegram"] = TelegramChannel(...)
    except ImportError as e:
        logger.warning(f"Telegram channel not available: {e}")
```

```python
# nanobot/cli/commands.py:284
def _make_provider(config: Config):
    from nanobot.providers.litellm_provider import LiteLLMProvider
    ...
```

> 解析：
> - 重依赖（python-telegram-bot、lark、mcp 等）**只在需要时导入**，未启用的通道零启动开销。
> - `try/except ImportError` 让"依赖缺失"降级为警告而非崩溃。
> - 代价：牺牲一点静态分析能力（mypy 对函数内 import 支持较弱），但对 CLI 应用收益远大于成本。

**最佳实践**
- 包级 `__init__.py` 只 re-export 公共 API，别放业务代码。
- 入口脚本用 `__main__.py` + `if __name__ == "__main__"`。
- 可选依赖/重型依赖用函数内 import + ImportError 兜底。

---

### 9. 函数进阶

**① 默认参数与可变对象陷阱（必考）**

```python
# ❌ 错误示范：默认参数 [] 只在定义时创建一次，所有调用共享
def add_item(item, lst=[]):
    lst.append(item)

# ✅ 项目正确写法（dataclass 的 field）
media: list[str] = field(default_factory=list)
# ✅ 普通函数正确写法
def add_item(item, lst=None):
    if lst is None:
        lst = []
```

**② `**kwargs` 透传**

```python
# nanobot/agent/tools/base.py:43
async def execute(self, **kwargs: Any) -> str:

# nanobot/agent/tools/registry.py:60
return await tool.execute(**params)
```

> 工具接收 LLM 传来的任意参数字典，`**kwargs` 解包后转发——动态参数的标准做法。

**③ 装饰器（高阶函数）**

```python
# nanobot/cli/commands.py:931 —— 装饰器注册表
_LOGIN_HANDLERS: dict[str, callable] = {}

def _register_login(name: str):
    def decorator(fn):
        _LOGIN_HANDLERS[name] = fn
        return fn
    return decorator

@_register_login("openai_codex")
def _login_openai_codex() -> None: ...
```

> 用装饰器 + 全局注册表实现"声明式注册"，新增 OAuth 登录只需写一个带装饰器的函数。

**④ lambda 与回调**

```python
# nanobot/agent/subagent.py:87
bg_task.add_done_callback(lambda _: self._running_tasks.pop(task_id, None))
```

**⑤ 默认参数求值时机**：`default_factory=datetime.now`（不调用）vs `=datetime.now()`（调用）——前面已讲，重点复习。

---

### 10. global 全局变量

```python
# nanobot/cli/commands.py:37
_PROMPT_SESSION: PromptSession | None = None
_SAVED_TERM_ATTRS = None

def _init_prompt_session() -> None:
    global _PROMPT_SESSION, _SAVED_TERM_ATTRS
    _PROMPT_SESSION = PromptSession(...)
```

**解析**：函数内赋值会创建局部变量，要修改模块级变量必须声明 `global`。

**最佳实践**
- 全局状态尽量少用（本项目只用于 CLI 的单例会话），**模块级可变状态是并发 bug 的温床**（见第九部分全局状态反模式）。
- 如果必须全局，用 `global` 声明并在文档注释里说明谁在何时初始化。

---

## 第三部分 面向对象与设计模式

### 3.1 继承与多态 —— 9 个通道同构

```python
# nanobot/channels/telegram.py:101
class TelegramChannel(BaseChannel):
    name = "telegram"
    async def start(self) -> None: ...   # 长轮询实现
    async def send(self, msg: OutboundMessage) -> None: ...

# nanobot/channels/whatsapp.py:15
class WhatsAppChannel(BaseChannel):
    name = "whatsapp"
    async def start(self) -> None: ...   # WebSocket 桥实现
```

核心代码只依赖 `BaseChannel` 抽象，`ChannelManager` 用同一个 `start/send/stop` 接口驱动所有通道——这就是**多态**：同样的调用，不同的行为。

### 3.2 注册表模式 —— providers 单一事实源

```python
# nanobot/providers/registry.py:69
PROVIDERS: tuple[ProviderSpec, ...] = (
    ProviderSpec(name="openrouter", keywords=("openrouter",), env_key="OPENROUTER_API_KEY", ...),
    ProviderSpec(name="anthropic",  keywords=("anthropic", "claude"), env_key="ANTHROPIC_API_KEY", ...),
    ...
)

def find_by_model(model: str) -> ProviderSpec | None:
    model_lower = model.lower()
    for spec in PROVIDERS:
        if spec.is_gateway or spec.is_local:
            continue
        if any(kw in model_lower for kw in spec.keywords):
            return spec
    return None
```

**解析**
- "注册表" = 一个集中存放"元数据/可扩展项"的容器（这里是 tuple + 查找函数，`ToolRegistry` 是 dict 版）。
- 效果：**加一个 provider 不需要改任何 if-else**，只需在注册表加一条。这是"数据驱动"而非"代码驱动"。
- `LiteLLMProvider` 里因此**没有一条 if-elif 链**（`providers/litellm_provider.py` 全部逻辑由 spec 驱动）。

**最佳实践**
- 当"要支持的东西种类很多且行为差异大"时，考虑注册表模式。
- 注册表条目用 frozen dataclass 保证不可变。
- 查找逻辑集中成函数（`find_by_model` / `find_gateway` / `find_by_name`），复用而非各处复制。

### 3.3 策略模式 —— 模型提供方

`LLMProvider` 抽象 + 三个实现（LiteLLM 通用路由 / Custom 直连 / Codex OAuth）就是策略模式：`AgentLoop` 持有一个 `provider` 引用，运行时按配置切换具体策略，算法（chat）的接口不变。

```python
# nanobot/cli/commands.py:282 —— 工厂函数选择策略
def _make_provider(config: Config):
    provider_name = config.get_provider_name(model)
    if provider_name == "openai_codex":
        return OpenAICodexProvider(default_model=model)
    if provider_name == "custom":
        return CustomProvider(...)
    return LiteLLMProvider(...)
```

### 3.4 模板方法 —— 通道消息处理流水线

```python
# nanobot/channels/base.py:86 —— 基类定义流程骨架
async def _handle_message(self, sender_id, chat_id, content, ...):
    if not self.is_allowed(sender_id):          # 步骤1：鉴权（子类可覆盖）
        logger.warning(...)
        return
    msg = InboundMessage(channel=self.name, ...) # 步骤2：构造事件
    await self.bus.publish_inbound(msg)          # 步骤3：入队
```

所有通道共用这条"鉴权→构造→入队"流水线，只需调用 `self._handle_message(...)`（如 `telegram.py:369`），鉴权逻辑不会在各通道重复。

### 3.5 依赖注入与组合根

```python
# nanobot/cli/commands.py:356 —— 组合根：所有依赖在此装配
agent = AgentLoop(
    bus=bus,
    provider=provider,
    workspace=config.workspace_path,
    cron_service=cron,
    session_manager=session_manager,
    ...
)
# 之后再注入回调，避免循环依赖
cron.on_job = on_cron_job
```

**解析**：对象需要的依赖通过构造函数传入（依赖注入），而不是在内部自己 `new`。装配动作集中在一个"组合根"（composition root）——本项目是 `gateway()`。

**最佳实践**：依赖通过 `__init__` 参数注入；回调先建对象再赋值（如 cron 的 `on_job`）是打破循环依赖的常用手段。

### 3.6 适配器模式 —— MCP 工具包装

```python
# nanobot/agent/tools/mcp.py:12
class MCPToolWrapper(Tool):
    def __init__(self, session, server_name: str, tool_def):
        self._name = f"mcp_{server_name}_{tool_def.name}"
        ...
    async def execute(self, **kwargs: Any) -> str:
        result = await self._session.call_tool(self._original_name, arguments=kwargs)
        ...
```

把外部系统（MCP 服务器）的接口**适配**成项目内部的 `Tool` 接口，核心代码无需感知 MCP 的存在。

### 3.7 发布-订阅 / 消息队列模式

`MessageBus`（`bus/queue.py`）是发布-订阅的变体：生产者（通道）`publish_inbound`，消费者（AgentLoop）`consume_inbound`，两者通过 `asyncio.Queue` 解耦——**发送方不知道接收方是谁**，接收方也不知道消息从哪来。

---

## 第四部分 异步编程 asyncio（重点）

nanobot 是**纯 asyncio 应用**，这是本项目最值得深度学习的地方。

### 4.1 事件循环与 async/await 的本质

```python
# nanobot/__main__.py → cli/commands.py:435
asyncio.run(run())     # 创建事件循环并运行
```

- `async def` 定义协程；`await` 挂起当前协程让出控制权；`asyncio.run()` 是唯一入口。
- **阻塞 ≠ 等待**：`time.sleep()` 会卡死整个事件循环，必须用 `await asyncio.sleep()`（项目里所有延时都是 `await asyncio.sleep(...)`，见 `heartbeat/service.py:94`、`cron/service.py:199`）。

### 4.2 并发三兄弟：create_task / gather / wait_for

```python
# nanobot/channels/manager.py:157 —— create_task 并发启动多个通道
tasks = []
for name, channel in self.channels.items():
    tasks.append(asyncio.create_task(self._start_channel(name, channel)))
await asyncio.gather(*tasks, return_exceptions=True)   # :163 等全部启动完成
```

```python
# nanobot/agent/loop.py:314 —— create_task 后台执行（不阻塞当前流程）
asyncio.create_task(_consolidate_and_cleanup())
```

```python
# nanobot/bus/queue.py:59 —— wait_for 带超时地等待
msg = await asyncio.wait_for(self.outbound.get(), timeout=1.0)
```

**解析**
- `create_task()`：立即调度协程，**不等它完成**，返回 `asyncio.Task`；配合 `add_done_callback`（`subagent.py:87`）做完成回调。
- `gather()`：并发执行多个协程并**等全部完成**；`return_exceptions=True` 让单个失败不打断整体。
- `wait_for()`：给 await 加超时，超时抛 `asyncio.TimeoutError`，用 `except TimeoutError: continue` 实现"轮询不阻塞"（`agent/loop.py:256`）。

**三者分工**：gather = 我要等齐；create_task = 我不等，后台跑；wait_for = 我只等 N 秒。

### 4.3 异步队列 —— 生产者/消费者

```python
# nanobot/bus/queue.py:19
class MessageBus:
    def __init__(self):
        self.inbound: asyncio.Queue[InboundMessage] = asyncio.Queue()
        self.outbound: asyncio.Queue[OutboundMessage] = asyncio.Queue()

    async def publish_inbound(self, msg) -> None:   # 生产者：入队
        await self.inbound.put(msg)

    async def consume_inbound(self) -> InboundMessage:  # 消费者：阻塞取出
        return await self.inbound.get()
```

**解析**：`asyncio.Queue` 是**线程安全的异步队列**（在单事件循环内天然安全）；`get()` 无元素时挂起等待——这是通道与 Agent 解耦的核心。`qsize()` 可查看积压量（`inbound_size`）。

### 4.4 超时与取消的正确姿势

```python
# nanobot/agent/tools/shell.py:78 —— 命令超时则杀进程
try:
    stdout, stderr = await asyncio.wait_for(process.communicate(), timeout=self.timeout)
except asyncio.TimeoutError:
    process.kill()
```

```python
# nanobot/heartbeat/service.py:97 —— 循环体响应取消
while self._running:
    try:
        await asyncio.sleep(self.interval_s)
        if self._running:
            await self._tick()
    except asyncio.CancelledError:
        break
```

**最佳实践**
- 外部 IO（网络、子进程）一律加超时，防止永久挂起。
- 长时间运行的循环任务要 `try/except asyncio.CancelledError` 做清理，否则 `task.cancel()` 后资源不释放。
- `stop()` 方法只置 `_running = False` 标志 + `task.cancel()`（`cron/service.py:162`、`heartbeat/service.py:83`）是标准停机模式。

### 4.5 异步子进程

```python
# nanobot/agent/tools/shell.py:70
process = await asyncio.create_subprocess_shell(
    command,
    stdout=asyncio.subprocess.PIPE,
    stderr=asyncio.subprocess.PIPE,
    cwd=cwd,
)
stdout, stderr = await process.communicate()
```

> 注意：**永远不要**用同步的 `subprocess.run` 跑长命令（会阻塞事件循环）；异步写法用 `create_subprocess_shell`/`create_subprocess_exec` + `communicate()`。同步短命令（如 CLI 构建桥）可以用 `subprocess.run(..., capture_output=True)`（`cli/commands.py:671`），但那是 CLI 一次性场景。

### 4.6 to_thread —— 把阻塞 IO 扔出事件循环

```python
# nanobot/channels/email.py:79
inbound_items = await asyncio.to_thread(self._fetch_new_messages)
```

> `imaplib` 是同步阻塞库，用 `asyncio.to_thread` 把它放到线程池执行，事件循环不被卡住。**这是"项目里既有同步库又有异步框架"时的标准桥接手段**（同理 `openai_codex_provider.py:38` 的 `asyncio.to_thread(get_codex_token)`）。

### 4.7 AsyncExitStack —— 管理多个异步资源

```python
# nanobot/agent/loop.py:132
self._mcp_stack = AsyncExitStack()
await self._mcp_stack.__aenter__()
await connect_mcp_servers(self._mcp_servers, self.tools, self._mcp_stack)
# ...
# 关闭时：
await self._mcp_stack.aclose()
```

```python
# nanobot/agent/tools/mcp.py:59
read, write = await stack.enter_async_context(stdio_client(params))
session = await stack.enter_async_context(ClientSession(read, write))
```

> `AsyncExitStack` 是"动态收集任意多个异步上下文管理器、最后统一关闭"的容器——MCP 连接数量运行时才知道，无法用固定 `async with`。

### 4.8 异步生成器 —— SSE 流式解析

```python
# nanobot/providers/openai_codex_provider.py:225
async def _iter_sse(response: httpx.Response) -> AsyncGenerator[dict[str, Any], None]:
    buffer: list[str] = []
    async for line in response.aiter_lines():
        if line == "":
            if buffer:
                data_lines = [l[5:].strip() for l in buffer if l.startswith("data:")]
                ...
                yield json.loads(data)
            continue
        buffer.append(line)
```

**解析**：`async for` + `yield` 定义**异步生成器**——边接收网络流边产出事件，内存占用恒定；调用方 `async for event in _iter_sse(response)`（同文件 251 行）逐个消费。这是处理流式 API（SSE、WebSocket 数据流）的标准范式。

### 4.9 竞态与并发教训（进阶）

项目里的 `asyncio.create_task(_consolidate_memory(...))`（`agent/loop.py:322`）在后台跑记忆整合，同时 `_process_message` 继续对同一 session 写入——**两个协程并发修改同一对象，无锁**。这是 asyncio 最常见的坑：**单线程 ≠ 无竞态**，因为 await 点之间会切换协程。正确做法是 per-session 加锁或队列（见第九部分）。

---

## 第五部分 数据、序列化与配置

### 5.1 JSON 与 JSONL

```python
# nanobot/config/loader.py:35 —— 读 JSON 配置
data = json.load(f)
return Config.model_validate(data)
```

```python
# nanobot/session/manager.py:150 —— JSONL：每行一个 JSON 对象
with open(path, "w") as f:
    f.write(json.dumps(metadata_line) + "\n")   # 首行是会话元数据
    for msg in session.messages:
        f.write(json.dumps(msg) + "\n")         # 之后每行一条消息
```

**解析**
- **JSONL（JSON Lines）** = 每行一个独立 JSON。优点：可追加写、可流式读、损坏一行不影响其余行、方便 grep。适合日志/会话历史。
- 对比：cron 用单个 JSON 文件（`cron/service.py:151` 整体 `json.dumps(indent=2)` 保存），因为 jobs 是小规模结构化数据。

**容错解析 json_repair**

```python
# nanobot/providers/litellm_provider.py:180
args = json_repair.loads(args)      # LLM 返回的 JSON 经常不合法
# nanobot/agent/loop.py:470
result = json_repair.loads(text)
```

> **LLM 生成的 JSON 不可信**（会多逗号、单引号、markdown 包裹）。`json_repair` 是专门修复脏 JSON 的库；再配合 `text.split("\n",1)[-1].rsplit("```",1)[0]` 剥掉 markdown 代码围栏（`agent/loop.py:469`）。这是所有 LLM 应用的标准防御。

**最佳实践**
- 日志/历史用 JSONL（追加友好）；配置/快照用单个 JSON（整体读整体写）。
- 解析不可信 JSON 用 json_repair + 类型检查（`if not isinstance(result, dict): skip`）。
- `json.dumps(..., ensure_ascii=False)` 保留中文可读（`agent/loop.py:221`）。

### 5.2 base64 与 mimetypes —— 图片进消息

```python
# nanobot/agent/context.py:171
p = Path(path)
mime, _ = mimetypes.guess_type(path)
if not p.is_file() or not mime or not mime.startswith("image/"):
    continue
b64 = base64.b64encode(p.read_bytes()).decode()
images.append({"type": "image_url", "image_url": {"url": f"data:{mime};base64,{b64}"}})
```

**解析**：`base64.b64encode(bytes).decode()` 把图片二进制转成文本；`mimetypes.guess_type` 猜 MIME 类型；拼接成 `data:image/png;base64,...` 的 data URL 喂给多模态模型。`p.read_bytes()` 一次性读二进制。

### 5.3 Pydantic 配置体系（config/schema.py 全文精华）

```python
# nanobot/config/schema.py:9
class Base(BaseModel):
    """Base model that accepts both camelCase and snake_case keys."""
    model_config = ConfigDict(alias_generator=to_camel, populate_by_name=True)

class TelegramConfig(Base):
    enabled: bool = False
    token: str = ""
    allow_from: list[str] = Field(default_factory=list)
    proxy: str | None = None
```

```python
# nanobot/config/schema.py:271
class Config(BaseSettings):
    agents: AgentsConfig = Field(default_factory=AgentsConfig)
    channels: ChannelsConfig = Field(default_factory=ChannelsConfig)
    ...
    model_config = ConfigDict(env_prefix="NANOBOT_", env_nested_delimiter="__")
```

**解析**
- **`BaseModel`**（Pydantic v2）：声明式字段 + 自动类型校验 + `model_validate()`（从 dict 构造）+ `model_dump()`（转 dict）。
- **`alias_generator=to_camel`**：JSON 里写 `allowFrom`（camelCase），代码里写 `allow_from`（snake_case），两者都认——兼容"外部配置格式"与"Python 习惯"。
- **`BaseSettings` + `env_prefix`**：自动从环境变量读配置，`NANOBOT_TOOLS__EXEC__TIMEOUT` 对应 `tools.exec.timeout`（`__` 分隔嵌套）——**敏感信息（API key）不进配置文件**。
- `Field(default_factory=list)` 保证每个实例独立列表。

**最佳实践**：复杂应用用 Pydantic 做配置/数据校验（类型错误在启动时暴露而非运行中爆炸）；`validate` 失败要降级（`loader.py:38` 捕获异常回退默认配置并打印警告）。

### 5.4 环境变量管理

```python
# nanobot/providers/litellm_provider.py:64
if self._gateway:
    os.environ[spec.env_key] = api_key
else:
    os.environ.setdefault(spec.env_key, api_key)   # 不覆盖已有环境变量
```

> `setdefault` 语义：已存在则不覆盖（尊重用户显式设置）。注意 `os.environ[...] = v` 是**进程级全局副作用**（第九部分会批评）。

---

## 第六部分 系统编程、正则与安全

### 6.1 正则表达式实战

**① 提取与替换（去除 think 块）**

```python
# nanobot/agent/loop.py:155
return re.sub(r"<think>[\s\S]*?</think>", "", text).strip() or None
```

**② 非贪婪 + 多行 + 分组（Markdown 转换器）**

```python
# nanobot/channels/telegram.py:31
text = re.sub(r'```[\w]*\n?([\s\S]*?)```', save_code_block, text)
# 用占位符 \x00CB{index}\x00 保护代码块，避免被后续规则误伤 → 最后再还原
```

**③ 标志位与前瞻（telegram.py:58 避免匹配变量名里的下划线）**

```python
text = re.sub(r'(?<![a-zA-Z0-9])_([^_]+)_(?![a-zA-Z0-9])', r'<i>\1</i>', text)
```

**④ 安全检测（黑名单）**

```python
# nanobot/agent/tools/shell.py:25
deny_patterns = [
    r"\brm\s+-[rf]{1,2}\b",          # rm -r, rm -rf, rm -fr
    r"\bdd\s+if=",                   # dd
    r":\(\)\s*\{.*\};\s*:",          # fork bomb
]
```

**最佳实践**：正则用于"提取/替换/检测"三类任务；复杂转换用**占位符保护法**（先摘出再还原，telegram 转换器的精髓）；安全正则只是兜底（见反模式部分）。

### 6.2 时间与时区

```python
# nanobot/cron/service.py:37 —— 带时区的 cron 计算
tz = ZoneInfo(schedule.tz) if schedule.tz else datetime.now().astimezone().tzinfo
base_dt = datetime.fromtimestamp(base_time, tz=tz)
cron = croniter(schedule.expr, base_dt)
next_dt = cron.get_next(datetime)
```

```python
# nanobot/session/manager.py:38 —— 时间戳
"timestamp": datetime.now().isoformat()
```

**最佳实践**：时间统一存 ISO 字符串（`isoformat()`）可排序可解析；跨时区调度用 `zoneinfo.ZoneInfo`（Python 3.9+ 标准库，替代已废弃的 pytz）；时间戳计算用毫秒整数（`cron/service.py:16` 的 `int(time.time()*1000)`）避免浮点精度问题。

### 6.3 安全编程三件套

**① 路径穿越防护**（`agent/tools/filesystem.py:9`，见第 7 节）——`resolve()` 后再做前缀判断。

**② SSRF 防护**（`agent/tools/web.py:33`）

```python
def _validate_url(url: str) -> tuple[bool, str]:
    p = urlparse(url)
    if p.scheme not in ('http', 'https'):
        return False, f"Only http/https allowed, got '{p.scheme or 'none'}'"
    if not p.netloc:
        return False, "Missing domain"
    return True, ""
```

> `urlparse` 解析 URL 后校验协议与域名，限制重定向次数（`MAX_REDIRECTS = 5`）——防止 LLM 被诱导访问内网/文件协议。

**③ 访问白名单**（`channels/base.py:61` `is_allowed`）

```python
allow_list = getattr(self.config, "allow_from", [])
if not allow_list:
    return True
sender_str = str(sender_id)
if sender_str in allow_list: return True
if "|" in sender_str:          # 支持 "id|username" 复合 ID 分段匹配
    for part in sender_str.split("|"):
        if part and part in allow_list: return True
```

**最佳实践**：对不可信输入（LLM 参数、用户消息、外部 API）做**白名单/黑名单 + 规范化**双重校验；安全校验失败返回明确错误信息（本项目统一 `f"Error: ..."` 字符串，方便 LLM 理解并自我修正）。

### 6.4 终端与信号处理（CLI 专属）

```python
# nanobot/cli/commands.py:519 —— Ctrl+C 时恢复终端
signal.signal(signal.SIGINT, _exit_on_sigint)

# nanobot/cli/commands.py:41 —— 丢弃模型生成期间用户误输入
import termios
termios.tcflush(fd, termios.TCIFLUSH)
```

> `signal` 注册信号处理函数；`termios` 直接操作终端属性（保存/恢复、清空输入缓冲）——交互式 CLI 的高级话题。

---

## 第七部分 第三方生态实战

| 库 | 项目用法 | 一句话点评 |
|---|---|---|
| **typer** | `@app.command()` 定义 CLI（`cli/commands.py:157`） | 基于类型注解自动生成命令行参数/帮助/补全，写 CLI 首选 |
| **rich** | `console.print`、`Table`、`Markdown`（cli 全文） | 终端美化输出，表格/进度/语法高亮 |
| **prompt_toolkit** | `PromptSession` + `FileHistory` + `patch_stdout`（`cli/commands.py:93`） | 交互输入：历史记录、粘贴、多行编辑、与异步输出共存 |
| **loguru** | `logger.info/error/warning` 全项目 | 零配置日志，自动记录调用位置，比 logging 标准库省心 |
| **httpx** | `httpx.AsyncClient`（`agent/tools/web.py`、`openai_codex_provider.py`） | 异步 HTTP 客户端，`async with` + `raise_for_status()` |
| **litellm** | `acompletion` 统一调用 100+ 模型（`providers/litellm_provider.py:160`） | 一个接口接所有 LLM，`drop_params`/`extra_headers` 兼容各家 |
| **pydantic** | 配置体系（见 5.3） | 声明式数据校验 |
| **croniter** | cron 表达式计算下次执行（`cron/service.py:39`） | 解析 `0 9 * * *` 类表达式 |
| **mcp** | `ClientSession`/`stdio_client`（`agent/tools/mcp.py`） | 连接 MCP 服务器获取外部工具 |

**httpx 标准姿势**（项目里最典型）：

```python
# nanobot/agent/tools/web.py:70
async with httpx.AsyncClient() as client:
    r = await client.get(url, params={"q": query, "count": n},
                         headers={"X-Subscription-Token": self.api_key},
                         timeout=10.0)
    r.raise_for_status()      # 非 2xx 抛异常
results = r.json()            # 直接解析 JSON
```

---

## 第八部分 工程化与测试

### 8.1 pyproject.toml（现代打包标准）

```toml
[project]
name = "nanobot-ai"
version = "0.1.4"
requires-python = ">=3.11"
dependencies = ["typer>=0.9.0", "litellm>=1.0.0", ...]

[project.optional-dependencies]   # 可选依赖分组
dev = ["pytest>=7.0.0", "pytest-asyncio>=0.21.0", "ruff>=0.1.0"]

[project.scripts]                  # 安装后生成命令行入口
nanobot = "nanobot.cli.commands:app"

[build-system]                     # hatchling 构建后端
requires = ["hatchling"]

[tool.ruff]                        # 工具配置集中管理
line-length = 100
```

**解析**：`pyproject.toml` 是 Python 打包/工具配置的现代单一入口；`[project.scripts]` 把 `app`（typer 实例）暴露成 `nanobot` 命令；`optional-dependencies` 区分运行时/开发依赖；ruff 配置跟随项目。

### 8.2 测试（pytest + pytest-asyncio）

```toml
# pyproject.toml:94
[tool.pytest.ini_options]
asyncio_mode = "auto"      # 自动识别 async 测试函数
testpaths = ["tests"]
```

```python
# tests/test_consolidate_offset.py —— 项目测试示例风格
# （测试记忆整合的 last_consolidated 偏移逻辑）
```

**最佳实践**
- 异步代码用 `pytest-asyncio`，`asyncio_mode = "auto"` 免装饰器。
- 纯逻辑（工具校验、consolidate 偏移、CLI 解析）优先测——不依赖网络的测试才有价值。
- 本项目测试覆盖偏薄（6 个文件），**你应该做得更好**：`AgentLoop`、provider、通道都要有 mock 测试。

### 8.3 代码规范与静态检查

```toml
[tool.ruff.lint]
select = ["E", "F", "I", "N", "W"]   # 错误、未使用、导入排序、命名、警告
ignore = ["E501"]                     # 忽略行长度（项目用 100 列）
```

- **ruff**：极快的 lint + 格式化 + 导入排序（`I` 规则自动排 import）。
- 建议再加 **mypy / pyright** 做类型检查——本项目类型注解完备，`mypy --strict` 基本可零错误跑通，是极佳练习。

---

## 第九部分 实战教训与反模式

学习"别人踩过的坑"比学习"正确的写法"更能长记性。

### ❌ 反模式 1：上帝对象（God Object）
`AgentLoop`（`agent/loop.py`，472 行）承担 9 项职责：总线消费、斜杠命令、LLM 循环、记忆整合、系统消息、MCP 生命周期、CLI 直通……
**教训**：职责单一原则（SRP）。改进：拆出 `MessageHandler`、`MemoryConsolidator`、`McpManager`。

### ❌ 反模式 2：代码复制（复制粘贴循环）
`SubagentManager._run_subagent`（`agent/subagent.py:130`）复制了主循环 `_run_agent_loop`（`agent/loop.py:187`）几乎全部逻辑。
**教训**：DRY（不要重复自己）。两处已经分叉（子代理没有 `_strip_think`、没有 reasoning 处理）。改进：抽出共享的 `run_tool_loop(messages, tools, max_iterations)`。

### ❌ 反模式 3：全局可变状态
`litellm.api_base = api_base`（`litellm_provider.py:46`）、`os.environ[...] = ...`（:64）是进程级副作用。
**教训**：全局状态让"多个实例并存"成为不可能（库化使用时必炸）。改进：优先传参/实例字段；env 只在进程启动时设置一次。

### ❌ 反模式 4：错误转字符串（语义丢失）
`LiteLLMProvider.chat` 把异常转成 `content="Error calling LLM: ..."`（`litellm_provider.py:163`），Agent 无法区分"真回复"与"错误"，错误文本最终会发给用户。
**教训**：错误要有类型/状态（项目已有 `finish_reason="error"` 字段但没人检查）。改进：调用方检查 `finish_reason` 再决定是否继续迭代。

### ❌ 反模式 5：正则黑名单安全
`ExecTool._guard_command` 用正则黑名单拦截危险命令（`shell.py:25`）——`python -c`、`curl | sh`、编码混淆均可绕过。
**教训**：黑名单只能防手滑，不能防攻击。改进：白名单允许列表 / 容器沙箱 / 交互确认高风险操作。

### ❌ 反模式 6：字符串协议（Stringly Typed）
会话键 `"channel:chat_id"` 在 `loop.py`、`session/manager.py:77`（`replace("_", ":")`）、`list_sessions` 三处隐式约定。
**教训**：用常量、枚举或 dataclass 表达协议，别靠"记得格式"。

### ❌ 反模式 7：并发无锁修改
`asyncio.create_task(self._consolidate_memory(session))` 与 `_process_message` 同时写 session。
**教训**：单线程也有竞态（await 切换点）。改进：per-session `asyncio.Lock` 或串行化到专用队列。

### ✅ 值得学习的正面实践
- **优雅降级**：通道 ImportError 只警告；单消息异常不退出主循环（`agent/loop.py:249`）。
- **超时全覆盖**：所有外部 IO 都有 `wait_for`/`timeout`。
- **防御式解析**：`json_repair` + 类型检查 + markdown 围栏剥离。
- **上下文预算管理**：`memory_window`、工具输出截断（`shell.py:102`）、技能渐进加载。
- **资源统一清理**：`finally` + `stop()` 标志 + 任务取消。

---

## 第十部分 学习路径建议

### 阶段一：语言地基（1-2 周）
1. 精读 [第二部分](#第二部分-语言基础)：类型注解、dataclass、ABC、property、异常、Pathlib。
2. 练习：把 `bus/events.py` 和 `cron/types.py` 抄写一遍并改造（加字段、加校验）。
3. 工具：安装 ruff，跑 `ruff check nanobot/` 看它报什么。

### 阶段二：OOP 与设计模式（1 周）
1. 精读 [第三部分](#第三部分-面向对象与设计模式)。
2. 练习：自己实现一个 `BaseChannel` 子类（如"文件通道"：把消息写到本地文件），挂进 `ChannelManager` 试试。

### 阶段三：asyncio（2-3 周，本项目核心）
1. 精读 [第四部分](#第四部分-异步编程-asyncio重点)，逐个知识点手写最小示例：
   ```python
   # 练习：实现一个简易版 MessageBus（两个队列 + 生产者/消费者）
   import asyncio
   async def producer(q):
       for i in range(5):
           await q.put(i); await asyncio.sleep(0.1)
   async def consumer(q):
       while True:
           item = await q.get()
           print(item)
   async def main():
       q = asyncio.Queue()
       await asyncio.gather(producer(q), consumer(q))
   asyncio.run(main())
   ```
2. 练习：给 `ExecTool` 写一个用 `asyncio.create_subprocess_shell` 的小工具。
3. 进阶：读懂 `openai_codex_provider.py` 的 SSE 异步生成器。

### 阶段四：数据与配置（1 周）
1. 精读 [第五部分](#第五部分-数据序列化与配置)。
2. 练习：用 Pydantic 给一个"Todo 应用"写配置 + 数据模型，支持环境变量覆盖。

### 阶段五：工程化（1-2 周）
1. 精读 [第八部分](#第八部分-工程化与测试)。
2. 练习：给 `agent/tools/base.py` 的 `validate_params` 写 pytest 测试（覆盖所有校验分支）；
   再给 `cron/service.py` 的 `_compute_next_run` 写测试（三种 schedule 类型）。
3. 跑 `mypy --strict nanobot/bus/` 体验类型检查。

### 阶段六：实战改造（产出作品）
挑一个"改进项目"作为毕业设计：
- 给 `MessageBus` 加每会话并发处理（per-session 队列）；
- 把 `_run_agent_loop` 抽成共享工具循环（消除子代理复制）；
- 给某个通道（如 whatsapp）实现语音转写；
- 写一个新的 `Tool`（如天气工具）并注册。

### 推荐资源
- 官方教程：https://docs.python.org/3/tutorial/
- asyncio 权威指南：https://docs.python.org/3/library/asyncio.html
- Real Python asyncio 系列：https://realpython.com/async-io-python/
- 类型检查：https://mypy.readthedocs.io/ ｜ https://mypy-play.net/
- Pydantic：https://docs.pydantic.dev/
- Typer：https://typer.tiangolo.com/

---

*本指南由 AI 基于 nanobot 项目源码（`F:\ai-agent\nanobot\nanobot`）分析生成，所有代码引用均来自当前代码库。*
