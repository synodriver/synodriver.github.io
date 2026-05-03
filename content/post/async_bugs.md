+++
date = '2026-04-25T03:11:51+08:00'
draft = false
title = '异步三坑'
author = 'synodriver'
+++

# 异步三坑

### 1. 论cancel

*asyncio.CancelledError* 简直是神出鬼没，任何await的地方都可能冒出来，对他们的恰当处理是重中之重

### 2. 论Future

*Future.set_result()* 和 *Future.set_exception()* 这两个方法是Future的核心，然而，调用的时候是否判断了他的状态？
如果Future已经完成了，再调用*set_result()*或者*set_exception()*，就会抛出*InvalidStateError*，
与上一个坑结合起来，*await future*的时候被cancel了，抛出CancelledError，future的状态已经是cancelled，这时候就不能再对future搞事了。

### 3. 论callback

*Future.add_done_callback()* 可以往future里面塞各种稀奇古怪的函数，哪怕这个函数会改变future的自己和依赖的状态，理论上来说他们会在set_result后
的下一个loop调用(多亏了call_soon)，不过天知道这些callback会惹出什么破事。

<!-- 
└── (T) Task-1
    └──  HypercornAsyncioWorker._asyncio_serve /root/miniconda3/envs/cp314t/lib/python3.14t/site-packages/hypercorn/workers.py:385
        └──  wait /root/miniconda3/envs/cp314t/lib/python3.14t/asyncio/tasks.py:432
            └──  _wait /root/miniconda3/envs/cp314t/lib/python3.14t/asyncio/tasks.py:521
                ├── (T) Task-2
                │   └──  serve /root/miniconda3/envs/cp314t/lib/python3.14t/site-packages/hypercorn/asyncio/__init__.py:44
                │       └──  worker_serve /root/miniconda3/envs/cp314t/lib/python3.14t/site-packages/hypercorn/asyncio/run.py:153
                │           └──  TaskGroup.__aexit__ /root/miniconda3/envs/cp314t/lib/python3.14t/asyncio/taskgroups.py:72
                │               └──  TaskGroup._aexit /root/miniconda3/envs/cp314t/lib/python3.14t/asyncio/taskgroups.py:121
                │                   ├── (T) Task-5
                │                   │   └──  raise_shutdown /root/miniconda3/envs/cp314t/lib/python3.14t/site-packages/hypercorn/utils.py:173
                │                   │       └──  Event.wait /root/miniconda3/envs/cp314t/lib/python3.14t/asyncio/locks.py:213
                │                   └── (T) Task-6
                │                       └──  raise_shutdown /root/miniconda3/envs/cp314t/lib/python3.14t/site-packages/hypercorn/utils.py:173
                │                           └──  EventWrapper.wait /root/miniconda3/envs/cp314t/lib/python3.14t/site-packages/hypercorn/asyncio/worker_context.py:45
                │                               └──  Event.wait /root/miniconda3/envs/cp314t/lib/python3.14t/asyncio/locks.py:213
                └── (T) Task-3
                    └──  HypercornAsyncioWorker.asyncio_callback_notify /root/miniconda3/envs/cp314t/lib/python3.14t/site-packages/hypercorn/workers.py:425
                        └──  sleep /root/miniconda3/envs/cp314t/lib/python3.14t/asyncio/tasks.py:702
└── (T) Task-4
    └──  Lifespan.handle_lifespan /root/miniconda3/envs/cp314t/lib/python3.14t/site-packages/hypercorn/asyncio/lifespan.py:55
        └──  ASGIWrapper.__call__ /root/miniconda3/envs/cp314t/lib/python3.14t/site-packages/hypercorn/app_wrappers.py:34
            └──  FastAPI.__call__ /root/miniconda3/envs/cp314t/lib/python3.14t/site-packages/fastapi/applications.py:1134
                └──  Starlette.__call__ /root/miniconda3/envs/cp314t/lib/python3.14t/site-packages/starlette/applications.py:113
                    └──  ServerErrorMiddleware.__call__ /root/miniconda3/envs/cp314t/lib/python3.14t/site-packages/starlette/middleware/errors.py:151
                        └──  ExceptionMiddleware.__call__ /root/miniconda3/envs/cp314t/lib/python3.14t/site-packages/starlette/middleware/exceptions.py:49
                            └──  AsyncExitStackMiddleware.__call__ /root/miniconda3/envs/cp314t/lib/python3.14t/site-packages/fastapi/middleware/asyncexitstack.py:18
                                └──  Router.__call__ /root/miniconda3/envs/cp314t/lib/python3.14t/site-packages/starlette/routing.py:716
                                    └──  Router.app /root/miniconda3/envs/cp314t/lib/python3.14t/site-packages/starlette/routing.py:725
                                        └──  Router.lifespan /root/miniconda3/envs/cp314t/lib/python3.14t/site-packages/starlette/routing.py:701
                                            └──  Lifespan.asgi_receive /root/miniconda3/envs/cp314t/lib/python3.14t/site-packages/hypercorn/asyncio/lifespan.py:108
                                                └──  Queue.get /root/miniconda3/envs/cp314t/lib/python3.14t/asyncio/queues.py:186
-->

