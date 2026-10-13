---
title: "深入 Python asyncio：事件循环的工作原理"
date: 2026-10-14 00:30:00 +0800
categories: [Python, 并发编程]
tags: [Python, asyncio, 异步编程, 事件循环, 协程, IO多路复用, Task]
description: "从协程协议、Future/Task 与 CPython 源码出发，拆解 asyncio 事件循环的调度机制与 IO 多路复用，并给出一批可直接运行的案例代码。"
---

如果你写过 `await asyncio.sleep(1)`，可能想过一个问题：**单线程为什么能"同时"等待几十个网络请求？** `await` 到底在等什么——等 IO，还是等调度器？

这篇文章沿着「问题 → 心智模型 → 源码 → 动手实现 → 工程实践」的路线，把 Python `async` 事件循环的机制讲清楚。文中所有代码都在 **CPython 3.12.3 / Linux** 上实际运行过，输出为真实结果（用到 3.11+ 才有的 `TaskGroup`、`asyncio.timeout` 的地方会单独标注）。

先给结论，后文逐一展开：

1. `await` 的本质，是把"我在等一个还没完成的东西"这件事**交回给事件循环**，由后者决定下一步跑谁；
2. 事件循环内部只有三件家具：**就绪队列**（deque）、**定时器堆**（堆）、**IO 多路复用器**（selector）；
3. 单线程并发的秘密不在线程，而在 `select`/`epoll` 这类系统调用——"同时等一万个连接"是内核的能力；
4. 协程是**协作式**调度：任何一处同步阻塞代码，都会冻住整个循环。

## 0. 事件循环要解决什么问题

先回到问题的起点。程序等待一个网络响应，这段时间 CPU 是空闲的。处理"同时有很多等待"有三种经典路线：

| 路线 | 代表 | 代价 |
| --- | --- | --- |
| 一请求一线程/进程 | 传统同步服务器 | 每线程 MB 级内存；上万连接时上下文切换开销巨大 |
| 回调（Callback） | Node.js、Twisted | 性能好，但控制流被切碎，回调地狱难写难读 |
| 协程（Coroutine） | asyncio、Go goroutine、Kotlin | 写法接近同步代码，由运行时负责在等待点切换 |

asyncio 走的是第三条路：**程序员用 `await` 显式标出"这里可能等很久，可以切走"，事件循环据此在单线程内并发调度成千上万个任务。** 注意是"并发"（concurrent）而不是"并行"（parallel）：同一时刻仍然只有一个协程在 CPU 上运行。

整体结构可以先用一张图概括：

```text
                    ┌───────────────────────────────┐
   await ... ──────►│            协程                 │
                    │   async def / await 表达挂起点   │
                    └───────────────┬───────────────┘
                                    │ 挂起时 yield 出一个 Future
                    ┌───────────────▼───────────────┐
                    │             Task               │
                    │   驱动协程 / 挂完成回调 / 存结果  │
                    └───────────────┬───────────────┘
                                    │ call_soon / add_done_callback
                    ┌───────────────▼───────────────┐
                    │           事件循环              │
                    │  _ready     就绪回调队列（FIFO） │
                    │  _scheduled 定时器最小堆         │
                    │  selector   IO 多路复用          │
                    └───────────────────────────────┘
```

## 1. 心智模型：调度器 + 就绪队列

在看 asyncio 之前，先用一个 **40 行的手写调度器**建立最小的心智模型。它只用生成器模拟协程：每个任务在 `yield` 处"暂停"，把控制权还给调度器；调度器把没跑完的任务排回队尾，循环往复。

```python
from collections import deque


class Scheduler:
    """一个最简事件循环：维护就绪队列，反复推进其中的任务。"""

    def __init__(self):
        self.ready = deque()

    def call_soon(self, task):
        self.ready.append(task)

    def run(self):
        while self.ready:
            task = self.ready.popleft()      # 1. 从队首取一个任务
            try:
                task.send(None)              # 2. 推进它一步
            except StopIteration:            #    任务结束，丢弃
                continue
            else:
                self.call_soon(task)         # 3. 还没结束，排回队尾


def countdown(name, n):
    """一个协作式任务：每打印一次，就 yield 让出控制权。"""
    while n > 0:
        print(f"{name}: {n}")
        yield                                # ← 在这里暂停自己
        n -= 1
    print(f"{name}: 完成")


sched = Scheduler()
sched.call_soon(countdown("A", 3))
sched.call_soon(countdown("B", 3))
sched.run()
```

输出：

```text
A: 3
B: 3
A: 2
B: 2
A: 1
B: 1
A: 完成
B: 完成
```

这就是"协作式多任务"的全部本质：**任务轮流用 CPU，切换点由任务自己声明（`yield`/`await`），调度器只负责维护"接下来轮到谁"。** 后面会看到，asyncio 的 Task 驱动协程用的就是同一个协议，只不过推进方式从 `send(None)` 换成了对 `Future` 的精确追踪。

## 2. 协程协议：`async def` 到底是什么

建完模型，来回答一个更底层的问题：`async def` 定义的函数，调用时发生了什么？

答案是：**什么都不发生。** 调用一个 `async def` 函数不会执行函数体，只返回一个**协程对象**（coroutine object）——一个可以被"推进"的状态机。真正的执行权，掌握在驱动方（事件循环的 Task）手里。

协程的核心协议只有三个动作：

| 动作 | 含义 |
| --- | --- |
| `coro.send(value)` | 推进协程；协程在下一个 `await` 的挂起点暂停并 `yield` 出等待对象 |
| `coro.throw(exc)` | 在挂起点注入异常（取消、超时就是这么实现的） |
| `coro.close()` | 关闭协程（内部抛 `GeneratorExit`） |

`await` 表达式的语义，就是把"等待对象"通过 `yield` 交给驱动方；驱动方把事情办完后，再把结果 `send()` 回来，协程从挂起点继续。下面的实验可以完整观察到这个过程：

```python
class FakeFuture:
    """模拟一个"尚未完成的操作"，__await__ 是协程能 await 它的关键。"""

    def __await__(self):
        result = yield self         # 把自己交给驱动方，暂停
        return result               # 被唤醒时把驱动方送来的值作为结果


async def add(a, b):
    print(f"  协程开始: {a} + {b}")
    result = await FakeFuture()
    print(f"  挂起处被唤醒，收到: {result!r}")
    return a + b


# 1) 调用 async 函数不会执行函数体
coro = add(1, 2)
print("调用 add(1, 2) 得到:", coro)
print("函数体没有执行（上面没有 '协程开始'）")
coro.close()

# 2) 用 send() 手动驱动：挂起 → 唤醒 → 跑完
coro = add(40, 2)
suspended = coro.send(None)
print("协程挂起，yield 出:", suspended)
try:
    coro.send("IO 完成的结果")
except StopIteration as e:
    print("协程跑完，返回值 =", e.value)
```

运行结果：

```text
调用 add(1, 2) 得到: <coroutine object add at 0x...>
函数体没有执行（上面没有 '协程开始'）
  协程开始: 40 + 2
协程挂起，yield 出: <__main__.FakeFuture object at 0x...>
  挂起处被唤醒，收到: 'IO 完成的结果'
协程跑完，返回值 = 42
```

三个值得记住的细节：

1. **协程的返回值放在 `StopIteration.value` 里**——这是生成器/协程协议的一部分，Task 就是从这里拿到协程的最终结果的；
2. 能被 `await` 的对象必须实现 `__await__`（或 `__iter__`），`Future`、`Task` 都实现了它；
3. 协议由驱动方定义。真实 asyncio 的 Task 只发送 `None`——因为它约定"结果存在 Future 里，不靠 send 传值"。

### 2.1 `await` 在字节码层面就是 `yield from`

口说无凭，用 `dis` 反汇编看看。下面是两个函数：一个用 `await` 等待协程，一个用 `yield from` 等待生成器：

```python
import dis

async def inner():
    return 42

async def with_await():
    x = await inner()          # 等待一个协程
    return x + 1

def inner_gen():
    yield
    return 42

def with_yield_from():
    x = yield from inner_gen()  # 等待一个生成器
    return x + 1

dis.dis(with_await)
dis.dis(with_yield_from)
```

关键指令对比（Python 3.12，节选）：

```text
==== with_await ====
 24 GET_AWAITABLE            0
 26 LOAD_CONST               0 (None)
 28 SEND                     3 (to 38)      ← 把子协程推进，等待它的产出
 32 YIELD_VALUE              2             ← 挂起自己，交还控制权
 36 JUMP_BACKWARD_NO_INTERRUPT     5 (to 28)
 38 END_SEND

==== with_yield_from ====
 24 GET_YIELD_FROM_ITER
 26 LOAD_CONST               0 (None)
 28 SEND                     3 (to 38)      ← 完全相同的循环
 32 YIELD_VALUE              2
 36 JUMP_BACKWARD_NO_INTERRUPT     5 (to 28)
 38 END_SEND
```

两者只有"获取等待对象"的一步不同（`GET_AWAITABLE` vs `GET_YIELD_FROM_ITER`），驱动循环完全一致。**Python 3.5 起，`async/await` 就是把 `yield from` 协议包装成类型安全的语法糖。**

## 3. asyncio 的四大件

有了协程协议的基础，asyncio 的组件就不神秘了。整个库本质上只有四个概念：

| 组件 | 角色 | 一句话理解 |
| --- | --- | --- |
| Coroutine | 可挂起的函数 | "可以被暂停执行的任务体" |
| Future | 结果占位符 | "一个现在还没有、迟早会有的值" |
| Task | 协程的驱动器 | "把协程跑起来，并在被唤醒时推它一步" |
| EventLoop | 调度器 | "谁可以跑、什么时候跑、等什么" |

四者的关系：**循环调度 Task，Task 驱动协程，协程等待 Future，Future 完成时把 Task 排回循环。**

### 3.1 Future：结果占位符

`Future` 表示一个尚未完成的操作，有三个状态：`PENDING`、`FINISHED`（有结果或有异常）、`CANCELLED`。它最关键的代码是 `__await__`（下面是 `futures.py` 中的纯 Python 版本，C 加速版行为一致）：

```python
def __await__(self):
    if not self.done():
        self._asyncio_future_blocking = True
        yield self          # 告诉 Task：我在等这个 Future
    if not self.done():
        raise RuntimeError("await wasn't used with future")
    return self.result()    # 被唤醒后取出结果（或抛出异常）
```

当协程 `await future` 时，协程体把自己挂起，并把 **future 对象本身** yield 给 Task。Task 看到 `_asyncio_future_blocking` 标记，就给 future 挂上"完成后叫醒我"的回调，然后撒手不管——直到有人调用 `future.set_result()`。

另外，Future 支持挂多个完成回调（`add_done_callback`），这构成了 asyncio 世界里一切"等待链"的基础。

### 3.2 Task：协程的驱动器

`Task` 是 `Future` 的子类：对等待它的代码来说，它就是一个 future（await 一个 Task 就是等它跑完）；对内部协程来说，它是驱动者。核心逻辑在 `Task.__step` 里（节选，注释为本文所加）：

```python
def __step_run_and_handle_result(self, exc):
    coro = self._coro
    try:
        if exc is None:
            result = coro.send(None)      # ← 推进协程：一直运行到下一个挂起点
        else:
            result = coro.throw(exc)      # ← 注入异常（取消/超时走这条路）
    except StopIteration as exc:
        super().set_result(exc.value)     # 协程结束 → Task 完成，广播给等待者
    except CancelledError as exc:
        super().cancel()                  # 协程被取消 → Task 变 cancelled
    except BaseException as exc:
        super().set_exception(exc)        # 协程抛异常 → 异常暂存在 Task 里
    else:
        blocking = getattr(result, '_asyncio_future_blocking', None)
        if blocking is not None and blocking:
            result._asyncio_future_blocking = False
            result.add_done_callback(self.__wakeup)   # 协程在等它，等它完成叫我
            self._fut_waiter = result
        elif result is None:
            # 裸 yield：本轮先不继续，下一轮再排进就绪队列
            self._loop.call_soon(self.__step)
```

这段 20 行代码解释了 `await` 的全部行为模式，可以归结为四种情况：

1. **协程跑完了**（`StopIteration`）→ 把返回值设为 Task 的结果，唤醒所有等它的人；
2. **协程在等一个 Future**（yield 出带 `_asyncio_future_blocking` 标记的对象）→ 挂完成回调，Task 转入等待；
3. **协程裸 yield 了 `None`**（比如 `await asyncio.sleep(0)`）→ 把 `self.__step` 排进就绪队列，等于"让出一次，下轮继续"；
4. **协程抛异常 / 被取消** → 状态存进 Task，等 `await` 它的代码把异常取走。

注意第 3 种情况：**这是一种"自愿让路"**。协程并不总是等 IO，`await asyncio.sleep(0)` 的唯一作用就是"主动重新排队"，给其他就绪任务一个运行机会。

### 3.3 EventLoop：三件家具，一个循环

事件循环是唯一真正"运行"的东西。它的主循环 `run_forever()` 反复调用 `_run_once()`，后者是理解一切的关键。下面是 CPython 3.12 `base_events.py` 的精简节选（注释为本文所加）：

```python
def _run_once(self):
    # ... 前置：清理定时器堆里被取消的句柄（略）

    # 1. 计算 selector 的阻塞时长
    timeout = None
    if self._ready or self._stopping:
        timeout = 0                # 还有就绪回调，不阻塞，先干活
    elif self._scheduled:
        when = self._scheduled[0]._when       # 堆顶 = 最早到期的定时器
        timeout = min(max(0, when - self.time()), MAXIMUM_SELECT_TIMEOUT)

    # 2. 阻塞等待 IO 事件（或超时、或被跨线程唤醒）
    event_list = self._selector.select(timeout)
    self._process_events(event_list)          # IO 就绪的回调 → 排进 _ready

    # 3. 到点的定时器 → 从堆里取出，转入就绪队列
    end_time = self.time() + self._clock_resolution
    while self._scheduled:
        handle = self._scheduled[0]
        if handle._when >= end_time:
            break
        handle = heapq.heappop(self._scheduled)
        handle._scheduled = False
        self._ready.append(handle)

    # 4. 执行本轮快照内的所有就绪回调
    #    （回调中新排入的下一轮再执行，保证单轮不饿死其他人）
    ntodo = len(self._ready)
    for i in range(ntodo):
        handle = self._ready.popleft()
        if handle._cancelled:
            continue
        handle._run()                          # 唯一的"真正调用回调"的地方
```

整个循环的运转逻辑：

- **`_ready`**：一个 `collections.deque`（FIFO），装的是"现在就可以跑"的回调句柄。`loop.call_soon(cb)` 就是往它尾部追加；
- **`_scheduled`**：一个最小堆，装的是定时器句柄（`TimerHandle`），按到期时间排序。`loop.call_later(delay, cb)` 就是往堆里放东西；
- **`_selector`**：IO 多路复用器（Linux 上是 `epoll`，macOS 上是 `kqueue`，Windows 上的 Proactor 用 IOCP）。循环没有活干时，就在这里"睡到第一个定时器到期"或"有 IO 事件到来"。

一个容易忽略的细节是第 4 步的**快照语义**：只执行本轮开始时就绪的回调，回调自身再排入的回调留到下一轮。这保证了回调链不会无限吞占一个 tick，也保证其他任务总有机会。

#### 跨线程与信号：循环是怎么被"叫醒"的

单线程调试 asyncio 很轻松，但它免不了和外界打交道：别的线程要给循环投递任务怎么办？

- `loop.call_soon_threadsafe(cb)`：线程安全版的 `call_soon`。内部通过 **self-pipe 技巧**（Unix 上结合 `signal.set_wakeup_fd`）向循环自己写一个字节，让阻塞在 `select()` 里的循环立刻醒来处理新回调；
- `asyncio.run_coroutine_threadsafe(coro, loop)`：反向场景——非循环线程提交协程到循环上运行并等待结果，返回一个 `concurrent.futures.Future`；
- `loop.add_signal_handler(sig, cb)`：Unix 上把信号处理接进循环（需要主线程），把异步信号变成普通回调。

## 4. 完整旅程：`await asyncio.sleep(0.1)` 都经历了什么

现在把前面所有零件串起来。`asyncio.sleep` 在 3.12 的实现（节选）：

```python
async def sleep(delay, result=None):
    if delay <= 0:
        await __sleep0()            # 裸 yield：让出一次，下轮继续
        return result
    loop = events.get_running_loop()
    future = loop.create_future()
    h = loop.call_later(delay, futures._set_result_unless_cancelled,
                        future, result)
    try:
        return await future
    finally:
        h.cancel()                  # 被取消/异常时，顺手摘掉定时器
```

一次 `await asyncio.sleep(0.1)` 的完整旅程：

1. 协程执行到 `await asyncio.sleep(0.1)`：`sleep()` 是个协程函数，`await` 通过 `SEND` 把它推进第一步；
2. `sleep` 内部创建 `future`，并用 `loop.call_later(0.1, ...)` 注册一个定时器——`TimerHandle` 进入 `_scheduled` 堆；
3. 执行到 `await future`：`Future.__await__` 把 future **yield 出来**，协程挂起；
4. 控制权回到 `Task.__step`：检测到"协程在等一个 Future"，于是 `future.add_done_callback(self.__wakeup)`，Task 记为挂起（`_fut_waiter = future`），本轮结束；
5. 控制权回到 `_run_once` 的主循环。本轮 ready 队列跑完，准备进入"睡眠"；
6. `_run_once` 计算 `timeout`：约 0.1 秒（定时器堆顶的剩余时间），调用 `self._selector.select(0.1)` 阻塞。**这 0.1 秒期间，CPU 完全可以让出去跑别的任务；如果没有任务，就真的在 `select` 里睡着，不空转；**
7. 时间到，`select` 返回（也可能中途被 IO 事件或跨线程唤醒提前叫醒）；循环从 `_scheduled` 堆里弹出到期的 `TimerHandle`，放进 `_ready`；
8. 下一轮执行这个句柄，回调 `_set_result_unless_cancelled(future, None)` 调用 `future.set_result(None)`；
9. `set_result` 遍历该 future 的回调列表，把 `Task.__wakeup` 用 `call_soon` 排进 `_ready`；
10. 循环执行 `Task.__wakeup` → `Task.__step()` → `coro.send(None)`：协程从 `await future` 处恢复（拿到 `None` 作为结果），`sleep` 协程返回，外层协程继续向下执行。

一个只有 10 步、却支撑起所有上层用法的机制。**"单线程并发"的秘密在第 6 步：不是轮流抢 CPU，而是集体告诉内核"我在等哪些事件"，然后一次 `select` 等所有。**

## 5. 调度顺序实验：谁先跑？

理论讲完，动手验证一下调度顺序。下面这个实验观察 `call_soon`（就绪队列）和 `call_later`（定时器堆）的区别，顺便偷看一眼循环的内部状态（教学用途，生产代码不要依赖私有属性）：

```python
import asyncio


async def main():
    loop = asyncio.get_running_loop()

    # call_soon：把回调排进"就绪队列"，下一次 tick 按 FIFO 执行
    loop.call_soon(print, "call_soon 1")
    loop.call_soon(print, "call_soon 2")

    # call_later：定时器先进"定时器堆"，到点后再转入就绪队列
    loop.call_later(0.01, print, "call_later 10ms")

    print("main 的函数体（同步执行，不经过队列）")

    print("ready 队列类型:", type(loop._ready).__name__,
          "长度:", len(loop._ready))
    print("定时器堆:", [f"{h._when - loop.time():.3f}s"
                      for h in loop._scheduled])

    await asyncio.sleep(0.05)
    print("main 结束")


asyncio.run(main())
print("asyncio.run 返回，循环关闭")
```

运行结果：

```text
main 的函数体（同步执行，不经过队列）
ready 队列类型: deque 长度: 2
定时器堆: ['0.010s']
call_soon 1
call_soon 2
call_later 10ms
main 结束
asyncio.run 返回，循环关闭
```

值得注意的点：

- `main` 的函数体是**同步执行**的——协程一被调度就一路跑到第一个真正的挂起点（这里第一个挂起点是 `await asyncio.sleep(0.05)`），中间不切换；
- `call_soon` 的回调按 FIFO 顺序在下一轮执行，先于 10ms 后的定时器；
- `await asyncio.sleep(0.05)` 给了定时器 10ms 的回调执行窗口，所以 `call_later 10ms` 排在 `main 结束` 之前。

## 6. 用不到 90 行实现迷你 asyncio

理解一个系统最好的方式是重写它。下面这个迷你实现包含了 asyncio 的每个核心概念：`Future`、`Task`、事件循环、`sleep`、`create_task`。它没有 IO（用不到 selector，定时器堆扮演了"唯一的 IO"），但驱动协议与真实 asyncio 完全一致：

```python
import heapq
import time
from collections import deque


class Future:
    """结果占位符：表示"未来会有一个值"。"""

    def __init__(self, loop):
        self.loop = loop
        self.done = False
        self.result = None
        self._callbacks = []

    def set_result(self, value):
        assert not self.done
        self.done = True
        self.result = value
        for cb in self._callbacks:
            self.loop.call_soon(cb)          # 回调统一排进就绪队列

    def add_done_callback(self, cb):
        self._callbacks.append(cb)

    def __await__(self):
        if not self.done:
            yield self                        # 把自己"交"给 Task
        return self.result

    __iter__ = __await__


class Task(Future):
    """把协程包装成 Future，并负责在它挂起/完成之间"推进"它。"""

    def __init__(self, loop, coro):
        super().__init__(loop)
        self.coro = coro
        self.step()                           # 立刻推进第一步

    def step(self):
        try:
            fut = self.coro.send(None)        # 推进协程到下一个挂起点
        except StopIteration as e:
            if not self.done:
                self.set_result(e.value)      # 协程结束 → 拿到返回值
        else:
            fut.add_done_callback(self.step)  # 协程在等 fut：它完成时回来推进


class Loop:
    """就绪队列 + 定时器堆，撑起整个调度。"""

    def __init__(self):
        self._ready = deque()
        self._scheduled = []

    def call_soon(self, cb):
        self._ready.append(cb)

    def call_later(self, delay, cb):
        heapq.heappush(self._scheduled, (time.monotonic() + delay, cb))

    def create_task(self, coro):
        return Task(self, coro)

    def run_forever(self):
        while self._ready or self._scheduled:
            # 1) 到期的定时器 → 转入就绪队列
            now = time.monotonic()
            while self._scheduled and self._scheduled[0][0] <= now:
                _, cb = heapq.heappop(self._scheduled)
                self._ready.append(cb)
            # 2) 就绪回调跑一轮（长度快照，新回调下一轮再跑）
            for _ in range(len(self._ready)):
                self._ready.popleft()()
            # 3) 没活干就睡到下一个定时器 —— 对应真实循环的 selector.select
            if self._scheduled and not self._ready:
                time.sleep(max(0, self._scheduled[0][0] - time.monotonic()))


_loop = None  # 供 sleep() 找到当前循环（真实 asyncio 用线程局部变量）


def sleep(delay):
    fut = Future(_loop)
    _loop.call_later(delay, lambda: fut.set_result(None))
    return fut


async def worker(name, n, delay):
    t0 = time.perf_counter()
    for i in range(n):
        print(f"  {name}: 第 {i + 1} 次，t={time.perf_counter() - t0:.2f}s")
        await sleep(delay)
    print(f"  {name}: 完成")
    return name


async def main():
    print("main: 启动两个任务，分别每 0.3s / 0.5s 醒一次")
    t1 = _loop.create_task(worker("A", 3, 0.3))
    t2 = _loop.create_task(worker("B", 2, 0.5))
    r1 = await t1                     # await 一个 Task，就是等它的结果
    r2 = await t2
    print(f"main: 拿到结果 {r1!r}, {r2!r}")


_loop = Loop()
_loop.create_task(main())
_loop.run_forever()
print("事件循环退出")
```

运行结果（一次真实的交错执行）：

```text
main: 启动两个任务，分别每 0.3s / 0.5s 醒一次
  A: 第 1 次，t=0.00s
  B: 第 1 次，t=0.00s
  A: 第 2 次，t=0.30s
  B: 第 2 次，t=0.50s
  A: 第 3 次，t=0.60s
  A: 完成
  B: 完成
main: 拿到结果 'A', 'B'
事件循环退出
```

对照真实 asyncio 看这个迷你版，会发现概念一一对应：

| 迷你版 | 真实 asyncio | 差异 |
| --- | --- | --- |
| `loop._ready` + `_scheduled` | 同名结构 | 真实版还有 selector 处理 IO |
| `Task.step` | `Task.__step` | 真实版处理取消、异常、context 等边界 |
| `loop.run_forever` 里的 `time.sleep` | `selector.select(timeout)` | 真实版睡在 IO 多路复用上，能被 IO 事件唤醒 |
| `sleep()` 里的全局 `_loop` | `get_running_loop()` | 真实版用线程局部变量，支持多线程多循环 |

## 7. IO 多路复用：事件循环的"睡觉机制"

上面所有例子里，循环等的都是定时器。真实世界的等待大头是 IO——而"同时等一万个连接"的能力，来自操作系统提供的 IO 多路复用接口：

| 平台 | 接口 | 特点 |
| --- | --- | --- |
| Linux | `epoll` | O(就绪事件数)，不受连接数影响 |
| macOS/BSD | `kqueue` | 类似 epoll |
| Windows | IOCP（Proactor） | 完成端口模型，asyncio 3.8+ 默认 |

Python 用 `selectors` 模块统一了 epoll / kqueue / select 这些接口（Windows 的 Proactor 走单独的 IOCP 实现）。**Unix 上事件循环的 IO 部分，就是一个放在 `selectors.DefaultSelector` 上的阻塞循环。** 为了看得更清楚，我们不用 asyncio，只用 `selectors` 裸写一个单线程 echo 服务器——它同时服务 5 个并发客户端（完整示例还包含一个多线程客户端脚本，用于同时发起连接）：

```python
import selectors, socket

sel = selectors.DefaultSelector()
server = socket.socket()
server.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
server.bind(("127.0.0.1", 9107))
server.listen()
server.setblocking(False)
sel.register(server, selectors.EVENT_READ, data=None)

while True:
    # 阻塞等待：只要有任一 fd 可读/可写就返回 —— 单线程并发的核心
    for key, _ in sel.select(timeout=0.1):
        if key.data is None:                    # 新连接到达
            conn, addr = key.fileobj.accept()
            conn.setblocking(False)
            sel.register(conn, selectors.EVENT_READ, data=b"")
        else:                                   # 已有连接上收到数据
            conn = key.fileobj
            data = conn.recv(1024)
            if data:
                conn.sendall(b"echo: " + data)
            else:
                sel.unregister(conn)
                conn.close()
```

（完整版对非阻塞 `recv` 增加了 `BlockingIOError` 兜底，并对 `sendall` 做了部分写处理，这里省略。）

用 asyncio 的 Streams API 写同样的服务器，IO 多路复用被隐藏在 StreamReader/StreamWriter 背后：

```python
import asyncio

async def handle(reader, writer):
    while True:
        data = await reader.read(1024)      # 挂起，交给循环的 selector
        if not data:
            break
        writer.write(b"echo: " + data)
        await writer.drain()
    writer.close()
    await writer.wait_closed()

async def main():
    server = await asyncio.start_server(handle, "127.0.0.1", 9108)
    async with server:
        await server.serve_forever()
```

两份实现跑起来的输出（节选）：

```text
[selector] 监听 127.0.0.1:9107
--- 5 个客户端并发连 selectors 服务器 ---
[selector] 接受连接 ('127.0.0.1', 52662)
[selector] 接受连接 ('127.0.0.1', 52666)
...
  客户端 port=9107 收到: 'echo: hello-2'
...
--- 5 个客户端并发连 asyncio 服务器 ---
[asyncio] 监听 127.0.0.1:9108
[asyncio] 接受连接 ('127.0.0.1', 36606)
...
两种实现都完成了同样的工作。
```

两者在等待模型上是同一个东西，区别只在抽象层级：asyncio 把"注册 fd → 事件回调 → 恢复协程"的繁琐细节封装成了 `await reader.read()`。**记住这个对照，就再也不会把 asyncio 想象成某种魔法线程。**

## 8. 顺序还是并发：`await` 与 `gather` 的区别

初学者最常见的性能事故是"写了异步代码，但效果没变快"。原因通常只有一个：**在循环里逐个 `await` 会串行执行。** 实验对比一下：

```python
import asyncio
import time


async def io_task(name, seconds):
    print(f"  {name} 开始（需要 {seconds}s）")
    await asyncio.sleep(seconds)
    print(f"  {name} 结束")
    return seconds


async def sequential():
    t0 = time.perf_counter()
    await io_task("请求1", 0.5)     # await 会一路等它做完，才轮到下一个
    await io_task("请求2", 0.5)
    await io_task("请求3", 0.5)
    return time.perf_counter() - t0


async def concurrent():
    t0 = time.perf_counter()
    # gather 把三个协程包装成 Task 立刻排进循环，然后一起等
    await asyncio.gather(
        io_task("请求1", 0.5),
        io_task("请求2", 0.5),
        io_task("请求3", 0.5),
    )
    return time.perf_counter() - t0


async def main():
    print("顺序 await：")
    t_seq = await sequential()
    print("并发 gather：")
    t_con = await concurrent()
    print(f"\n顺序耗时 {t_seq:.2f}s，并发耗时 {t_con:.2f}s")
    print("理论：顺序 ≈ 3×0.5=1.5s；并发 ≈ max(0.5)=0.5s")


asyncio.run(main())
```

运行结果：

```text
顺序 await：
  请求1 开始（需要 0.5s）
  请求1 结束
  请求2 开始（需要 0.5s）
  请求2 结束
  请求3 开始（需要 0.5s）
  请求3 结束
并发 gather：
  请求1 开始（需要 0.5s）
  请求2 开始（需要 0.5s）
  请求3 开始（需要 0.5s）
  请求1 结束
  请求2 结束
  请求3 结束

顺序耗时 1.50s，并发耗时 0.50s
理论：顺序 ≈ 3×0.5=1.5s；并发 ≈ max(0.5)=0.5s
```

关键区别：**`await coro` 只是"等它做完"；`gather`/`create_task` 才是"把它注册进循环，和别人一起跑"。** 协程对象在被调度前只是"待办事项"，`create_task` 才是"交给循环"的动作：

```python
t = asyncio.create_task(io_task("请求1", 0.5))   # 立刻排进循环
t2 = asyncio.create_task(io_task("请求2", 0.5))
await t
await t2            # 两个请求已经在并行等待了
```

## 9. Task 生命周期：状态、取消与超时

任务一旦交给循环，就有自己的生命周期。下面的实验覆盖日常最常用的四个操作——查看状态、取消、超时、取异常：

```python
import asyncio


async def main():
    # 状态流转
    t = asyncio.create_task(asyncio.sleep(0.1, result="ok"))
    print("刚创建 done:", t.done(), "| cancelled:", t.cancelled())
    val = await t
    print("完成后 结果:", val, "| done:", t.done())

    # 取消：向协程注入 CancelledError
    async def slow():
        try:
            await asyncio.sleep(10)
        except asyncio.CancelledError:
            print("slow 做清理后重新抛出")
            raise
    t2 = asyncio.create_task(slow())
    await asyncio.sleep(0)
    t2.cancel()
    try:
        await t2
    except asyncio.CancelledError:
        print("调用方捕获 CancelledError | cancelled:", t2.cancelled())

    # 超时（3.11+）：等价于"到点就取消内部任务"
    try:
        async with asyncio.timeout(0.05):
            await asyncio.sleep(100)
    except TimeoutError:
        print("timeout 到点 → TimeoutError")

    # 异常暂存在 Task 里，await 或 result() 时才抛出
    async def boom():
        raise ValueError("任务内部爆炸")
    t3 = asyncio.create_task(boom())
    await asyncio.sleep(0)
    print("异常暂存在 Task 里:", repr(t3.exception()))
    try:
        t3.result()
    except ValueError as e:
        print("result() 重新抛出:", e)


asyncio.run(main())
```

运行结果：

```text
刚创建 done: False | cancelled: False
完成后 结果: ok | done: True
slow 做清理后重新抛出
调用方捕获 CancelledError | cancelled: True
timeout 到点 → TimeoutError
异常暂存在 Task 里: ValueError('任务内部爆炸')
result() 重新抛出: 任务内部爆炸
```

几个必须知道的语义：

- **取消不是杀死，而是"在挂起点注入 `CancelledError`"**。协程有机会在 `except` 里做清理，但清理完必须重新 `raise`，否则取消会被吞掉；
- `CancelledError` 在 Python 3.8+ 继承自 `BaseException`——这是刻意设计，防止被宽泛的 `except Exception:` 误吞；
- 任务里的异常不会主动炸掉程序，而是**暂存在 Task 里**，直到有人 `await` 它或调用 `result()`。没人取走？垃圾回收时你会看到 "Task exception was never retrieved" 的提示；
- `asyncio.timeout`（3.11+）是上下文管理器，内部对所在任务设了定时取消，超时后表现为 `TimeoutError`。之前的写法是 `asyncio.wait_for(awaitable, timeout)`。

## 10. 阻塞循环的代价

协作式调度有一个铁律：**循环只在协程 `await` 让出时才能切换任务。** 如果任何一段代码在协程里同步阻塞（`time.sleep`、密集计算、同步数据库调用），整个循环——以及循环上所有任务——都会被冻住。实验为证：

```python
import asyncio
import time

T0 = time.perf_counter()


def log(msg):
    print(f"[t={time.perf_counter() - T0:5.2f}s] {msg}")


async def ticker(tag):
    for i in range(3):
        await asyncio.sleep(1)
        log(f"{tag} 心跳 {i + 1}（预期 t≈{i + 1}.0s）")


async def sync_blocker():
    await asyncio.sleep(0.1)
    log("开始 time.sleep(2)，整个循环被冻住")
    time.sleep(2)                       # ← 没有 await，循环在这里卡死
    log("time.sleep(2) 结束，循环恢复")


async def good_citizen():
    await asyncio.sleep(0.1)
    log("开始 await asyncio.sleep(2)，循环照常运行")
    await asyncio.sleep(2)              # 挂起自己，把控制权还给循环
    log("等待结束，循环从头到尾没卡过")


async def main():
    log("=== 实验 1：心跳 + 同步阻塞 ===")
    await asyncio.gather(ticker("同步版"), sync_blocker())

    log("=== 实验 2：心跳 + 正确 await ===")
    await asyncio.gather(ticker("异步版"), good_citizen())


asyncio.run(main())
```

运行结果：

```text
[t= 0.00s] === 实验 1：心跳 + 同步阻塞 ===
[t= 0.10s] 开始 time.sleep(2)，整个循环被冻住
[t= 2.10s] time.sleep(2) 结束，循环恢复
[t= 2.10s] 同步版 心跳 1（预期 t≈1.0s）
[t= 3.10s] 同步版 心跳 2（预期 t≈2.0s）
[t= 4.10s] 同步版 心跳 3（预期 t≈3.0s）
[t= 4.10s] === 实验 2：心跳 + 正确 await ===
[t= 4.20s] 开始 await asyncio.sleep(2)，循环照常运行
[t= 5.10s] 异步版 心跳 1（预期 t≈1.0s）
[t= 6.11s] 异步版 心跳 2（预期 t≈2.0s）
[t= 6.20s] 等待结束，循环从头到尾没卡过
[t= 7.11s] 异步版 心跳 3（预期 t≈3.0s）
```

实验 1 里，`time.sleep(2)` 把整个循环冻住，心跳第一次打卡被推迟到 2.1 秒；实验 2 里，同样的"睡 2 秒"用 `await asyncio.sleep(2)` 表达，心跳严格按 +1.0s / +2.0s / +3.0s 准点打卡。

这条铁律在实践中对应着最常见的事故：**在异步服务里调用 `requests.get()`。** 一个请求卡 500ms，就等于服务端所有并发请求一起卡 500ms。

## 11. 桥接同步世界：`to_thread` 与 `run_in_executor`

现实世界不可能全是异步库。老 SDK、`requests`、同步数据库驱动、CPU 密集计算，都需要出路。标准方案是**把阻塞调用扔进线程池/进程池，用 `run_in_executor` 或 `asyncio.to_thread`（3.9+）包装成 awaitable**：

```python
import asyncio
import hashlib
import time
from concurrent.futures import ProcessPoolExecutor


def blocking_io(name, seconds):
    """模拟没有 async 版本的阻塞调用（如 requests.get）。"""
    time.sleep(seconds)
    return f"{name} 完成"


def cpu_heavy(seed):
    """纯 CPU 计算：反复哈希。"""
    h = f"seed-{seed}".encode()
    for _ in range(400_000):
        h = hashlib.sha256(h).digest()
    return h.hex()[:8]


async def main():
    # 三个阻塞调用并发跑在线程池，总耗时 ≈ 0.5s 而非 1.5s
    t0 = time.perf_counter()
    results = await asyncio.gather(
        asyncio.to_thread(blocking_io, "A", 0.5),
        asyncio.to_thread(blocking_io, "B", 0.5),
        asyncio.to_thread(blocking_io, "C", 0.5),
    )
    print(results, f"→ 线程池并发耗时 {time.perf_counter() - t0:.2f}s")

    # 纯 CPU：串行 vs 多进程
    t0 = time.perf_counter()
    serial = [cpu_heavy(s) for s in range(4)]
    t_serial = time.perf_counter() - t0

    t0 = time.perf_counter()
    with ProcessPoolExecutor() as pool:
        loop = asyncio.get_running_loop()
        futs = [loop.run_in_executor(pool, cpu_heavy, s) for s in range(4)]
        parallel = await asyncio.gather(*futs)
    t_parallel = time.perf_counter() - t0

    print("结果一致:", serial == parallel)
    print(f"CPU 串行 {t_serial:.2f}s vs 多进程 {t_parallel:.2f}s")


asyncio.run(main())
```

运行结果（本机 22 逻辑核）：

```text
['A 完成', 'B 完成', 'C 完成'] → 线程池并发耗时 0.50s
结果一致: True
CPU 串行 0.40s vs 多进程 0.15s
```

选型要点：

| 阻塞类型 | 方案 | 原因 |
| --- | --- | --- |
| 网络/文件 IO（无 async 版） | `asyncio.to_thread` | 等待期间释放 GIL，线程池接近满效率 |
| 纯 Python CPU 计算 | `ProcessPoolExecutor` | GIL 下多线程无法并行计算，必须多进程 |
| 已有 async 版本 | 直接 `await` | 不占用线程池，扩展性最好 |

线程池默认大小是 `min(32, os.cpu_count() + 4)`，需要调大可以 `loop.set_default_executor(ThreadPoolExecutor(max_workers=...))`。但注意：**`to_thread` 是逃生舱，不是默认姿势**——如果线程池被占满，新的 `to_thread` 调用也要排队。

## 12. 工程模式：限流、背压与结构化并发

真实服务里直接 `gather` 一万个请求会把对端打挂，也会把自己的内存打爆。asyncio 提供了对应的原语：`Semaphore` 控制并发上限，`Queue` 做背压，`TaskGroup`（3.11+）做结构化并发——任何子任务失败时，其余子任务自动取消，不会留下孤儿任务：

```python
import asyncio
import random
import time


async def fetch(url, sem):
    """模拟网络请求：用 sem 控制全局并发上限。"""
    async with sem:                       # 拿到令牌才允许"发请求"
        cost = random.uniform(0.05, 0.2)
        await asyncio.sleep(cost)
        return f"{url} (耗时{cost:.2f}s)"


async def producer(urls, sem, queue):
    results = await asyncio.gather(*(fetch(u, sem) for u in urls))
    for r in results:
        await queue.put(r)                # 队列有界时在这里反压生产端
    await queue.put(None)                 # 结束哨兵


async def consumer(queue, out):
    while (item := await queue.get()) is not None:
        out.append(item)
        await asyncio.sleep(0)            # 模拟少量处理开销
        queue.task_done()


async def main():
    urls = [f"https://example.com/page/{i}" for i in range(20)]
    sem = asyncio.Semaphore(5)            # 最多 5 个并发请求
    queue = asyncio.Queue(maxsize=8)      # 有界队列 → 背压
    out = []

    t0 = time.perf_counter()
    async with asyncio.TaskGroup() as tg:  # 任一子任务失败会取消其余
        tg.create_task(producer(urls, sem, queue))
        tg.create_task(consumer(queue, out))
    elapsed = time.perf_counter() - t0

    print(f"完成 {len(out)} 个请求，总耗时 {elapsed:.2f}s")


asyncio.run(main())
```

运行结果（随机数导致每次略有不同）：

```text
完成 20 个请求，总耗时 0.57s
```

对比：如果顺序执行 20 个平均 0.125s 的请求，大约需要 2.5 秒；限流到 5 并发后，理论下限约 `20/5 × 0.125 ≈ 0.5s`——实测 0.57s，符合预期。这里的三层保护值得记住：**Semaphore 限制对外压力，有界 Queue 控制内存，TaskGroup 保证异常时不留孤儿。**

另外，`async with`（异步上下文管理器）和 `async for`（异步迭代器）是这层的语法原料——数据库连接池、session、事务都是 `async with` 的典型应用：

```python
async with AsyncTimer("任务A"):     # __aenter__ / __aexit__ 都可以 await
    await asyncio.sleep(0.1)

async for page in page_reader(4):   # 每次迭代可以挂起（翻页、拉流）
    process(page)
```

## 13. 任务隔离：每个 Task 一份 ContextVar

多任务并发时，"这次请求的上下文"（trace id、用户身份）不能被别的任务串台。Python 用 `contextvars` 解决：**创建 Task 时会复制当前上下文，任务内对 ContextVar 的修改只影响自己的副本**——这也是 Web 框架在异步中间件里传递请求上下文的基础：

```python
import asyncio
import contextvars

request_id = contextvars.ContextVar("request_id", default="<无>")


async def handler(name):
    request_id.set(name)
    await asyncio.sleep(0.01)
    print(f"  {name} 任务内看到: {request_id.get()}")


async def main():
    request_id.set("main")
    tasks = [asyncio.create_task(handler(f"request-{i}")) for i in range(3)]
    await asyncio.gather(*tasks)
    print(f"  main 任务内仍是: {request_id.get()}（未被子任务污染）")


asyncio.run(main())
```

运行结果：

```text
  request-0 任务内看到: request-0
  request-1 任务内看到: request-1
  request-2 任务内看到: request-2
  main 任务内仍是: main（未被子任务污染）
```

## 14. 常见坑与调试技巧

最后集中说几个实际开发中代价最高的坑，以及 asyncio 自带的排查工具。

**坑 1：忘记 `await`。** 协程对象创建后没人驱动，函数体永远不会执行。Python 会发出警告：

```text
RuntimeWarning: coroutine 'foo' was never awaited
RuntimeWarning: Enable tracemalloc to get the object allocation traceback
```

**坑 2：创建 Task 后不保存引用。** `asyncio.create_task()` 返回的 Task 如果没有任何强引用，可能被垃圾回收，任务"凭空消失"：

```text
Task was destroyed but it is pending!
task: <Task pending name='Task-2' coro=<background() ...> wait_for=<Future pending ...>>
```

官方建议：把 fire-and-forget 的 Task 存进一个集合，任务结束时从集合里移除：

```python
background_tasks = set()

t = asyncio.create_task(some_coro())
background_tasks.add(t)
t.add_done_callback(background_tasks.discard)
```

**坑 3：吞掉 `CancelledError`。** 清理逻辑必须保证取消继续向上传播：

```python
try:
    await operation()
except asyncio.CancelledError:
    cleanup()        # 可以做清理
    raise            # 但必须重新抛出，让取消继续传播
```

**坑 4：在异步代码里用同步阻塞库。** 就是第 10 节讲的问题，用 `to_thread` 解决。

**调试开关**：`asyncio.run(main(), debug=True)`（或环境变量 `PYTHONASYNCIODEBUG=1`）会打开调试模式，效果包括：

- 检测到"回调执行超过 `loop.slow_callback_duration`（默认 0.1s）"时输出告警（这条告警经 logging 的兜底通道直接打到 stderr，无需配置日志）：

```text
Executing <Task ...> took 0.150 seconds
```

- 对"协程从未被 await"等情况提供更详细的对象创建位置信息；
- 打开额外的内部检查（如线程安全检查、协程原始位置追踪）。调试模式下 asyncio 还会通过 `asyncio` logger 输出 DEBUG 级信息（例如使用哪种 selector、事件循环关闭等），需要把 logging 配置到 DEBUG 级别才能看到。

**3.12 的新玩具：eager task factory。** 默认 `create_task` 只把协程的第一步排进队列；用 `asyncio.eager_task_factory` 可以让 `create_task` 立即同步执行协程体直到第一个挂起点，省掉一次调度往返，对短任务密集的场景有可观收益：

```python
loop = asyncio.get_running_loop()
loop.set_task_factory(asyncio.eager_task_factory)
task = asyncio.create_task(coro())   # 协程体此时已经执行到第一个挂起点
```

## 15. 总结

回到开头的问题：`await` 在等什么？**等一个 Future 完成；而"完成"的通知，由事件循环把回调排进就绪队列来投递。** 单线程之所以能并发，秘密不在这段循环代码里，而在 `select`/`epoll` 系统调用——把"我在等什么"交给内核，让出一份 CPU 去跑别的任务。

整篇文章的核心结论压缩成一张清单：

1. **协程协议**：`async def` 产生的协程是"可以被 send/throw/close 驱动的状态机"；`await` 在字节码层面就是 `yield from`；
2. **Task**：`Task.__step` 用 `coro.send(None)` 推进协程，遇到 Future 就挂完成回调，遇到裸 yield 就下轮重排——所有调度行为都在这 20 行里；
3. **事件循环**：三件家具（就绪 deque、定时器堆、selector）+ 两个约束（单轮快照、非阻塞回调）构成全部机制；
4. **IO 多路复用**：是循环的"睡眠机制"，也是 asyncio 单线程并发的物理基础；
5. **协作式调度的边界**：任何同步阻塞都会冻住整个循环；CPU 密集用多进程，阻塞库用 `to_thread`；
6. **工程化要点**：`gather`/`TaskGroup` 才是并发入口；限流用 `Semaphore`，背压用有界 `Queue`，取消要传播，fire-and-forget 要保存引用。

最后说选型。asyncio 最适合 **IO 密集、高并发、等待时间远大于计算时间**的场景：爬虫、API 网关、WebSocket 服务、数据库中间层。它不适合 CPU 密集计算（请用多进程）；也不适合"团队完全不熟悉异步心智"的项目——一个同步阻塞调用就能把收益清零。生态上，追求性能可以换 `uvloop`（libuv 实现，drop-in 替换），需要更严格的并发语义可以了解 `trio`/`anyio`，而理解本文这些机制，是使用它们中任何一个的前提。
