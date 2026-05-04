+++
date = '2026-05-04T12:51:00+08:00'
draft = false
title = 'Gunicorn Dirty Arbiters：异步脏调度器详解'
author = 'synodriver'
+++
# Gunicorn Dirty Arbiters：异步脏调度器详解

> **Beta 特性**：Dirty Arbiters 是 Gunicorn 25.0.0 中引入的测试版功能。虽然已经过测试，但 API 和行为可能在后续版本中变更。如有问题请在 GitHub 上反馈。

## 引言

Dirty Arbiters（脏调度器）为 Gunicorn 提供了一个**独立于 HTTP Worker 的进程池**，专门用于执行长时间运行的阻塞操作（如 AI 模型加载、重度计算），而不会阻塞 HTTP Worker。其核心思想源自 Erlang 的脏调度器（dirty schedulers），通过进程隔离、消息传递 IPC 和 asyncio 事件循环，实现了以下关键能力：

- **进程隔离**：脏 Worker 崩溃或内存泄漏不影响 HTTP Worker
- **有状态计算**：资源（如 ML 模型）在 Worker 生命周期内常驻内存，避免重复加载
- **流式传输**：支持生成器/异步生成器的 token 级流式响应
- **共享状态**：通过 Stash 机制实现跨 Worker 的消息传递式状态共享
- **动态伸缩**：支持运行时通过信号动态增减 Worker 数量

---

## 概述

传统 Gunicorn Worker 设计用于快速处理 HTTP 请求。长时间运行的操作（如加载 ML 模型或执行重度计算）会阻塞这些 Worker，降低服务器处理并发请求的能力。

Dirty Arbiters 通过提供以下能力解决此问题：

- **独立 Worker 池** — 完全独立于 HTTP Worker，可以独立杀死/重启
- **有状态 Worker** — 已加载的资源常驻在脏 Worker 内存中
- **消息传递 IPC** — 通过 Unix 套接字使用二进制 TLV 协议通信
- **显式 API** — 清晰的 `execute()` 调用（无隐藏 IPC）
- **基于 asyncio** — 干净的并发处理，支持流式传输

---

## 设计哲学

### 独立进程层次

与线程或进程内池不同，Dirty Arbiters 使用完全独立的进程树：

- **隔离性** — 脏 Worker 中的崩溃或内存泄漏不会影响 HTTP Worker
- **独立生命周期** — 脏 Worker 可以独立杀死/重启，不影响请求处理
- **资源核算** — 可对每个进程应用操作系统级内存限制
- **干净关闭** — 每个进程树可以独立发信号和终止

### Erlang 启发

名称和概念源自 Erlang 的"脏调度器"——专门处理会阻塞正常调度器的操作的特殊调度器。在 Erlang 中，脏调度器运行无法让步的 NIF（原生实现函数）。类似地，Gunicorn 的 Dirty Arbiters 处理会阻塞 HTTP Worker 的 Python 操作。

### 为什么使用 asyncio

Dirty Arbiter 使用 asyncio 作为核心循环，而非主 Arbiter 的基于 select 的方式：

- **非阻塞 IPC** — 可高效处理多个并发客户端连接
- **并发请求路由** — 多个请求可以同时分派到 Worker
- **流式传输支持** — 原生异步生成器用于流式响应
- **干净信号处理** — 信号通过 `loop.add_signal_handler()` 干净地集成

### 有状态应用

传统 WSGI 应用是请求作用域的——按请求调用，请求之间不维护状态。脏应用则不同：

- **长期存活** — 应用在 Worker 生命周期内持久驻留
- **预加载资源** — 模型、连接和缓存保持加载
- **显式状态管理** — 应用通过 `init()` 和 `close()` 控制自己的生命周期

这使脏应用非常适合 ML 推理场景——加载一次模型并为多个请求复用。

---

## 架构

```
                     +-------------------+
                     |   Main Arbiter    |
                     | (manages both)    |
                     +--------+----------+
                              |
                SIGTERM/SIGHUP/SIGUSR1 (forwarded)
                              |
       +----------------------+----------------------+
       |                                             |
 +-----v-----+                                +------v------+
 | HTTP      |                                | Dirty       |
 | Workers   |                                | Arbiter     |
 +-----------+                                +------+------+
       |                                             |
       |    Unix Socket IPC                   SIGTERM/SIGHUP
       |    /tmp/gunicorn_dirty_<pid>.sock          |
       +------------------>---------------------->---+
                                     +-----------+-----------+
                                     |           |           |
                               +-----v---+ +-----v---+ +-----v---+
                               | Dirty   | | Dirty   | | Dirty   |
                               | Worker  | | Worker  | | Worker  |
                               +---------+ +---------+ +---------+
                                  ^   |        ^   |       ^   |
                                  |   |        |   |       |   |
                            Heartbeat (mtime every dirty_timeout/2)
                                  |   |        |   |       |   |
                                  +---+--------+---+-------+---+
                                               |
                             Workers load apps based on allocation
                             Worker 1: [MLApp, ImageApp, HeavyApp]
                             Worker 2: [MLApp, ImageApp, HeavyApp]
                             Worker 3: [MLApp, ImageApp]  (HeavyApp workers=2)
```

### 进程关系

| 组件 | 父进程 | 通信方式 |
|------|--------|----------|
| Main Arbiter | init/systemd | 操作系统信号 |
| HTTP Workers | Main Arbiter | 管道、信号 |
| Dirty Arbiter | Main Arbiter | 信号、退出状态 |
| Dirty Workers | Dirty Arbiter | Unix 套接字、信号、WorkerTmp |

---

## 配置

在 Gunicorn 配置文件或命令行中添加以下设置：

```python
# gunicorn.conf.py
dirty_apps = [
    "myapp.ml:MLApp",
    "myapp.images:ImageApp",
]
dirty_workers = 2          # 脏 Worker 数量
dirty_timeout = 300        # 任务超时时间（秒）
dirty_threads = 1          # 每个 Worker 的线程数
dirty_graceful_timeout = 30  # 优雅关闭超时时间
```

或通过命令行：

```bash
gunicorn myapp:app \
    --dirty-app myapp.ml:MLApp \
    --dirty-app myapp.images:ImageApp \
    --dirty-workers 2 \
    --dirty-timeout 300
```

### 配置选项

| 设置 | 默认值 | 描述 |
|------|--------|------|
| `dirty_apps` | `[]` | 脏应用导入路径列表 |
| `dirty_workers` | `0` | 脏 Worker 数量（0 = 禁用） |
| `dirty_timeout` | `300` | 任务超时时间（秒） |
| `dirty_threads` | `1` | 每个脏 Worker 的线程数 |
| `dirty_graceful_timeout` | `30` | 优雅关闭超时时间 |

---

## 每应用 Worker 分配

默认情况下，所有脏 Worker 加载所有已配置的应用。对于消耗大量内存的应用（如大型 ML 模型），可以限制加载特定应用的 Worker 数量。

### 为什么要每应用分配？

考虑一个 10GB ML 模型和 8 个脏 Worker 的场景：

- 默认行为：8 个 Worker × 10GB = **80GB 内存**
- 设置 `workers=2`：2 个 Worker × 10GB = **20GB 内存**（节省 75%）

对受限应用的请求仅路由到已加载该应用的 Worker。

### 配置方法

**方法一：类属性**

```python
from gunicorn.dirty import DirtyApp

class HeavyModelApp(DirtyApp):
    workers = 2  # 仅 2 个 Worker 加载此应用

    def init(self):
        self.model = load_10gb_model()

    def predict(self, data):
        return self.model.predict(data)

    def close(self):
        pass
```

**方法二：配置覆盖**

在配置中使用 `module:class:N` 格式：

```python
# gunicorn.conf.py
dirty_apps = [
    "myapp.light:LightApp",           # 所有 Worker（默认）
    "myapp.heavy:HeavyModelApp:2",    # 仅 2 个 Worker
    "myapp.single:SingletonApp:1",    # 仅 1 个 Worker
]
dirty_workers = 4
```

配置覆盖优先于类属性。

### Worker 分配

当 Worker 启动时，应用根据其限制进行分配：

以 `dirty_workers=4` 为例：
- LightApp (workers=None)：加载到 Worker 1, 2, 3, 4
- HeavyModelApp (workers=2)：加载到 Worker 1, 2
- SingletonApp (workers=1)：加载到 Worker 1

```
Worker 1: [LightApp, HeavyModelApp, SingletonApp]
Worker 2: [LightApp, HeavyModelApp]
Worker 3: [LightApp]
Worker 4: [LightApp]
```

### 请求路由

请求自动路由到拥有目标应用的 Worker：

```python
client = get_dirty_client()

# 可路由到 4 个 Worker 中的任意一个（轮询）
client.execute("myapp.light:LightApp", "action")

# 仅路由到 Worker 1 或 2（在这些 Worker 间轮询）
client.execute("myapp.heavy:HeavyModelApp", "predict", data)

# 始终路由到 Worker 1
client.execute("myapp.single:SingletonApp", "process")
```

### 错误处理

如果没有 Worker 加载了请求的应用，将抛出 `DirtyNoWorkersAvailableError`：

```python
from gunicorn.dirty import get_dirty_client
from gunicorn.dirty.errors import DirtyNoWorkersAvailableError

def my_view(request):
    client = get_dirty_client()
    try:
        result = client.execute("myapp.heavy:HeavyModelApp", "predict", data)
    except DirtyNoWorkersAvailableError as e:
        # 拥有此应用的所有 Worker 都已宕机，或应用未配置
        return {"error": "Service temporarily unavailable", "app": e.app_path}
```

### Worker 崩溃恢复

当 Worker 崩溃时，其替代者会获得与已死 Worker 相同的应用：

```
时间线：
  t=0: Worker 1 崩溃（拥有 HeavyModelApp）
  t=1: Arbiter 检测到崩溃，排队重启
  t=2: 新 Worker 5 启动，加载与 Worker 1 相同的应用
  t=3: 空档期间 HeavyModelApp 仍在 Worker 2 上可用
```

这确保了：

- 现有 Worker 不需要重新分配内存
- 可预测的替换行为
- 重模型仅在新 Worker 上加载

### 最佳实践

1. **设置合理的限制** — 除非必要，不要设置 `workers=1`（单点故障）
2. **监控内存** — 跟踪每个 Worker 的内存以调整分配
3. **处理不可用** — 优雅地捕获 `DirtyNoWorkersAvailableError`
4. **使用类属性设置应用特定的限制** — 使限制成为应用定义的一部分
5. **使用配置进行部署特定的覆盖** — 开发和生产环境使用不同的限制

---

## 创建脏应用

脏应用继承自 `DirtyApp` 并实现三个方法：

```python
# myapp/dirty.py
from gunicorn.dirty import DirtyApp

class MLApp(DirtyApp):
    """ML 工作负载的脏应用。"""

    def __init__(self):
        self.models = {}

    def init(self):
        """在脏 Worker 启动时调用一次。"""
        # 预加载常用模型
        self.models['default'] = self._load_model('base-model')

    def __call__(self, action, *args, **kwargs):
        """分发到 action 方法。"""
        method = getattr(self, action, None)
        if method is None:
            raise ValueError(f"Unknown action: {action}")
        return method(*args, **kwargs)

    def load_model(self, name):
        """将模型加载到内存。"""
        if name not in self.models:
            self.models[name] = self._load_model(name)
        return {"loaded": True, "name": name}

    def inference(self, model_name, input_text):
        """在已加载的模型上运行推理。"""
        model = self.models.get(model_name)
        if not model:
            raise ValueError(f"Model not loaded: {model_name}")
        return model.predict(input_text)

    def _load_model(self, name):
        import torch
        model = torch.load(f"models/{name}.pt")
        return model

    def close(self):
        """关闭时清理。"""
        for model in self.models.values():
            del model
```

### DirtyApp 接口

| 方法/属性 | 描述 |
|-----------|------|
| `workers` | 类属性。加载此应用的 Worker 数量（None = 所有 Worker） |
| `init()` | 脏 Worker 启动时调用一次（实例化之后）。在此加载资源 |
| `__call__(action, *args, **kwargs)` | 处理来自 HTTP Worker 的请求 |
| `close()` | 脏 Worker 关闭时调用。清理资源 |

### 初始化序列

脏 Worker 启动时，初始化按以下顺序进行：

1. **Fork** — Worker 进程从 Dirty Arbiter fork
2. **dirty_post_fork(arbiter, worker)** — fork 后立即调用钩子
3. **应用实例化** — 每个脏应用类被实例化（`__init__`）
4. **app.init()** — 实例化后为每个应用调用（加载模型、资源）
5. **dirty_worker_init(worker)** — 所有应用完成初始化后调用钩子
6. **运行循环** — Worker 开始接受来自 HTTP Worker 的请求

这意味着：

- 使用 `__init__` 进行基本设置（初始化空容器、存储配置）
- 使用 `init()` 进行重度加载（ML 模型、数据库连接、大文件）
- `dirty_worker_init` 钩子仅在所有应用完成其 `init()` 调用后才触发

---

## 从 HTTP Worker 调用

### 同步 Worker（sync, gthread）

```python
from gunicorn.dirty import get_dirty_client

def my_view(request):
    client = get_dirty_client()

    # 加载模型
    client.execute("myapp.ml:MLApp", "load_model", "gpt-4")

    # 运行推理
    result = client.execute(
        "myapp.ml:MLApp",
        "inference",
        "gpt-4",
        input_text=request.data
    )
    return result
```

### 异步 Worker（ASGI）

```python
from gunicorn.dirty import get_dirty_client_async

async def my_view(request):
    client = await get_dirty_client_async()

    # 非阻塞执行
    await client.execute_async("myapp.ml:MLApp", "load_model", "gpt-4")

    result = await client.execute_async(
        "myapp.ml:MLApp",
        "inference",
        "gpt-4",
        input_text=request.data
    )
    return result
```

---

## 流式传输

Dirty Arbiters 支持流式响应，适用于 LLM token 生成等增量产生数据的场景，无需等待完整执行即可实时交付结果。

### 使用生成器进行流式传输

任何返回生成器（同步或异步）的脏应用 action 会自动将数据块流式传输到客户端：

```python
# myapp/llm.py
from gunicorn.dirty import DirtyApp

class LLMApp(DirtyApp):
    def init(self):
        from transformers import pipeline
        self.generator = pipeline("text-generation", model="gpt2")

    def generate(self, prompt):
        """同步流式传输 - 产出 token。"""
        for token in self.generator(prompt, stream=True):
            yield token["generated_text"]

    async def generate_async(self, prompt):
        """异步流式传输 - 产出 token。"""
        import openai
        client = openai.AsyncOpenAI()
        stream = await client.chat.completions.create(
            model="gpt-4",
            messages=[{"role": "user", "content": prompt}],
            stream=True
        )
        async for chunk in stream:
            if chunk.choices[0].delta.content:
                yield chunk.choices[0].delta.content

    def close(self):
        pass
```

### 客户端流式 API

同步 Worker 使用 `stream()`，异步 Worker 使用 `stream_async()`：

**同步 Worker（sync, gthread）：**

```python
from gunicorn.dirty import get_dirty_client

def generate_view(request):
    client = get_dirty_client()

    def generate_response():
        for chunk in client.stream("myapp.llm:LLMApp", "generate", request.prompt):
            yield chunk

    return StreamingResponse(generate_response())
```

**异步 Worker（ASGI）：**

```python
from gunicorn.dirty import get_dirty_client_async

async def generate_view(request):
    client = await get_dirty_client_async()

    async def generate_response():
        async for chunk in client.stream_async("myapp.llm:LLMApp", "generate", request.prompt):
            yield chunk

    return StreamingResponse(generate_response())
```

### 流式传输协议

流式传输使用包含三种消息类型的简单协议：

1. **Chunk**（type: "chunk"）— 包含部分数据
2. **End**（type: "end"）— 标识流完成
3. **Error**（type: "error"）— 标识流传输过程中的错误

```
Client -> Arbiter -> Worker: request
Worker -> Arbiter -> Client: chunk (data: "Hello")
Worker -> Arbiter -> Client: chunk (data: " ")
Worker -> Arbiter -> Client: chunk (data: "World")
Worker -> Arbiter -> Client: end
```

---

## 二进制协议

脏 Worker IPC 使用受 OpenBSD msgctl/msgsnd 启发的二进制协议，用于高效数据传输。这消除了图像、音频或模型权重等二进制数据的 base64 编码开销。

### 头部格式（16 字节）

```
+--------+--------+--------+--------+--------+--------+--------+--------+
|  Magic (2B)     | Ver(1) | MType  |        Payload Length (4B)        |
+--------+--------+--------+--------+--------+--------+--------+--------+
|                       Request ID (8 bytes)                            |
+--------+--------+--------+--------+--------+--------+--------+--------+
```

- **Magic**: `0x47 0x44`（"GD"，代表 Gunicorn Dirty）
- **Version**: `0x01`
- **MType**: 消息类型（`0x01`=REQUEST, `0x02`=RESPONSE, `0x03`=ERROR, `0x04`=CHUNK, `0x05`=END）
- **Length**: 负载大小（大端序 uint32，最大 64MB）
- **Request ID**: uint64 标识符

### TLV 负载编码

负载数据使用 Type-Length-Value 编码：

| 类型 | 代码 | 描述 |
|------|------|------|
| None | `0x00` | 无值字节 |
| Bool | `0x01` | 1 字节（`0x00`/`0x01`） |
| Int64 | `0x05` | 8 字节大端序有符号整数 |
| Float64 | `0x06` | 8 字节 IEEE 754 浮点数 |
| Bytes | `0x10` | 4 字节长度 + 原始字节 |
| String | `0x11` | 4 字节长度 + UTF-8 编码 |
| List | `0x20` | 4 字节计数 + 元素 |
| Dict | `0x21` | 4 字节计数 + 键值对 |

### 二进制数据优势

二进制协议允许直接传递原始字节而无需编码：

```python
# 使用二进制数据进行图像处理
def resize(self, image_data, width, height):
    """调整图像大小 - image_data 是原始字节。"""
    img = Image.open(io.BytesIO(image_data))
    resized = img.resize((width, height))
    buffer = io.BytesIO()
    resized.save(buffer, format='PNG')
    return buffer.getvalue()  # 返回原始字节

# 从 HTTP Worker 调用
thumbnail = client.execute(
    "myapp.images:ImageApp",
    "thumbnail",
    raw_image_bytes,  # 无需 base64 编码
    size=256
)
```

### 流中的错误处理

流式传输过程中的错误以错误消息形式传递：

```python
def generate_view(request):
    client = get_dirty_client()

    try:
        for chunk in client.stream("myapp.llm:LLMApp", "generate", prompt):
            yield chunk
    except DirtyError as e:
        # 流传输中途发生错误
        yield f"\n[Error: {e.message}]"
```

### 流式传输最佳实践

1. **I/O 密集型流式传输使用异步生成器** — 例如调用外部服务的 API
2. **CPU 密集型流式传输使用同步生成器** — 例如本地模型推理
3. **频繁产出** — 流式传输期间会发送心跳以保持 Worker 存活
4. **保持数据块较小** — 更小的数据块提供更好的感知延迟
5. **处理客户端断连** — 即使客户端断开连接，流也会继续；需相应设计

---

## Stash（通过消息传递实现的共享状态）

Stash 提供脏 Worker 之间的共享状态，类似 Erlang 的 ETS（Erlang Term Storage）。Worker 保持完全隔离——所有状态访问通过消息传递到 Arbiter。

### 架构

```
                +------------------+
                |   Dirty Arbiter  |
                |                  |
                |  stash_tables:   |
                |    sessions: {}  |
                |    cache: {}     |
                +--------+---------+
                         |
          Unix Socket IPC (message passing)
                         |
     +-------------------+-------------------+
     |                   |                   |
+-----v-----+       +-----v-----+       +-----v-----+
|  Worker 1 |       |  Worker 2 |       |  Worker 3 |
|           |       |           |       |           |
| (isolated)|       | (isolated)|       | (isolated)|
+-----------+       +-----------+       +-----------+

Workers have NO shared memory.
All stash operations are IPC messages to arbiter.
```

### 工作原理

1. Worker 调用 `stash.put("sessions", "user:1", data)`
2. Worker 通过 Unix 套接字向 Arbiter 发送消息
3. Arbiter 在其内存中存储数据（`self.stash_tables`）
4. Arbiter 向 Worker 发送响应
5. Worker 接收确认

这不是共享内存——Worker 保持完全隔离。Arbiter 充当中心化存储，Worker 通过消息传递与其通信。这与 Erlang 的模型一致，ETS 表由一个进程拥有。

### 基本用法

```python
from gunicorn.dirty import stash

# 存储值（表自动创建）
# 向 Arbiter 发送消息，由 Arbiter 存储
stash.put("sessions", "user:123", {"name": "Alice", "role": "admin"})

# 检索值
# 向 Arbiter 发送请求，Arbiter 返回值
user = stash.get("sessions", "user:123")

# 删除键
stash.delete("sessions", "user:123")

# 检查存在性
if stash.exists("sessions", "user:123"):
    print("Session exists")

# 使用模式匹配列出键
keys = stash.keys("sessions", pattern="user:*")
```

### 类字典接口

更 Pythonic 的访问方式，使用表接口：

```python
from gunicorn.dirty import stash

# 获取表引用
sessions = stash.table("sessions")

# 类字典操作（每个都是一条 IPC 消息）
sessions["user:123"] = {"name": "Alice"}
user = sessions["user:123"]
del sessions["user:123"]

# 迭代
for key in sessions:
    print(key, sessions[key])

# 长度
count = len(sessions)
```

### 表管理

```python
from gunicorn.dirty import stash

# 显式创建表（幂等）
stash.ensure("cache")

# 获取表信息
info = stash.info("sessions")
print(f"Table has {info['size']} entries")

# 清空表中的所有条目
stash.clear("sessions")

# 删除整个表
stash.delete_table("sessions")

# 列出所有表
tables = stash.tables()
```

### 在 DirtyApp 中使用 Stash

使用 `stashes` 类属性声明应用使用的表：

```python
from gunicorn.dirty import DirtyApp, stash

class SessionApp(DirtyApp):
    # 在此声明的表会在启动时自动创建
    stashes = ["sessions", "counters"]

    def init(self):
        # 如需初始化计数器
        if not stash.exists("counters", "requests"):
            stash.put("counters", "requests", 0)

    def login(self, user_id, user_data):
        """存储会话 - 任何 Worker 都可以通过 Arbiter 读取。"""
        stash.put("sessions", f"user:{user_id}", {
            "data": user_data,
            "logged_in_at": time.time(),
        })
        self._increment_counter()
        return {"status": "ok"}

    def get_session(self, user_id):
        """获取会话 - 请求发送到 Arbiter。"""
        return stash.get("sessions", f"user:{user_id}")

    def _increment_counter(self):
        """通过 Arbiter 递增全局计数器。"""
        current = stash.get("counters", "requests", 0)
        stash.put("counters", "requests", current + 1)

    def close(self):
        pass
```

### API 参考

| 函数 | 描述 |
|------|------|
| `stash.put(table, key, value)` | 存储值（表自动创建） |
| `stash.get(table, key, default=None)` | 检索值 |
| `stash.delete(table, key)` | 删除键，如果删除成功返回 True |
| `stash.exists(table, key=None)` | 检查表/键是否存在 |
| `stash.keys(table, pattern=None)` | 列出键，可选 glob 模式 |
| `stash.clear(table)` | 删除表中所有条目 |
| `stash.info(table)` | 获取表信息（大小等） |
| `stash.ensure(table)` | 如不存在则创建表 |
| `stash.delete_table(table)` | 删除整个表 |
| `stash.tables()` | 列出所有表名 |
| `stash.table(name)` | 获取类字典接口 |

### 模式与用例

**会话存储：**

```python
# 登录时存储会话（Worker 1）
stash.put("sessions", f"user:{user_id}", session_data)

# 请求时检查会话（可能是 Worker 2）
session = stash.get("sessions", f"user:{user_id}")
if session is None:
    raise AuthError("Not logged in")
```

**共享缓存：**

```python
def get_expensive_result(key):
    # 先检查缓存（通过 Arbiter）
    cached = stash.get("cache", key)
    if cached is not None:
        return cached

    # 计算并缓存
    result = expensive_computation()
    stash.put("cache", key, result)
    return result
```

**全局计数器：**

```python
def increment_counter(name):
    # 注意：非原子操作 - 两个 Worker 可能读取到相同的值
    current = stash.get("counters", name, 0)
    stash.put("counters", name, current + 1)
    return current + 1
```

**特性标志：**

```python
# 设置标志（从管理端点）
stash.put("flags", "new_feature", True)

# 检查标志（从任何 Worker）
if stash.get("flags", "new_feature", False):
    enable_new_feature()
```

### Stash 错误处理

```python
from gunicorn.dirty.stash import (
    StashError,
    StashTableNotFoundError,
    StashKeyNotFoundError,
)

try:
    info = stash.info("nonexistent")
except StashTableNotFoundError as e:
    print(f"Table not found: {e.table_name}")

# 使用带默认值的 get() 避免 KeyNotFoundError
value = stash.get("table", "key", default="fallback")
```

### Stash 最佳实践

1. **使用描述性的表名** — `user_sessions`、`ml_cache`，而非 `data`
2. **使用键前缀** — `user:123`、`cache:model:v1` 以便组织
3. **处理缺失数据** — 始终提供默认值或检查存在性
4. **不要存储大数据** — 每次访问都是一次 IPC 往返
5. **记住数据是临时的** — Arbiter 重启后数据丢失

### Stash 优势

- **Worker 隔离** — Worker 保持完全隔离，无共享内存 Bug
- **简单 API** — 类字典接口，无需加锁
- **二进制支持** — 高效存储字节（图像、模型权重）
- **模式匹配** — `keys(pattern="user:*")` 用于查询
- **零配置** — 与脏 Worker 自动配合工作
- **基于表** — 将数据组织到逻辑命名空间

### Stash 局限性

- **无持久化** — 数据仅存在于 Arbiter 内存中
- **无事务** — 无原子读-修改-写操作
- **无 TTL** — 条目不会自动过期
- **IPC 开销** — 每次操作都是一次网络往返
- **单 Arbiter** — 不跨多台机器分布

对于持久化或分布式状态，请使用 Redis、PostgreSQL 或类似的外部系统。

---

## Flask 示例

```python
from flask import Flask, Response
from gunicorn.dirty import get_dirty_client

app = Flask(__name__)

@app.route("/chat", methods=["POST"])
def chat():
    prompt = request.json.get("prompt")
    client = get_dirty_client()

    def stream():
        for token in client.stream("myapp.llm:LLMApp", "generate", prompt):
            yield f"data: {token}\n\n"

    return Response(stream(), content_type="text/event-stream")
```

## FastAPI 示例

```python
from fastapi import FastAPI
from fastapi.responses import StreamingResponse
from gunicorn.dirty import get_dirty_client_async

app = FastAPI()

@app.post("/chat")
async def chat(prompt: str):
    client = await get_dirty_client_async()

    async def stream():
        async for token in client.stream_async("myapp.llm:LLMApp", "generate", prompt):
            yield f"data: {token}\n\n"

    return StreamingResponse(stream(), media_type="text/event-stream")
```

---

## 生命周期钩子

Dirty Arbiters 提供自定义钩子：

```python
# gunicorn.conf.py

def on_dirty_starting(arbiter):
    """在 Dirty Arbiter 启动前调用。"""
    print("Dirty arbiter starting...")

def dirty_post_fork(arbiter, worker):
    """在脏 Worker fork 后立即调用。"""
    print(f"Dirty worker {worker.pid} forked")

def dirty_worker_init(worker):
    """在脏 Worker 初始化所有应用后调用。"""
    print(f"Dirty worker {worker.pid} initialized")

def dirty_worker_exit(arbiter, worker):
    """在脏 Worker 退出时调用。"""
    print(f"Dirty worker {worker.pid} exiting")

on_dirty_starting = on_dirty_starting
dirty_post_fork = dirty_post_fork
dirty_worker_init = dirty_worker_init
dirty_worker_exit = dirty_worker_exit
```

---

## 信号处理

Dirty Arbiters 与主 Arbiter 的信号处理集成。信号从主 Arbiter 转发到 Dirty Arbiter，然后传播到 Worker。

### 信号流

```
Main Arbiter                    Dirty Arbiter                 Dirty Workers
     |                                |                             |
SIGTERM/SIGHUP/SIGUSR1 ------>  signal_handler()                    |
     |                                |                             |
     |                          call_soon_threadsafe()              |
     |                                |                             |
     |                          handle_signal()                     |
     |                                |                             |
     |                                +------> os.kill(worker, sig) |
     |                                                              |
```

### 信号参考

| 信号 | 在 Dirty Arbiter | 在 Dirty Worker | 备注 |
|------|------------------|-----------------|------|
| SIGTERM | 设置 `self.alive = False`，等待优雅关闭 | 完成当前请求后退出 | 带超时的优雅关闭 |
| SIGQUIT | 通过 `sys.exit(0)` 立即退出 | 立即杀死 | 快速关闭，不清理 |
| SIGHUP | 杀死所有 Worker，重新生成 | 立即退出 | Worker 热重载 |
| SIGUSR1 | 重新打开日志文件，转发到 Worker | 重新打开日志文件 | 日志轮转支持 |
| SIGTTIN | Worker 数量增加 1 | 不适用 | 动态扩容 |
| SIGTTOU | Worker 数量减少 1 | 不适用 | 动态缩容 |
| SIGCHLD | 由事件循环处理，触发 reap | 不适用 | Worker 死亡检测 |
| SIGINT | 同 SIGTERM | 同 SIGTERM | Ctrl-C 处理 |

### 使用 TTIN/TTOU 动态伸缩

可以在运行时通过信号动态调整脏 Worker 数量，无需重启 Gunicorn：

```bash
# 找到 Dirty Arbiter 进程
ps aux | grep dirty-arbiter
# 或使用 PID 文件
cat /tmp/gunicorn-dirty-myapp.pid

# 增加一个脏 Worker
kill -TTIN <dirty-arbiter-pid>

# 减少一个脏 Worker
kill -TTOU <dirty-arbiter-pid>
```

**最小 Worker 约束**：Dirty Arbiter 不会减少到低于应用配置所需的最小 Worker 数。例如，如果有一个 `workers = 3` 的应用，则不能将脏 Worker 缩减到 3 以下。达到此限制时，会记录警告：

```
WARNING: SIGTTOU: Cannot decrease below 3 workers (required by app specs)
```

**使用场景**：

- **突发处理** — 预期高负载时扩容
- **成本优化** — 低流量期间缩容
- **恢复** — Worker 忙于长任务时扩容

### 转发的信号

主 Arbiter 向 Dirty Arbiter 进程转发以下信号：

- **SIGTERM** — 整个进程树的优雅关闭
- **SIGHUP** — Worker 重载（主 Arbiter 重载 HTTP Worker，Dirty Arbiter 重载脏 Worker）
- **SIGUSR1** — 所有进程的日志轮转

### 异步信号处理

Dirty Arbiter 使用 asyncio 的信号集成，确保在事件循环中安全处理：

```python
# 信号注册到事件循环
loop.add_signal_handler(signal.SIGTERM, self.signal_handler, signal.SIGTERM)

def signal_handler(self, sig):
    # 使用 call_soon_threadsafe 实现线程安全的事件循环集成
    self.loop.call_soon_threadsafe(self.handle_signal, sig)
```

此模式确保信号不会在 asyncio 操作执行中途中断，防止竞态条件和部分状态更新。

---

## 存活与健康监控

Dirty Arbiters 实现多层健康监控，确保 Worker 保持响应，孤立进程被清理。

### 心跳机制

每个脏 Worker 维护一个"worker tmp"文件，其 mtime 作为心跳：

```
Worker 生命周期：
  1. Worker 启动，创建 WorkerTmp 文件
  2. Worker 每 (dirty_timeout / 2) 秒 touch 文件
  3. Arbiter 每 1 秒检查所有 Worker 的 mtime
  4. 如果 mtime 超过 dirty_timeout 秒未更新，Worker 被杀死
```

基于文件的心跳有几个优势：

- **操作系统级跟踪** — 无需 IPC，即使 Worker 卡在 C 代码中也有效
- **崩溃检测** — Worker 停止更新时 Arbiter 立即察觉
- **优雅恢复** — Worker 被 SIGKILL 杀死，Arbiter 生成替代者

### 超时检测

Arbiter 的监控循环每秒检查 Worker 健康：

```python
# Worker 监控伪代码
for worker in self.workers:
    mtime = worker.tmp.last_update()
    if time.time() - mtime > self.dirty_timeout:
        log.warning(f"Worker {worker.pid} timed out, killing")
        os.kill(worker.pid, signal.SIGKILL)
```

Worker 被杀死时：

1. SIGCHLD 传递给 Arbiter
2. Arbiter 回收 Worker 进程
3. 调用 `dirty_worker_exit` 钩子
4. 生成新 Worker 以维持 `dirty_workers` 数量

### 父进程死亡检测

Dirty Arbiter 监控其父进程（主 Arbiter）以检测孤儿状态：

```python
# 在 Dirty Arbiter 的主循环中
if os.getppid() != self.parent_pid:
    log.info("Parent died, shutting down")
    self.alive = False
```

此检查在事件循环的每次迭代中运行（通常亚毫秒级）。检测到父进程死亡时：

1. Arbiter 设置 `self.alive = False`
2. 所有 Worker 被发送 SIGTERM
3. Arbiter 等待优雅关闭（最多 `dirty_graceful_timeout`）
4. 剩余 Worker 被发送 SIGKILL
5. Arbiter 退出

### 孤儿清理

为处理 Dirty Arbiter 自身崩溃的边缘情况，使用已知 PID 文件：

```
PID 文件位置: /tmp/gunicorn_dirty_<main_arbiter_pid>.pid
```

启动时，Dirty Arbiter：

1. 检查 PID 文件是否存在
2. 如存在，读取旧 PID 并尝试杀死它（SIGTERM）
3. 短暂等待清理
4. 写入自己的 PID
5. 退出时删除 PID 文件

这确保了如果 Dirty Arbiter 崩溃且主 Arbiter 重启它，旧的孤立进程会被终止。

### 重启行为

| 组件 | 重启触发 | 重启行为 |
|------|----------|----------|
| Dirty Worker | 退出、超时或崩溃 | 立即重启以维持 `dirty_workers` 数量 |
| Dirty Arbiter | 退出或崩溃 | 主 Arbiter 在非关闭状态下重启 |

Dirty Arbiter 维护目标 Worker 数量，持续生成 Worker 直到达到目标：

```python
while len(self.workers) < self.num_workers:
    self.spawn_worker()
```

### 监控建议

生产部署建议：

1. **日志监控** — 关注 "Worker timed out" 消息，指示 Worker 挂起
2. **进程监控** — 使用 systemd 或 supervisord 监控主 Arbiter
3. **指标** — 跟踪重启频率以检测不稳定的 Worker

```bash
# 检查最近的 Worker 超时
grep "Worker.*timed out" /var/log/gunicorn.log | tail -20

# 监控进程树
watch -n 1 'pstree -p $(cat gunicorn.pid)'
```

---

## 错误处理

Dirty 客户端抛出特定异常：

```python
from gunicorn.dirty.errors import (
    DirtyError,
    DirtyTimeoutError,
    DirtyConnectionError,
    DirtyAppError,
    DirtyAppNotFoundError,
    DirtyNoWorkersAvailableError,
)

try:
    result = client.execute("myapp.ml:MLApp", "inference", "model", data)
except DirtyTimeoutError:
    # 操作超时
    pass
except DirtyAppNotFoundError:
    # 应用未在脏 Worker 中加载
    pass
except DirtyNoWorkersAvailableError as e:
    # 没有 Worker 拥有此应用（全部崩溃或应用被限制为 0 个 Worker）
    print(f"No workers for app: {e.app_path}")
except DirtyAppError as e:
    # 应用执行期间的错误
    print(f"App error: {e.message}, traceback: {e.traceback}")
except DirtyConnectionError:
    # 连接到 Dirty Arbiter 失败
    pass
```

---

## 最佳实践

1. 在 `init()` 中预加载常用资源以避免冷启动
2. 根据工作负载设置适当的超时时间
3. 优雅地处理错误——脏 Worker 可能会重启
4. 使用有意义的 action 名称以便调试
5. 保持响应可序列化——结果通过二进制 IPC 传递（直接支持 bytes）

---

## 监控

使用标准进程监控监控脏 Worker：

```bash
# 检查 Dirty Arbiter 和 Worker
ps aux | grep "dirty"

# 查看日志
tail -f gunicorn.log | grep dirty
```

---

## 示例：图像处理

```python
# myapp/images.py
from gunicorn.dirty import DirtyApp
from PIL import Image
import io

class ImageApp(DirtyApp):
    def init(self):
        # 预导入重型库
        import cv2
        self.cv2 = cv2

    def resize(self, image_data, width, height):
        """调整图像大小。"""
        img = Image.open(io.BytesIO(image_data))
        resized = img.resize((width, height))
        buffer = io.BytesIO()
        resized.save(buffer, format='PNG')
        return buffer.getvalue()

    def thumbnail(self, image_data, size=128):
        """创建缩略图。"""
        img = Image.open(io.BytesIO(image_data))
        img.thumbnail((size, size))
        buffer = io.BytesIO()
        img.save(buffer, format='JPEG')
        return buffer.getvalue()

    def close(self):
        pass
```

使用方式：

```python
from gunicorn.dirty import get_dirty_client

def upload_image(request):
    client = get_dirty_client()

    # 在脏 Worker 中创建缩略图
    thumbnail = client.execute(
        "myapp.images:ImageApp",
        "thumbnail",
        request.files['image'].read(),
        size=256
    )

    return save_thumbnail(thumbnail)
```
