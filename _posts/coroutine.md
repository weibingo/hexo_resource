---
title: python协程
date: 2019-01-20 10:31:06
tags: IO
categories: CS
---
这些天，在标签调度上，可预见性的标签调度会越来越多，如果全由调度系统承载，调度系统压力增大，负责调度系统的同事担心会影响其他任务，遂在讨论下，决策开发一个简版的标签执行服务/脚本（相比现有调度系统，只保留任务执行控制，例如池化，并发控制等，任务编排，启动时间都由调度系统控制，整个执行服务就是调度系统的一个task）。而我对此持保持意见，我认为应该增强调度系统能力，2333。

既然决策已定，加上部门现在主要使用python，而调度系统也使用的是airflow，所以主要开发语言也定为python。对于此执行服务/脚本，希望能够并行的执行任务，并且能够控制每次在跑的任务数。调研了下python的并发模型，以及多线程的知识。很多说python多线程是[鸡肋](https://www.zhihu.com/question/23474039)，而自己也不想引入第三方并发的库，又看到python支持协程，且相比线程轻量很多，代码易理解，遂最终选定使用协程来实现这个需求。
### 协程
网上对协程的解释众说纷纭，有说协程是一种用户态的轻量级线程，协程的调度完全由用户控制。wikipedia的定义：
协程是一个无优先级的子程序调度组件，允许子程序在特点的地方挂起恢复。但大家对协程的作用倒挺统一的：
* 占用资源少，开个协程只需要K级别内存，而新开个线程则需要M级别
* 线程之间的上下文切换，性能消耗大，而协程间切换非常快
* 相比事件驱动模型的回调复杂性，协程易于理解，写协程代码就像写同步代码一样。
* 协程的调度是协作式调度，需要协程自己主动把控制权转让出去之后，其他协程才能被执行到（很难像抢占式调度那样做到强制的 CPU 控制权切换到其他进程/线程）

### 历史的宿命
在互联网行业面临C10K问题时，线程方案不足以扛住大量的并发，这时的解决方案是epoll() 式的事件循环，nginx在这波潮流中顺利换掉apache上位。同一时间的开发社区为nginx的成绩感到震撼，出现了很多利用事件循环的应用框架，如tornado/ nodejs，也确实能够跑出更高的分数。而且python/ruby 社区受GIL之累，几乎没有并发支持，这时事件循环是一种并发的解放。然而事件循环的异步控制流对开发者并不友好。业务代码中随处可见的mysql/memcache调用，迅速地膨胀成一坨callback hell。这时社区发现了协程，在用户态实现上下文切换的工具，把epoll()事件循环隐藏起来，而且成本不高：用每个协程一个用户态的栈，代替手工的状态管理。似乎同时得到了事件循环和线程同步控制流的好处，既得到了epoll()的高性能，又易于开发。甚至通过monkey patch，旧的同步代码可以几乎无缝地得到异步的高性能，真是太完美了。
<!--more-->
### python协程的实现
#### gevent
gevent是基于协程（greenlet）的网络库，底层的事件轮询基于libev（早期是libevent），当一个greenlet遇到IO操作时，比如访问网络，就自动切换到其他的greenlet，等到IO操作完成，再在适当的时候切换回来继续执行。由于IO操作非常耗时，经常使程序处于等待状态，有了gevent为我们自动切换协程，就保证总有greenlet在运行，而不是等待IO（<font color=red size=2>所以当你的代码没有IO操作，协程便不会进行任务切换，所以你会看到是顺序执行的</font>）。gevent的API概念和Python标准库一致(如事件，队列)。gevent有一个很有意思的东西 monkey-patch，能够使python标准库中的阻塞操作变成异步，如socket的读写。下图是gevent与其他网络库的对比图。

![gevent与其他网络库的对比图](https://hexo-1256892004.cos.ap-beijing.myqcloud.com/coroutine/gevent.png)

gevent代码也很通俗易懂：
```python
def foo():
    print('Running in foo')
    gevent.sleep(0)
    print('Explicit context switch to foo again')

def bar():
    print('Explicit context to bar')
    gevent.sleep(0)
    print('Implicit context switch back to bar')

gevent.joinall([
    gevent.spawn(foo),
    gevent.spawn(bar),
])

# output
Running in foo
Explicit context to bar
Explicit context switch to foo again
Implicit context switch back to bar
```
定义foo和bar两个方法，将foo和bar放入gevent中（gevent.spawn(func)），并注册入轮询事件中。当遇到异步阻塞操作时，如代码中的gevent.sleep(0)，变会切换到其他gevent，一直反反复复，直到所有的gevent执行完成。而整个code也和写同步程序流程一致：定义两个方法，执行两个方法。而gevent内部遇到阻塞IO进行协程切换，而不是等此IO。

**monkey-patch**

写gevent程序几乎都会加一句code
```python
from gevent import monkey
monkey.patch_all()
```
这是干嘛用的呢？我们知道协程切换是在IO操作时自动完成，所以gevent需要修改Python自带的一些标准库，将python自带的标准库由阻塞IO改成非阻塞IO，这一过程在启动时通过monkey patch完成。
#### asyncio
asyncio是Python 3.4版本引入的标准库，直接内置了对异步IO的支持。asyncio的编程模型就是一个消息循环。我们从asyncio模块中直接获取一个EventLoop的引用，然后把需要执行的协程扔到EventLoop中执行，就实现了异步IO。

在gevent中，我们不能自己定义协程，只能通过控制非阻塞IO，gevent识别到非阻塞IO，自动切换协程。而gevent支持的非阻塞IO有限（以及通过monkey patch替换某些库），且不能很好的自定义。而且gevent是第三方库，在GitHub上看，以及四五年未更新了。

asyncio模块中引入了以下几个定义：
* event_loop 事件循环：程序开启一个无限的循环，程序员会把一些函数注册到事件循环上。当满足事件发生的时候，调用相应的协程函数。
* coroutine 协程：协程对象，指一个使用async关键字定义的函数，它的调用不会立即执行函数，而是会返回一个协程对象。协程对象需要注册到事件循环，由事件循环调用。
* task 任务：一个协程对象就是一个原生可以挂起的函数，任务则是对协程进一步封装，其中包含任务的各种状态。
* future： 代表将来执行或没有执行的任务的结果。它和task上没有本质的区别
* async/await 关键字：python3.5 用于定义协程的关键字，async定义一个协程，await用于挂起阻塞的异步调用接口。

##### asyncio.coroutine/yield from
在python3.4中，asyncio.coroutine修饰器用来标记作为协程的函数，这里的协程是和asyncio及其事件循环一起使用的。结合python之前的yield语法，通过yield from将协程交给其他对象继续执行，代码示例如下：
```python
import asyncio

@asyncio.coroutine
def foo():
    print('Running in foo')
    yield from asyncio.sleep(1)
    print('Explicit context switch to foo again')

def bar():
    print('Explicit context to bar')
    yield from asyncio.sleep(1)
    print('Implicit context switch back to bar')
 
loop = asyncio.get_event_loop()
tasks = [
    asyncio.ensure_future(foo()),
    asyncio.ensure_future(bar())]
loop.run_until_complete(asyncio.wait(tasks))
loop.close()
```
##### await/async
Python 3.5 在处理协程时增加了一些特殊的语法（await/async）。新功能中很大一部分在3.5 之前的版本就已经有了，不过之前的语法并不算最好的，因为生成器和协程的概念本身就有点容易混淆。PEP-0492 通过使用 async 关键字显式的对生成器和协程做了区分。对于python迭代器，生成器和协程的知识，可以自己去网上查，也可以参考[从迭代器、生成器到协程](https://juejin.im/entry/585e6f62570c3500693a1058)。这里就不讲解了。

而await/async也可以简单看成将@asyncio.coroutine替换成await，yield from替换成await
```python
# This also works in Python 3.5.
@asyncio.coroutine
def py34_coro():
    yield from stuff()
````
与
```python
async def py35_coro():
    await stuff()
```
在async/await模型中，协程函数内部也可以调用多个协程函数。

如何更好的理解event loop和await/async代码呢？
```python
import asyncio
import random
import time

async def worker(name, queue):
    while True:
        # Get a "work item" out of the queue.
        sleep_for = await queue.get()

        # Sleep for the "sleep_for" seconds.
        await asyncio.sleep(sleep_for)

        # Notify the queue that the "work item" has been processed.
        queue.task_done()

        print(f'{name} has slept for {sleep_for:.2f} seconds')


async def main():
    # Create a queue that we will use to store our "workload".
    queue = asyncio.Queue()

    # Generate random timings and put them into the queue.
    total_sleep_time = 0
    for _ in range(20):
        sleep_for = random.uniform(0.05, 1.0)
        total_sleep_time += sleep_for
        queue.put_nowait(sleep_for)

    # Create three worker tasks to process the queue concurrently.
    tasks = []
    for i in range(3):
        task = asyncio.create_task(worker(f'worker-{i}', queue))
        tasks.append(task)

    # Wait until the queue is fully processed.
    started_at = time.monotonic()
    await queue.join()
    total_slept_for = time.monotonic() - started_at

    # Cancel our worker tasks.
    for task in tasks:
        task.cancel()
    # Wait until all worker tasks are cancelled.
    await asyncio.gather(*tasks, return_exceptions=True)

    print('====')
    print(f'3 workers slept in parallel for {total_slept_for:.2f} seconds')
    print(f'total expected sleep time: {total_sleep_time:.2f} seconds')
    
loop = asyncio.get_event_loop()
loop.run_until_complete(main())
loop.close()
```
首先需要loop = asyncio.get_event_loop()获取一个事件loop，loop是个死循环，轮询遍历task。然后将协程task注册到loop中，在上面的示例中，main()有三个task。当每个task执行协程函数遇到await时，切换到其他task执行，如此反复，直到loop中的task全部执行完毕，退出。

### 总结
本来简单介绍了协程的优点，以及python对协程的支持，以及如何写python协程代码。但未对gevent或者async/await用在服务中，从网上搜索来看，似乎python的协程坑还是挺多的，如果需要使用，还需要多多测试。对于我的需求，使用python协程方案已经能够不错满足了，而且本来也只是利用其并发优势来执行任务，不涉及高并发请求。python现在的定位还是：开发速度快，解释语言，脚本处理，数据处理。如果是高并发服务的话，需要多斟酌下。

### 后续
协程这个概念几乎是由golang语言带火的，golang从底层语言特性添加了对协程的支持，也是现在对协程支持最完善的语言了。后续有时间也研究下~

### 参考
[为什么觉得协程是趋势？](https://www.zhihu.com/question/32218874)

[linux进程-线程-协程上下文环境的切换与实现](https://blog.csdn.net/runner668/article/details/80512664)

https://docs.python.org/3/whatsnew/3.5.html#whatsnew-pep-492

https://www.liaoxuefeng.com/wiki/001374738125095c955c1e6d8bb493182103fac9270762a000/0013868328689835ecd883d910145dfa8227b539725e5ed000
