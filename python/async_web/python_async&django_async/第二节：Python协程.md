# 第二节：Python协程

## 一、第一个协程代码
在Python3.4中添加了asyncio库，这让我们利用Python编写协程（coroutine）代码变得更加简单，以下是一个简单的协程代码：

```python
import asyncio
#import time
#time.sleep(1)

async def main():
    print('hello')
    await asyncio.sleep(1)
    print('world')

if __name__ == '__main__':
		# main()：创建一个协程，并不是直接运行main函数，并且这个协程是不会自己运行的
    asyncio.run(main())

# 以下是注释，不是代码，方便理解事件循环
# 有一个协程队列，存储所有需要执行的协程
queue = [cor1, cor2, ...]
while True:
	cor = queue.pop()
	result = await cor
	# 其他代码
```

以上代码有以下几点需要说明：

1. 协程不会自己运行，需要加入到事件循环中，让事件循环调度运行。我们可以通过asyncio.run 函数来将一个协程放到事件循环中执行。
2. 通过在函数前面加上async 关键字，将一个普通的函数变为一个协程。
3. 在协程中，使用await 关键字等待一个协程执行完成，协程运行，必须在前面加上await。关键字，await 关键字也必须放到async 定义的函数中，否则会报错。
4. 以上asyncio.sleep 函数不能用time.sleep 来代替，后者是同步的，如果放到异步函数中，将无法发挥异步编程的优势。

## 二、计算时间装饰器
在后续的测试代码中，经常需要计算程序的运行时间，这里我们提前写好一个装饰器，代码如下：

```python
import asyncio
import time
from functools import wraps

def async_timed(func):
    @wraps(func)
    async def wrapped(*args, **kwargs):
        print(f'开始执行{func}，参数为：{args}, {kwargs}')
        start = time.time()
        try:
            return await func(*args, **kwargs)
        finally:
            end = time.time()
            total = end - start
            print(f'结束执行{func}，耗时：{total:.4f}秒')

    return wrapped
```

## 三、并发（concurrency）多个协程
### 1. 使用任务
如果想要并发运行多个协程，那么可以将任务创建成Task/Future 对象，示例代码如下：

```python
async def greet(name, delay):
    await asyncio.sleep(delay)
    return f"Hello, {name}!"

@async_timed
async def main():
    task1 = asyncio.create_task(greet('xx', 2))
    task2 = asyncio.create_task(greet('yy', 3))

    # Wait for tasks to complete
    await task1
    await task2

if __name__ == "__main__":
    asyncio.run(main())
```

以上程序的运行时间将在4s即可结束，说明两个任务是并发运行了的。**千万要注意，任务要并发运行，必须先全部都创建好，再进行****await**** ，否则将同步执行**，比如以下代码将耗时3s：

```python
async def greet(name, delay):
    await asyncio.sleep(delay)
    return f"Hello, {name}!"

@async_timed
async def main():
    await asyncio.create_task(greet('xx', 1))
    await asyncio.create_task(greet('yy', 2))
```

### 2. 使用任务组
除了创建完任务后，一个个await ，还可以使用TaskGroup 创建一个任务组，然后在任务组中创建多个任务，最终统一await 这个任务组，示例代码如下：

```python
async def greet(name, delay):
    await asyncio.sleep(delay)
    return f"Hello, {name}!"

@async_timed
async def main():
    async with asyncio.TaskGroup() as group:
        task1 = group.create_task(greet('xx', 2))
        task2 = group.create_task(greet('yy', 4))
    print(task1.result())
    print(task2.result())

if __name__ == "__main__":
    asyncio.run(main())
```

TaskGroup 中的任务，如果在运行期间有任何一个任务抛出异常，那么其他任务将全部被取消。比如以下代码：

```python
async def greet(name, delay):
    await asyncio.sleep(delay)
    if name == 'xx':
        raise ValueError("操作错误！")
    return f"Hello, {name}!"

@async_timed
async def main():
    try:
        async with asyncio.TaskGroup() as group:
            task1 = group.create_task(greet('xx',2))
            task2 = group.create_task(greet('yy',4))
    except Exception as e:
        print(task2.cancelled())
```

由于task1 任务抛出异常了，那么会导致task2 被取消。

### 3. 使用gather
以上代码是手动创建任务后运行，另外还可以通过一个更高级的API来实现并发运行，即asyncio.gather ，这个函数的底层实际上也是将协程封装成Future 对象，然后再并发执行。

```python
import asyncio
import time

async def greet(name, delay):
    await asyncio.sleep(delay)
    return f"Hello, {name}!"

@async_timed
async def main():
    results = await asyncio.gather(
        greet("张三", 2),
        greet("李四", 1),
        greet("王五", 3),
    )
    print(results)

if __name__ == "__main__":
    asyncio.run(main())
```

以上代码，我们通过使用asyncio.gather 函数，将多个协程并发运行，总共耗时才3秒。

补充：

+ 如果gather中的协程出现异常，那么会抛出异常，如果不想抛出异常，那么可以设置return_exceptions=True ，就会把异常作为返回值，而不会抛出异常。
+ gather在将所有协程全部都执行完后，会按照协程入队的顺序，将协程的返回值存放在results中。
+ 与TaskGroup相比：asyncio.gather 函数即使其中有任务抛出异常，也不会取消后面的任务；而TaskGroup 则是只要有一个任务抛出异常，后续任务将会被取消。

示例代码如下：

```python
@async_timed
async def main():
    try:
        results = await asyncio.gather(
            greet_group('xx', 1),
            greet('李四', 3),
            greet('王五', 2),
        )
    except Exception as e:
        pass
    # 获取所有正在运行的任务
    tasks = asyncio.all_tasks()
    for task in tasks:
        # 这里说明：gather中有的任务发生异常了，剩余还没执行完的任务也不会被取消
        # print(task.get_name(), task.cancelled())

        # 继续等待还未执行完的任务
        # Task-1：代表的是main这个协程
        # 不能在协程中await自身，比如在main协程中，不能awiat main这样
        if task.get_name() == 'Task-1':
            continue
        result = await task
        print(result)
```

+ 与as_completed相比：asyncio.gather会等待所有任务都执行完成后才会返回，而as_completed是执行完一个任务后就立即返回。

### 4. 使用as_completed
使用asyncio.as_completed 方法也可以实现协程的并发运行，与asyncio.gather不同的是，asyncio.as_completed会每运行完一个协程后就返回。使用方式如下：

```python
async def main():
    aws = [
        greet('张三', 2),
        greet('李四', 1)
    ]
    for coro in asyncio.as_completed(aws):
        result = await coro
        print(result)
```

上述代码中，会并发执行aws 中的协程，并且可以通过遍历的形式进行等待，最先执行完的会先返回结果。以上返回结果为"world" ，然后再是"hello" 。as_completed 方法在其中某个任务抛出异常，或者超时后，剩余的任务都不会被取消掉。比如以下代码：

```python
async def greet_group(name, delay):
    await asyncio.sleep(delay)
    if name == 'xx':
        raise ValueError("执行错误！")
    return f'hello {name}'

@async_timed
async def main():
    try:
        tasks = asyncio.as_completed([
            greet_group('xx', 1),
            greet('李四', 3)
        ], timeout=4)
        for task in tasks:
            result = await task
            print(result)
    except Exception as e:
        pass
    # 其中一个任务出现异常后，剩余任务不会取消，可以继续等待其完成。
    for task in asyncio.all_tasks():
        if task.get_name() != 'Task-1':
            result = await task
            print(result)
```

## 三、等待
有时候，我们期望某个协程或者任务最多运行多长时间，就可以使用wait_for 或wait 函数。

### 1. wait_for(aw, timeout)函数
wait_for 函数只能用于等待一个协程或者任务，可以指定超时时间。

```python
async def enternity(what):
    await asyncio.sleep(5)
    return what

async def main():
    try:
        await asyncio.wait_for(enternity('hello'), timeout=1)
    except asyncio.TimeoutError:
        print('timeout!')
```

wait_for在超时后，会抛出TimeoutError。超时后的任务，没法继续让其运行了。

### 2. wait(aws, timeout=None, return_when=ALL_COMPLETED)函数
这个函数可用于等待多个Task或Future对象，并且可以指定在什么情况下才会返回，默认是ALL_COMPLETED ，并且注意，这个函数并不会触发TimeoutError，而是将执行完的，以及超时的，通过元组的形式返回。示例代码如下：

```python
async def enternity(delay, what):
    await asyncio.sleep(delay)
    return what

async def main():
    done_tasks, pending_tasks = await asyncio.wait([
        asyncio.create_task(enternity(1, 'hello')),
        asyncio.create_task(enternity(3, 'python')),
        asyncio.create_task(enternity(4, 'asyncio'))
    ], timeout=2)
    # 循环打印已经执行完的任务的返回值
    for task in done_tasks:
        print(task.result())
		# 将剩余的任务并发执行
    results = await asyncio.gather(*pending_tasks)
    print(results)
```

如果没有指定timeout参数，那么就永远不会超时。

其中，return_when 除了默认的ALL_COMPLETED 外，还有以下可选值：

+ ALL_COMPLETED ：等所有任务都执行完成后再返回。
+ FIRST_EXCEPTION ：有任何任务发生异常后就立即返回，即使没有超时也会返回。
+ FIRST_COMPLETED ：第一个任务执行完后就立即返回。

## 四、超时
asyncio提供了专门的超时API，用于限制某些任务的最大执行时间。超时API有两个，分别是：asyncio.timeout 以及asyncio.timeout_at ，以下分别进行讲解。

### 1. asyncio.timeout(delay)
这个方法有以下几点：

+ 该函数返回一个异步上下文管理器，也就意味着我们可以使用async with 进行使用。
+ 其中delay 可以为具体的秒数，也可以为None ，如果为None ，那么代表永远不会超时。
+ 如果超过delay 的时间，那么下面的所有任务都将被取消，并会抛出TimeoutError 异常。

示例代码如下：

```python
async def main():
    try:
        async with asyncio.timeout(2):
            task1 = asyncio.create_task(enternity(1, 'hello'), name='AA')
            task2 = asyncio.create_task(enternity(3, 'world'), name='BB')

            await task1
            await task2
    except asyncio.TimeoutError:
        print('超时了')
```

### 2. asyncio.timeout_at(when)
与asyncio.timeout 不同的是，asyncio.timeout_at 中的when 参数是一个绝对时间，或者为None。示例代码如下：

```python
async def main():
    try:
        loop = asyncio.get_running_loop()
        deadline = loop.time() + 2
        async with asyncio.timeout_at(deadline):
            task1 = asyncio.create_task(enternity(1, 'hello'), name='AA')
            task2 = asyncio.create_task(enternity(3, 'world'), name='BB')

            await task1
            await task2
    except asyncio.TimeoutError:
        print('超时了')
```

## 五、在线程中运行
对于一些很耗时的同步代码，我们可以将其放到单独的线程中运行，这样就不会阻塞事件循环了。我们可以通过asyncio.to_thread 来实现这个需求，示例代码如下：

```python
# 同步的阻塞函数
def blocking_function():
    print('blocking开始')
    time.sleep(2)
    print('blocking结束')
    return "success"

async def main():
    print('main开始')
    results = await asyncio.gather(
        asyncio.to_thread(blocking_function),
        asyncio.sleep(1)
    )
    print(results)
    print('main结束')
```

以上代码运行只需要2s，如果将以上代码的asyncio.to_thread 删掉，那么运行需要3s，因为time.sleep(2) 是阻塞的，会直接阻塞事件循环。

## 6. Task对象
Task对象是用于封装和管理协程的运行的，可以将协程并发执行。Task对象有以下方法：

+ done() ：用于获取该Task对象是否执行完成（正常完成、异常、被取消都算done）。
+ result() ：用于获取该Task执行完后的返回值。
+ exception() ：如果Task对象执行过程中发生异常，则该方法会返回异常信息。如果任务没有发生异常，那么调用exception()方法将抛出asyncio.exceptions.InvalidStateError: Exception is not set.异常。示例代码如下：

```python
async def task_will_fail():
    await asyncio.sleep(1)
    raise ValueError('发生异常啦！')

async def main():
    task = asyncio.create_task(task_will_fail())
    await asyncio.sleep(2)
    print(task.exception())
```

+ add_done_callback ：添加任务执行完成后的回调。示例代码如下：

```python
async def enternity(delay, what):
    await asyncio.sleep(delay)
    return what
    
def callback(delay, future):
    print('回调')
    print('delay:', delay)
    print('执行结果：', future.result())

async def main():
    print('main开始')
    task = asyncio.create_task(enternity(1, 'hello'))
    task.add_done_callback(partial(callback, 2))
    await task
    print('main结束')
```

+ cancel ：取消任务的执行。

```python
async def something():
    print('something start')
    await asyncio.sleep(20)

async def main():
    print('main开始')
    task = asyncio.create_task(something())
    await asyncio.sleep(1)
    task.cancel()
    try:
        await task
    except asyncio.CancelledError:
        print('main 取消错误！')
    print('main结束')
```

+ cancelled ：判断任务是否被取消。
+ get_name ：获取任务的名称。
+ set_name ：设置任务的名称。



> 原文: <https://www.yuque.com/hynever/async-django/kqfuxyog87pnd8bx>