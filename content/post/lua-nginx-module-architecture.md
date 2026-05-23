---
date: '2026-05-15T00:26:40+08:00'
draft: false
title: 'lua-nginx-module 设计思路与架构深度剖析'
author: 'synodriver'
tags: ["nginx", "lua", "c"]
---

# lua-nginx-module 设计思路与架构深度剖析

## 概述

[lua-nginx-module](https://github.com/openresty/lua-nginx-module)（又称 `ngx_lua`）是一个将 LuaJIT 嵌入 Nginx HTTP 服务器的 C 语言模块，是 [OpenResty](https://openresty.org) 的核心组件。它通过将 Lua 协程与 Nginx 事件模型深度融合，实现了 **100% 非阻塞** 的网络 I/O 操作，使得在 Nginx 中编写高性能 Web 应用成为可能。本文基于模块源码（v0.10.29），从设计哲学、核心架构、关键机制等多个维度进行深入剖析。

---

## 设计哲学

### 1. 非阻塞 I/O 为第一要务

传统 Apache `mod_lua` 和 Lighttpd `mod_magnet` 嵌入脚本语言后，脚本中的阻塞 I/O 操作会拖慢整个服务器。`ngx_lua` 的核心设计目标就是：**只要使用模块提供的 Lua API，所有网络操作都必须是非阻塞的**。

这通过以下方式实现：
- Lua 代码中遇到网络 I/O 时，主动让出（yield）当前 Lua 协程
- 控制权交还 Nginx 事件循环，由事件循环在 I/O 就绪时恢复（resume）协程
- 整个过程对 Lua 代码完全透明——开发者看到的是同步风格的代码，底层却是异步非阻塞的

### 2. 协程驱动的请求处理

模块利用 Lua 协程（coroutine）来隔离每个请求的执行上下文。每个请求在对应的 Nginx 处理阶段中，都会创建一个独立的 Lua 协程来运行用户代码。这种设计的优势：

- **轻量级上下文切换**：Lua 协程的 yield/resume 开销极小，远低于操作系统线程
- **请求隔离**：每个请求在自己的协程中执行，全局环境通过沙箱机制隔离
- **协作式调度**：协程在 I/O 等待时主动让出，避免抢占式调度的复杂性

### 3. 单 Lua VM 共享

每个 Nginx Worker 进程只创建一个 LuaJIT VM 实例（`lua_State`），所有请求共享该 VM。这带来两个关键优势：

- **极低的内存开销**：加载的 Lua 模块在 Worker 进程级别只存在一份拷贝
- **零拷贝的代码共享**：利用操作系统的 Copy-on-Write（COW）机制，`init_by_lua_block` 阶段加载的代码和数据在 fork 后被所有 Worker 进程共享

### 4. 与 Nginx 处理阶段的深度集成

模块不是简单地在某个阶段嵌入 Lua 脚本，而是几乎覆盖了 Nginx HTTP 请求处理的全部阶段，使得 Lua 代码可以在请求生命周期的每一个关键节点介入。

---

## 整体架构

```
┌──────────────────────────────────────────────────────────────────────┐
│                         Nginx Master Process                         │
│   ┌────────────────────────────────────────────────────────────┐     │
│   │              init_by_lua_block (配置加载阶段)                │     │
│   │        创建 Lua VM → 预加载模块 → 初始化共享字典            │     │
│   └────────────────────────────────────────────────────────────┘     │
│                                 │ fork() + COW                       │
├─────────────────────────────────┼────────────────────────────────────┤
│                    Nginx Worker Process                              │
│                                 ▼                                    │
│   ┌──────────────────────────────────────────────────────────────┐   │
│   │              init_worker_by_lua_block (Worker 启动阶段)      │   │
│   │          创建定时器 → 初始化 Worker 级数据 → 健康检查          │   │
│   └──────────────────────────────────────────────────────────────┘   │
│                                                                      │
│   ┌──────────────────────────────────────────────────────────────┐   │
│   │                    单个 LuaJIT VM 实例                        │   │
│   │   ┌─────────┐ ┌──────────┐ ┌──────────┐ ┌───────────────┐   │   │
│   │   │代码缓存  │ │协程注册表│ │Socket池  │ │共享字典(SHM)  │   │   │
│   │   └─────────┘ └──────────┘ └──────────┘ └───────────────┘   │   │
│   └──────────────────────────────────────────────────────────────┘   │
│                                                                      │
│   ┌───────────── HTTP 请求处理流水线 ──────────────────────────┐    │
│   │                                                            │    │
│   │  ┌──────────────────┐   ┌───────────────────────┐         │    │
│   │  │ server_rewrite  │──▶│server_rewrite_by_lua  │         │    │
│   │  │ _by_lua_block    │   └───────────┬───────────┘         │    │
│   │  └──────────────────┘               │                     │    │
│   │                                     ▼                     │    │
│   │  ┌──────────────────┐   ┌───────────────────────┐         │    │
│   │  │ rewrite         │──▶│ rewrite_by_lua_block  │         │    │
│   │  │ _by_lua_block    │   └───────────┬───────────┘         │    │
│   │  └──────────────────┘               │                     │    │
│   │                                     ▼                     │    │
│   │  ┌──────────────────┐   ┌───────────────────────┐         │    │
│   │  │ access          │──▶│ access_by_lua_block    │         │    │
│   │  │ _by_lua_block    │   └───────────┬───────────┘         │    │
│   │  └──────────────────┘               │                     │    │
│   │                                     ▼                     │    │
│   │  ┌──────────────────┐   ┌───────────────────────┐         │    │
│   │  │ precontent      │──▶│ precontent_by_lua      │         │    │
│   │  │ _by_lua_block    │   └───────────┬───────────┘         │    │
│   │  └──────────────────┘               │                     │    │
│   │                                     ▼                     │    │
│   │  ┌──────────────────┐   ┌───────────────────────┐         │    │
│   │  │ content          │──▶│ content_by_lua_block  │         │    │
│   │  │ _by_lua_block    │   └───────────┬───────────┘         │    │
│   │  └──────────────────┘               │                     │    │
│   │                          ┌───────────┴───────────┐         │    │
│   │                          ▼                       ▼         │    │
│   │              ┌──────────────────┐  ┌──────────────────┐   │    │
│   │              │ header_filter    │  │ body_filter      │   │    │
│   │              │ _by_lua_block   │  │ _by_lua_block    │   │    │
│   │              └────────────────┘  └──────────────────┘   │    │
│   │                                                          │    │
│   │              ┌──────────────────┐                         │    │
│   │              │ log_by_lua_block │                         │    │
│   │              └──────────────────┘                         │    │
│   └──────────────────────────────────────────────────────────┘    │
│                                                                      │
│   ┌─────────────────── SSL/TLS 钩子 ───────────────────────────┐   │
│   │ ssl_client_hello_by_lua* → ssl_certificate_by_lua*        │   │
│   │ ssl_session_store_by_lua* → ssl_session_fetch_by_lua*     │   │
│   └────────────────────────────────────────────────────────────┘   │
│                                                                      │
│   ┌─────────────────── 负载均衡 ────────────────────────────────┐   │
│   │ balancer_by_lua_block → 动态选择后端服务器                    │   │
│   └────────────────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────────────────┘
```

---

## 源码结构

模块的 C 源码位于 `src/` 目录下，按功能划分为约 70 个源文件。下面是核心文件及其职责的分类梳理：

### 核心框架

| 文件 | 职责 |
|------|------|
| `ngx_http_lua_module.c` | 模块入口：定义指令数组、配置创建/合并、postconfiguration 钩子、Lua VM 初始化 |
| `ngx_http_lua_common.h` | 核心头文件：定义所有关键数据结构（main_conf、srv_conf、loc_conf、ctx、co_ctx） |
| `ngx_http_lua_util.c/h` | 工具函数集：Lua VM 初始化、协程创建/调度、请求清理、输出链管理 |
| `ngx_http_lua_directive.c/h` | 指令解析通用逻辑 |
| `ngx_http_lua_config.c/h` | nginx 配置在 Lua 侧的接口 |

### 各阶段处理器

| 文件 | 对应指令 |
|------|---------|
| `ngx_http_lua_initby.c` | `init_by_lua_block` |
| `ngx_http_lua_initworkerby.c` | `init_worker_by_lua_block` |
| `ngx_http_lua_exitworkerby.c` | `exit_worker_by_lua_block` |
| `ngx_http_lua_setby.c` | `set_by_lua_block` |
| `ngx_http_lua_server_rewriteby.c` | `server_rewrite_by_lua_block` |
| `ngx_http_lua_rewriteby.c` | `rewrite_by_lua_block` |
| `ngx_http_lua_accessby.c` | `access_by_lua_block` |
| `ngx_http_lua_precontentby.c` | `precontent_by_lua_block` |
| `ngx_http_lua_contentby.c` | `content_by_lua_block` |
| `ngx_http_lua_headerfilterby.c` | `header_filter_by_lua_block` |
| `ngx_http_lua_bodyfilterby.c` | `body_filter_by_lua_block` |
| `ngx_http_lua_logby.c` | `log_by_lua_block` |
| `ngx_http_lua_balancer.c` | `balancer_by_lua_block` |

### 网络 I/O

| 文件 | 职责 |
|------|------|
| `ngx_http_lua_socket_tcp.c/h` | TCP cosocket 实现（非阻塞 TCP 连接、收发数据、连接池） |
| `ngx_http_lua_socket_udp.c/h` | UDP cosocket 实现 |
| `ngx_http_lua_sleep.c/h` | `ngx.sleep` 非阻塞睡眠 |

### Lua API 暴露

| 文件 | 职责 |
|------|------|
| `ngx_http_lua_args.c/h` | 请求参数解析 |
| `ngx_http_lua_headers.c/h` | 请求/响应头操作 |
| `ngx_http_lua_headers_in.c/h` | 请求头解析 |
| `ngx_http_lua_headers_out.c/h` | 响应头设置 |
| `ngx_http_lua_output.c/h` | `ngx.say`/`ngx.print` 等输出 API |
| `ngx_http_lua_control.c/h` | `ngx.exit`/`ngx.redirect`/`ngx.exec` 等控制 API |
| `ngx_http_lua_req_body.c/h` | 请求体读取 |
| `ngx_http_lua_shdict.c/h` | 共享字典（`lua_shared_dict`） |
| `ngx_http_lua_regex.c` | 正则表达式（`ngx.re.match` 等） |
| `ngx_http_lua_log.c/h` | 日志 API |
| `ngx_http_lua_ctx.c/h` | `ngx.ctx` 请求上下文 |
| `ngx_http_lua_coroutine.c/h` | 协程 API |
| `ngx_http_lua_timer.c` | `ngx.timer.at` 定时器 |
| `ngx_http_lua_semaphore.c/h` | 信号量 |
| `ngx_http_lua_pipe.c/h` | 子进程管道 |

### 基础设施

| 文件 | 职责 |
|------|------|
| `ngx_http_lua_cache.c/h` | Lua 代码缓存 |
| `ngx_http_lua_clfactory.c/h` | 闭包工厂（Closure Factory）：代码加载器 |
| `ngx_http_lua_capturefilter.c/h` | 子请求响应体捕获过滤器 |
| `ngx_http_lua_exception.c/h` | Lua 异常处理 |
| `ngx_http_lua_pcrefix.c/h` | PCRE 正则集成 |
| `ngx_http_lua_ssl*.c/h` | SSL/TLS 相关钩子 |
| `ngx_http_lua_uthread.c` | 用户轻线程（`ngx.thread.spawn`） |
| `ngx_http_lua_input_filters.c/h` | 输入过滤器 |

---

## 核心数据结构

### 1. `ngx_http_lua_main_conf_t` — 主配置

```c
struct ngx_http_lua_main_conf_s {
    lua_State *lua;           // 唯一的 LuaJIT VM 实例
    ngx_str_t lua_path;       // lua_package_path
    ngx_str_t lua_cpath;      // lua_package_cpath
    ngx_pool_t *pool;

    // 定时器管理
    ngx_int_t max_pending_timers;
    ngx_int_t pending_timers;
    ngx_int_t max_running_timers;
    ngx_int_t running_timers;

    // 代码缓存
    ngx_int_t lua_thread_cache_max_entries;
    ngx_queue_t free_lua_threads;     // 空闲 Lua 线程队列
    ngx_queue_t cached_lua_threads;   // 缓存的 Lua 线程队列

    // 正则缓存
    ngx_int_t regex_cache_entries;
    ngx_int_t regex_cache_max_entries;

    // 共享字典
    ngx_array_t *shm_zones;
    ngx_array_t *shdict_zones;

    // 各阶段初始化处理器
    ngx_http_lua_main_conf_handler_pt init_handler;
    ngx_http_lua_main_conf_handler_pt init_worker_handler;
    ngx_http_lua_main_conf_handler_pt exit_worker_handler;

    // 阶段需求标志位
    unsigned requires_header_filter:1;
    unsigned requires_body_filter:1;
    unsigned requires_capture_filter:1;
    unsigned requires_rewrite:1;
    unsigned requires_access:1;
    unsigned requires_log:1;
    unsigned requires_shm:1;
    unsigned requires_capture_log:1;
    unsigned requires_server_rewrite:1;
    unsigned requires_precontent:1;
    // ...
};
```

**关键设计点**：
- `lua` 字段指向 Worker 进程中唯一的 LuaJIT VM 实例
- `requires_*` 标志位采用位域（bitfield），用于惰性注册阶段处理器——只有当配置中实际使用了对应的 `*_by_lua*` 指令时，才注册相应的 Nginx 阶段处理器，避免不必要的性能开销
- `free_lua_threads` / `cached_lua_threads` 构成了 Lua 协程对象的缓存池，通过 `lua_resetthread` API 复用已终止的协程对象

### 2. `ngx_http_lua_loc_conf_t` — 位置配置

```c
struct ngx_http_lua_loc_conf_s {
    // 各阶段源码与处理器
    ngx_http_handler_pt rewrite_handler;
    ngx_http_handler_pt access_handler;
    ngx_http_handler_pt precontent_handler;
    ngx_http_handler_pt content_handler;
    ngx_http_handler_pt log_handler;
    ngx_http_handler_pt header_filter_handler;
    ngx_http_output_body_filter_pt body_filter_handler;

    // 各阶段 Lua 源码（内联/文件路径）
    ngx_http_complex_value_t rewrite_src;
    ngx_http_complex_value_t access_src;
    ngx_http_complex_value_t content_src;
    // ...

    // 缓存键（MD5 摘要）与引用
    u_char *rewrite_src_key;
    int     rewrite_src_ref;   // Lua registry 引用
    u_char *rewrite_chunkname;
    // ...

    // cosocket 配置
    ngx_msec_t keepalive_timeout;
    ngx_msec_t connect_timeout;
    ngx_msec_t send_timeout;
    ngx_msec_t read_timeout;
    size_t     buffer_size;
    ngx_uint_t pool_size;

    // 功能开关
    ngx_flag_t enable_code_cache;
    ngx_flag_t http10_buffering;
    ngx_flag_t check_client_abort;
    // ...
};
```

**关键设计点**：
- 每个阶段的 Lua 代码都有三要素：`src`（源码/文件路径）、`src_key`（缓存键，基于源码的 MD5 摘要）、`src_ref`（Lua registry 引用，用于快速查找已编译的闭包）
- `*_handler` 是 C 函数指针，在对应 Nginx 处理阶段被回调时执行 Lua 代码
- cosocket 相关的超时和连接池配置可以精确到 location 级别

### 3. `ngx_http_lua_ctx_t` — 请求上下文

```c
typedef struct ngx_http_lua_ctx_s {
    ngx_http_lua_vm_state_t *vm_state;  // 非 code_cache 模式下的独立 VM
    ngx_http_request_t *request;

    // 协程调度
    ngx_http_handler_pt resume_handler; // 协程恢复时的回调
    ngx_http_lua_co_ctx_t *cur_co_ctx;  // 当前运行的协程上下文
    ngx_list_t *user_co_ctx;            // 用户协程上下文列表
    ngx_http_lua_co_ctx_t entry_co_ctx; // 入口协程上下文
    ngx_http_lua_co_ctx_t *on_abort_co_ctx;

    int ctx_ref;               // Lua registry 中 ngx.ctx 的引用
    uint32_t context;          // 当前执行的指令上下文（阶段标志）

    // 输出缓冲
    ngx_chain_t *out;
    ngx_chain_t *free_bufs;
    ngx_chain_t *busy_bufs;

    // 输出缓冲管理
    unsigned flushing_coros;          // 等待 ngx.flush(true) 的协程数

    // 子请求响应体收集
    ngx_chain_t *body;               // 缓冲的子请求响应体链
    ngx_chain_t **last_body;

    // 状态标志
    unsigned exited:1;
    unsigned eof:1;
    unsigned capture:1;
    unsigned buffering:1;
    unsigned header_sent:1;
    // ...
} ngx_http_lua_ctx_t;
```

**关键设计点**：
- `cur_co_ctx` 指向当前正在执行的协程，在 yield/resume 时切换
- `context` 字段（32位位掩码）标记当前所处的 Nginx 阶段，用于 API 上下文检查（如 cosocket 不能在 `set_by_lua*` 中使用）
- 子请求相关字段（`sr_statuses`/`sr_bodies`/`sr_headers`）位于 `ngx_http_lua_co_ctx_t` 中，而非本结构体

### 4. `ngx_http_lua_co_ctx_t` — 协程上下文

```c
struct ngx_http_lua_co_ctx_s {
    void *data;                   // 用户数据（如 cosocket 状态）
    lua_State *co;                // Lua 协程的 lua_State
    ngx_http_lua_co_ctx_t *parent_co_ctx;  // 父协程

    ngx_http_lua_posted_thread_t *zombie_child_threads;       // 已结束的子线程链表
    ngx_http_lua_posted_thread_t **next_zombie_child_thread;  // 子线程链表尾指针
    ngx_http_cleanup_pt cleanup;  // 清理回调

    ngx_int_t *sr_statuses;       // 子请求状态码数组
    ngx_http_headers_out_t **sr_headers;  // 子请求响应头数组
    ngx_str_t *sr_bodies;          // 子请求响应体数组
    uint8_t *sr_flags;             // 子请求标志位数组

    unsigned nresults_from_worker_thread; // worker thread 回调的返回值数量
    unsigned nrets;                // ngx_http_lua_run_thread 的 nrets 参数
    unsigned nsubreqs;             // 子请求数量
    unsigned pending_subreqs;      // 等待中的子请求数量

    ngx_event_t sleep;            // ngx.sleep 用的事件
    ngx_queue_t sem_wait_queue;   // 信号量等待队列

    int co_ref;                   // Lua registry 中的引用，防止 GC 回收
    unsigned co_status:3;         // 协程状态（RUNNING/SUSPENDED/NORMAL/DEAD/ZOMBIE）
    unsigned is_uthread:1;        // 是否是 ngx.thread.spawn 创建的用户线程
    unsigned flushing:1;          // 是否在等待 ngx.flush(true)
    unsigned waited_by_parent:1;  // 是否被父协程等待
    unsigned thread_spawn_yielded:1;  // 是否从 ngx.thread.spawn() 调用中 yield
    unsigned is_wrap:1;           // 是否通过 coroutine.wrap 创建
    unsigned propagate_error:1;   // 是否向父协程传播错误
    // ...
};
```

**关键设计点**：
- 协程通过 `co_ref` 锚定在 Lua registry 的 `coroutines_key` 表中，防止被 GC 回收
- `co_status` 定义了 5 种状态：`RUNNING`、`SUSPENDED`、`NORMAL`、`DEAD`、`ZOMBIE`，其中 `ZOMBIE` 状态表示协程已结束但其返回值尚未被父协程收集
- `parent_co_ctx` 构成了协程的树形结构，支持 `ngx.thread.spawn` 创建的用户线程
- `zombie_child_threads` 链表管理已结束但尚未被父协程收集返回值的子线程，避免子线程返回值丢失
- `sr_statuses`/`sr_headers`/`sr_bodies`/`sr_flags` 存储子请求的结果，供 `ngx.location.capture` 使用

---

## 核心机制

### 1. Lua VM 初始化与生命周期

```
                    Nginx Master 进程
                          │
               ┌──────────┴──────────┐
               │  ngx_http_lua_init  │  (postconfiguration)
               │  创建全局 Lua VM    │
               │  加载 resty.core    │
               │  执行 init_by_lua*  │
               └──────────┬──────────┘
                          │ fork()
            ┌─────────────┼─────────────┐
            ▼             ▼             ▼
      Worker 1       Worker 2       Worker N
      (COW 共享       (COW 共享       (COW 共享
       Lua VM)         Lua VM)        Lua VM)
            │             │             │
     init_worker_    init_worker_    init_worker_
     by_lua*         by_lua*         by_lua*
```

VM 初始化的关键步骤（`ngx_http_lua_init_vm`）：

1. **创建 `lua_State`**：调用 `luaL_newstate()`
2. **设置路径**：根据 `lua_package_path` / `lua_package_cpath` 设置模块搜索路径
3. **加载 `resty.core`**：强制加载 `lua-resty-core` 库，该库通过 FFI 重新实现了许多纯 Lua API，大幅提升性能
4. **注册预加载钩子**：通过 `preload_hooks` 数组注册模块预加载器
5. **执行 `init_by_lua*`**：运行用户指定的初始化代码

当 `lua_code_cache off` 时，每个请求会创建独立的 VM 实例（`vm_state` 字段），请求结束后销毁。这是开发模式，生产环境绝不推荐使用。

### 2. 闭包工厂（Closure Factory）与代码缓存

`ngx_http_lua_clfactory.c` 实现了一个精巧的代码加载机制——**闭包工厂**。

对于标准 Lua（非 LuaJIT），加载用户代码时会在前后包装：

```lua
-- 实际加载的代码
return function()
    -- 用户的原始代码
end
```

这样，`lua_load()` 编译出的不是用户代码本身，而是一个返回用户代码闭包的工厂函数。缓存这个工厂函数后，每次执行时调用它即可获得全新的闭包，避免全局变量污染。

**对于 LuaJIT（OpenResty 版本）**，`lua_load` 编译产生的函数可以直接作为闭包被缓存和调用，无需工厂包装。代码缓存查找命中后，OpenResty LuaJIT 直接返回缓存的函数引用；而标准 Lua 则需要调用缓存的工厂函数来生成新闭包。这也是 `#ifdef OPENRESTY_LUAJIT` 条件编译的典型场景——在 `ngx_http_lua_cache_load_code` 中，OpenResty LuaJIT 路径直接使用缓存的函数，而标准 Lua 路径需要 `lua_pcall` 调用工厂函数。

代码缓存的工作流程：

```
                     代码缓存查找流程

  ngx_http_lua_cache_loadbuffer()
            │
            ▼
  ┌─────────────────────┐
  │ 生成缓存键 cache_key │   内联代码: tag + MD5(源码)
  │                     │   文件代码: tag + MD5(文件路径)
  └──────────┬──────────┘
             │
             ▼
  ┌─────────────────────┐     命中
  │ 在 Lua registry     │──────────▶ 返回缓存的闭包
  │ code_cache_key 表中  │
  │ 查找 cache_key      │
  └──────────┬──────────┘
             │ 未命中
             ▼
  ┌─────────────────────┐
  │ clfactory_loadbuffer│   加载代码（带闭包包装）
  │ 或 clfactory_loadfile│
  └──────────┬──────────┘
             │
             ▼
  ┌─────────────────────┐
  │ 存入 code cache 表   │
  │ cache_store_code()  │
  └──────────┬──────────┘
             │
             ▼
        返回新闭包
```

### 3. 协程调度引擎

这是 `ngx_lua` 最核心也最复杂的部分。整个调度模型围绕 `ngx_http_lua_run_thread` 函数构建。

#### 3.1 入口协程的创建与执行

当请求到达某个 `*_by_lua*` 阶段时，处理流程为：

1. 获取请求的 `ngx_http_lua_ctx_t`，若不存在则创建
2. 从主 VM 创建新的 Lua 协程（`lua_newthread`）
3. 将编译好的 Lua 闭包压入新协程的栈
4. 调用 `ngx_http_lua_run_thread` 执行协程

#### 3.2 `ngx_http_lua_run_thread` 调度循环

```
ngx_http_lua_run_thread()
        │
        ▼
  ┌───────────────────┐
  │ lua_resume(co,   │─── 协程正常运行
  │ nret)             │
  └────────┬──────────┘
           │
     ┌─────┴─────┐
     │ Lua yield  │─── 协程主动让出
     │ 或 I/O 等待│
     └─────┬─────┘
           │
           ▼
  ┌──────────────────────────────────────────┐
  │         根据 yield 原因分发处理            │
  ├──────────────────────────────────────────┤
  │                                          │
  │  ┌─ NGX_HTTP_LUA_USER_CORO_YIELD ──┐    │
  │  │  用户协程 yield: 挂起当前协程,     │    │
  │  │  等待 resume 唤醒                  │    │
  │  └──────────────────────────────────┘    │
  │                                          │
  │  ┌─ 子请求完成 ─────────────────────┐    │
  │  │  收集子请求结果, 恢复父协程        │    │
  │  └──────────────────────────────────┘    │
  │                                          │
  │  ┌─ cosocket I/O 就绪 ──────────────┐   │
  │  │  恢复正在等待 I/O 的协程            │   │
  │  └──────────────────────────────────┘    │
  │                                          │
  │  ┌─ ngx.sleep 超时 ──────────────────┐   │
  │  │  恢复正在睡眠的协程                 │   │
  │  └──────────────────────────────────┘    │
  │                                          │
  │  ┌─ 协程正常结束 ───────────────────┐    │
  │  │  处理返回值, 通知父协程             │    │
  │  └──────────────────────────────────┘    │
  │                                          │
  │  ┌─ 协程出错 ───────────────────────┐    │
  │  │  报错, 清理, 退出当前阶段           │    │
  │  └──────────────────────────────────┘    │
  └──────────────────────────────────────────┘
```

#### 3.3 协程状态机

协程在 C 层面的状态转换：

```
                    lua_resume()
  RUNNING ◄──────────────────────────┐
      │                               │
      │ yield (I/O等待/主动让出)        │
      ▼                               │
  SUSPENDED ──── 事件就绪 ────────────┘
      │
      │ ngx.thread.spawn 创建子线程
      ▼
  NORMAL (等待子线程)
      │
      │ 子线程结束
      ▼
  RUNNING
      │
      │ 执行完毕
      ▼
  DEAD / ZOMBIE (等待父协程收集返回值)
```

#### 3.4 用户线程（User Thread）

`ngx.thread.spawn` 创建的用户线程在 C 层面也是一个 Lua 协程，但有以下特殊处理：

- 协程创建后立即被调度执行（不是等待 resume）
- 父协程可以通过 `ngx.thread.wait` 等待子线程完成
- 子线程结束时不会立即被 GC，而是进入 `ZOMBIE` 状态，直到父协程收集其返回值
- 请求结束时，所有未完成的用户线程会被强制终止

### 4. Cosocket 机制

Cosocket（Coroutine Socket）是 `ngx_lua` 实现非阻塞网络 I/O 的核心抽象。

#### 4.1 架构概览

```
  Lua 代码层
  ┌──────────────────────────────┐
  │  local sock = ngx.socket.tcp()│
  │  sock:connect("127.0.0.1", 80)│
  │  sock:send("hello")           │
  │  local data = sock:receive() │
  │  sock:setkeepalive()          │
  └──────────────┬───────────────┘
                 │ Lua → C 调用
  ┌──────────────▼───────────────┐
  │   C 层 cosocket 实现          │
  │                              │
  │  ngx_http_lua_socket_tcp.c   │
  │                              │
  │  ┌────────────────────────┐  │
  │  │ 连接状态机             │  │
  │  │ connect → send → recv  │  │
  │  │     ↑                  │  │
  │  │     └── setkeepalive ──┘  │ (连接回池)
  │  └────────────────────────┘  │
  │                              │
  │  ┌────────────────────────┐  │
  │  │ 连接池                 │  │
  │  │ ngx_http_lua_socket_  │  │
  │  │ pool_t                 │  │
  │  │  ├── cache 队列        │  │
  │  │  ├── free 队列         │  │
  │  │  └── wait_connect_op  │  │
  │  └────────────────────────┘  │
  └──────────────┬───────────────┘
                 │ Nginx 事件驱动
  ┌──────────────▼───────────────┐
  │       Nginx Event Core       │
  │  epoll/kqueue/eventport      │
  └──────────────────────────────┘
```

#### 4.2 非阻塞原理

以 TCP 连接为例：

```
sock:connect(host, port)
        │
        ▼
  ┌──────────────────────┐
  │ 创建 ngx_connection_t│
  │ 发起非阻塞 connect()│
  └──────────┬───────────┘
             │ EINPROGRESS (非阻塞连接进行中)
             ▼
  ┌──────────────────────┐
  │ 注册写事件到         │
  │ Nginx 事件循环       │
  └──────────┬───────────┘
             │
             ▼
  ┌──────────────────────┐
  │ 当前协程 yield      │   ◄── 控制权交还 Nginx 事件循环
  │ 保存 co_ctx 状态     │
  └──────────┬───────────┘
             │
             │ ... Nginx 处理其他请求 ...
             │
             ▼
  ┌──────────────────────┐
  │ 连接就绪事件触发     │
  │ (epoll_wait 返回)    │
  └──────────┬───────────┘
             │
             ▼
  ┌──────────────────────┐
  │ resume 协程           │
  │ 返回连接成功          │   ◄── Lua 代码继续执行
  └──────────────────────┘
```

关键数据结构 `ngx_http_lua_socket_tcp_upstream_t` 中的字段分工：

- `read_co_ctx` / `write_co_ctx`：分别记录等待读/写操作的协程
- `read_event_handler` / `write_event_handler`：I/O 事件就绪时的回调
- `socket_pool`：指向连接池，`setkeepalive` 时将连接放回池中

#### 4.3 连接池

连接池以 `{host}:{port}` 为键，存储在 Lua VM registry 的 `socket_pool_key` 表中：

```
  ngx_http_lua_socket_pool_t
  ┌────────────────────────────────────┐
  │ lua_vm: 指向所属 Lua VM            │
  │ size: 30 (pool_size 配置)          │
  │ connections: 5 (活跃连接数)         │
  │ backlog: 0 (积压的连接等待数)       │
  │ key: "127.0.0.1:8080"             │
  │                                    │
  │ ┌────────┐  ┌────────┐            │
  │ │ cache  │  │ free   │            │
  │ │ 队列   │  │ 队列   │            │
  │ └───┬────┘  └───┬────┘            │
  │     │            │                  │
  │  ┌──▼──┐     ┌──▼──┐              │
  │  │conn1│     │item1│  (空闲连接)   │
  │  │conn2│     │item2│              │
  │  └─────┘     └─────┘              │
  │                                    │
  │ ┌─────────────────┐               │
  │ │ cache_connect_op│ (待执行的连接) │
  │ │ wait_connect_op │ (等待连接的)   │
  │ └─────────────────┘               │
  └────────────────────────────────────┘
```

`setkeepalive` 将连接标记为可复用并放回池中，下次 `connect` 同一目标时直接复用，避免了 TCP 三次握手开销。

### 5. 子请求（Subrequest）机制

`ngx.location.capture` 通过 Nginx 子请求实现：

1. Lua 代码调用 `ngx.location.capture(uri)`
2. C 层创建 Nginx 子请求，挂起当前协程
3. 子请求执行完毕后，通过 `post_subrequest` 回调收集结果
4. 恢复挂起的协程，将子请求的 status/headers/body 作为返回值传递

`ngx.location.capture_multi` 支持并发发起多个子请求：

```
  local res1, res2, res3 = ngx.location.capture_multi({
      {"/uri1", {args = "foo=bar"}},
      {"/uri2", {method = ngx.HTTP_POST}},
      {"/uri3", {}}
  })
```

C 层面通过 `pending_subreqs` 计数器跟踪未完成的子请求数量，所有子请求完成后才恢复父协程。

### 6. 共享字典（Shared Dict）

`lua_shared_dict` 通过 Nginx 的共享内存（`ngx_shm_zone_t`）实现，具有以下特性：

- **进程间共享**：所有 Worker 进程共享同一块内存区域
- **原子操作**：`set`/`get`/`incr` 等操作通过 Nginx 自旋锁保证原子性
- **LRU 淘汰**：可设置 `ttl` 过期时间，过期条目被自动清理
- **非阻塞**：`get` 操作不涉及磁盘 I/O，不会阻塞事件循环

共享内存中的数据结构基于 Nginx 的 `ngx_slab_pool` 内存分配器，使用红黑树 + 链表组织：

```
  ngx_shm_zone_t
  ┌────────────────────────────────────┐
  │  ngx_slab_pool_t (内存分配器)       │
  │  ┌──────────────────────────────┐  │
  │  │ 红黑树: 按 key 排序           │  │
  │  │  ├── "key1" → value1, ttl   │  │
  │  │  ├── "key2" → value2        │  │
  │  │  └── "key3" → value3, ttl   │  │
  │  │                              │  │
  │  │ LRU 队列: 按访问时间排序      │  │
  │  │  → key3 → key1 → key2 →     │  │
  │  └──────────────────────────────┘  │
  └────────────────────────────────────┘
```

### 7. 定时器（Timer）

`ngx.timer.at(delay, callback)` 创建零延迟或延时的定时器回调：

```
  定时器创建流程:

  ngx.timer.at(0, handler)
        │
        ▼
  ┌─────────────────────┐
  │ 在共享内存中注册    │
  │ pending_timers++    │ (如果达到上限则拒绝)
  └──────────┬──────────┘
             │
             ▼
  ┌─────────────────────┐
  │ 创建 Nginx 定时事件 │
  │ ngx_add_timer()     │
  └──────────┬──────────┘
             │
             │ delay 时间后
             ▼
  ┌─────────────────────┐
  │ 定时器事件触发      │
  │ running_timers++    │ (如果达到上限则拒绝)
  │ 创建伪请求          │
  │ 在新协程中执行      │
  │ callback(premature) │
  └─────────────────────┘
```

定时器回调在独立的协程中运行，可以执行任何可 yield 的 API。定时器有两个计数器保护：
- `max_pending_timers`：等待中的定时器上限（默认 1024）
- `max_running_timers`：同时运行的定时器上限（默认 256）

### 8. SSL/TLS 钩子

模块提供了 4 个 SSL 相关的 Lua 钩子，在 TLS 握手的不同阶段介入：

| 钩子 | 触发时机 | 典型用途 |
|------|---------|---------|
| `ssl_client_hello_by_lua*` | 收到 ClientHello 消息后 | 基于 SNI 动态选择证书 |
| `ssl_certificate_by_lua*` | 选择证书时 | 动态加载证书/密钥 |
| `ssl_session_store_by_lua*` | 存储会话时 | 自定义会话存储（如存入 Redis） |
| `ssl_session_fetch_by_lua*` | 获取会话时 | 自定义会话获取 |

这些钩子通过 OpenSSL 的回调机制（如 `SSL_CTX_set_cert_cb`、`SSL_CTX_sess_set_new_cb`）注册，在回调中创建协程执行 Lua 代码。

---

## 请求上下文隔离机制

### 全局环境沙箱

每个请求的入口协程拥有独立的全局环境（globals table），但通过 `__index` 元方法指向原始全局表：

```lua
-- 每个请求的协程全局环境
local new_globals = {}
setmetatable(new_globals, {__index = _G})

-- 请求中对全局变量的读取会回溯到 _G
-- 但写入只在 new_globals 中，不会污染 _G
```

在 C 层面，通过 `ngx_http_lua_new_thread` 实现：

```c
// 为 OpenResty LuaJIT: 通过 lua_getexdata2 / lua_setexdata2 快速获取协程上下文
//   不再需要为每个协程创建独立的全局环境表
// 为标准 LuaJIT (非 OpenResty): 创建新的全局表并设置 __index 元方法
//   新全局表的 __index 指向原始 _G，确保读取回溯但写入隔离
```

### ngx.ctx

`ngx.ctx` 是请求级别的 Lua table，其底层基础设施由 C 层提供（`ctx_ref` 引用），但实际创建和访问逻辑已移至 `lua-resty-core` 的 `resty.core.ctx` 模块中实现：

- 在 `init_by_lua*` / `init_worker_by_lua*` 中不可用
- 在同一请求的不同阶段之间共享数据
- 子请求有独立的 `ngx.ctx`
- 请求结束后自动被 GC

---

## Nginx 处理阶段注册机制

`ngx_http_lua_init`（postconfiguration 回调）中，根据 `requires_*` 标志位动态注册阶段处理器：

```c
if (lmcf->requires_rewrite) {
    h = ngx_array_push(&cmcf->phases[NGX_HTTP_REWRITE_PHASE].handlers);
    *h = ngx_http_lua_rewrite_handler;
}

if (lmcf->requires_access) {
    h = ngx_array_push(&cmcf->phases[NGX_HTTP_ACCESS_PHASE].handlers);
    *h = ngx_http_lua_access_handler;
}
// ...
```

**对于 header filter 和 body filter**，模块使用 Nginx 的 filter 链机制：

```c
// header filter 链
ngx_http_lua_next_header_filter = ngx_http_top_header_filter;
ngx_http_top_header_filter = ngx_http_lua_header_filter;

// body filter 链
ngx_http_lua_next_body_filter = ngx_http_top_body_filter;
ngx_http_top_body_filter = ngx_http_lua_body_filter;
```

**对于 log phase**，模块的 handler 被插入到 handlers 数组的**开头**（通过 `memmove` 移位），确保在其他 log handler 之前执行。

---

## 内存管理策略

### 1. Lua 对象内存

LuaJIT 使用自己的内存分配器（基于 `mmap`），不受 Nginx 内存池管理。模块通过 `lua_malloc_trim` 指令定期调用 `malloc_trim()` 回收 libc 缓存的空闲内存给操作系统。

### 2. Nginx 内存池

模块中的 C 对象使用 Nginx 内存池（`ngx_pool_t`）分配：

- 请求级别的临时对象从 `r->pool` 分配，请求结束时自动释放
- 配置对象从 `cf->pool` 分配，进程生命周期内有效
- 共享内存使用 `ngx_slab_pool_t` 分配，进程间共享

### 3. 缓冲区管理

输出缓冲区使用链表（`ngx_chain_t`）管理，有三级缓存：

- `free_bufs`：空闲缓冲区链表，可复用
- `busy_bufs`：正在使用的缓冲区
- `out`：待发送的输出数据链

### 4. 协程对象缓存

通过 `lua_resetthread`（OpenResty LuaJIT 独有 API），已终止的协程对象可以被重置并复用，避免了频繁的 Lua 线程创建和 GC 开销：

```c
// 缓存池操作
ngx_http_lua_free_thread()   // 协程结束时放入缓存池
ngx_http_lua_new_cached_thread()  // 创建协程时优先从缓存池获取
```

---

## 性能优化策略

### 1. FFI 加速

`lua-resty-core` 使用 LuaJIT FFI 替代纯 Lua 实现，将大量 Lua-C 交互从 Lua C API 调用变为 FFI 直接操作 C 数据结构，性能提升数倍。

### 2. 代码缓存

所有 `*_by_lua_file` 和 `*_by_lua_block` 中的代码在首次执行后被缓存，后续请求直接使用缓存的闭包，避免重复编译。缓存键基于源码内容的 MD5 摘要。

### 3. 正则缓存

`ngx.re.match` 等正则 API 在指定 `o` 选项时，编译后的正则对象会被缓存在 Worker 进程级别，避免重复编译。默认最多缓存 1024 个正则对象。

### 4. 连接池复用

cosocket 连接池避免了频繁的 TCP 连接建立/断开开销，默认 keepalive 超时为 60 秒。

### 5. COW 优化

`init_by_lua_block` 在 Nginx Master 进程中执行，fork 后所有 Worker 进程共享已加载的代码和只读数据，利用操作系统的 Copy-on-Write 机制节省内存。

### 6. LuaJIT 字节码预编译

通过 `luajit -b` 将 `.lua` 文件预编译为字节码，减少运行时编译开销。甚至可以将字节码静态链接到 Nginx 可执行文件中。

---

## 已知限制与设计权衡

### 1. 不可 yield 的上下文

由于 Nginx 的内部限制或模块设计约束，以下上下文中不可执行 yield 操作（即不可使用 cosocket、`ngx.sleep`、子请求等）：

| 上下文 | 原因 |
|--------|------|
| `set_by_lua*` | 在 Nginx rewrite 模块中执行，不支持非阻塞 |
| `header_filter_by_lua*` | Nginx header filter 不支持挂起 |
| `body_filter_by_lua*` | Nginx body filter 不支持挂起 |
| `log_by_lua*` | Nginx log phase 不支持挂起，此时请求处理已接近尾声 |
| `init_by_lua*` | 在 Master 进程中执行，无事件循环 |
| `init_worker_by_lua*` | 可注册定时器并使用定时器中的 cosocket，但不能直接使用 cosocket（在 `init_worker` 阶段本身无法 yield） |

### 2. 进程级数据隔离

每个 Worker 进程有独立的 Lua VM，进程间不能通过 Lua 变量共享数据。跨进程数据共享必须使用：
- `lua_shared_dict`（共享内存）
- 外部存储（Redis、MySQL 等）

### 3. 协程不能跨 C 调用边界 yield

LuaJIT 中 `dofile` 和 `require` 是 C 函数实现，在这些函数的调用栈内 yield 会触发 "attempt to yield across C-call boundary" 错误。

### 4. 全局变量陷阱

每个请求的全局环境是独立的，在请求结束时被清理。通过 `require` 加载的模块中的全局变量会在后续请求中丢失。因此，必须使用 `local` 声明所有变量。

---

## 总结

`lua-nginx-module` 的设计展现了以下几个层次的精妙工程：

1. **语言嵌入层**：通过闭包工厂和代码缓存，高效地将 Lua 代码融入 Nginx 的配置驱动模型
2. **协程调度层**：将 Lua 协程的 yield/resume 语义与 Nginx 事件循环完美对接，实现看似同步实为异步的编程模型
3. **I/O 抽象层**：cosocket 将 Nginx 事件驱动的非阻塞 I/O 封装为 Lua 中自然易用的 socket API
4. **状态管理层**：共享字典、定时器、子请求等机制在单进程内和跨进程间提供了不同粒度的数据共享和任务调度能力
5. **阶段集成层**：几乎覆盖 Nginx 请求处理的所有阶段，使得 Lua 代码可以在请求生命周期的任何关键节点介入

这种分层架构使得 `ngx_lua` 既保持了 Nginx 高性能事件驱动的优势，又赋予了开发者使用脚本语言快速构建复杂 Web 应用的灵活性，堪称 C 模块与脚本语言集成的典范。

---

## 参考资源

- [lua-nginx-module GitHub 仓库](https://github.com/openresty/lua-nginx-module)
- [OpenResty 官方网站](https://openresty.org)
- [lua-resty-core](https://github.com/openresty/lua-resty-core)
- [OpenResty LuaJIT2](https://github.com/openresty/luajit2)
- [Nginx 开发指南](https://nginx.org/en/docs/dev/development_guide.html)
