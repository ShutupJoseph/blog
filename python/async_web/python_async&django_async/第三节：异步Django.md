# 第三节：异步Django

在Django3.1版本开始，Django正式支持异步。以下来讲解异步Django项目的实现细节。

## 一、ASGI服务器
### 1. ASGI服务器介绍
在以前，我们开发同步代码，可以使用`wsgi` 协议的服务器来运行django项目，比如`uwsgi` 。而现在开发异步代码，则必须使用`asgi` 协议的服务器来运行，Django官方推荐的有三个`asgi` 服务器，分别是：`Daphne` 、`Uvicorn` 、`Hypercorn` 。其中`Daphne` 是Django团队开发的，可以非常方便的集成到Django的开发环境中，而`Uvicorn` 是基于`uvloop` ，其性能能媲美Go语言，基于以上两点考虑，我们在开发阶段使用`Daphne` ，在生产环境中使用`Uvicorn` 。

### 2. 集成Daphne
首先通过以下命令安装Daphne：

```python
pip install daphne==4.1.2
```

然后在`settings.py` 中添加以下配置：

```python
INSTALLED_APPS = [
    "daphne",
    ...,
]

ASGI_APPLICATION = "myproject.asgi.application"
```

这样以后即可通过`python manage.py runserver` 的方式运行项目了。

## 二、异步视图
### 1. 函数视图
如果是函数视图，则只需要在定义函数的时候，加上`async` 关键字即可，示例代码如下：

```python
async def index(request):
    return render(request, 'index.html')
```

### 2. 类视图
类视图可以在定义方法的时候，在前面加上`async` 关键字，示例代码如下：

```python
class AboutView(View):
    async def get(self, request):
        return render(request, 'about.html')
```

## 三、异步ORM
数据库操作是一个典型的可以用异步优化的模块。在异步ORM中，存在两类操作：

1. **返回QuerySet的方法**。在Django中，QuerySet对象是懒惰的，他只是构建一个查询对象，并不会立即执行`SQL`语句。比如`Book.objects.all()`返回的是QuerySet对象，只有对`QuerySet`对象执行以下操作时，才会真正执行`SQL`语句。
+ 迭代（循环）。
+ 切片。当步长不是`1`时，就会执行`SQL`语句。
+ Pickle序列化/缓存。
+ 使用`repr`函数。
+ 使用`len`函数获取元素个数。
+ 使用`list`函数强制转换为列表。
+ 使用`bool`函数、`if`判断。

 所以在异步模式中，不能直接使用以上操作，否则将运行阻塞的同步代码，阻塞事件循环。这种情况的处理方式，是通过`异步迭代`的方式来查找数据库。示例代码如下：

```python
queryset = Book.objects.all()
books = []
async for book in queryset:
    books.append(book)
```

 完整的返回`QuerySet` 对象的方法：[https://docs.djangoproject.com/en/5.0/ref/models/querysets/#methods-that-return-new-querysets](https://docs.djangoproject.com/en/5.0/ref/models/querysets/#methods-that-return-new-querysets)  
 

2. **不返回QuerySet的方法**。比如`first`、`save`、`delete`等。这类方法Django大多都提供了异步版本，比如`afirst`、`asave`、`adelete`。示例代码如下：

```python
# 查询
async for author in Author.objects.filter(name__startswith="A"):
        # afirst
    book = await author.books.afirst()

# 保存
async def make_book(*args, **kwargs):
    book = Book(...)
    # asave
    await book.asave()

async def make_book_with_tags(tags, *args, **kwargs):
        # acreate
    book = await Book.objects.acreate(...)
    # aset
    await book.tags.aset(tags)
```

完整的不返回`QuerySet`方法：[https://docs.djangoproject.com/en/5.0/ref/models/querysets/#methods-that-do-not-return-querysets](https://docs.djangoproject.com/en/5.0/ref/models/querysets/#methods-that-do-not-return-querysets)

**注意：事务目前还不支持异步模式，如果想要实现事务的异步模式，需要借助**`sync_to_async`** 来实现。**

## 四、异步适配函数
当在异步的环境中调用耗时的同步函数时，需要将同步函数转换为异步函数，这样才不会对事件循环进行阻塞。同理，如果在同步环境中调用异步函数，也需要将异步函数转换为同步函数。为此，Django团队开发了一个叫做`asgiref` 的包，里面提供了两个函数：`sync_to_async` 、`async_to_sync` 来实现同步代码和异步代码的相互转换。

### 1. sync_to_async(sync_func, thread_sensitive=True)
将同步函数作为异步函数执行。其实底层原理，本质上是将同步函数放到线程中执行的，这样就不会阻塞事件循环了。示例代码如下：

```python
class BaiduView(View):
    def get_page(self):
        resp = requests.get('https://www.baidu.com')
        return resp.text

    async def get(self, request):
        aget_page = sync_to_async(self.get_page)
        aget_qq = sync_to_async(self.get_qq_page, thread_sensitive=False)
        text = await aget_page()
        return HttpResponse(text)
```

其中`thread_sensitive` 参数用于控制同步函数的运行策略：

+ `thread_sensitive=True`：（默认值）代表该请求的所有同步函数都在同一个线程（不是主线程）中执行。如果多个同步函数需要访问相同的临界值，那么就使用这种方式，因为这种方式会让当次请求的所有同步函数都在同一个线程中之行，不会对临界值造成问题。
+ `thread_sensitive=False`：代表该同步函数在一个独立的线程中执行。同步函数间不会访问临界值，以及同步函数可能比较耗时，那么就可以用这种方式，因为这种方式会将同步函数放到新的线程中执行，可以提高并发性。

### 2. async_to_sync(async_func, force_new_loop=False)
将异步函数作为同步函数执行。如果当前线程有事件循环，则会直接将异步函数加入到事件循环中执行，如果没有事件循环，则会先创建一个事件循环，再该异步函数加入到事件循环中执行。同步函数会等待这个事件循环执行完后才继续往下执行。

## 五、异步中间件
在异步开发中，中间件必须是异步的，如果中间件是同步的，那么django会先通过线程的方式切换到同步模式来执行中间件代码，再切换到异步模式来执行视图函数代码。这样会抵消采用异步编程带来的性能提升。

### 1. 函数中间件
如果用的是函数中间件，那么可以采用`sync_only_middleware` 、`async_only_middleware` 、`sync_and_async_middleware` 三个装饰器来构建中间件的运行模式。比如我们使用`sync_and_async_middleware` 装饰器来实现一个既支持同步又支持异步的中间件，示例代码如下：

```python
from asgiref.sync import iscoroutinefunction
from django.utils.decorators import sync_and_async_middleware

@sync_and_async_middleware
def simple_middleware(get_response):
    # One-time configuration and initialization goes here.
    if iscoroutinefunction(get_response):

        async def middleware(request):
            # 执行视图函数之前执行的
            response = await get_response(request)
            # 执行完视图函数之后执行的
            return response

    else:

        def middleware(request):
            # Do something here!
            response = get_response(request)
            return response

    return middleware
```

+ `@sync_and_async_middleware`：那么具体执行异步还是同步，依赖于部署的web服务器是什么，如果是异步的，那么就只会执行异步的部分，否则就只会执行同步的部分。
+ `@sync_only_middleware`：则只会执行同步的部分。
+ `@async_only_middleware`：则只会执行异步的部分。

### 2. 类中间件
如果采用的是类中间件，那么可以通过`async_capable` 、`sync_capable` 两个类属性来控制该类中间件支持的模式，默认是只支持同步的。用类中间件实现一个仅支持异步的中间件示例代码如下：

```python
from asgiref.sync import iscoroutinefunction, markcoroutinefunction

class AsyncMiddleware:
    async_capable = True
    sync_capable = False

    def __init__(self, get_response):
        self.get_response = get_response
        if iscoroutinefunction(self.get_response):
            markcoroutinefunction(self)

    async def __call__(self, request):
            # 进入视图函数之前的代码
        response = await self.get_response(request)
        # 执行完视图函数之后的代码
        return response
```

## 六、异步装饰器
下列装饰器既支持同步，也支持异步（django5.0后添加的）：

+ `cache_control()`
+ `never_cache()`
+ `no_append_slash()`
+ `csrf_exempt()`
+ `csrf_protect()`
+ `ensure_csrf_cookie()`
+ `requires_csrf_token()`
+ `sensitive_variables()`
+ `sensitive_post_parameters()`
+ `gzip_page()`
+ `condition()`
+ `conditional_page()`
+ `etag()`
+ `last_modified()`
+ `require_http_methods()`
+ `require_GET()`
+ `require_POST()`
+ `require_safe()`
+ `vary_on_cookie()`
+ `vary_on_headers()`
+ `xframe_options_deny()`
+ `xframe_options_sameorigin()`
+ `xframe_options_exempt()`

`  
`



> 原文: <https://www.yuque.com/hynever/async-django/pekgpgdc4l6sn5zl>