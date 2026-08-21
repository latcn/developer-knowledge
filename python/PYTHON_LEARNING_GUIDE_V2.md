# 🐈 基于 nanobot v0.3.0 的 Python 学习指南（V2）

> **本指南以 nanobot 当前源码（`F:\ai-agent\nanobot\nanobot`，v0.3.0，非测试代码约 9.5 万行）为真实教材**，逐知识点讲解
> **"代码长什么样 → 为什么这么写 → 易错点 → 最佳实践"**。
> 它是旧版 `PYTHON_LEARNING_GUIDE.md`（基于 v0.1.4）的全面升级：旧版引用的 `litellm_provider.py`、`heartbeat/service.py`、`subscribe_outbound` 等已在当前代码中删除或重构，本指南所有示例均经实际读取验证，附当前文件与行号。
> 阅读时建议同时打开对应源码文件边看边读。

---

## 目录

- [第一部分 项目速览与学习地图](#第一部分-项目速览与学习地图)
- [第二部分 语言基础（14 个知识点）](#第二部分-语言基础)
- [第三部分 面向对象与设计模式](#第三部分-面向对象与设计模式)
- [第四部分 异步编程 asyncio（重点）](#第四部分-异步编程-asyncio重点)
- [第五部分 数据、序列化与配置](#第五部分-数据序列化与配置)
- [第六部分 系统编程、正则与安全](#第六部分-系统编程正则与安全)
- [第七部分 第三方生态实战](#第七部分-第三方生态实战)
- [第八部分 工程化与测试](#第八部分-工程化与测试)
- [第九部分 实战教训与反模式（v0.3.0 更新版）](#第九部分-实战教训与反模式)
- [第十部分 学习路径建议](#第十部分-学习路径建议)

---

## 第一部分 项目速览与学习地图

v0.3.0 的 nanobot 是一个 **纯 asyncio + 严格类型注解（basedpyright strict）** 的个人 AI Agent 运行时，几乎涵盖 Python 中高级全部知识点，且比旧版多了不少"进阶武器"：

| 技术领域                           | 涉及模块（当前）                                                                                      | 本指南章节    |
| ------------------------------ | --------------------------------------------------------------------------------------------- | -------- |
| 类型注解、dataclass、ABC、Protocol    | `bus/events.py`、`providers/base.py`、`channels/base.py`、`agent/tools/context.py`               | 第二 §1-§4 |
| ContextVar 请求作用域               | `agent/tools/context.py`、`agent/tools/file_state.py`、`security/workspace_access.py`           | 第二 §13   |
| asyncio 全家族（锁/信号量/事件/队列/子进程/流） | `bus/queue.py`、`agent/loop.py`、`agent/runner.py`、`cron/service.py`                            | 第四       |
| 插件注册表与元数据驱动                    | `channels/plugin.py`、`providers/registry.py`、`agent/tools/loader.py`                          | 第三       |
| 数据序列化与原子写                      | `session/manager.py`（JSONL）、`cron/service.py`（WAL）、`utils/helpers.py`                         | 第五 §1、§5 |
| Pydantic v2 配置                 | `config/schema.py`（717 行）、`config_base.py`                                                    | 第五 §3    |
| 正则、subprocess、时区               | `agent/tools/shell.py`、`utils/helpers.py`、`cron/service.py`                                   | 第六       |
| 安全编程                           | `security/network.py`、`security/workspace_policy.py`、`agent/tools/web.py`                     | 第六 §3    |
| 系统编程                           | `_windows_job.py`（ctypes）、`process_runtime.py`、`triggers/local_store.py`（filelock）            | 第六 §4-§5 |
| 第三方生态                          | typer、rich、loguru、httpx、pydantic、croniter、mcp、websockets、openai、anthropic、json-repair、dulwich | 第七       |
| 工程化                            | `pyproject.toml`、ruff、basedpyright、pytest 300+ 文件、coverage 75%                                | 第八       |

**建议学习顺序**：第二（语言基础）→ 第三（OOP/模式）→ 第四（asyncio，本指南重点）→ 第五~八 → 第九（吸取教训）→ 第十（路线图）。

---

## 第二部分 语言基础

### 1. 类型注解体系（PEP 484 / 585 / 604 / 586 / 589）

**① PEP 585 —— 内置泛型（Python 3.9+）**

```python
# nanobot/bus/queue.py:17-18 —— asyncio.Queue 带上元素类型
self.inbound: asyncio.Queue[InboundMessage] = asyncio.Queue()
self.outbound: asyncio.Queue[OutboundMessage] = asyncio.Queue()

# nanobot/agent/loop.py:407 —— 嵌套泛型：键为 str、值为「task 集合」的字典
self._active_tasks: dict[str, set[asyncio.Task[Any]]] = {}
```

> 解析：旧写法 `Dict[str, Set[Task]]` 来自 `typing`，3.9+ 直接用 `dict[str, set[...]]`（PEP 585）。`asyncio.Queue[X]` 的 `X` 是**类型参数**——任何类都能当类型参数，不需要它本身是泛型。

**② PEP 604 —— 联合类型 `|`（Python 3.10+）**

```python
# nanobot/utils/llm_runtime.py:27
model_preset: str | None = None

# nanobot/agent/tools/context.py:34
message_id: str | None = None
```

> 解析：`str | None` ≡ 旧 `Optional[str]`；`A | B` ≡ `Union[A, B]`。项目要求 Python ≥3.11，全程使用。**易错点**：`|` 联合只适用于类型注解语境；运行时 `isinstance(x, str | int)` 在 3.10+ 才合法。

**③ PEP 586 —— Literal 字面量类型**

```python
# nanobot/bus/events.py:37
input_role: Literal["user", "system"] | None = None

# nanobot/config/schema.py:212
api_type: Literal["auto", "chat_completions", "responses"] = "auto"
```

> 解析：`Literal["user", "system"]` 限定取值只能是这两个字符串。写错值静态检查器直接报错——比 `str` 强得多。**最佳实践**：有限枚举值优先 `Literal`，需要运行时常量集合时用 `Enum`（见 §12）。

**④ Callable / Awaitable / Coroutine —— 回调签名**

```python
# nanobot/bus/runtime_events.py:123
RuntimeEventHandler = Callable[[Any], Awaitable[None] | None]

# nanobot/cron/service.py（构造参数）
on_job: Callable[[CronJob], Coroutine[Any, Any, str | None]] | None = None
```

> 解析：`Callable[[参数], 返回]` 描述可调用对象；`Awaitable[T]` 表示"可 await 的对象"；`Coroutine[Any, Any, T]` 更精确。项目里用**类型别名**（`RuntimeEventHandler = ...`）让复杂签名可复用。

**⑤ Any 与逃生舱**

```python
# nanobot/bus/events.py:34
metadata: dict[str, Any] = field(default_factory=dict)
```

> 解析：`Any` 关闭类型检查。工具参数来自 LLM 的 JSON，天然动态，用 `Any` 合理；但**能用具体类型就不用 Any**——项目在"已知结构"处都用 TypedDict/dataclass（见 §11）。

**最佳实践**：所有函数签名写全注解（项目 100% 覆盖，`basedpyright strict` 零错误跑通）；用 `| None`、`list[str]`、`Literal`；回调签名写全参数与返回；复杂回调类型用 `TypeAlias` 命名。

---

### 2. 泛型：运行时"不检查"的两面性

```python
# nanobot/agent/tools/context.py:51 —— ContextVar 的泛型
def bind_request_context(ctx: RequestContext) -> Token[RequestContext | None]:
    return _CURRENT_REQUEST_CONTEXT.set(ctx)
```

**易错点（必考）**
- Python 泛型**运行时完全不检查**：`asyncio.Queue[int]()` 里塞字符串不会报错，只是给 mypy/pyright/IDE 看的元数据。想要运行时校验得自己写（如子类化做 `isinstance`）。
- 普通类不能下标：`SomeClass[int]` 抛 `TypeError: type object is not subscriptable`，除非继承 `Generic` 或实现 `__class_getitem__`。
- `TypeVar` 带约束：`T = TypeVar("T", bound=BaseChannel)` 限定 T 必须是 BaseChannel 子类，可用于写"泛型池/泛型工厂"。

**项目中的约束型泛型**

```python
# nanobot/utils/helpers.py:30 —— 泛型 + overload 风格的薄封装
def sanitize_surrogates(text: _T) -> _T: ...
def sanitize_surrogates(text: Any) -> Any: ...
```

> 解析：先声明一个"仅注解"的泛型签名（`_T -> _T` 保持类型），再写实现体（`Any -> Any`）。这是**"重载声明 + 实现"**的常见简化写法——调用方拿到精确类型，实现体免检查。

---

### 3. dataclass 数据类全套

**① 基础 + 可变默认值陷阱**

```python
# nanobot/bus/events.py:24-50
@dataclass
class InboundMessage:
    channel: str
    sender_id: str
    chat_id: str
    content: str
    timestamp: datetime = field(default_factory=datetime.now)
    media: list[str] = field(default_factory=list)
    metadata: dict[str, Any] = field(default_factory=dict)
    session_key_override: str | None = None

    @property
    def session_key(self) -> str:
        return self.session_key_override or f"{self.channel}:{self.chat_id}"
```

**易错点**
- `field(default_factory=list)`：可变默认值的正确写法。直接 `= []` 会让**所有实例共享同一个列表**（定义时只创建一次）。
- `default_factory=datetime.now` **不写括号**——传"工厂函数"，每次实例化调用；写 `datetime.now()` 则所有实例共享创建时刻。
- `field(default_factory=dict, repr=False)`：`repr=False` 让敏感字段（如 `ProviderConversationState.payload` 可能含加密 reasoning，`providers/base.py:171`）不进日志/报错信息。

**② frozen / slots / compare / __post_init__**

```python
# nanobot/utils/llm_runtime.py:14-15 —— 不可变运行时快照
@dataclass(frozen=True, slots=True)
class LLMRuntime:
    provider: LLMProvider
    model: str
    generation: GenerationSettings
    context_window_tokens: int
    model_preset: str | None = None
```

```python
# nanobot/session/manager.py:163-169 —— frozen + frozenset 默认值
@dataclass(frozen=True)
class SessionPolicy:
    persist: bool = True
    log_content: bool = True
    disabled_tools: frozenset[str] = frozenset()
```

> 解析：`frozen=True` 实例不可变（`__setattr__` 抛 FrozenInstanceError），适合"配置快照/常量表"；`slots=True` 用 `__slots__` 省内存、去 `__dict__`（3.10+）。`compare=False` 排除比较字段（`Session.policy`）。

```python
# nanobot/session/manager.py:185-197 —— 构造后防御性校验
def __post_init__(self) -> None:
    if not isinstance(cast(object, self.metadata), dict):
        self.metadata = {}
    last_consolidated = cast(object, self.last_consolidated)
    if (
        isinstance(last_consolidated, bool)
        or not isinstance(last_consolidated, int)
        or not 0 <= last_consolidated <= len(self.messages)
    ):
        self.last_consolidated = 0
```

> `__post_init__` 在 `__init__` 生成后运行，是"反序列化自不可信数据时的兜底修复"标准位置（注意：`bool` 是 `int` 子类，`isinstance(x, int)` 会把 `True` 也判为 int——所以要先排除 bool，这是经典易错点）。

**③ dataclasses.replace —— 不可变对象的"改一份"**

```python
# nanobot/agent/loop.py:1329 —— 局部替换字段并保持其余不变
msg = dataclasses.replace(msg, session_key_override=session_key)
# nanobot/utils/llm_runtime.py:60-82 —— 派生"带覆盖的运行时"
def with_generation_overrides(self, *, temperature=None, ...) -> LLMRuntime:
    return replace(self, generation=GenerationSettings(...))
```

> `dataclasses.replace` 按字段名替换并**深拷贝其余字段**（拷贝 `__dict__` 再 setattr），是 frozen dataclass 做"更新"的唯一正规途径。

**最佳实践**：纯数据容器一律 `@dataclass`；不可变配置/快照加 `frozen=True`；敏感字段 `repr=False`；从外部数据构造时用 `__post_init__` 修复 + 显式校验。

---

### 4. ABC 抽象基类 + Protocol 结构化子类型

**① ABC 模板方法（三个扩展点）**

```python
# nanobot/channels/base.py:21-102
class BaseChannel(ABC):
    name: str = "base"

    @abstractmethod
    async def start(self) -> None: ...

    @abstractmethod
    async def stop(self) -> None: ...

    @abstractmethod
    async def send(self, msg: OutboundMessage) -> None: ...

    def is_allowed(self, sender_id: str) -> bool: ...   # 公共逻辑，非抽象
    async def _handle_message(self, ...) -> None: ...   # 模板方法
```

> 解析：`@abstractmethod` 方法子类不实现就无法实例化；基类可含**具体方法**（`is_allowed`/`_handle_message`）构成模板方法骨架。项目 3 个核心扩展点：`BaseChannel`→17 通道、`LLMProvider`（`providers/base.py:308`）→9 backend、`Tool`（`agent/tools/base.py:159`）→22 工具。

**② 抽象属性 + 装饰器顺序**

```python
# nanobot/agent/tools/base.py:171-187
@property
@abstractmethod
def name(self) -> str: ...

@property
@abstractmethod
def parameters(self) -> dict[str, Any]: ...
```

> **易错点**：`@property` 在上、`@abstractmethod` 在下（离函数最近）。顺序反了会报 `TypeError` 或属性不抽象。

**③ Protocol —— 鸭子类型的显式契约**

```python
# nanobot/agent/tools/context.py:45-48 —— 兼容旧工具协议的接口
@runtime_checkable
class ContextAware(Protocol):
    def set_context(self, ctx: RequestContext) -> None: ...
```

> 解析：`Protocol` 声明"只要对象有这些方法就算满足接口"，无需继承；`@runtime_checkable` 允许 `isinstance(obj, ContextAware)` 在运行时探测。项目用它兼容旧式 setter 协议工具（`registry.py:128`）。**易错点**：`runtime_checkable` 的 `isinstance` 只检查方法**存在**，不检查签名。

**④ TypeGuard —— 让类型检查器信任自定义守卫**

```python
# nanobot/agent/tools/self.py:36-56（引用）—— TypeGuard + Protocol 组合
def _is_runtime_control_target(value: object) -> TypeGuard[_RuntimeControlTarget]: ...
```

> `TypeGuard[T]` 声明"此函数返回 True 时，参数可收窄为 T"。自定义 `isinstance` 式校验（如检查 dict 结构）时用 TypeGuard 让类型检查器相信收窄结果。

---

### 5. property 属性

```python
# nanobot/bus/events.py:39-49 —— 派生字段
@property
def session_key(self) -> str:
    return self.session_key_override or f"{self.channel}:{self.chat_id}"

# nanobot/agent/loop.py:194-204 —— 只读状态
@property
def tool_names(self) -> list[str]:
    return self.tools.tool_names
```

> 解析：`property` 把方法伪装成属性（无括号访问），语义是"派生/只读状态"。好处是未来可换成存储字段而不破坏调用方。**易错点**：property 里做重计算或 IO 会误导调用方以为"访问很便宜"（项目 `cli_apps.py:103` 的 `description` property 每次读盘实例化 `CliAppManager`，已被点名批评——反例）。

**最佳实践**：派生值用 property 保持数据单一来源；只读状态用 property 暴露；不要在 property 里放昂贵 IO。

---

### 6. 异常处理

**① 精确捕获 + 兜底记录 + 异常转结果（项目特色）**

```python
# nanobot/agent/runner.py（引用）：工具错误转字符串喂回模型
# nanobot/agent/tools/registry.py:187-201
def execute(self, name, params):
    try:
        return await tool.execute(**params)
    except Exception as e:
        return ToolResult.error(f"Error executing {name}: {e}")   # 错误即值
```

> 项目哲学：**"错误不抛出，转成给 LLM 看的内容"**——`ToolResult` 是携带 `is_error` 标志的 `str` 子类（`agent/tools/base.py:144-156`），错误以内容形式回流让模型自我修正。这是 LLM 应用的特色模式，但注意**别让错误冒充成功回复**（见第九反模式）。

**② CancelledError 与 BaseExceptionGroup**

```python
# nanobot/agent/loop.py:1239-1247 —— 取消信号的特殊处理
except asyncio.CancelledError:
    if not self._running or task_is_cancelling():
        raise
    logger.warning("Ignoring leaked CancelledError while consuming inbound messages")
    continue

# nanobot/agent/loop.py:1487-1490 —— 3.11 异常组
if len(errors) == 1:
    raise errors[0]
if errors:
    raise BaseExceptionGroup("failed to close agent resources", errors)
```

**易错点**
- Python 3.8 起 `CancelledError` 继承自 **`BaseException`** 而非 `Exception`——裸 `except Exception` **捕获不到它**；捕获后要么重抛要么妥善处理，否则取消语义被破坏。
- `asyncio.Task.cancelling()`（3.11+）判断"任务是否正被取消"，项目 `utils/cancellation.py` 封装成 `task_is_cancelling()`。
- `BaseExceptionGroup`（3.11）配合 `except*` 语法；项目在资源清理时**聚合多个清理错误**再统一抛出。

**③ contextlib.suppress —— 优雅忽略**

```python
# nanobot/bus/runtime_events.py:146-148
def _unsubscribe() -> None:
    with contextlib.suppress(ValueError):   # 已移除则忽略
        self._handlers.remove(entry)
```

> `contextlib.suppress(ValueError)` 等价 try/except pass，但意图更清晰——"移除可能不存在的元素"是幂等操作。

**最佳实践**：能精确捕获就精确捕获；`except Exception` 兜底必须记录日志（loguru）；取消单独处理；清理失败聚合抛出（BaseExceptionGroup）；吞异常前问自己"它还能被观测到吗"。

---

### 7. 字符串与 f-string

**① f-string 插值 + 三元 + 切片**

```python
# nanobot/agent/loop.py:1667-1670 —— 长内容截断预览
preview = final_content[:120] + "..." if len(final_content) > 120 else final_content
```

**② 海象运算符 `:=`（3.8+）**

```python
# nanobot/agent/loop.py:1419-1426（引用）—— 赋值表达式
if (mt := self.tools.get("message")) and isinstance(mt, MessageTool) and mt._sent_in_turn:
    ...
```

> 解析：`:=` 在条件表达式里"赋值并复用"，避免先 `mt = self.tools.get(...)` 再判断。**易错点**：海象优先级低于比较运算，注意加括号。

**③ 多行字符串做 prompt 模板 + 大括号转义**

```python
# nanobot/agent/memory.py（引用）—— LLM prompt 模板
prompt = f"""You are a memory consolidation agent. Process this conversation and return a JSON object with exactly two keys:

1. "history_entry": ...
"""
```

> **易错点**：f-string 里的 JSON 示例大括号必须 `{{}}` 转义——这是 f-string 模板最常见翻车点。

**④ UTF-16 代理项修复（Windows 实战）**

```python
# nanobot/utils/helpers.py:33-58
def sanitize_surrogates(text: Any) -> Any:
    if not isinstance(text, str):
        return text
    for ch in text:
        cp = ord(ch)
        if 0xD800 <= cp <= 0xDFFF:
            break
    else:
        return text                       # 快速路径：无代理项，原样返回
    return text.encode("utf-16-le", errors="surrogatepass").decode(
        "utf-16-le", errors="replace"
    )
```

> 解析：Windows 控制台输入的 emoji 会产生孤立 UTF-16 代理项（`U+D800..DFFF`），直接 UTF-8 序列化会抛 `UnicodeEncodeError`。`errors="surrogatepass"` 编码→`errors="replace"` 解码：真代理对重建、孤立代理替换为 `U+FFFD`。**知识点**：`for...else` —— else 在循环**未 break** 时执行（快速路径）；`errors=` 参数是编码/解码的容错开关。`sanitize_surrogates_deep`（:61-97）递归清理 list/dict/tuple，且**只在发生变更时重建容器**（`new_item is not item` 判断身份）避免无谓分配——性能意识范例。

**最佳实践**：拼接优先 f-string；大量拼接用 `"".join(...)`；模板大括号转义；跨平台文本处理记住 surrogate 问题；`for...else` 用于"找得到/找不到"分支。

---

### 8. Pathlib 文件系统操作

**① 全程 Path，零 os.path.join**

```python
# nanobot/session/manager.py:1027-1049 —— 文件名编码
def storage_key(key: str) -> str:
    return base64.urlsafe_b64encode(key.encode("utf-8")).decode("ascii").rstrip("=")
```

> 会话键 `channel:chat_id` 含 `:`、`/` 等非法文件名字符，项目用 **base64url 编码**做存储文件名（无碰撞、可逆），替换了旧版 `_` 替换的有损方案——"文件名安全"的工程答案。

**② 原子写 + fsync（数据安全核心）**

```python
# nanobot/session/manager.py:1239-1279（节选）
def _save_unlocked(self, session, *, fsync=False):
    tmp_path = path.with_name(f".{path.name}.{secrets.token_hex(8)}.tmp")
    try:
        with open(tmp_path, "x", encoding="utf-8") as f:
            f.write(json.dumps(metadata_line, ensure_ascii=False) + "\n")
            for msg in session.messages:
                f.write(json.dumps(msg, ensure_ascii=False) + "\n")
            if fsync:
                f.flush(); os.fsync(f.fileno())
        os.replace(tmp_path, path)          # 原子替换
        if fsync:
            fd = os.open(str(path.parent), os.O_RDONLY)  # fsync 目录确保文件名持久
            ...
    finally:
        tmp_path.unlink(missing_ok=True)
```

> 解析：**"临时文件 + `os.replace` 原子替换"**是防半写文件的黄金模式；`fsync` 把缓冲区刷到磁盘（`open("x")` 排他创建防并发冲突；`secrets.token_hex` 生成随机临时名）。**易错点**：`os.replace` 跨文件系统会失败（所以 tmp 必须同目录）；`missing_ok=True` 让清理幂等。

**③ 安全路径解析（防穿越）**

```python
# nanobot/security/workspace_policy.py（引用）
def resolve_allowed_path(path, base, *, allow_extra=()):
    resolved = Path(path).expanduser().resolve()   # 规范化：解析 .. 与符号链接
    ...
```

> `resolve()` 把相对路径、`..`、符号链接解析成绝对规范路径——**做路径边界校验的前提**（不 resolve 的话 `a/../../etc/passwd` 能绕过前缀检查）。

**最佳实践**：一律 `from pathlib import Path`；读写显式 `encoding="utf-8"`（Windows 默认 gbk 会乱码）；落盘数据走"tmp + os.replace + 可选 fsync"；路径安全校验用 `resolve()` 后的结果。

---

### 9. 模块与包

**① `__init__.py` 双职责：门面 re-export**

```python
# nanobot/bus/__init__.py
from nanobot.bus.events import InboundMessage, OutboundMessage
from nanobot.bus.queue import MessageBus
```

> 调用方写 `from nanobot.bus import MessageBus` 而非深路径——`__init__.py` 作为包的公共门面。

**② PEP 562 —— 模块级 `__getattr__` 惰性导出（本项目亮点）**

```python
# nanobot/__init__.py:57-91（节选）
_LAZY_EXPORTS = {
    "Nanobot": ".nanobot",
    "RequestContext": ".agent.tools.context",
    "StreamEvent": ".nanobot",
    ...
}

def __getattr__(name: str) -> Any:
    module_path = _LAZY_EXPORTS.get(name)
    if module_path is None:
        raise AttributeError(f"module {__name__!r} has no attribute {name!r}")
    from importlib import import_module
    mod = import_module(module_path, __name__)
    val = getattr(mod, name)
    globals()[name] = val
    return val
```

> 解析：模块级 `__getattr__`（PEP 562）在"访问不存在的属性"时被调用——`import nanobot; nanobot.Nanobot` 首次访问时才真正导入 `.nanobot` 模块。**效果**：`import nanobot` 零成本，只有用到某符号才加载对应模块（重依赖延迟）。`globals()[name] = val` 缓存结果避免重复导入。**易错点**：`__all__` 必须与惰性表一致，否则 `from nanobot import *` 行为错乱。

**③ 入口与 `if __name__ == "__main__"`**

```python
# nanobot/__main__.py
from nanobot.cli.entry import main
if __name__ == "__main__":
    main()
```

> `python -m nanobot` 即可运行；配合 `pyproject.toml [project.scripts] nanobot = "nanobot.cli.entry:main"` 安装后生成 `nanobot` 命令。

**④ 函数内惰性导入 + 模块级 `__getattr__`（CLI 快路径）**

```python
# nanobot/cli/entry.py:35-51（引用）：第一个参数是 agent 时走最小依赖路径
# nanobot/cli/agent.py:25-51：模块级 __getattr__ 惰性代理经典栈符号
```

> 项目把"启动性能"当工程问题：`nanobot agent` 只加载必要依赖，重 provider/channel 依赖全部延迟。**易错点**：函数内 import 牺牲静态分析（pyright 支持有限），且每调用一次查一次 sys.modules——热路径注意。

**⑤ importlib / pkgutil 运行时发现**

```python
# nanobot/channels/registry.py:20-27 —— 包扫描
def _channel_package_names() -> list[str]:
    import nanobot.channels as package
    return [name for _, name, is_package in pkgutil.iter_modules(package.__path__)
            if is_package and has_channel_package(name)]
```

> `pkgutil.iter_modules` 枚举包内子模块（不导入）；`importlib.import_module` 动态导入（`channels/plugin.py:79`）。这是**插件自动发现**的基础设施。

---

### 10. 函数进阶

**① 默认参数可变对象陷阱（必考）**

```python
# ❌ 错误：def f(x, lst=[]) —— 所有调用共享同一个列表
# ✅ dataclass 正确写法：media: list[str] = field(default_factory=list)
# ✅ 普通函数正确写法：
def f(x, lst=None):
    if lst is None:
        lst = []
```

**② `**kwargs` 透传**

```python
# nanobot/agent/tools/base.py:227 —— 工具接收 LLM 动态参数
@abstractmethod
async def execute(self, **kwargs: Any) -> Any: ...
# nanobot/agent/tools/registry.py（引用）：解包转发
return await tool.execute(**params)
```

**③ 装饰器 + 注册表（声明式注册）**

```python
# nanobot/providers/registry.py（引用）：provider 注册表是数据表
# nanobot/cli/provider.py（旧版 _LOGIN_HANDLERS 的演化）：
# OAuth 登录处理器按 provider 名分发
```

> 装饰器 + 全局注册表实现"声明式扩展"：新扩展 = 写一个带装饰器的函数/条目，核心零改动。项目当前更倾向**数据表注册**（ProviderSpec），见第三部分 §3.2。

**④ functools.lru_cache —— 记忆化**

```python
# nanobot/channels/plugin.py:134-135 —— 发现结果缓存
@lru_cache(maxsize=None)
def load_channel_package(name: str) -> ChannelPlugin | None:
    ...
```

> `lru_cache` 按参数缓存返回值（无界/有界）。**易错点**：参数必须是可哈希的；缓存对象若可变会引入陈旧状态（项目用它包装"只读发现"函数是安全用法）；`utils/helpers.py:100` 的 `@lru_cache(maxsize=1)` 缓存 tiktoken 编码器（重初始化昂贵）。

**⑤ lambda 与回调**

```python
# nanobot/agent/loop.py:1321 —— task 完成回调
task.add_done_callback(active_tasks.discard)
# nanobot/gateway/runtime.py（引用）：
# monitor_gateway_clients(GatewayClientLease(gateway_runtime, kind="gateway-monitor"), ...)
```

---

### 11. 类型工具：cast / TYPE_CHECKING / TypedDict / TypeGuard

**① `if TYPE_CHECKING:` —— 仅类型检查期导入**

```python
# nanobot/agent/tools/context.py:10-21
if TYPE_CHECKING:
    from nanobot.agent.subagent import SubagentManager
    from nanobot.agent.tools.exec_session import ExecSessionManager
    ...
```

> 解析：`TYPE_CHECKING` 为 True 仅存在于类型检查器眼中——用于**打破运行时循环导入**：A 只在类型注解里引用 B，B 又 import A。项目全树大量使用（`bus/events.py`、`providers/base.py` 等）。**易错点**：类型注解里引用的名字若只在 TYPE_CHECKING 导入，运行时求值会 NameError——所以配合 `from __future__ import annotations`（注解延迟为字符串）使用（`context.py:2` 顶部）。

**② cast —— 让类型检查器相信你**

```python
# nanobot/session/manager.py:186
if not isinstance(cast(object, self.metadata), dict):
```

> `cast(T, x)` 纯声明、零运行时开销，用于"检查器不知道但程序员确定"的场景。**易错点**：cast 是"责任转嫁"——骗过了检查器就要对运行时负责，过度 cast 会让检查形同虚设。

**③ TypedDict —— 字典的结构化类型**

```python
# nanobot/session/manager.py:510-525（引用）
class SessionPayload(TypedDict):
    key: str
    created_at: str
    ...
```

> `TypedDict` 给 dict 定义键与类型，让"字典当结构体"的代码获得补全与检查。**易错点**：TypedDict 是运行时普通 dict 子类，不强制键存在；`total=False` 控制可选键。

---

### 12. 枚举与字面量（Enum vs Literal）

```python
# nanobot/agent/loop.py:112-114
class TurnKind(Enum):
    USER = auto()
    SYSTEM = auto()
```

> `Enum + auto()` 自动编号。项目在**需要运行时取值/遍历**时用 Enum（`TurnKind`），在**纯类型约束**时用 `Literal`（§1③）。**易错点**：`Enum` 成员比较用 `is`（单例）；别把 `Enum` 当 `Literal` 用（多了运行时开销），反之亦然。

---

### 13. ContextVar 请求作用域（本项目进阶重点）

```python
# nanobot/agent/tools/context.py:23-70（全文精华）
_CURRENT_REQUEST_CONTEXT: ContextVar["RequestContext | None"] = ContextVar(
    "nanobot_tool_request_context", default=None,
)

def bind_request_context(ctx: RequestContext) -> Token[RequestContext | None]:
    return _CURRENT_REQUEST_CONTEXT.set(ctx)

def reset_request_context(token: Token[RequestContext | None]) -> None:
    _CURRENT_REQUEST_CONTEXT.reset(token)

@contextmanager
def request_context(ctx: RequestContext):
    token = bind_request_context(ctx)
    try:
        yield ctx
    finally:
        reset_request_context(token)

def current_request_context() -> RequestContext | None:
    return _CURRENT_REQUEST_CONTEXT.get()
```

> 解析：
> - `ContextVar` 是"**任务/协程局部变量**"——每个 async task 有自己的值，互不串扰。项目里**工具实例是共享的**（一个 ToolRegistry 服务所有会话），但请求上下文（channel/chat_id/turn_id）必须按会话隔离，于是放 ContextVar。
> - `set()` 返回 `Token`，`reset(token)` 恢复旧值——**必须成对**（用 `contextmanager` 包成 `with request_context(ctx):` 保证 finally 恢复）。
> - 读取方（工具内部）调 `current_request_context()` 取当前回合上下文。
> - 项目同款用法：`agent/tools/file_state.py`（FileStates 绑定）、`security/workspace_access.py`（WorkspaceScope 绑定）、`message.py`（turn 内发送状态）。

**易错点**
- ContextVar 按**任务**隔离：`asyncio.create_task` 新任务里 set 的值不会污染父任务，但父任务也看不到子任务的 set。
- **忘记 reset 是 bug 温床**：token 泄漏 → 后续回合拿到上一回合的上下文。务必用 `contextmanager`/`try-finally` 包裹。
- ContextVar 是线程安全之外的另一维度：线程池里的 `asyncio.to_thread` 代码看不到事件循环协程的 ContextVar（项目因此把阻塞 IO 尽量留在异步边界内）。

**最佳实践**：跨协程共享的"每请求状态"用 ContextVar；共享单例 + 每请求上下文 = 服务化架构的标准组合；绑定/恢复永远成对。

---

### 14. weakref 弱引用与缓存

```python
# nanobot/agent/loop.py:411-413 —— 会话锁的弱引用缓存
self._session_locks: weakref.WeakValueDictionary[str, asyncio.Lock] = (
    weakref.WeakValueDictionary()
)

def _get_session_lock(self, session_key: str) -> asyncio.Lock:
    lock = self._session_locks.get(session_key)
    if lock is None:
        lock = asyncio.Lock()
        self._session_locks[session_key] = lock
    return lock
```

> 解析：`WeakValueDictionary` 的**值**是弱引用——没有其他强引用时条目自动消失。会话结束、锁不再被任何 task 持有时，条目自动回收，**无需手动清理**。这是"按需缓存的每会话锁"的正确实现（否则 dict 无限膨胀）。
> **易错点**：`WeakValueDictionary` 只弱引用**值**；值必须是可弱引用对象（普通类实例可以，`int`/`str` 等内建不行）；get 时条目可能已被回收（所以要 `if None: 重建`）。`session/manager.py` 还用了 `OrderedDict` LRU 缓存（`SESSION_CACHE_MAX_SIZE`）——两种缓存策略并存，各有适用场景。

---

## 第三部分 面向对象与设计模式

### 3.1 多态 —— 17 个通道同构

```python
# nanobot/channels/telegram/runtime.py（引用）
class TelegramChannel(BaseChannel): ...

# nanobot/channels/websocket/runtime.py（引用）
class WebSocketChannel(BaseChannel): ...
```

核心代码只依赖 `BaseChannel` 抽象，`ChannelManager` 用同一套 `start/stop/send/send_delta` 驱动全部通道——同样的调用、不同的行为。

### 3.2 插件注册表（元数据驱动，项目最佳模式）

**通道版：自包含包 + manifest 声明**

```python
# nanobot/channels/telegram/manifest.py（全文）
PLUGIN = ChannelPlugin(
    name="telegram",
    display_name="Telegram",
    runtime=f"{__package__}.runtime:TelegramChannel",   # 惰性加载的 "module:attr"
    setup=SETUP_SPEC,
    dependencies=("python-telegram-bot[socks,webhooks]>=22.6,<23.0", ...),
    webui="webui/index.ts",
)
```

```python
# nanobot/channels/plugin.py:74-94 —— 用到才导入运行时
def load_channel_class(self) -> type[BaseChannel]:
    module_name, _, attr_name = self.runtime.partition(":")
    module = importlib.import_module(module_name)
    channel_cls = getattr(module, attr_name, None)
    if not (isinstance(channel_cls, type) and issubclass(channel_cls, BaseChannel)):
        raise ImportError(...)
    return channel_cls
```

> 解析：通道是"自包含包"，`manifest.py` 声明元数据（发现阶段**不导入 SDK**），`ChannelPlugin` 的 `__post_init__`（`plugin.py:43-72`）做声明校验（name 正则、runtime 必须指向包内模块、webui 入口存在）。`ChannelManager._init_channels` 用 `pkgutil` 扫描自动装配——**加通道 = 新建一个包，零改动核心**（对比旧版"改 4 处"）。

**Provider 版：数据表 + 查找函数**

```python
# nanobot/providers/registry.py（引用）：~50 个 ProviderSpec 的元数据表
PROVIDERS: tuple[ProviderSpec, ...] = (
    ProviderSpec(name="openrouter", keywords=("openrouter",),
                 env_key="OPENROUTER_API_KEY", is_gateway=True, ...),
    ...
)
```

> "注册表" = 集中存放元数据的容器 + 查找函数（`find_by_name`/`find_by_model`）。效果：加 provider 不需要任何 if-else，只需在表里加一条（核心 2 处改动）。backend 实现里因此**没有 if-elif 长链**。

**最佳实践**：扩展点种类多且差异大 → 注册表；条目用 frozen dataclass；发现（descriptor）与加载（runtime）分离；声明校验前置（`__post_init__`）。

### 3.3 策略模式 + 抽象工厂 —— provider 选择

```python
# nanobot/providers/factory.py:168-231（节选）—— 按 backend 分发构造
if backend == "openai_codex":
    provider = OpenAICodexProvider(default_model=model, ...)
elif backend == "anthropic":
    provider = AnthropicProvider(api_key=..., ...)
elif backend == "bedrock":
    provider = BedrockProvider(region=..., ...)
else:
    provider = OpenAICompatProvider(api_key=..., ...)
```

> `LLMProvider` 抽象 + 7 种 backend 实现 = 策略模式；`make_provider` 是抽象工厂；`FallbackProvider`（`fallback_provider.py:96`）是**装饰器型 provider**——在主 provider 外套一层故障切换。

### 3.4 模板方法 —— 通道消息处理流水线

```python
# nanobot/channels/base.py（引用）：_handle_message 统一做
# 鉴权（is_allowed）→ 配对码回复 → 构造 InboundMessage → bus.publish_inbound
```

所有通道共用这条"鉴权→构造→入队"流水线，鉴权逻辑不重复。

### 3.5 依赖注入与组合根

```python
# nanobot/cli/gateway_runtime.py:426-443（节选）—— 唯一装配点
agent = AgentLoop.from_config(
    config, bus,
    provider=provider_snapshot.provider,
    cron_service=cron,
    session_manager=session_manager,
    runtime_events=runtime_events,
    turn_delivery_factory=turn_delivery_factory,
    ...
)
cron.on_job = on_cron_job          # 后置注入回调，打破循环依赖
```

> 依赖通过构造参数注入（不在内部 `new`），装配集中在组合根 `_run_gateway`。**最佳实践**：回调先建对象再赋值（cron `on_job`）；`create_gateway_app` 工厂把依赖参数化使 CLI 可测（`cli/gateway.py:48-58`）。

### 3.6 适配器 —— MCP 工具包装

```python
# nanobot/agent/tools/mcp.py（引用）：MCPToolWrapper/MCPResourceWrapper/MCPPromptWrapper
class MCPToolWrapper(Tool):
    def __init__(self, session, server_name, tool_def):
        self._name = f"mcp_{server_name}_{tool_def.name}"
```

> 把外部系统（MCP 服务器）接口适配成内部 `Tool` 接口，核心无需感知 MCP 存在。同一模式：`outbound_events._legacy_event_from_metadata`（旧事件桥）、`_LegacyErrorPrefixTool`（旧插件错误契约）。

### 3.7 门面 —— SDK

```python
# nanobot/nanobot.py:66-141
class Nanobot:
    """Programmatic facade for running the nanobot agent."""
    @classmethod
    def from_config(cls, ...) -> Nanobot: ...
    async def run(self, message, ...) -> RunResult: ...
```

> 对外只暴露 `run/run_streamed/stream/sessions/memory/runtime`，内部组装 AgentLoop+MCPProvider+SDK clients——隐藏复杂度。

### 3.8 观察者 —— RuntimeEventBus（进程内 pub/sub）

```python
# nanobot/bus/runtime_events.py:127-169
class RuntimeEventBus:
    def subscribe(self, handler, event_type=None) -> Callable[[], None]:
        entry = (event_type, handler)
        self._handlers.append(entry)
        def _unsubscribe() -> None:
            with contextlib.suppress(ValueError):
                self._handlers.remove(entry)
        return _unsubscribe

    async def publish(self, event: RuntimeEvent) -> None:
        for event_type, handler in list(self._handlers):
            if event_type is not None and not isinstance(event, event_type):
                continue
            try:
                result = handler(event)
                if inspect.isawaitable(result):
                    await result
            except Exception:
                logger.exception(...)
```

> 解析：`subscribe` 返回**退订闭包**（同时是幂等删除）；`publish` 支持同步/异步混合 handler（`inspect.isawaitable` 探测）；按注册顺序 await 保证时序。WebUI 经 `WebuiTurnCoordinator` 订阅它把 turn 生命周期投影成 WS 帧。

### 3.9 不可变值对象

`LLMRuntime`（`utils/llm_runtime.py:14`）、`RequestContext`（`tools/context.py:29`）、`TurnRoute` 都是 `frozen dataclass`——把"一次回合的执行配置"冻结成值，防止运行中漂移（见第二 §3②）。

---

## 第四部分 异步编程 asyncio（重点）

v0.3.0 是**纯 asyncio 应用**，且比旧版多了一大票并发设施。这是全项目最值得学的部分。

### 4.1 async/await 本质

```python
# nanobot/cli/gateway_runtime.py:964-965 —— 唯一入口
with gateway_runtime.foreground_instance(gateway_start_options):
    asyncio.run(run())
```

- `async def` 定义协程；`await` 挂起让出控制权；`asyncio.run()` 是唯一入口。
- **阻塞 ≠ 等待**：`time.sleep()` 卡死事件循环，必须 `await asyncio.sleep()`。**易错点**：事件循环里出现 `time.sleep`/同步 `requests`/同步 `subprocess.run` 都是阻塞事件循环的信号。

### 4.2 create_task / gather / wait_for 三兄弟

```python
# nanobot/channels/manager.py:577-595 —— gather 并发启动通道
tasks: list[asyncio.Task[None]] = []
for name, channel in self.channels.items():
    tasks.append(self._start_channel_task(name, channel))
await asyncio.gather(*tasks, return_exceptions=True)

# nanobot/agent/loop.py:1318-1321 —— create_task 后台跑 + 完成回调
task = asyncio.create_task(self._dispatch(msg))
active_tasks = self._active_tasks.setdefault(effective_key, set())
active_tasks.add(task)
task.add_done_callback(active_tasks.discard)

# nanobot/agent/loop.py:1235 —— wait_for 带超时轮询
msg = await asyncio.wait_for(self.bus.consume_inbound(), timeout=1.0)
```

> 分工：`gather` = 等齐；`create_task` = 后台跑不等；`wait_for` = 只等 N 秒。**易错点**：`gather` 默认一个失败就传播（其他任务仍运行但结果被丢弃）——`return_exceptions=True` 让单个失败不打断整体；`create_task` 的 task 必须被引用（否则可能被 GC 回收，3.11+ 已缓解）。

**任务跟踪模式（项目标准）**

```python
# nanobot/agent/loop.py:1492-1496 —— 后台任务集合防 GC + 关停时 drain
def schedule_background(self, coro) -> None:
    task = asyncio.create_task(coro)
    self._background_tasks.add(task)
    task.add_done_callback(self._background_tasks.discard)
# 关停：await asyncio.gather(*self._background_tasks, return_exceptions=True)
```

### 4.3 asyncio.Queue 生产者/消费者

```python
# nanobot/bus/queue.py:8-44 —— 通道与 Agent 解耦的核心
class MessageBus:
    def __init__(self):
        self.inbound: asyncio.Queue[InboundMessage] = asyncio.Queue()
        self.outbound: asyncio.Queue[OutboundMessage] = asyncio.Queue()

    async def publish_inbound(self, msg) -> None:
        await self.inbound.put(msg)
    async def consume_inbound(self) -> InboundMessage:
        return await self.inbound.get()
```

> `asyncio.Queue` 在单事件循环内天然线程安全；`get()` 空时挂起等待。**易错点**：无界队列无背压——项目 SDK 流用 `asyncio.Queue(maxsize=256)`（`nanobot.py:224`）限背压，而 `MessageBus` 是无界的（报告点名批评）；`QueueFull` 时 `put_nowait` 抛异常要处理（`loop.py:1305` 捕获后回退）。

### 4.4 asyncio.Lock / Semaphore —— 并发控制

```python
# nanobot/agent/loop.py:411-413 + 1325-1340 —— per-session 锁 + 全局并发门
lock = self._get_session_lock(session_key)          # WeakValueDictionary 缓存
gate = self._concurrency_gate or nullcontext()      # Semaphore(3) 或空操作
async with lock, gate:                              # 同时进入两个上下文管理器
    pending = asyncio.Queue(maxsize=20)
    self._pending_queues[session_key] = pending
    ...
# nanobot/agent/loop.py:435-439 —— 全局并发门
_max = int(os.environ.get("NANOBOT_MAX_CONCURRENT_REQUESTS", "3"))
self._concurrency_gate = asyncio.Semaphore(_max) if _max > 0 else None
```

> 解析：**per-session 串行 + 跨会话并发**：同一会话的消息必须排队（锁），不同会话并行（gate 限流总并发）。`async with lock, gate:` 同时进入两个异步上下文管理器（Python 3.10+ 支持 `asyncio.Lock` 的裸 await / async with）。`nullcontext()`（`contextlib`）在 gate 关闭时提供空操作——**用 nullcontext 统一有无门控的代码路径**。

**易错点**：`asyncio.Lock` 不是线程锁；同一任务重入会死锁（无 RLock 语义）；锁持有期间不要做无界等待。

### 4.5 asyncio.Event —— 关停编排

```python
# nanobot/cli/gateway_runtime.py:851-924（节选）—— shutdown_event 编排
shutdown_event = asyncio.Event()
...
shutdown_task = asyncio.create_task(shutdown_event.wait(), name="nanobot-gateway-shutdown")
done, _pending = await asyncio.wait({runtime_tasks, shutdown_task}, return_when=asyncio.FIRST_COMPLETED)
if runtime_tasks in done:
    await runtime_tasks
else:
    runtime_tasks.cancel()
```

> `asyncio.Event`：`wait()` 挂起直到 `set()`。信号处理函数 `set()` 事件 → 主编排 `FIRST_COMPLETED` 任一完成即进入关停。这是"信号驱动优雅关停"的标准姿势。

### 4.6 超时与取消的正确姿势

```python
# nanobot/agent/tools/shell.py（引用）：命令超时杀进程
try:
    stdout, stderr = await asyncio.wait_for(process.communicate(), timeout=self.timeout)
except asyncio.TimeoutError:
    process.kill()
```

```python
# nanobot/agent/loop.py:1239-1247：取消语义的精细处理
except asyncio.CancelledError:
    if not self._running or task_is_cancelling():
        raise            # 真取消：向上传播
    logger.warning("Ignoring leaked CancelledError ...")
    continue
```

**易错点**：`wait_for` 超时**会取消**被等待的协程（不是简单地放弃等待）；协程内部要响应取消做清理（`finally`/`except CancelledError`）。`task.cancelling()`（3.11+）判断任务是否真的在被取消。

### 4.7 异步子进程 + 线程桥接

```python
# nanobot/agent/tools/shell.py（引用）：异步子进程 + 进程树清理
process = await asyncio.create_subprocess_shell(command, stdout=PIPE, stderr=PIPE, ...)
# Unix: start_new_session=True + killpg；Windows: _windows_job.py 的 Job Object
```

```python
# nanobot/channels/email/runtime.py（引用）：同步库扔线程池
inbound_items = await asyncio.to_thread(self._fetch_new_messages)
```

> **易错点**：同步阻塞库（imaplib、boto3 同步 API、tiktoken）在事件循环里直接调用会卡死循环——用 `asyncio.to_thread`（项目 `bedrock_provider.py:743` 把同步 boto3 converse 桥接到线程池）。注意 `to_thread` 里不能碰事件循环对象。

### 4.8 AsyncExitStack —— 动态异步资源管理

```python
# nanobot/agent/tools/mcp.py（引用）：MCP 连接数量运行时才知道
read, write = await stack.enter_async_context(stdio_client(params))
session = await stack.enter_async_context(ClientSession(read, write))
```

> `AsyncExitStack` 动态收集任意多个异步上下文管理器、最后统一 `aclose()`——无法用固定 `async with` 的场景（连接数运行时决定）用它。**易错点**：`enter_async_context` 进入后必须配对退出，否则资源泄漏；项目 `_OwnedMCPConnection` 更进一步让连接归属打开的 task，避免 anyio cancel scope 跨 task 泄漏。

### 4.9 异步生成器 —— 流式处理

```python
# nanobot/providers/openai_responses/parsing.py（引用）
async def consume_sse_with_reasoning(response, ...):
    async for event in iter_sse(response): ...
```

> `async for` + `yield` = 异步生成器：边收网络流边产出事件，内存恒定。`providers/base.py:786-823` 的 `chat_stream()` 默认实现 = 非流式 + 单 delta 回调——**流式回调三件套**：`on_content_delta(text)`/`on_thinking_delta(text)`/`on_tool_call_delta({index, call_id, name, arguments_delta})`，全项目 provider 统一这套回调契约。

### 4.10 contextmanager / asynccontextmanager

```python
# nanobot/agent/tools/context.py:59-66 —— 同步 contextmanager（含 await 体）
@contextmanager
def request_context(ctx: RequestContext):
    token = bind_request_context(ctx)
    try:
        yield ctx
    finally:
        reset_request_context(token)
```

> 注意：`@contextmanager` 的生成器**可以包含 await**（生成器函数是异步的也没关系，因为 await 在调用方的协程里执行）——但若需要"进入时 await"，则必须 `@asynccontextmanager` + `async with`（如 `nanobot/nanobot.py:353` 的 `__aenter__`）。**易错点**：混用两者会 `TypeError: 'async_generator' object is not an iterator`。

### 4.11 竞态教训：单线程 ≠ 无竞态

```python
# nanobot/agent/loop.py（引用）：history.jsonl 的 append 与整文件重写竞态
# append_history 用 threading.Lock（memory.py:108）保护，
# 但 compact_history → _write_entries 是"读全量→os.replace"且不取同一把锁
```

**易错点（本项目最典型）**：asyncio 单线程，但 **await 点之间会切换协程**——两个协程对同一对象"读-改-写"仍是竞态。正确做法：per-session `asyncio.Lock`（项目已用于 turn 处理）、每资源一把锁、或串行化队列。**结论**：并发正确性不因"单线程"而自动成立。

---

## 第五部分 数据、序列化与配置

### 5.1 JSON 与 JSONL

```python
# nanobot/session/manager.py:1243-1261 —— JSONL：首行元数据 + 逐行消息
with open(tmp_path, "x", encoding="utf-8") as f:
    metadata_line = {"_type": "metadata", "key": session.key, ...}
    f.write(json.dumps(metadata_line, ensure_ascii=False) + "\n")
    if session.provider_state is not None:
        f.write(json.dumps({"_type": "provider_state", "state": ...}) + "\n")
    for msg in session.messages:
        f.write(json.dumps(msg, ensure_ascii=False) + "\n")
```

> **JSONL** = 每行一个独立 JSON：可追加写、损坏一行不影响其余、方便 grep。**易错点**：`ensure_ascii=False` 保留中文可读（否则 `\uXXXX` 转义）；逐行读写注意 `\n` 结尾统一。

**脏 JSON 容错（LLM 应用防御）**

```python
# nanobot/providers/base.py:100-120（引用）—— 工具参数解析
def parse_tool_arguments(arguments: Any):
    # 只解析合法 JSON 对象；坏 JSON/数组原样保留交给注册表拒绝
```

> LLM 生成的 JSON 不可信（多逗号/单引号/markdown 包裹）——`json_repair.loads` 修复 + 类型检查（`isinstance(result, dict)`）是标准防御。项目 `providers/base.py` 的 `parse_tool_arguments` 采用"解析失败不抛错、原样保留"的宽容策略。

### 5.2 base64 与 data URL（多模态）

```python
# nanobot/agent/context.py（引用）
b64 = base64.b64encode(p.read_bytes()).decode()
images.append({"type": "image_url", "image_url": {"url": f"data:{mime};base64,{b64}"}})
```

> `base64.b64encode(bytes).decode()` 二进制→文本；拼 `data:image/png;base64,...` 喂多模态模型。`utils/media_decode.py` 做反向：`data:...;base64` 解码落盘（API 与 websocket 共用）。

### 5.3 Pydantic v2 配置体系（config/schema.py 全文精华）

**① 别名生成（camelCase/snake_case 双兼容）**

```python
# nanobot/config_base.py:12-15
class Base(BaseModel):
    model_config = ConfigDict(alias_generator=to_camel, populate_by_name=True)
```

> JSON 里写 `allowFrom`（camelCase），代码里写 `allow_from`（snake_case），两者都认。`AliasChoices` 兼容历史键（`schema.py:444-448` 的 `modelPresets` 等）。

**② BaseSettings + 环境变量前缀**

```python
# nanobot/config/schema.py:674-677
model_config = SettingsConfigDict(
    env_prefix="NANOBOT_", env_nested_delimiter="__", ...
)
```

> `NANOBOT_PROVIDERS__OPENAI__API_KEY` 自动映射 `providers.openai.api_key`——**敏感信息不进配置文件**。

**③ Field 校验与 lazy 默认（破循环导入）**

```python
# nanobot/config/schema.py:387-391
def _lazy_default(module_path: str, attr: str) -> Callable[[], Any]:
    def factory() -> Any:
        module = importlib.import_module(module_path)   # 用到才导入
        return getattr(module, attr)()
    return factory
# schema.py:402-406 —— 工具子配置延迟解析避免 schema→tools 导入环
web: WebToolsConfig = Field(default_factory=lambda: _lazy_default("nanobot.agent.tools.web", "WebToolsConfig"))
```

> **设计巧思**：`schema.py` 与 `tools/*` 互相依赖会成环——用"延迟导入的 factory"让 schema 定义时不需要 tools 已导入。文件尾部 `_resolve_tool_config_refs()`（:680-717）再统一回填模块命名空间。

**④ model_validator / PrivateAttr**

```python
# nanobot/config/schema.py（引用）：timezone 自动探测、model preset 引用完整性校验
# nanobot/config/schema.py:435 —— 实例私有状态
_source_path: Path | None = PrivateAttr(default=None)
```

> `PrivateAttr` 声明"不进序列化/不进 model_dump"的实例状态；`model_validator(mode="after")` 做跨字段校验。**易错点**：Pydantic v2 的 `ConfigDict` 替代 v1 的 `class Config`；`model_validate`/`model_dump` 替代 v1 的 `parse_obj`/`dict`。

### 5.4 tomllib 读取打包元数据

```python
# nanobot/__init__.py:37-43 —— 源树版本回退
def _read_pyproject_version() -> str | None:
    pyproject = Path(__file__).resolve().parent.parent / "pyproject.toml"
    data = tomllib.loads(pyproject.read_text(encoding="utf-8"))
    return data.get("project", {}).get("version")
```

> `tomllib`（3.11+ 标准库）只读解析 TOML。`importlib.metadata.version("nanobot-ai")` 优先，源树 checkout（无 dist-info）回退读 pyproject——**健壮的版本解析双路径**。

### 5.5 dataclass 序列化（camelCase 双读）

```python
# nanobot/cron/types.py + nanobot/triggers/local_types.py（引用）
# from_dict 用 utils/dict_keys.get_camel_snake 兼容 camelCase/snake_case 双键
# to_dict 输出 camelCase —— 数据文件与 WebUI 契约一致
```

```python
# nanobot/utils/dict_keys.py:8-17
def get_camel_snake(data, camel, snake, default=None):
    return data.get(camel, data.get(snake, default))
```

> **易错点**：手写序列化成对出现（from/to），字段一多就易漂移——项目这两处已被点名"模板化重复"，学习时对比 Pydantic 方案思考取舍。

### 5.6 secrets / hmac / hashlib

```python
# nanobot/session/manager.py:1241 —— 随机临时文件名
tmp_path = path.with_name(f".{path.name}.{secrets.token_hex(8)}.tmp")

# nanobot/api/server.py:496（引用）—— 常量时间比较
hmac.compare_digest(token, expected)

# nanobot/pairing/store.py（引用）—— 配对码生成
_ALPHABET = string.ascii_uppercase + string.digits   # 大写字母 + 数字
_CODE_LENGTH = 8
raw = "".join(secrets.choice(_ALPHABET) for _ in range(_CODE_LENGTH))  # 8 位安全随机
code = f"{raw[:4]}-{raw[4:]}"                         # ABCD-EFGH 格式
```

> `secrets`（密码学安全随机）用于安全场景（token/临时名/配对码），**不要**用 `random`（可预测）；`secrets.choice` 从显式字母表取字符（排除易混淆字符如 O/0 是常见增强，本处为全字母+数字）；`hmac.compare_digest` 防时序攻击（普通 `==` 会因字符串比较短路泄露信息）；`hashlib.sha1` 用于指纹（`channels/manager.py:659` 内容去重指纹）。

---

## 第六部分 系统编程、正则与安全

### 6.1 正则表达式实战

**① 剥离思考块（非贪婪）**

```python
# nanobot/utils/helpers.py（引用）
def strip_think(text): return re.sub(r"<think>[\s\S]*?</think>", "", text)
```

**② 占位符保护法（先摘出再还原）**

```python
# nanobot/channels/telegram/runtime.py（引用）：Markdown→HTML 转换
text = re.sub(r'```[\w]*\n?([\s\S]*?)```', save_code_block, text)  # 占位符 \x00CB{n}\x00
# ... 后续规则不会误伤代码块 ... 最后还原
```

**③ 安全黑名单（best-effort）**

```python
# nanobot/agent/tools/shell.py:233-251（引用）
deny_patterns = [r"\brm\s+-[rf]{1,2}\b", r"\bdd\s+if=", r":\(\)\s*\{.*\};\s*:", ...]
```

**易错点**：正则黑名单可被 `rm -fr` 变体、Unicode 同形、`$()` 命令替换绕过——**只能防手滑**；安全必须靠白名单/沙箱/路径边界（项目 `security/` 模块是真正的防线）。

### 6.2 时间与时区

```python
# nanobot/cron/service.py（引用）
tz = ZoneInfo(schedule.tz) if schedule.tz else datetime.now().astimezone().tzinfo
cron = croniter(schedule.expr, datetime.fromtimestamp(base_time, tz=tz))
next_dt = cron.get_next(datetime)
```

> `zoneinfo.ZoneInfo`（3.9+ 标准库，替代 pytz）；时间戳用毫秒整数（`int(time.time()*1000)`）避免浮点精度问题；存 ISO 字符串（`isoformat()`）可排序可解析。

### 6.3 安全编程

**① SSRF 三层防护（security/network.py）**

```python
# nanobot/security/network.py（引用）
_BLOCKED_NETWORKS = (...RFC1918、环回、链路本地、云元数据 169.254.0.0/16、::/128、fc00::/7...)

def resolve_url_target(url, ...):
    # scheme/hostname/DNS 解析 + 逐 IP 校验

# PinnedDNSAsyncTransport：DNS 钉住到已校验的 IP，防 DNS 重绑定
# 逐跳重定向校验（上限 5 跳）
```

> **易错点**：SSRF 防护必须"解析后逐 IP 校验"——只校验域名能被 DNS 重绑定绕过；IPv6-mapped IPv4（`::ffff:127.0.0.1`）需归一化（`network.py:56-68` `_normalize_addr`）。

**② 路径穿越（resolve + 前缀判断）**

```python
# nanobot/security/workspace_policy.py（引用）
resolved = Path(path).expanduser().resolve()
if not is_path_within(resolved, base): raise WorkspaceBoundaryError(...)
```

**③ fail-closed 原则**

```python
# nanobot/pairing/store.py:181-186（引用）：读失败=拒绝
# nanobot/utils/evaluator.py:136-138：评估器失败=不打扰用户
```

> 安全判断失败时**向拒绝侧倒**（fail-closed），而不是放行。

### 6.4 subprocess 与进程管理

```python
# nanobot/agent/tools/_windows_job.py（引用）：ctypes 调用 Windows Job Object
# CreateJobObject + KILL_ON_JOB_CLOSE + 挂起创建 + 主线程恢复 —— 拥有整个进程树
# nanobot/agent/tools/shell.py（引用）：Unix start_new_session=True + killpg
```

> **易错点**：`subprocess.Popen(..., shell=True)` 的子进程树可能逃逸 kill——Windows 用 Job Object、Unix 用进程组（`start_new_session`）才能"连锅端"；僵尸进程要 `_reap_pid`（`os.waitpid`/`communicate`）。

### 6.5 文件锁与跨进程协调

```python
# nanobot/triggers/local_store.py:52（引用）
self._lock = FileLock(str(self.root / ".lock"))
# cron 的 WAL：非运行实例先追加 action.jsonl 操作日志，拥有者下次 tick 合并
```

> `filelock`（第三方）提供跨进程文件锁；cron 用 **WAL 式操作日志**（`service.py:620-628`）让多进程协同改 jobs.json，配合 `_store_dirty` 标志防"旧磁盘快照重放已执行任务"（`service.py:312-313`）——**跨进程一致性**的工程范例。

---

## 第七部分 第三方生态实战

| 库 | 项目用法 | 一句话点评 |
|---|---|---|
| **typer** | `@app.command()` 定义 CLI（`cli/commands.py:85`） | 类型注解驱动 CLI，`create_gateway_app` 工厂注入依赖可测 |
| **rich** | `console.print`/`Table`/`Markdown`/`Live`（cli 全文） | 终端美化 + 流式渲染（`cli/stream.py` 的 `StreamRenderer`） |
| **prompt_toolkit** | `PromptSession` + `SafeFileHistory` + `patch_stdout`（`cli/terminal.py`） | 交互输入与异步输出共存 |
| **loguru** | `logger.info/error/exception` 全项目 | 零配置日志；`logger.bind(channel=...)` 上下文绑定（`channels/base.py:44`） |
| **httpx** | `httpx.AsyncClient(proxy=..., trust_env=False)`（provider 层） | 异步 HTTP；`event_hooks` 做逐请求 SSRF 校验（`mcp.py:299-306`） |
| **pydantic v2** | 配置体系（见 5.3） | 声明式校验 + 别名 + 环境变量 |
| **croniter** | cron 表达式下次执行（`cron/service.py`） | 解析 `0 9 * * *` |
| **mcp** | `stdio_client`/`ClientSession`/`streamablehttp_client` | MCP 工具/资源/提示词三包装 |
| **websockets** | `websockets.serve` + `process_request` 分流（websocket 通道） | WS 服务端 + HTTP 同端口分流 |
| **openai / anthropic SDK** | `AsyncOpenAI`/`AsyncAnthropic`，`max_retries=0` 由基类统一重试 | 原生 SDK + 自有重试层 |
| **json-repair** | `json_repair.loads`（provider/base 引用） | LLM 脏 JSON 修复 |
| **dulwich** | `GitStore`（`utils/gitstore.py`） | **纯 Python Git**：Dream 记忆版本化审计 |
| **watchfiles** | `config/watcher.py` | 配置文件热监听 |
| **filelock** | triggers/session 跨进程锁 | 文件锁标准库替代 |

**httpx 标准姿势**

```python
# nanobot/providers/openai_compat_provider.py:561-581（引用）
# 本地端点禁用 keepalive 并关代理；远程走 proxy + trust_env=False
client = httpx.AsyncClient(proxy=..., trust_env=False, event_hooks={"request": [...]})
```

**易错点**：`trust_env=False` 表示**不自动读环境代理**（防代理污染）；`event_hooks` 在每请求前挂 SSRF 校验钩子；`asyncio.to_thread` 里的同步 httpx 与异步 httpx 不能混用同一 client。

---

## 第八部分 工程化与测试

### 8.1 pyproject.toml（现代打包单一入口）

```toml
[project]
name = "nanobot-ai"
version = "0.3.0"
requires-python = ">=3.11"
dependencies = ["typer>=0.20.0", "anthropic>=0.100.0", "websockets>=15.0,<17.0", ...]

[project.optional-dependencies]   # 可选依赖分组：api/azure/bedrock/dev/...
dev = ["pytest>=9.0.0", "pytest-asyncio>=1.3.0", "ruff>=0.1.0", "basedpyright>=1.39.0", ...]

[project.scripts]
nanobot = "nanobot.cli.entry:main"

[tool.hatch.build.hooks.custom]   # hatch_build.py 自定义构建（打包 WebUI 产物）
artifacts = ["nanobot/web/dist/**/*"]   # git-ignored 但进 wheel
```

> **要点**：`[project.scripts]` 生成命令行入口；optional-dependencies 区分场景；hatch 把 `webui` 构建产物（git-ignored）打进 wheel（`artifacts` 列表）。

### 8.2 静态检查与格式化

```toml
[tool.ruff.lint]
select = ["E", "F", "I", "N", "W"]   # 错误/未使用/导入排序/命名/警告
ignore = ["E501"]                     # 行长度 100

[tool.basedpyright]
include = ["nanobot"]
typeCheckingMode = "strict"           # 严格模式
```

> 项目类型注解完备到 **basedpyright strict 零错误**——这是学习类型系统的极佳范本。

### 8.3 测试（pytest + pytest-asyncio，300+ 文件）

```toml
[tool.pytest.ini_options]
asyncio_mode = "auto"                 # 自动识别 async 测试
testpaths = ["tests", "nanobot/channels"]

[tool.coverage.report]
fail_under = 75                       # 覆盖率门槛
```

**可测试性设计（值得学）**

```python
# nanobot/cli/gateway.py:48-58 —— 工厂注入依赖
def create_gateway_app(console, log_handler_id, load_runtime_config, run_gateway,
                       runtime_factory, service_factory, prepare_webui_bundle):
    ...
```

> 把副作用全部参数化 → 测试用假实现替换真实依赖（`tests/cli/test_gateway_commands.py`）。**测试分层**：纯逻辑（工具校验/consolidate 偏移）→ 模块级（session/cron）→ 集成（loop/runner）→ 协议级（websocket）。项目测试从旧版 6 个文件反转为 300+，是"测试驱动成长"的范例。

---

## 第九部分 实战教训与反模式（v0.3.0 更新版）

学习"别人踩过的坑"比学习"正确的写法"更能长记性。以下按**当前代码**重述，标注哪些是已修复、哪些仍存在：

### ❌ 反模式 1：上帝对象（God Object）
`AgentLoop`（`agent/loop.py`，2,419 行、`__init__` ~40 参数）仍承担调度/持久化/检查点/命令/协调/运行时选择 8 类职责；`AgentRunner._run_core`（`runner.py:420-874`）是 450 行单方法。
**已缓解**：执行循环已下沉到 `AgentRunner`、turn 处理拆成 7 阶段流水线。**教训**：职责单一 + 阶段拆分是驯服大类的两板斧。

### ❌ 反模式 2：代码复制（DRY）
**已修复**：子代理不再复制主循环——`SubagentManager` 直接 `self.runner.run(AgentRunSpec(...))`（`subagent.py:405`）。
**仍存在**：`_merge_message_content`（context.py vs runner.py 逐行相同）、token 预算计算两处、原子写+fsync 在 6 个模块各写一遍、`_handle_error` 在 3 个 provider 近似重复。

### ❌ 反模式 3：全局可变状态
`os.environ["AWS_BEARER_TOKEN_BEDROCK"] = api_key`（`bedrock_provider.py:72-73`）、Copilot 运行时改写 `client.api_key`、模块级 `AsyncOpenAI` 全局占位。**教训**：进程级副作用让"多实例并存"不可能，且 env 可被 /proc 读取。

### ❌ 反模式 4：错误转字符串（语义丢失）
provider 异常 → `content="Error calling LLM: ..."` 的 LLMResponse；API 层 `_response_text` 只取 `.content`，error 以 HTTP 200 + 错误文本返回。**已部分修复**：`finish_reason="error"` + 结构化错误元数据已进重试/熔断控制流。**教训**：错误要有类型/状态，别让错误文本冒充回复。

### ❌ 反模式 5：正则黑名单安全
`ExecTool` deny 正则（`shell.py:233-251`）可被 `rm -fr` 变体/Unicode 同形/`$()` 绕过。**已改善**：新增 allow 白名单优先、内网 URL 检测、环境最小化、Windows Job Object 进程树。**教训**：黑名单防手滑，白名单/沙箱/路径边界才是防线。

### ❌ 反模式 6：字符串协议（Stringly Typed）
会话键 `channel:chat_id` 的拼接/拆分散落多处；`Session.metadata` 内部 `_` 保留键靠约定。**已改善**：`session/keys.py` 集中构造、base64url 存储键。**教训**：协议用常量/枚举/专用函数表达。

### ❌ 反模式 7：并发无锁修改
**已修复（turn 层）**：per-session `asyncio.Lock` + 并发门 + mid-turn 注入队列。
**仍存在**：`history.jsonl` append（threading.Lock）与整文件重写（os.replace）不共用锁；`AgentLoop._current_iteration` 跨会话共享；后台压缩与下一 turn 并发写 session。**教训**：单线程也有竞态（await 切换点），每资源一把锁。

### ❌ 反模式 8：死代码与内省式兼容
`loader.py::_resolve_env_vars` 定义未用；`cli/models.py` 空壳但 `onboard.py` 仍调用（功能静默失效）；`inspect.signature` 探测回调参数 3 处。**教训**：兼容层要标记生命周期，内省式探测脆弱（显式接口更可靠）。

### ✅ 值得学习的正面实践（v0.3.0）
- **插件注册表三件套**（通道/工具/provider）：发现与加载分离、声明校验前置、加扩展零改核心。
- **错误即值 + 统一重试 + 熔断**：429 分类/欠费识别/图片剥离重试全收敛在基类 `_run_with_retry`。
- **数据安全**：原子写 + fsync + WAL 防重放 + 快照校验迁移 + fail-closed。
- **不可变运行时快照**：`LLMRuntime` 冻结配置防回合中漂移。
- **ContextVar 请求作用域**：共享单例 + 每请求上下文。
- **优雅关停**：Event + FIRST_COMPLETED + 后台任务 drain + 资源 finally 清理。
- **双流式链路**：channel（bus 事件）与 SDK（StreamEvent）分离，文本不重复。

---

## 第十部分 学习路径建议

### 阶段一：语言地基（1-2 周）
1. 精读 [第二部分](#第二部分-语言基础) §1-§12：类型注解、dataclass、ABC/Protocol、异常、Pathlib、模块。
2. 练习：抄写 `bus/events.py` + `utils/llm_runtime.py` 并改造（加字段、加校验、改 frozen）。
3. 练习：把 `bus/runtime_events.py` 的 pub/sub 手写一遍（subscribe 返回退订闭包）。

### 阶段二：进阶语法（1 周）
1. 精读 §13-§14：ContextVar、weakref、TypeGuard、PEP 562 惰性导出。
2. 练习：用 ContextVar 给"共享工具实例 + 每请求上下文"写最小示例；给类实现 `__getattr__` 惰性属性。
3. 练习：`weakref.WeakValueDictionary` 写"每会话锁"并验证自动回收。

### 阶段三：asyncio（2-3 周，本项目核心）
1. 精读 [第四部分](#第四部分-异步编程-asyncio重点)，逐个知识点手写最小示例：
   ```python
   # 练习：per-session 锁 + 并发门
   import asyncio
   async def worker(name, lock, gate):
       async with lock, gate:
           await asyncio.sleep(0.1)
           print(name)
   async def main():
       lock, gate = asyncio.Lock(), asyncio.Semaphore(2)
       await asyncio.gather(*(worker(i, lock, gate) for i in range(5)))
   asyncio.run(main())
   ```
2. 练习：用 `asyncio.create_subprocess_exec` + `start_new_session` 写"超时杀进程树"工具。
3. 进阶：读懂 `providers/base.py` 的 `_run_with_retry`（重试引擎）与 `openai_responses/parsing.py`（SSE 状态机）。

### 阶段四：数据与配置（1 周）
1. 精读 [第五部分](#第五部分-数据序列化与配置)。
2. 练习：用 Pydantic v2 写一个带 `alias_generator` + `env_prefix` + `model_validator` 的配置模型；用"tmp + os.replace + fsync"实现原子 JSONL 存储。

### 阶段五：安全与系统编程（1 周）
1. 精读 [第六部分](#第六部分-系统编程正则与安全)：SSRF 三层、路径 resolve、fail-closed、进程树。
2. 练习：给 `security/network.py` 的 `resolve_url_target` 写测试（RFC1918/环回/云元数据/IPv6-mapped 用例）。

### 阶段六：工程化与实战改造（1-2 周）
1. 精读 [第八部分](#第八部分-工程化与测试)；跑 `ruff check nanobot/`、`basedpyright` 体验静态检查。
2. 练习：给 `agent/tools/base.py` 的 `validate_params` 写 pytest（覆盖全部校验分支）。
3. 毕业设计（任选）：写一个新 `Tool` 并注册（走 pkgutil 自动发现）；写一个新通道包（manifest.py + runtime.py）；给 `MessageBus` 加有界队列背压；修复 `history.jsonl` 的 append/rewrite 竞态（统一锁）。

### 推荐资源
- 官方教程：https://docs.python.org/3/tutorial/
- asyncio 权威指南：https://docs.python.org/3/library/asyncio.html
- Real Python asyncio：https://realpython.com/async-io-python/
- 类型检查：https://mypy.readthedocs.io/ ｜ https://mypy-play.net/
- Pydantic v2：https://docs.pydantic.dev/
- Typer：https://typer.tiangolo.com/

---

*本指南由 AI 基于 nanobot 当前源码（`F:\ai-agent\nanobot\nanobot`，v0.3.0）分析生成，所有代码引用均来自当前代码库并经验证。配套架构分析见 `PROJECT_ANALYSIS_V2.md`。*
