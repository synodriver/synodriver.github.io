---
date: '2026-10-25T00:00:00+08:00'
draft: false
title: 'NoneBot2 源码内幕'
author: 'synodriver'
tags: ["python", "asyncio", "nonebot2", "bot"]
---

# NoneBot2 源码内幕

NoneBot 2 的插件写起来很短：`on_command()` 声明条件，`@matcher.handle()` 编写响应。然而一次消息从网络进入处理函数，中间经过驱动器、协议适配器、事件对象、规则、权限、依赖注入和调度器。本文以本机安装的 **nonebot2 2.5.0** 源码为准，按调用方向拆开这些环节。具体平台协议由单独安装的适配器实现，以下讨论的是 NoneBot 核心提供的接口与调度行为。

## 1. 从一个插件开始

```python
# bot.py
import nonebot
from nonebot.adapters.onebot.v11 import Adapter as OneBotV11Adapter

nonebot.init()
driver = nonebot.get_driver()
driver.register_adapter(OneBotV11Adapter)
nonebot.load_plugins("plugins")

if __name__ == "__main__":
    nonebot.run()
```

```python
# plugins/ping.py
from nonebot import on_command
from nonebot.adapters import Message
from nonebot.params import CommandArg

ping = on_command("ping", priority=5, block=True)

@ping.handle()
async def _(args: Message = CommandArg()):
    name = args.extract_plain_text().strip() or "你"
    await ping.finish(f"pong, {name}")
```

这里的 `nonebot.adapters.onebot.v11` 来自**额外安装的 OneBot V11 适配器包**，并不由 `nonebot2` 核心包自带。消息 `/ping Alice` 到来后，命令规则解析参数，`CommandArg()` 从事件状态取出命令后的 `Message`，处理函数回复并结束。

`nonebot.init()` 在 `nonebot/__init__.py` 中加载配置、解析 `config.driver` 并创建全局 Driver；`register_adapter()` 创建协议适配器实例；`load_plugins()` 导入插件模块，模块执行时就完成 Matcher 注册；`nonebot.run()` 最终调用 `get_driver().run()`。因此，**插件导入是声明响应器的阶段，收到事件是运行响应器的阶段**。漏掉插件加载，装饰器根本不会执行。

## 2. 模块与职责

| 源码位置 | 关键对象或函数 | 职责 |
| --- | --- | --- |
| `nonebot/__init__.py`、`config.py` | `init`、`get_driver`、`run`、`Config` | 初始化配置和驱动器，提供公共入口 |
| `internal/driver/abstract.py`、`drivers/` | `Driver`、`ASGIMixin`、HTTP/WS 客户端接口 | 管理进程生命周期、网络能力、Bot 连接和适配器注册 |
| `internal/adapter/` | `Adapter`、`Bot`、`Event`、`Message` | 把平台协议转换成统一事件接口，并把发送/API 调用交回平台实现 |
| `plugin/load.py`、`plugin/on.py` | `load_plugins`、`on_command` 等 | 导入插件并创建 Matcher 类 |
| `internal/matcher/` | `Matcher`、`MatcherManager` | 保存响应器定义、实例状态与执行逻辑 |
| `message.py` | `handle_event`、`check_and_run_matcher` | 按优先级分发事件，运行前后处理器 |
| `rule.py`、`internal/rule.py`、`internal/permission.py` | `TrieRule`、`Rule`、`Permission` | 命令解析、内容匹配、触发权限 |
| `dependencies/`、`internal/params.py` | `Dependent`、参数解析器 | 在调用规则或处理函数前注入 Bot、Event、State 等参数 |

### 模块依赖与调用图

下面的箭头表示**运行时调用、创建或交付数据**，不是 Python 的 `import` 关系。上半部分是启动时加载 Matcher 定义，下半部分是收到事件后的处理；注册表把两条路径连接起来。

![NoneBot 模块依赖与调用关系图](/images/nonebot-module-dependencies.svg)

`Driver.register_adapter()` 创建适配器实例，适配器又使用 Driver 的网络接口；图中用一条主线表示它们协作，避免把这种双向关系误读成单向的 Python 模块依赖。`Matcher.new()` 在插件加载时写入注册表，`handle_event()` 在运行时从注册表读取、检查并执行 Matcher。核心库**不负责理解每个平台的原始 JSON、消息段和发送 API**；那是适配器的工作。

### 事件与响应的数据流图

这张数据流图沿一条消息展示数据形态怎样变化，并在 handler 处画出发送响应和多轮续接两条路径。普通 `finish()` 不会注册临时 Matcher；`pause()`/`reject()` 会让下一条事件重新进入同一个 `handle_event()` 入口。

![NoneBot 事件接收、Matcher 响应和多轮续接数据流图](/images/nonebot-event-data-flow.svg)

图中「适配器交给 `handle_event`」描述的是适配器实现应完成的路径，抽象基类本身没有替所有协议实现解析器。不同协议可能选择 HTTP 回调、正向 WebSocket 或反向 WebSocket，具体入口以所装适配器源码为准；OneBot V11 则由具体 `Bot.handle_event()` 完成预处理后调用核心函数。

## 3. Driver 和 Adapter 的边界

`Driver` 管**怎么连接**：启动和关闭服务、注册路由、建立 HTTP/WebSocket 客户端连接、维护 Bot 列表以及生命周期钩子。以 `drivers/fastapi.py` 为例，`setup_http_server()` 调用 FastAPI 的 `add_api_route()`，`setup_websocket_server()` 调用 `add_api_websocket_route()`；它们只把传输入口接入框架，不解释某个平台的事件字段。`nonebot.run()` 把控制权交给驱动器，FastAPI Driver 再运行 ASGI 服务。

`Adapter` 管**连接里的协议是什么意思**。`internal/adapter/adapter.py` 提供 `setup_http_server()`、`setup_websocket_server()`、`request()` 和 `websocket()`，内部根据 Driver 是否具备相应 Mixin 来调用传输能力。具体适配器负责认证、连接或回调处理、构造平台 Bot、解析事件、调用 `handle_event()`，并实现 `_call_api()`。`Adapter.bot_connect()`/`bot_disconnect()` 同步 Driver 的 Bot 列表；一个 Adapter 可以管理多个 `self_id`。

`Bot` 是某个已连接机器人的协议 API 门面：`bot.call_api("send_msg", ...)` 最终委托适配器 `_call_api()`；`bot.send(event, message)` 是由具体协议 Bot 实现的基础发送方法。`Matcher.send()` 从当前运行上下文拿到 Bot 和 Event，再调用 `bot.send()`。这种分工让插件多数时候只需认识事件与消息接口，但平台特有的图片段、API 名称及参数仍要看适配器文档。

Driver 的传输能力也可组合。`_resolve_combine_expr()` 解析 `配置中的 driver` 字符串，`combine_driver()` 组合主 Driver 与 Mixin；能否使用某种正向或反向通信，取决于最终 Driver 是否提供对应接口。选择适配器时，应同时核对它要求的驱动能力。

## 4. 事件模型：统一接口之下仍保留协议差异

`internal/adapter/event.py` 中的抽象 `Event` 继承 Pydantic `BaseModel`，要求适配器实现 `get_type()`、`get_event_name()`、`get_user_id()`、`get_session_id()`、`get_message()`、`is_tome()` 等方法。`get_plaintext()` 默认调用 `get_message().extract_plain_text()`。因此 `on_message()` 匹配的不是 Python 子类名，而是 `event.get_type() == "message"`；通知、请求和元事件分别使用 `notice`、`request`、`meta_event` 等类型。平台细分字段和事件类仍由适配器决定。

`Message` 是消息段序列，段可以是文本、图片或其他协议内容。`extract_plain_text()` 只提取文本，不能代替检查完整消息段。`get_session_id()` 则是会话键，后续交互的临时 Matcher 用它锁定下一条事件；若适配器的会话定义不符合业务需求，就需要重新考虑对应事件类或权限策略。

以本文示例所用的 OneBot V11 适配器为例，具体 `Bot.handle_event()` 先规整消息段、检查回复和 @ 机器人等信息，然后调用核心的 `handle_event(self, event)`。它的私聊 `MessageEvent.get_session_id()` 返回用户 ID，群消息 `GroupMessageEvent.get_session_id()` 返回 `group_{group_id}_{user_id}`。这说明会话范围最终由适配器事件类定义；同一个用户在不同群的追问会被区分，但不同 Bot 处理相同会话 ID 时仍需自行考虑隔离。

消息被适配器构造成 `(bot, event)` 后，适配器通常通过 Driver 的任务组调度 `handle_event(bot, event)`。核心的 `message.py` 在此创建**本次事件的** `state`、依赖缓存和 `AsyncExitStack`，先执行事件预处理器，再由 `TrieRule.get_value()` 解析命令前缀与参数，随后才进入 Matcher 调度。`TrieRule` 的结果写在事件 state 中；命令规则和 `CommandArg()` 读取的是这份结果。因此 `/ping Alice` 的参数解析发生在匹配处理函数之前，而不是在插件里重新切字符串。

## 5. Matcher 怎样注册、匹配和响应

`plugin/on.py` 中的 `on()` 把传入的规则包成 `Rule`，把权限包成 `Permission`，然后调用 `Matcher.new()`。`on_message()`、`on_notice()`、`on_command()` 等都是在此基础上加事件类型或特定规则。`on_command()` 还会注册命令前缀；它默认把 `block` 设为 `False`，而 `on_message()` 默认是 `True`，所以是否继续向低优先级传播最好显式指定。

`Matcher.new()` 用 `type(cls.__name__, (cls,), {...})` **生成一个新的 Matcher 子类**。`type`、`rule`、`permission`、`handlers`、`priority`、`block`、`temp`、默认 state 和源码位置都作为这个新类的配置，然后 `matchers[priority].append(NewMatcher)`。`MatcherManager` 的默认后端本质是按整数优先级索引的列表。`@ping.handle()` 调用类方法 `append_handler()`，把函数解析成 `Dependent`，追加到这个新类的 `handlers`；装饰器执行时不会处理任何真实事件。

事件到来后，`handle_event()` 按数字从小到大遍历优先级；同一优先级的 Matcher 在 AnyIO TaskGroup 中并发检查和运行。`_check_matcher()` 先剔除过期定义，再分别执行 `check_perm()` 和 `check_rule()`：两者都检查 `event.get_type()` 是否符合；`Permission` 的多个 checker 是“或”，`Rule` 的多个 checker 是“且”。`_run_matcher()` 对命中的类执行 `matcher = Matcher()`，接着运行 run 前处理器、`matcher.run()`、run 后处理器；实例的 `simple_run()` 按顺序执行处理函数。`block` 或 `stop_propagation()` 阻止后续**更低优先级**继续处理，不能把同优先级并发任务理解为严格串行的抢占式处理。

`check_perm()` 和 `check_rule()` 的类型判断都写成 `event_type == (cls.type or event_type)`，这个表达式很简洁：`cls.type` 非空时，比较事件类型与指定类型；`cls.type == ""` 时，`or` 返回 `event_type`，比较就变成 `event_type == event_type`，自然为真。它等价于“未指定类型就跳过类型限制，否则要求类型相同”的 `if` 分支，也解释了基础 `on()` 为何能用空字符串表示匹配任意事件类型。

`Matcher.__init__()` 复制类上的 handlers 列表到 `remain_handlers`，并复制默认 state。事件调度给每个候选 Matcher 的也是 `state.copy()`。这些都是**浅拷贝**：通常可隔离顶层键的增删，但如果多个响应器共用某个可变嵌套对象，仍可能相互影响。处理函数自己需要避免把跨事件的可变数据放在类属性或共享默认 state 中。

### 为什么是新类，不是新实例？元类究竟做了什么？

这里要区分三个操作：

```python
ping = on_command("ping")  # Matcher.new() 返回一个新子类：注册定义

@ping.handle()              # 修改该子类的 handlers：补充定义
async def handler():
    ...

# 事件匹配成功后，message._run_matcher() 才执行 ping()：创建运行实例
```

如果 `on_command()` 立刻返回一次性的实例，后续装饰器虽然也能修改对象，却必须自行解决“定义对象”和“每次运行状态”的隔离、临时会话的派生，以及按类携带规则和来源信息的问题。当前设计让**类充当可注册的响应器定义，实例充当单次执行上下文**：类可直接作为装饰器入口；每次命中才有独立的 `remain_handlers` 和 `state`；`pause()`/`reject()` 还可从当前类派生新的临时类，把剩余 handlers、会话权限和状态注册回调度表。

但不要把这些能力全部归功于元类。2.5.0 的 `MatcherMeta(type)` **没有重写 `__new__` 或 `__call__`**，运行时代码只自定义了类对象的 `__repr__`，使日志中的响应器定义带有类型、模块和行号；另有少量 `TYPE_CHECKING` 类型声明。真正创建并注册新类的是 `Matcher.new()` 这个普通 `@classmethod`，真正创建实例的是 `_run_matcher()` 中的 `Matcher()`。由于基类使用 `MatcherMeta`，动态生成的子类自然继承该元类。选择动态类主要是为了“定义与运行分离”和可派生的会话续接；此处元类更多承担展示与类型层面的辅助职责。

## 6. 多轮会话：临时 Matcher 如何保存上下文

NoneBot 这里的「session」不是一个一直挂起、等待输入的协程，也没有独立的持久化 Session 对象。`MatcherManager` 默认把 Matcher 类放在进程内存的优先级表中；一次处理结束后，框架靠**新注册的临时 Matcher 类**保存下一步需要的 handlers 和 state。下一条事件仍从 `handle_event()` 的入口重新分发。

### `finish`、`pause`、`reject` 的控制流

这三个方法都通过专用异常退出当前 handler，`Matcher.run()` 再决定后续动作：

| 方法 | 当前事件结束后 | 后续事件 |
| --- | --- | --- |
| `finish(message)` | 可选地发送消息，抛 `FinishedException` | 不注册续接 Matcher；本次执行结束 |
| `pause(prompt)` | 可选地发送提示，抛 `PausedException` | 从**下一个**尚未执行的 handler 继续 |
| `reject(prompt)` | 可选地发送提示，抛 `RejectedException` | 把当前 handler 插回队首，重新执行它 |

这里的“下一条”有条件：必须有新事件通过临时 Matcher 的类型和权限检查，并且尚未过期。`skip()` 则只跳过当前 handler，继续本次事件中的下一个 handler；它不是多轮会话操作。

`simple_run()` 在调用 handler 前先从实例的 `remain_handlers` 弹出它，并用 `current_handler` 记录；因此 `resolve_reject()` 能把刚弹出的 handler 插回队首。`run()` 处理 `PausedException` 或 `RejectedException` 时调用 `self.new(...)`，把 `remain_handlers`、当前 `state`、来源信息和更新函数放进新类，同时设置 `temp=True`、`priority=0`、`block=True`、`rule=Rule()`。此时原实例已完成，不会被重新唤醒。临时类命中后，`message._run_matcher()` **先调用 `destroy()` 从注册表移除它，再创建新实例执行**，所以通常只消费一次；如果后续再次拒绝，又会创建一个新的临时类。由于删除发生在运行前，run 前处理器取消执行或 handler 抛出普通异常时，框架不会自动恢复这一步会话。

默认 `update_type()` 把续接类型设成 `"message"`。默认 `update_permission()` 构造 `Permission(User.from_event(event, perm=self.permission))`：`User` 以原事件的 `get_session_id()` 为键，后续事件必须返回同一个会话 ID，且仍满足原权限。连续追问时，如果原权限本身只是 `User` 包装，`User.from_event()` 会先去掉旧的会话限制，再绑定当前会话，避免一层层嵌套。开发者可以用 `@matcher.type_updater` 和 `@matcher.permission_updater` 修改这两个默认值。会话键**不自动包含 Bot ID**；是否区分不同 Bot、群和用户，取决于适配器的 `get_session_id()` 实现以及你额外设置的权限。

### `got` 和 `receive` 怎样接住下一条消息

`@matcher.got("name", prompt="你的名字？")` 并不是单纯给函数加一个标记。它通过 `Depends(_key_getter)` 在 handler 前执行检查，并把参数存到 `matcher.state["name"]`：

1. 首次运行时，`set_target("name")` 把目标写入 `_next_target`；若 `state` 已有 `name`，直接运行 handler，否则 `reject(prompt)` 发送提示并退出。
2. `resolve_reject()` 把 `_next_target` 复制到 `_current_target`，重新注册带有当前 handler 和 state 的临时 Matcher。
3. 下一条同会话消息命中临时类时，`_key_getter` 发现 `_current_target == "name"`，调用 `set_arg("name", event.get_message())`，然后执行 handler。`Arg("name")`、`ArgStr("name")`、`ArgPlainText("name")` 从这个 state 里注入参数。
4. 如果内容不合格，handler 调用 `reject_arg("name", prompt)`；它把目标再次设为 `name`，`run()` 再创建临时类。下一条消息会覆盖旧的 `name` 值，然后重新执行当前 handler。`got` 本身只负责采集，不会替业务验证输入。

`@matcher.receive(id)` 走相同的目标与拒绝流程，但 `set_receive()` 保存的是**整个 Event**，同时更新最近一次接收事件；处理函数可用 `Received(id)` 或 `LastReceived()` 注入。`got` 保存的是 `event.get_message()`，适合收一个参数；`receive` 适合检查下一次事件的其他字段。两者装饰器都在 handler 前追加依赖，若装饰的是已经通过 `@handle()` 注册的同一个函数，源码会替换对应的 `Dependent`，避免把同一函数重复加入 handlers。

下面的例子连续收集名字和城市，同时展示“命令里已有名字就跳过第一次追问；校验失败则重问当前字段”：

```python
from nonebot import on_command
from nonebot.adapters import Message
from nonebot.matcher import Matcher
from nonebot.params import ArgPlainText, CommandArg

register = on_command("register", block=True)

@register.handle()
async def prepare(matcher: Matcher, args: Message = CommandArg()):
    if args.extract_plain_text().strip():
        matcher.set_arg("name", args)

@register.got("name", prompt="请输入名字")
async def receive_name(name: str = ArgPlainText("name")):
    name = name.strip()
    if not name:
        await register.reject_arg("name", "名字不能为空，请重新输入")

@register.got("city", prompt="请输入城市")
async def receive_city(matcher: Matcher, city: str = ArgPlainText("city")):
    city = city.strip()
    if not city:
        await register.reject_arg("city", "城市不能为空，请重新输入")
    name = matcher.get_arg("name").extract_plain_text().strip()
    await register.finish(f"已登记：{name}，{city}")
```

当用户只发送 `/register` 时，`prepare` 运行后，第一个 `got` 因没有 `name` 而提示并拒绝；下一条同会话消息进入临时 Matcher，完成名字处理后，第二个 `got` 再提示城市并注册另一个临时 Matcher。若最初发送 `/register Alice`，`prepare` 已经调用 `set_arg()`，第一个 `got` 不会追问。每次续接都保留此前的 `name`，所以最后一个 handler 能读取它。不要把 `CommandArg()` 与 `got` 自动等同：命令参数来自 `TrieRule`，`got` 的参数来自 Matcher state，此例通过 `set_arg()` 明确衔接两者。

### 超时到底何时发生

`config.session_expire_timeout` 默认是两分钟。`Matcher.new()` 在**创建临时类时**把 `timedelta` 转成 `datetime.now() + timeout`，存为类的 `expire_time`。`message._check_matcher()` 在以后检查该类时发现当前时间超过它，调用 `destroy()` 删除并跳过。源码没有为每个会话启动定时清理任务：如果之后没有任何事件，过期类可能仍留在注册表里，直到后续事件检查它；过期后到来的回复也不会续接。若用户在有效期内又触发一次 `reject` 或 `pause`，产生的是**新的**临时类和新的过期时间。

源码用 `if expire_time` 和 `if Matcher.expire_time` 判断是否设置、检查超时，因此把超时配置为零 `timedelta` 时，会被当成假值而**不会触发过期检查**；不要把零当作“立刻超时”。

这份状态默认只存在当前 Python 进程内：进程重启会丢失；多个 Worker 各有自己的注册表，不能假设回复一定落在保存临时类的 Worker 上。同一会话若有多条事件并发到达，临时类的检查与移除也不能被当成可靠的消息队列或事务锁。需要跨进程、长时间或严格串行的会话，应自行设计持久化状态和路由策略。

## 7. 依赖注入、钩子与开发注意点

### 签名反射发生在注册时

从实现上看，你可以把依赖注入理解为**预先解析函数签名，运行时按签名动态传参**。但 NoneBot 没有在启动时把所有可能的实参都计算好。`@matcher.handle()` 调用 `append_handler()`，后者把函数交给 `Dependent.parse()`；`get_typed_signature()` 使用 `inspect.signature()` 读取形参，并解析类型注解。每个形参由允许的 `Param` 子类按顺序识别：`DependParam` 处理 `Depends(...)`，`BotParam`、`EventParam`、`StateParam`、`MatcherParam` 识别框架对象，`ArgParam` 识别 `ArgPlainText()` 等标记，最后 `DefaultParam` 处理普通默认值。若一个必需参数无法识别，通常在**注册阶段**就抛出 `Unknown parameter`，不必等事件到来才发现。

`dependencies/__init__.py` 的 `Dependent.parse_params()` 可以缩成下面几行；`get_typed_signature()` 内部调用 `inspect.signature()`，解析得到的 `field_info` 则决定运行时使用哪个 `Param._solve()`：

```python
# 源码节选并省略 ModelField 构造等细节
params = get_typed_signature(call).parameters.values()
for param in params:
    if isinstance(param.default, Param):
        field_info = param.default
    else:
        for allow_type in allow_types:
            if field_info := allow_type._check_param(param, allow_types):
                break
        else:
            raise ValueError(f"Unknown parameter {param.name}")
```

解析结果保存在 `Dependent(call, params, parameterless)` 中：`params` 是形参与其 `Param` 解析器的描述，`parameterless` 是 `got`/`receive` 这类不占用 handler 形参、但需要先执行的依赖。Matcher 类的 `handlers` 列表保存这些 `Dependent`；规则、权限和处理器钩子也以类似方式保存自己的可调用对象。也就是说，**预存的是“取值计划”，并非每种参数的现成值**。这比每次事件都重新做一次签名反射更省事。

### 运行时的参数从哪里来

`message.handle_event(bot, event)` 为每次事件创建 `state = {}`、`dependency_cache = {}` 和 `AsyncExitStack`。调度器把 `bot`、`event`、`state.copy()`、退出栈和缓存传给 Matcher；`simple_run()` 再把实例自己的 `state`、`matcher=self` 等作为关键字参数交给 handler 的 `Dependent`。`Dependent.__call__()` 先做检查，再调用各 `Param._solve()` 得到仅对应**目标函数签名**的 `values`，最后执行 `call(**values)`；同步函数则经 `run_sync()` 执行。

这两处源码连起来，恰好能看到“传入完整上下文”和“只给 handler 传它声明的参数”的分界：

```python
# internal/matcher/matcher.py，Matcher.simple_run() 中的调用
await handler(
    matcher=self,
    bot=bot,
    event=event,
    state=self.state,
    stack=stack,
    dependency_cache=dependency_cache,
)

# dependencies/__init__.py，Dependent.__call__() 的异步分支，省略异常处理
await self.check(**kwargs)
values = await self.solve(**kwargs)
return await self.call(**values)
```

`kwargs` 是框架提供的候选上下文；`solve()` 遍历注册时保存的 `ModelField`，逐个调用其 `Param._solve(**kwargs)`，生成以形参名为键的 `values`。例如 `BotParam._solve()` 返回 `bot`，`MatcherParam._solve()` 返回 `matcher`，最终用户函数看不到它没有声明的 `stack` 和 `dependency_cache`。

| 要注入的值 | 所在位置或取值方式 |
| --- | --- |
| `Bot`、`Event` | 适配器交给 `handle_event()` 的对象，沿调用链作为关键字参数传下去 |
| 当前 `Matcher` | 命中后创建的实例，由 `simple_run()` 传入 |
| `State` | 本次事件的字典；Matcher 实例另有自己的 `state`，执行前会合入事件 state |
| `CommandArg()` | `TrieRule` 先写入事件 state 中的 `_prefix.command_arg`，依赖函数运行时从中读取 |
| `ArgPlainText("name")` | `ArgParam` 从当前 Matcher 的 `state["name"]` 取出消息，再提取纯文本 |
| `EventPlainText()` | `Depends` 调用 `_event_plain_text(event)`，在需要时从 Event 计算 |
| 自定义 `Depends(func)` | 递归解析 `func` 的参数，运行时执行 `func`，把返回值交给父函数 |

`CommandArg()` 是“数据在哪里”的直接例子。`TrieRule.get_value()` 先把命令分析结果写到 `state[PREFIX_KEY]`，`params.py` 则把取值函数包装成 `Depends`；直到 handler 运行时，`_command_arg()` 才读取它：

```python
# nonebot/params.py
def _command_arg(state: T_State) -> Message:
    return state[PREFIX_KEY][CMD_ARG_KEY]

def CommandArg() -> Any:
    return Depends(_command_arg)
```

因此，这里没有一个“所有参数都预先放好的全局字典”。**本次事件的上下文**在 `handle_event()` 的局部变量与沿途传递的关键字参数中；命令结果等共享事件数据放在 `state`；`got` 收集的跨轮数据放在临时 Matcher 的默认 state；依赖函数的结果单独放进 `dependency_cache`。`BotParam._solve()` 只是返回收到的 `bot`，而 `EventPlainText()` 要到调用时才从 `event` 计算字符串，这两类来源不能混为一谈。

`Depends` 更进一步形成递归依赖图。`DependParam._solve()` 先求子依赖的参数，再调用子函数；同一次事件的缓存是“可调用对象 → `DependencyCache`”的映射，默认 `use_cache=True` 时，相同子依赖即使被多个参数或 Matcher 使用也能复用结果。缓存是**每次 `handle_event()` 新建**的，下一条事件会重新计算；生成器依赖的清理由同一个 `AsyncExitStack` 管理。`Dependent` 还会检查参数类型，必要时抛 `TypeMisMatch`，所以它不只是把一批值随意塞进函数。

`DependParam._solve()` 中的关键分支如下。`DependencyCache` 含有等待事件和结果；同一事件中两个并发调用依赖同一个函数时，后者可以等前者算完，而不必重复执行。源码先求子依赖参数，再检查当前依赖函数的缓存。下一条事件会创建新缓存。

```python
# internal/params.py，省略子依赖求值、异常处理及同步/生成器分支
if use_cache and call in dependency_cache:
    return await dependency_cache[call].wait()

target = call(**sub_values)
dependency_cache[call] = cache = DependencyCache()
result = await target
cache.set_result(result)
return result
```

插件代码最终可以写成下面这样。`Bot`、`Event`、`Matcher`、`State` 直接从当前调用上下文求值；`text` 则由 `Depends` 先调用 `command_text()`，后者再通过 `CommandArg()` 读取事件 state，形成两层依赖：

```python
from nonebot import on_command
from nonebot.adapters import Bot, Event, Message
from nonebot.matcher import Matcher
from nonebot.params import CommandArg, Depends
from nonebot.typing import T_State

async def command_text(args: Message = CommandArg()) -> str:
    return args.extract_plain_text().strip()

inspect_command = on_command("inspect")

@inspect_command.handle()
async def handle_inspect(
    bot: Bot,
    event: Event,
    matcher: Matcher,
    state: T_State,
    text: str = Depends(command_text),
):
    state["last_text"] = text
    await matcher.finish(f"{bot.self_id} 收到 {event.get_type()}：{text}")
```

规则与权限 checker 也使用这套依赖解析机制，但允许的参数种类有所不同，例如规则 checker 没有当前 Matcher 实例可注入。`Matcher.ensure_context()` 另外用 `ContextVar` 设置当前 Bot、Event、Matcher，供 `Matcher.send()`、`finish()` 这类快捷方法读取；它不是 handler 形参注入的主要存储处。脱离处理上下文直接调用这些快捷方法，也就没有同样的前提。

框架还提供 `event_preprocessor`/`event_postprocessor`、`run_preprocessor`/`run_postprocessor`，分别围住整次事件和单个 Matcher 运行过程。事件预处理器可以阻止后续调度；run 前处理器可以阻止当前响应器运行。它们适合做日志、审计或统一状态准备，但同组钩子以任务组运行，不应依赖钩子之间固定的执行顺序。一个 Matcher 的 handler 列表才是顺序执行的。

写插件时，可用下面几条原则判断代码该放在哪一层：

1. 只关心“什么事件触发、回复什么”的逻辑，放进插件 Matcher；先用 `on_message()`、`on_command()` 和规则组合表达条件。
2. 需要平台特殊事件字段或消息段时，读取具体适配器的 `Event`/`MessageSegment` 类型；不要假定所有平台消息都是纯文本。
3. 需要平台 API 时，通过具体 Bot 的 `call_api()` 或适配器定义的方法调用；抽象 `Bot.send()` 只保证基础发送接口，具体行为由协议实现。
4. 自己开发适配器时，实现协议解析、Bot/Event/Message 类型和 `_call_api()`，再利用 Driver 提供的网络接口，把事件交给 `handle_event()`；不要在 Driver 中硬编码平台协议。
5. 调试“收到了事件却没有响应”时，沿 `适配器是否上报 → event.get_type() → 优先级/过期时间 → Permission → Rule → handler` 逐层定位；命令类问题还要检查 `command_start`、首个消息段和 `TrieRule` 解析结果。

NoneBot 的主线可以概括为：**Driver 提供传输，Adapter 翻译协议，Event 提供统一视图，Matcher 类保存注册定义，Matcher 实例执行一次响应**。理解类和实例的分界，就能顺着源码解释插件加载、并发分发、状态隔离和多轮会话的多数行为。

源码入口：`nonebot/__init__.py`、`nonebot/plugin/on.py`、`nonebot/message.py`、`nonebot/internal/matcher/matcher.py`、`nonebot/internal/driver/abstract.py`、`nonebot/internal/adapter/`。公共 API 与适配器安装方式可对照 [NoneBot 官方文档](https://nonebot.dev/docs/)。
