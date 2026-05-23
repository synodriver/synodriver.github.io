---
date: '2026-05-23T12:39:09+08:00'
draft: false
title: 'NGINX Unit 架构与 Python 模块深度剖析：从 WSGI 到 ASGI 的 C 语言实现'
author: 'synodriver'
tags: ["nginx unit", "python", "c", "asgi"]
---

~~虽然NGINX Unit已经archive了，但它是少有的用C实现异步python调用对象的例子，这个验尸还是有意义的~~


## 1. NGINX Unit 整体架构

NGINX Unit 是一个开源的多语言应用服务器，由 NGINX Inc. 开发。与传统的 NGINX 不同，Unit 不仅支持静态文件服务和反向代理，还内建了多种语言运行时的直接支持，包括 Python、Go、Java、Node.js、Perl、Ruby 和 WebAssembly。

### 1.1 进程模型

Unit 采用**多进程架构**，核心进程包括：

- **Main Process（主进程）**：负责启动、配置管理和进程间协调
- **Controller Process（控制器进程）**：处理 JSON API 配置请求，验证并应用配置变更
- **Router Process（路由进程）**：核心的请求路由和分发进程，管理监听套接字、连接、HTTP 请求路由（`nxt_router.c`，162KB，是最大的源文件）
- **Application Process（应用进程）**：每种语言模块运行在独立的应用进程中，通过共享内存和 Unix 域套接字与 Router 通信

```
                    ┌─────────────────┐
                    │  Main Process   │
                    └────────┬────────┘
                             │
              ┌──────────────┼──────────────┐
              │              │              │
    ┌─────────▼────┐  ┌─────▼──────┐  ┌───▼───────────┐
    │ Controller   │  │  Router    │  │ App Process   │
    │ (API/Config) │  │ (Routing)  │  │ (Python/Go/…) │
    └──────────────┘  └─────┬──────┘  └───┬───────────┘
                            │              │
                      ┌─────▼──────────────▼─────┐
                      │  Shared Memory + Ports   │
                      │   (IPC via Unix Socket)  │
                      └──────────────────────────┘
```

### 1.2 进程间通信（IPC）

Router 和 App Process 之间通过 `nxt_port` 机制进行通信：

- **共享内存**：用于传递请求数据（HTTP 头、请求体等），避免数据拷贝
- **Port（端口）**：基于 Unix 域套接字的 IPC 通道，用于传递控制消息
- **`nxt_unit.c`（180KB）**：提供共享库 C API，是外部语言模块与 Unit 核心交互的桥梁

### 1.3 事件驱动引擎

Unit 核心采用事件驱动架构，支持多种平台的事件机制：

- Linux: `epoll`（`nxt_epoll_engine.c`）
- macOS: `kqueue`（`nxt_kqueue_engine.c`）
- Solaris: `/dev/poll`（`nxt_devpoll_engine.c`）
- 通用: `poll`/`select`

## 2. Python 模块架构

### 2.1 文件结构

Python 模块位于 `src/python/`，共 10 个文件：

```
src/python/
├── nxt_python.c                  — 主入口：初始化、协议检测、请求分发
├── nxt_python.h                  — 公共类型和接口声明
├── nxt_python_asgi.c             — ASGI 核心：事件循环、协议状态机、端口集成
├── nxt_python_asgi.h             — ASGI 数据结构和函数声明
├── nxt_python_asgi_http.c        — ASGI HTTP 协议处理
├── nxt_python_asgi_lifespan.c    — ASGI Lifespan 生命周期事件
├── nxt_python_asgi_websocket.c   — ASGI WebSocket 协议处理
├── nxt_python_asgi_str.c         — ASGI 字符串常量池
└── nxt_python_asgi_str.h         — 字符串常量声明
```

### 2.2 协议抽象层：`nxt_python_proto_t`

Python 模块通过一个协议抽象层来统一 WSGI 和 ASGI 的差异：

```c
// nxt_python.h
typedef struct {
    int   (*ctx_data_alloc)(void **pdata, int main);
    void  (*ctx_data_free)(void *data);
    int   (*startup)(void *data);
    int   (*run)(nxt_unit_ctx_t *ctx);
    void  (*done)(void);
} nxt_python_proto_t;
```

WSGI 和 ASGI 分别提供各自实现：

| 方法 | WSGI 实现 | ASGI 实现 |
|------|-----------|-----------|
| `ctx_data_alloc` | `nxt_python_wsgi_ctx_data_alloc` | `nxt_python_asgi_ctx_data_alloc` |
| `ctx_data_free` | `nxt_python_wsgi_ctx_data_free` | `nxt_python_asgi_ctx_data_free` |
| `startup` | `NULL` | `nxt_python_asgi_startup` |
| `run` | `nxt_python_wsgi_run` | `nxt_python_asgi_run` |
| `done` | `nxt_python_wsgi_done` | `nxt_python_asgi_done` |

### 2.3 协议自动检测

`nxt_python_start()` 中实现了 ASGI/WSGI 的自动检测逻辑：

```c
// nxt_python.c:322-335
proto = c->protocol;

if (proto.length == 0) {
    proto = nxt_python_asgi_check(targets->target[0].application)
            ? asgi : wsgi;
    // 所有 target 必须是同一协议
    for (i = 1; i < targets->count; i++) {
        probe_proto = nxt_python_asgi_check(targets->target[i].application)
                      ? asgi : wsgi;
        if (probe_proto.start != proto.start) {
            nxt_alert(task, "A mix of ASGI & WSGI targets is forbidden");
            goto fail;
        }
    }
}
```

检测逻辑在 `nxt_python_asgi_check()` 中实现：

```c
// nxt_python_asgi.c:67-92
int nxt_python_asgi_check(PyObject *obj)
{
    PyObject      *func;
    PyCodeObject  *code;

    func = nxt_python_asgi_get_func(obj);  // 提取底层函数对象
    if (func == NULL) return 0;

    code = (PyCodeObject *) PyFunction_GET_CODE(func);

    // 两个判断标准：
    // 1. 函数标记了 CO_COROUTINE 标志 → async def → ASGI 3.0
    // 2. 函数参数个数为 1（即 scope）→ ASGI 2.0 legacy 模式
    res = (code->co_flags & CO_COROUTINE) != 0 || code->co_argcount == 1;

    Py_DECREF(func);
    return res;
}
```

检测逻辑的精妙之处：ASGI 3.0 的应用是 `async def app(scope, receive, send)`，其 `CO_COROUTINE` 标志位为 1，参数个数为 3；而 WSGI 的 `def app(environ, start_response)` 参数个数为 2 且不是协程。此外，ASGI 2.0 legacy 模式的 `def app(scope)` 参数个数为 1，也能被正确识别。

### 2.4 多线程模型

Python 模块支持多线程运行，每个线程有独立的上下文数据：

```c
// nxt_python.c:607-659
static int nxt_python_init_threads(nxt_python_app_conf_t *c)
{
    // 主线程使用主事件循环，工作线程各自创建新的事件循环
    for (i = 0; i < c->threads - 1; i++) {
        ti = &nxt_py_threads[i];
        res = nxt_py_proto.ctx_data_alloc(&ti->ctx_data, 0);  // main=0 → new_event_loop
    }
}
```

当 `ready_handler` 被调用时，启动工作线程：

```c
// nxt_python.c:697-734
static void *nxt_python_thread_func(void *data)
{
    gstate = PyGILState_Ensure();
    ctx = nxt_unit_ctx_alloc(ti->ctx, ti->ctx_data);  // 创建新的 unit 上下文
    (void) nxt_py_proto.run(ctx);                      // 运行各自的事件循环
    nxt_unit_done(ctx);
    PyGILState_Release(gstate);
}
```

## 3. WSGI 服务器的 C 语言实现

WSGI（Web Server Gateway Interface，PEP 3333）是 Python 的同步 Web 应用接口。Unit 的 WSGI 实现完全集中在 `nxt_python_wsgi.c` 中。

### 3.1 核心数据结构

```c
// nxt_python_wsgi.c:41-51
typedef struct {
    PyObject_HEAD
    uint64_t                 content_length;
    uint64_t                 bytes_sent;
    PyObject                 *environ;       // WSGI environ 字典
    PyObject                 *start_resp;    // start_response 可调用对象
    PyObject                 *write;         // write() 可调用对象
    nxt_unit_request_info_t  *req;           // Unit 请求对象
    PyThreadState            *thread_state;  // 线程状态（用于 GIL 管理）
} nxt_python_ctx_t;
```

这个结构体同时充当了 Python 类型对象（`nxt_py_input_type`），其 `tp_basicsize` 被设为 `sizeof(nxt_python_ctx_t)`，因此它**既是 `wsgi.input` 输入流对象，也是请求上下文容器**。

### 3.2 请求处理流程

```
              Unit Core
                 │
    ┌────────────▼────────────┐
    │ nxt_python_request_handler │  ← 回调函数
    └────────────┬────────────┘
                 │
    ┌────────────▼────────────┐
    │ PyEval_RestoreThread()    │  ← 获取 GIL
    └────────────┬────────────┘
                 │
    ┌────────────▼────────────┐
    │ nxt_python_get_environ() │  ← 构建 environ 字典
    └────────────┬────────────┘
                 │
    ┌────────────▼────────────┐
    │ PyObject_CallObject(     │  ← 调用 WSGI 应用
    │   application,           │     app(environ, start_response)
    │   (environ, start_resp)) │
    └────────────┬────────────┘
                 │
    ┌────────────▼────────────┐
    │ 遍历迭代器写入响应       │  ← PyIter_Next + nxt_python_write
    └────────────┬────────────┘
                 │
    ┌────────────▼────────────┐
    │ PyEval_SaveThread()      │  ← 释放 GIL
    └────────────┬────────────┘
                 │
    ┌────────────▼────────────┐
    │ nxt_unit_request_done()  │  ← 完成请求
    └─────────────────────────┘
```

### 3.3 WSGI 运行模式：同步阻塞 + GIL 管理

WSGI 的运行方式是最直接的同步模式：

```c
// nxt_python_wsgi.c:278-293
static int nxt_python_wsgi_run(nxt_unit_ctx_t *ctx)
{
    nxt_python_ctx_t  *pctx = ctx->data;

    // 释放 GIL，让其他线程可以运行
    pctx->thread_state = PyEval_SaveThread();

    // 进入 Unit 的事件循环（阻塞等待请求）
    rc = nxt_unit_run(ctx);

    // 恢复 GIL
    PyEval_RestoreThread(pctx->thread_state);

    return rc;
}
```

关键设计：**WSGI 的主循环是 `nxt_unit_run()`**，这是 Unit 共享库提供的 C 语言事件循环。当有请求到来时，回调 `nxt_python_request_handler` 被调用，该回调内部通过 `PyEval_RestoreThread/SaveThread` 来管理 GIL，确保 Python 代码在持有 GIL 的状态下执行。

### 3.4 `start_response` 和 `write` 的 C 实现

这两个 WSGI 规范要求的可调用对象通过 `PyCFunction_New` 创建：

```c
// nxt_python_wsgi.c:232-245
pctx->start_resp = PyCFunction_New(nxt_py_start_resp_method, (PyObject *) pctx);
pctx->write = PyCFunction_New(nxt_py_write_method, (PyObject *) pctx);
```

`nxt_py_start_resp` 的关键逻辑：

1. 解析状态码和响应头
2. 调用 `nxt_unit_response_init()` 初始化响应
3. 逐个添加响应头字段
4. 遵循 PEP 3333 规范：仅在 `Content-Length: 0` 时立即发送响应头，否则等待响应体

`nxt_py_write` 的关键逻辑：

1. 检查内容长度约束
2. 调用 `nxt_unit_response_write()` 发送数据

### 3.5 `wsgi.input` 的 C 实现

`nxt_python_ctx_t` 本身就是 `wsgi.input` 对象（Python 类型 `unit._input`），支持 `read()`、`readline()`、`readlines()` 方法和迭代器协议。数据通过 `nxt_unit_request_read()` 从共享内存读取。

## 4. ASGI 服务器的 C 语言实现

ASGI（Asynchronous Server Gateway Interface）是 Python 的异步 Web 应用接口，Unit 的 ASGI 实现由 6 个文件协作完成，代码量约为 WSGI 的 3 倍。

### 4.1 核心数据结构

```c
// nxt_python_asgi.h:26-37
typedef struct {
    nxt_queue_t      drain_queue;               // 等待发送的请求队列
    PyObject         *loop_run_until_complete;   // loop.run_until_complete
    PyObject         *loop_create_future;        // loop.create_future
    PyObject         *loop_create_task;          // loop.create_task
    PyObject         *loop_call_soon;            // loop.call_soon
    PyObject         *loop_add_reader;           // loop.add_reader
    PyObject         *loop_remove_reader;        // loop.remove_reader
    PyObject         *quit_future;               // 退出信号 Future
    PyObject         *quit_future_set_result;    // quit_future.set_result
    PyObject         **target_lifespans;         // 每个 target 的 lifespan 对象
} nxt_py_asgi_ctx_data_t;
```

**这是整个 ASGI 子系统最关键的数据结构**。它将 Unit 的 C 层事件系统与 Python 的 asyncio 事件循环桥接起来。六个 `loop_*` 函数指针在初始化时从 asyncio 事件循环对象上获取并缓存。

### 4.2 ASGI HTTP 请求对象

```c
// nxt_python_asgi_http.c:18-31
typedef struct {
    PyObject_HEAD
    nxt_unit_request_info_t  *req;
    nxt_queue_link_t         link;              // drain 队列链接
    PyObject                 *receive_future;    // 等待中的 receive Future
    PyObject                 *send_future;       // 等待中的 send Future
    uint64_t                 content_length;
    uint64_t                 bytes_sent;
    PyObject                 *send_body;         // 待发送的 body 数据
    Py_ssize_t               send_body_off;     // 已发送偏移
    uint8_t                  complete;           // 响应完成标志
    uint8_t                  closed;             // 连接关闭标志
    uint8_t                  request_received;   // 请求体已接收标志
} nxt_py_asgi_http_t;
```

这个结构体注册为 Python 类型 `unit._asgi_http`，实现了 `__await__` 协议（通过 `tp_as_async`），以及 `receive`、`send`、`_done` 三个方法。

### 4.3 ASGI 请求处理流程

```
           Unit Core (C)
               │
  ┌────────────▼──────────────┐
  │ nxt_py_asgi_request_handler │  ← 请求回调
  └────────────┬──────────────┘
               │
  ┌────────────▼──────────────┐
  │ 创建 ASGI 对象              │  ← http 或 websocket
  │ (nxt_py_asgi_http_create)  │
  └────────────┬──────────────┘
               │
  ┌────────────▼──────────────┐
  │ 获取 receive/send/_done    │  ← PyObject_GetAttrString
  │ 方法的引用                  │
  └────────────┬──────────────┘
               │
  ┌────────────▼──────────────┐
  │ 构建 ASGI scope 字典       │  ← nxt_py_asgi_create_http_scope
  │ 注入 lifespan.state        │
  └────────────┬──────────────┘
               │
  ┌────────────▼──────────────┐
  │ 调用 ASGI 应用              │  ASGI 3.0:
  │ app(scope, receive, send)  │  → 返回协程
  │ 或 legacy:                 │  ASGI 2.0:
  │ app(scope)(receive, send)  │  → 两步调用
  └────────────┬──────────────┘
               │
  ┌────────────▼──────────────┐
  │ loop.create_task(coro)     │  ← 将协程包装为 Task
  └────────────┬──────────────┘
               │
  ┌────────────▼──────────────┐
  │ task.add_done_callback(    │  ← 注册完成回调
  │   done)                    │
  └───────────────────────────┘
```

与 WSGI 的关键区别：**请求处理函数不阻塞等待应用执行完毕**。它只是将应用的协程提交给 asyncio 事件循环（通过 `loop.create_task`），然后立即返回。应用的实际执行发生在 asyncio 事件循环内部。

### 4.4 ASGI 运行模式：asyncio 事件循环

```c
// nxt_python_asgi.c:389-411
static int nxt_python_asgi_run(nxt_unit_ctx_t *ctx)
{
    nxt_py_asgi_ctx_data_t  *ctx_data = ctx->data;

    // 阻塞运行 asyncio 事件循环，直到 quit_future 完成
    res = PyObject_CallFunctionObjArgs(ctx_data->loop_run_until_complete,
                                       ctx_data->quit_future, NULL);

    // 事件循环结束后，执行 lifespan shutdown
    nxt_py_asgi_lifespan_shutdown(ctx);

    return NXT_UNIT_OK;
}
```

**这是 ASGI 和 WSGI 运行方式的根本区别**：

- **WSGI**：运行 `nxt_unit_run()`，由 Unit 的 C 事件循环驱动
- **ASGI**：运行 `loop.run_until_complete(quit_future)`，由 Python asyncio 事件循环驱动

## 5. ASGI 异步函数（send/receive）的 C 语言实现

ASGI 规范要求 `receive` 和 `send` 都是异步可调用对象（awaitable）。Unit 通过 Python C API 实现了自定义的 awaitable 对象。

### 5.1 核心机制：Future + set_result_soon

Unit 的 ASGI 异步函数实现的核心模式是：

1. 创建一个 `asyncio.Future` 对象
2. 如果消息立即可用，通过 `loop.call_soon(future.set_result, msg)` 在下一个事件循环迭代中设置结果
3. 如果消息暂不可用，将 Future 保存到结构体中（如 `http->receive_future`），等待后续事件触发时设置结果
4. 返回 Future 对象（因为 Future 实现了 `__await__`，可以被 await）

```c
// nxt_python_asgi.c:1355-1403
PyObject *nxt_py_asgi_set_result_soon(nxt_unit_request_info_t *req,
    nxt_py_asgi_ctx_data_t *ctx_data, PyObject *future, PyObject *result)
{
    set_result = PyObject_GetAttrString(future, "set_result");
    // 使用 loop.call_soon 延迟设置结果，避免在回调上下文中直接修改 Future 状态
    res = PyObject_CallFunctionObjArgs(ctx_data->loop_call_soon, set_result,
                                       result, NULL);
    return future;
}
```

### 5.2 HTTP `receive` 的实现

```c
// nxt_python_asgi_http.c:112-159
static PyObject *nxt_py_asgi_http_receive(PyObject *self, PyObject *none)
{
    nxt_py_asgi_http_t  *http = (nxt_py_asgi_http_t *) self;

    if (http->closed || http->complete) {
        // 连接已关闭或响应已完成 → 返回 http.disconnect
        msg = nxt_py_asgi_new_msg(req, nxt_py_http_disconnect_str);
    } else {
        // 尝试读取请求体数据
        msg = nxt_py_asgi_http_read_msg(http);
    }

    future = PyObject_CallObject(ctx_data->loop_create_future, NULL);

    if (msg != Py_None) {
        // 消息立即可用 → 通过 call_soon 设置 Future 结果
        return nxt_py_asgi_set_result_soon(req, ctx_data, future, msg);
    }

    // 消息暂不可用 → 保存 Future，等待 C 层的数据到达回调
    http->receive_future = future;
    Py_INCREF(http->receive_future);

    return future;
}
```

三种场景的时序图：

**场景 1：请求体数据立即可用**

```
Python (await receive())          C 层
       │                            │
       ├── receive() 被调用 ───────►│
       │                            ├── http_read_msg() 读取数据
       │                            ├── 消息可用 (msg != Py_None)
       │◄── 返回 Future ───────────│
       │   (内部已通过 call_soon    │
       │    安排 set_result)        │
       ├── await Future 恢复 ──────│
       │   获得 http.request 消息   │
```

**场景 2：请求体数据暂不可用（如大请求体分段传输）**

```
Python (await receive())          C 层 (Router)              C 层 (数据到达)
       │                            │                          │
       ├── receive() 被调用 ───────►│                          │
       │                            ├── http_read_msg() 无数据 │
       │                            ├── msg == Py_None         │
       │◄── 返回 pending Future ───│                          │
       │   (Future 未设置结果)      │                          │
       │   await Future 挂起 ──────│                          │
       │                            │                          │
       │                            │    ┌─ data_handler ─────►│
       │                            │    │ 触发回调              │
       │                            │    │ http_data_handler()  │
       │                            │◄───┤ set_result(receive_future, msg)
       │◄── Future 完成 ───────────│    │                      │
       │   获得 http.request 消息   │                          │
```

**场景 3：连接关闭**

```
Python (await receive())          C 层
       │                            │
       ├── receive() 被调用 ───────►│
       │                            ├── http->closed == 1
       │◄── 返回 Future ───────────│
       │   (http.disconnect)       │
       │   或者                      │
       │   Future 挂起期间          │
       │                            ├── close_handler()
       │◄── emit_disconnect() ─────│
       │   set_result(disconnect)  │
```

### 5.3 HTTP `send` 的实现

```c
// nxt_python_asgi_http.c:253-295
static PyObject *nxt_py_asgi_http_send(PyObject *self, PyObject *dict)
{
    type = PyDict_GetItem(dict, nxt_py_type_str);

    if (nxt_unit_response_is_init(http->req)) {
        // 响应已初始化 → 期望 http.response.body
        return nxt_py_asgi_http_response_body(http, dict);
    }

    // 响应未初始化 → 期望 http.response.start
    return nxt_py_asgi_http_response_start(http, dict);
}
```

`send` 函数不是异步的（不返回 Future），它直接执行 C 层操作：

- `http.response.start`：初始化响应（状态码 + 响应头）
- `http.response.body`：发送响应体，如果共享内存缓冲区满了则返回一个 Future（背压机制）

背压机制的实现：

```c
// nxt_python_asgi_http.c:410-441 (response_body 中的关键片段)
while (body_len > 0) {
    sent = nxt_unit_response_write_nb(http->req, body_str, body_len, 0);

    if (sent == 0) {
        // 共享内存已满 → 创建 Future 并挂起
        future = PyObject_CallObject(ctx_data->loop_create_future, NULL);
        http->send_body = body;
        http->send_body_off = body_off;
        nxt_py_asgi_drain_wait(http->req, &http->link);  // 加入 drain 队列
        http->send_future = future;
        return future;  // 返回 Future，应用 await 此 Future 等待缓冲区可用
    }

    body_str += sent;
    body_len -= sent;
}
```

当共享内存缓冲区可用时，`shm_ack_handler` 触发 drain 处理：

```c
// nxt_python_asgi.c:1140-1159
static void nxt_py_asgi_shm_ack_handler(nxt_unit_ctx_t *ctx)
{
    while (!nxt_queue_is_empty(&ctx_data->drain_queue)) {
        lnk = nxt_queue_first(&ctx_data->drain_queue);
        rc = nxt_py_asgi_http_drain(lnk);  // 尝试继续发送
        if (rc == NXT_UNIT_AGAIN) return;   // 缓冲区又满了
        nxt_queue_remove(lnk);              // 发送完成，移出队列
    }
}
```

### 5.4 WebSocket 的 receive/send

WebSocket 的异步函数实现模式与 HTTP 类似，但更复杂：

- `receive` 需要处理五种状态：`INIT` → `CONNECT` → `ACCEPTED` → `DISCONNECTED` → `CLOSED`
- 帧数据可能分段到达，需要 `pending_frames` 队列缓存未完成的帧
- `send` 根据消息类型分发到 `accept`/`close`/`send_frame` 三种操作

## 6. asyncio 事件循环与 C 事件的集成

**这是 ASGI 实现中最精妙的部分**。asyncio 的事件循环是一个 Python 层面的死循环（`loop.run_forever()` 或 `loop.run_until_complete()`），而 Unit 的请求/响应事件由 C 层的 Router 进程通过共享内存和端口触发。如何在 asyncio 的事件循环中处理 C 层的事件？

### 6.1 核心问题

```
┌─────────────────────────────────────────────┐
│  asyncio Event Loop (Python)                │
│                                             │
│  while True:                                │
│      events = poll(registered_fds)          │
│      for event in events:                   │
│          callback(event)                    │
│                                             │
│  问题：C 层的端口事件如何进入这个循环？       │
└─────────────────────────────────────────────┘
```

### 6.2 解决方案：`loop.add_reader` 桥接

Unit 的解决方案是将 C 层的端口文件描述符注册到 asyncio 事件循环中：

```c
// nxt_python_asgi.c:1007-1028
static int nxt_py_asgi_add_port(nxt_unit_ctx_t *ctx, nxt_unit_port_t *port)
{
    // 将端口 fd 设为非阻塞
    nb = 1;
    ioctl(port->in_fd, FIONBIO, &nb);

    // 注册到 asyncio 事件循环
    return nxt_py_asgi_add_reader(ctx, port);
}
```

```c
// nxt_python_asgi.c:1031-1092
static int nxt_py_asgi_add_reader(nxt_unit_ctx_t *ctx, nxt_unit_port_t *port)
{
    fd = PyLong_FromLong(port->in_fd);
    py_ctx = PyLong_FromVoidPtr(ctx);
    py_port = PyLong_FromVoidPtr(port);

    // 关键调用：将 fd 注册到 asyncio 事件循环的 epoll/select 中
    // 当 fd 可读时，调用 nxt_py_port_read 函数
    res = PyObject_CallFunctionObjArgs(ctx_data->loop_add_reader,
                                       fd, nxt_py_port_read,
                                       py_ctx, py_port, NULL);
}
```

### 6.3 `unit_port_read`：C 事件的 Python 入口

```c
// nxt_python_asgi.c:53-54
static PyMethodDef nxt_py_port_read_method =
    {"unit_port_read", nxt_py_asgi_port_read, METH_VARARGS, ""};
```

当 asyncio 事件循环检测到端口 fd 可读时，调用 `nxt_py_asgi_port_read`：

```c
// nxt_python_asgi.c:1162-1222
static PyObject *nxt_py_asgi_port_read(PyObject *self, PyObject *args)
{
    // 从参数中恢复 C 指针
    ctx = PyLong_AsVoidPtr(arg0);
    port = PyLong_AsVoidPtr(arg1);

    // 处理端口消息（这是 C 层的逻辑）
    rc = nxt_unit_process_port_msg(ctx, port);

    if (rc == NXT_UNIT_OK) {
        // 如果还有消息未处理，通过 call_soon 再次调度自己
        res = PyObject_CallFunctionObjArgs(ctx_data->loop_call_soon,
                                           nxt_py_port_read,
                                           arg0, arg1, NULL);
    }

    Py_RETURN_NONE;
}
```

### 6.4 完整的事件流转

```
                    Unit Router Process
                          │
               ┌──────────▼──────────┐
               │  请求到达，写入      │
               │  共享内存 + 发送     │
               │  端口通知消息        │
               └──────────┬──────────┘
                          │ (Unix Socket)
                          ▼
    ┌───────────────────────────────────────────────┐
    │  asyncio Event Loop                           │
    │                                               │
    │  ┌──────────────────────────────────────────┐ │
    │  │ epoll/select 检测到 port fd 可读          │ │
    │  └──────────────────┬───────────────────────┘ │
    │                     │                         │
    │  ┌──────────────────▼───────────────────────┐ │
    │  │ 调用 loop.add_reader 注册的回调           │ │
    │  │ → nxt_py_asgi_port_read(ctx, port)        │ │
    │  └──────────────────┬───────────────────────┘ │
    │                     │                         │
    │  ┌──────────────────▼───────────────────────┐ │
    │  │ nxt_unit_process_port_msg(ctx, port)      │ │
    │  │ → 解析共享内存中的请求数据                 │ │
    │  │ → 调用注册的回调：                        │ │
    │  │   request_handler / data_handler /         │ │
    │  │   websocket_handler / close_handler        │ │
    │  └──────────────────┬───────────────────────┘ │
    │                     │                         │
    │  ┌──────────────────▼───────────────────────┐ │
    │  │ 回调中创建 ASGI 对象，调用 app()          │ │
    │  │ 设置 Future / 触发 receive/send 完成      │ │
    │  └──────────────────────────────────────────┘ │
    │                                               │
    └───────────────────────────────────────────────┘
```

### 6.5 关键设计洞察

**为什么 Unit 选择将 asyncio 事件循环作为主循环，而不是用自己的 C 事件循环？**

1. **ASGI 规范要求**：ASGI 应用是协程，必须在 asyncio 事件循环中运行
2. **避免嵌套循环**：如果 Unit 的 C 循环在外层，就需要在每次迭代中调用 `loop.run_once()`，这既复杂又低效
3. **优雅的集成**：通过 `loop.add_reader` 将 C 层的 fd 注册到 asyncio 的 poll 中，让 asyncio 统一管理所有 I/O 事件

**`call_soon` 的巧妙使用**：在 `nxt_py_asgi_port_read` 中，如果 `nxt_unit_process_port_msg` 返回 `NXT_UNIT_OK`，说明可能还有更多消息待处理。此时通过 `loop.call_soon` 将自身重新调度，确保在当前回调返回后继续处理剩余消息，而不会阻塞事件循环。

## 7. Lifespan 及 lifespan.state 扩展的实现

### 7.1 Lifespan 数据结构

```c
// nxt_python_asgi_lifespan.c:18-31
typedef struct {
    PyObject_HEAD
    nxt_py_asgi_ctx_data_t  *ctx_data;
    int                     disabled;           // lifespan 是否被禁用
    int                     startup_received;   // 是否已发送 startup 事件
    int                     startup_sent;       // 是否已收到 startup 响应
    int                     shutdown_received;  // 是否已发送 shutdown 事件
    int                     shutdown_sent;      // 是否已收到 shutdown 响应
    int                     shutdown_called;    // 是否调用了 shutdown
    PyObject                *startup_future;    // startup 同步 Future
    PyObject                *shutdown_future;   // shutdown 同步 Future
    PyObject                *receive_future;    // 等待中的 receive Future
    PyObject                *state;             // lifespan.state 字典
} nxt_py_asgi_lifespan_t;
```

### 7.2 Lifespan Startup 流程

Lifespan 的 startup 在 `nxt_python_asgi_startup()` 中执行，**这是在事件循环正式启动之前同步完成的**：

```c
// nxt_python_asgi_lifespan.c:133-318
static PyObject *nxt_py_asgi_lifespan_target_startup(
    nxt_py_asgi_ctx_data_t *ctx_data, nxt_python_target_t *target)
{
    // 1. 创建 lifespan 对象
    lifespan = PyObject_New(nxt_py_asgi_lifespan_t, &nxt_py_asgi_lifespan_type);

    // 2. 创建 startup_future（用于同步等待 startup 完成）
    lifespan->startup_future = PyObject_CallObject(ctx_data->loop_create_future, NULL);

    // 3. 创建 scope 和 state 字典
    scope = nxt_py_asgi_new_scope(NULL, nxt_py_lifespan_str, nxt_py_2_0_str);
    lifespan->state = PyDict_New();
    PyDict_SetItem(scope, nxt_py_state_str, lifespan->state);

    // 4. 调用 ASGI 应用：app(scope, receive, send)
    res = PyObject_CallFunctionObjArgs(target->application,
                                       scope, receive, send, NULL);

    // 5. 将协程包装为 Task 并注册完成回调
    py_task = PyObject_CallFunctionObjArgs(ctx_data->loop_create_task, res, NULL);
    PyObject_CallMethodObjArgs(py_task, nxt_py_add_done_callback_str, done, NULL);

    // 6. 同步等待 startup 完成
    //    这里 run_until_complete 会启动事件循环执行，
    //    直到 lifespan 的 receive 返回 startup 事件，
    //    应用处理完毕后通过 send 返回 startup.complete，
    //    此时 startup_future 的 set_result 被调用，解除阻塞
    res = PyObject_CallFunctionObjArgs(ctx_data->loop_run_until_complete,
                                       lifespan->startup_future, NULL);

    // 7. 检查 startup 是否成功
    if (lifespan->startup_sent == 1 || lifespan->disabled) {
        ret = lifespan;  // 返回 lifespan 对象，保存在 ctx_data->target_lifespans
    }
}
```

### 7.3 Lifespan Receive 的状态机

```c
// nxt_python_asgi_lifespan.c:402-443
static PyObject *nxt_py_asgi_lifespan_receive(PyObject *self, PyObject *none)
{
    future = PyObject_CallObject(ctx_data->loop_create_future, NULL);

    if (!lifespan->startup_received) {
        // 首次 receive → 返回 lifespan.startup 事件
        lifespan->startup_received = 1;
        msg = nxt_py_asgi_new_msg(NULL, nxt_py_lifespan_startup_str);
        return nxt_py_asgi_set_result_soon(NULL, ctx_data, future, msg);
    }

    if (lifespan->shutdown_called && !lifespan->shutdown_received) {
        // shutdown 被调用 → 返回 lifespan.shutdown 事件
        lifespan->shutdown_received = 1;
        msg = nxt_py_asgi_new_msg(NULL, nxt_py_lifespan_shutdown_str);
        return nxt_py_asgi_set_result_soon(NULL, ctx_data, future, msg);
    }

    // 等待后续事件（通常是等待 shutdown）
    lifespan->receive_future = future;
    Py_INCREF(future);
    return future;
}
```

### 7.4 Lifespan Send 的状态转换

```c
// nxt_python_asgi_lifespan.c:446-505
static PyObject *nxt_py_asgi_lifespan_send(PyObject *self, PyObject *dict)
{
    type_str = PyUnicode_AsUTF8AndSize(type, &type_len);

    // 根据消息类型分发
    if (== "lifespan.startup.complete")
        return nxt_py_asgi_lifespan_send_startup(lifespan, 0, NULL);
    if (== "lifespan.startup.failed")
        return nxt_py_asgi_lifespan_send_startup(lifespan, 1, message);
    if (== "lifespan.shutdown.complete")
        return nxt_py_asgi_lifespan_send_shutdown(lifespan, 0, NULL);
    if (== "lifespan.shutdown.failed")
        return nxt_py_asgi_lifespan_send_shutdown(lifespan, 1, message);

    // 未知类型 → 禁用 lifespan
    return nxt_py_asgi_lifespan_disable(lifespan);
}
```

`send_startup` 的核心逻辑：

```c
// nxt_python_asgi_lifespan.c:508-528
static PyObject *nxt_py_asgi_lifespan_send_startup(
    nxt_py_asgi_lifespan_t *lifespan, int v, PyObject *message)
{
    // v=0 表示 success, v=1 表示 failed
    return nxt_py_asgi_lifespan_send_(lifespan, v,
                                      &lifespan->startup_sent,
                                      &lifespan->startup_future);
}
```

`send_` 是通用的状态转换函数：

```c
// nxt_python_asgi_lifespan.c:531-563
static PyObject *nxt_py_asgi_lifespan_send_(
    nxt_py_asgi_lifespan_t *lifespan, int v, int *sent, PyObject **pfuture)
{
    if (*sent) {
        // 重复发送 → 禁用 lifespan
        return nxt_py_asgi_lifespan_disable(lifespan);
    }

    *sent = 1 + v;  // 1=success, 2=failed

    if (*pfuture != NULL) {
        future = *pfuture;
        *pfuture = NULL;

        // 设置 Future 结果，解除 startup/shutdown 的 run_until_complete 阻塞
        PyObject_CallMethodObjArgs(future, nxt_py_set_result_str, Py_None, NULL);
        Py_DECREF(future);
    }

    Py_INCREF(lifespan);
    return (PyObject *) lifespan;
}
```

### 7.5 Lifespan Shutdown 流程

Shutdown 在 `nxt_py_asgi_lifespan_shutdown()` 中执行，**在事件循环结束后同步完成**：

```c
// nxt_python_asgi_lifespan.c:345-399
static int nxt_py_asgi_lifespan_target_shutdown(
    nxt_py_asgi_lifespan_t *lifespan)
{
    if (lifespan == NULL || lifespan->disabled)
        return NXT_UNIT_OK;

    lifespan->shutdown_called = 1;

    // 如果应用正在 await receive()，立即发送 shutdown 事件
    if (lifespan->receive_future != NULL) {
        future = lifespan->receive_future;
        lifespan->receive_future = NULL;

        msg = nxt_py_asgi_new_msg(NULL, nxt_py_lifespan_shutdown_str);
        PyObject_CallMethodObjArgs(future, nxt_py_set_result_str, msg, NULL);
    }

    // 创建 shutdown_future 并同步等待 shutdown 完成
    lifespan->shutdown_future = PyObject_CallObject(
        ctx_data->loop_create_future, NULL);

    res = PyObject_CallFunctionObjArgs(ctx_data->loop_run_until_complete,
                                       lifespan->shutdown_future, NULL);
}
```

### 7.6 `lifespan.state` 扩展

`state` 属性通过 Python 的 `PyMemberDef` 机制暴露为只读属性：

```c
// nxt_python_asgi_lifespan.c:57-75
static PyMemberDef nxt_py_asgi_lifespan_members[] = {
    {
        .name   = "state",
        .type   = T_OBJECT_EX,
        .offset = offsetof(nxt_py_asgi_lifespan_t, state),
        .flags  = READONLY,
        .doc    = PyDoc_STR("lifespan.state")
    },
    { NULL, 0, 0, 0, NULL }
};
```

`state` 在 startup 时创建并注入 scope：

```c
// nxt_python_asgi_lifespan.c:198-212
lifespan->state = PyDict_New();
PyDict_SetItem(scope, nxt_py_state_str, lifespan->state);
```

在每个 HTTP/WebSocket 请求中，`state` 被复制一份：

```c
// nxt_python_asgi.c:499-530
lifespan = ctx_data->target_lifespans[req->request->app_target];
state = PyObject_GetAttr(lifespan, nxt_py_state_str);
newstate = PyDict_Copy(state);  // 浅拷贝 state 字典
PyDict_SetItem(scope, nxt_py_state_str, newstate);
```

这个设计遵循 ASGI 3.0 规范：`state` 是应用在 lifespan 启动期间设置的共享状态，每个请求获得一份拷贝，确保请求间不会互相干扰。

### 7.7 Lifespan 完整生命周期时序

```
启动阶段:
  nxt_python_asgi_startup()
    │
    ├── 创建 lifespan 对象
    ├── app(scope, receive, send) → 返回协程
    ├── create_task(coro)         → 协程开始执行
    │     │
    │     ├── 应用内部: await receive()
    │     │     └── 返回 lifespan.startup 事件
    │     │
    │     ├── 应用内部: await send({"type": "lifespan.startup.complete"})
    │     │     └── send_startup() → startup_future.set_result(None)
    │     │
    │     └── 应用内部: await receive()  ← 挂起，等待 shutdown
    │           └── 保存 receive_future
    │
    ├── run_until_complete(startup_future) ← 阻塞解除
    └── 保存 lifespan 对象

请求处理阶段:
  nxt_py_asgi_request_handler()
    └── 从 lifespan 获取 state 的拷贝注入 scope

关闭阶段:
  nxt_py_asgi_lifespan_shutdown()
    │
    ├── 设置 shutdown_called = 1
    ├── 如果有 receive_future:
    │     └── receive_future.set_result(lifespan.shutdown)
    │
    ├── 应用内部: 收到 shutdown 事件
    ├── 应用内部: await send({"type": "lifespan.shutdown.complete"})
    │     └── send_shutdown() → shutdown_future.set_result(None)
    │
    └── run_until_complete(shutdown_future) ← 阻塞解除
```

## 8. 总结：设计哲学

### 8.1 WSGI vs ASGI 实现对比

| 维度 | WSGI | ASGI |
|------|------|------|
| 事件循环 | `nxt_unit_run()`（C 循环） | `loop.run_until_complete()`（Python 循环） |
| GIL 管理 | `PyEval_SaveThread/RestoreThread` | 通过 asyncio 自动管理 |
| 请求处理 | 同步阻塞，handler 内完成整个请求 | 异步非阻塞，handler 仅启动 Task |
| 请求体读取 | `nxt_unit_request_read()` 直接读取 | 通过 Future + receive 回调 |
| 响应发送 | `nxt_unit_response_write()` 直接写入 | 通过 Future 实现背压控制 |
| 生命周期 | 无 | Lifespan 协议（startup/shutdown） |
| 多线程 | 通过 `thread_state` 管理 | 每线程独立事件循环 |
| 代码复杂度 | 1 个文件（39KB） | 6 个文件（120KB） |

### 8.2 关键设计决策

1. **共享库 API 模式**：通过 `nxt_unit.c` 提供的 C API，语言模块可以在自己的进程中运行，同时与 Unit 核心高效通信。这种设计使得 Unit 可以支持任意语言，而核心保持精简。

2. **asyncio 事件循环作为主循环**：ASGI 模式下，Unit 放弃了自己的 C 事件循环，转而将 C 层的 fd 注册到 asyncio 的 poll 中。这是一种"以应用为中心"的设计，确保 ASGI 应用的协程在原生 asyncio 环境中运行，而不是在某种模拟的异步环境中。

3. **Future 桥接异步世界**：C 层的事件通过 Python Future 与 asyncio 集成。C 回调通过 `future.set_result()` 或 `future.set_exception()` 来唤醒等待中的协程，而协程的异步操作（如 `send` 的背压控制）也通过 Future 来协调。

4. **`call_soon` 保证线程安全**：所有从 C 回调中对 Future 的操作都通过 `loop.call_soon` 调度到事件循环线程中执行，避免了多线程竞态条件。

5. **`state` 的拷贝语义**：每个请求获得 `lifespan.state` 的浅拷贝，既允许应用在启动时初始化共享状态（如数据库连接池），又保证了请求间的隔离性。
