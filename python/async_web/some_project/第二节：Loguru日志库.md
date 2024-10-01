# 第二节：Loguru日志库

loguru是Python中一个使用简单、性能高效的日志库。

## 一、安装
通过以下命令即可安装：

```shell
$ pip install loguru==0.7.2
```

## 二、配置日志
默认的日志是直接输出到控制台，我们也可以设置让其输出到文件中。配置示例如下：

```python
from loguru import logger

logger.add("info_{time:YYYY-MM-DD}.log",level="INFO", encoding="utf-8", retention="10 day")

logger.add("file_1.log", rotation="500 MB")    # Automatically rotate too big file
logger.add("file_2.log", rotation="12:00")     # New file is created each day at noon
logger.add("file_3.log", rotation="1 week")    # Once the file is too old, it's rotated

logger.add("file_X.log", retention="10 days")  # Cleanup after some time

logger.add("file_Y.log", compression="zip")    # Save some loved space

logger.add("file.log", format="{time:YYYY-MM-DD at HH:mm:ss} | {level} | {message}")
```

如果想要清除之前添加的日志输出方式，可以调用：

```python
logger.remove()
```

`loguru`库是线程安全的，但不是进程安全。在多进程和协程中，可以在`logger.add`中加上`enqueue`参数，那么数据将会先存放到队列中，最后在执行`logger.complete()`时才会将日志信息保存到文件中。

在`FastAPI`中，可以在`lifespan`中，先添配置好日志。示例代码如下：

```python
from contextlib import asynccontextmanager
from loguru import logger
import sys


@asynccontextmanager
async def lifespan(app: FastAPI):
    # FastAPI程序即将运行时执行的代码
    logger.remove()
    logger.add("logs/file_{time}.log", rotation="500 MB", enqueue=True)
    yield
    # FastAPI程序执行后，即将推出时执行的代码

app = FastAPI(lifespan=lifespan)
```

这样，在程序启动时，就会配置好日志。然后可以添加一个日志的中间件，在执行完视图函数后再执行`await logger.complete()`。示例代码如下：

```python
@app.middleware('http')
async def log_middleware(request: Request, call_next):
    response = await call_next(request)
    await logger.complete()
    return response

# 或
from starlette.middleware.base import BaseHTTPMiddleware
app.add_middleware(BaseHTTPMiddleware, dispatch=log_middleware)
```

## 三、记录日志
| **<font style="color:rgb(64, 64, 64);">Level name</font>** | **<font style="color:rgb(64, 64, 64);">Severity value</font>** | **<font style="color:rgb(64, 64, 64);">Logger method</font>** |
| --- | --- | --- |
| `<font style="color:rgb(231, 76, 60);">TRACE</font>` | <font style="color:rgb(64, 64, 64);">5</font> | [<font style="color:rgb(64, 64, 64);">logger.trace()</font>](https://loguru.readthedocs.io/en/stable/api/logger.html#loguru._logger.Logger.trace) |
| `<font style="color:rgb(231, 76, 60);">DEBUG</font>` | <font style="color:rgb(64, 64, 64);">10</font> | [<font style="color:rgb(64, 64, 64);">logger.debug()</font>](https://loguru.readthedocs.io/en/stable/api/logger.html#loguru._logger.Logger.debug) |
| `<font style="color:rgb(231, 76, 60);">INFO</font>` | <font style="color:rgb(64, 64, 64);">20</font> | [<font style="color:rgb(64, 64, 64);">logger.info()</font>](https://loguru.readthedocs.io/en/stable/api/logger.html#loguru._logger.Logger.info) |
| `<font style="color:rgb(231, 76, 60);">SUCCESS</font>` | <font style="color:rgb(64, 64, 64);">25</font> | [<font style="color:rgb(64, 64, 64);">logger.success()</font>](https://loguru.readthedocs.io/en/stable/api/logger.html#loguru._logger.Logger.success) |
| `<font style="color:rgb(231, 76, 60);">WARNING</font>` | <font style="color:rgb(64, 64, 64);">30</font> | [<font style="color:rgb(64, 64, 64);">logger.warning()</font>](https://loguru.readthedocs.io/en/stable/api/logger.html#loguru._logger.Logger.warning) |
| `<font style="color:rgb(231, 76, 60);">ERROR</font>` | <font style="color:rgb(64, 64, 64);">40</font> | [<font style="color:rgb(64, 64, 64);">logger.error()</font>](https://loguru.readthedocs.io/en/stable/api/logger.html#loguru._logger.Logger.error) |
| `<font style="color:rgb(231, 76, 60);">CRITICAL</font>` | <font style="color:rgb(64, 64, 64);">50</font> | [<font style="color:rgb(64, 64, 64);">logger.critical()</font>](https://loguru.readthedocs.io/en/stable/api/logger.html#loguru._logger.Logger.critical) |


## 四、异常记录
可以在可能发生异常的函数上，加上`logger.catch`装饰器。如果该函数在执行过程中发生异常了，那么就会记录到日志中。实例代码如下：

```python
@router.get('/test')
@logger.catch(reraise=True)
async def test_view(arg: str):
    a = 1
    b = 0
    c = a / b
    return 'test'
```

或者可以统一加载日志中间件上，示例代码如下：

```python
@app.middleware('http')
@logger.catch(reraise=True)
async def log_middleware(request: Request, call_next):
    response = await call_next(request)
    await logger.complete()
    return response
```

## 五、更多
更多请参考官方文档：[https://loguru.readthedocs.io/en/stable/overview.html#installation](https://loguru.readthedocs.io/en/stable/overview.html#installation)



> 原文: <https://www.yuque.com/hynever/shtqfp/fepqay0lpl7sci9z>