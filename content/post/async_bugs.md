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