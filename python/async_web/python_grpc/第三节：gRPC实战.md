# 第三节：gRPC实战

## 一、安装依赖
执行rpc，还需要安装`grpcio` ，命令如下：

```bash
$ pip install grpcio
```

## 二、服务端实现
### 1. 编写.proto文件：
在项目下创建一个`article.proto` 文件，代码如下：

```protobuf
// 指定版本号为3
syntax = "proto3";

message Article{
  int32 id = 1;
  string title = 2;
  string content = 3;
  string create_time = 4;
}

message ArticleListRequest{
  int32 page = 1;
  int32 page_size = 2;
}

message ArticleListResponse{
  // repeated Article：这是数据类型，文章的列表
  repeated Article articles = 1;
}

message ArticleDetailRequest {
  int32 pk = 1;
}

message ArticleDetailResponse {
  Article article = 1;
}

service ArticleService{
  rpc ArticleList(ArticleListRequest) returns (ArticleListResponse);
  rpc ArticleDetail(ArticleDetailRequest) returns (ArticleDetailResponse);
}
```

### 2. 生成Python代码
在`apps/rpc/article`下使用以下命令生成Python代码：

```bash
$ python -m grpc_tools.protoc -I. --python_out=. --grpc_python_out=. article.proto
```

使用以上命令会生成一个`article_pb2.py` 和一个`article_pb2_grpc.py` 文件。

### 3.启动gRPC服务
在项目的根路径下，创建一个`main.py`的文件，然后填入以下代码：

```python
import article_pb2_grpc
import article_pb2
import grpc
from concurrent.futures import ThreadPoolExecutor


class ArticleServer(article_pb2_grpc.ArticleServiceServicer):
    def ArticleList(self, request, context):
        page = request.page
        page_size = request.page_size
        print("page:", page, "page_size:", page_size)
        # 创建response对象
        response = article_pb2.ArticleListResponse()
        articles = [
            article_pb2.Article(id=100, title='xx', content='yy', create_time='2030-10-10'),
            article_pb2.Article(id=101, title='11', content='22', create_time='2030-10-10'),
        ]
        # 以下写法是错误的
        # response.articles = articles
        response.articles.extend(articles)
        return response

def main():
    server = grpc.server(ThreadPoolExecutor(max_workers=10))
    article_pb2_grpc.add_ArticleServiceServicer_to_server(ArticleServer(), server)
    server.add_insecure_port("0.0.0.0:5000")
    server.start()
    print('服务器已经启动！')
    server.wait_for_termination()

if __name__ == '__main__':
    main()
```

## 三、客户端实现
### 1.拷贝代码
由于我们客户端和服务端用的都是Python语言，所以我们可以直接将服务端生成的关于rpc的代码拷贝到客户端程序中，这里我们拷贝`article_pb2_grpc.py`和`article_pb2.py`拷贝到客户端项目中。

### 2.调用微服务
代码实现如下：

```python
import grpc
from article_pb2_grpc import ArticleServiceStub
from article_pb2 import ArticleListRequest

def main():
    with grpc.insecure_channel("127.0.0.1:5000") as channel:
        stub = ArticleServiceStub(channel=channel)
        request = ArticleListRequest()
        request.page = 1
        request.page_size = 10
        ret = stub.ArticleList(request)
        print(ret.articles)


if __name__ == '__main__':
    main()
```

## 四、异步gRPC
### 1.服务端
`grpcio`库也提供了异步的版本，只需在启动项目的时候使用`grpc.aio.server`即可使用异步的方式启动。然后在服务方法上，加上`async`关键字，并且在需要阻塞的地方加上`await`即可。以上同步服务端程序改为异步代码如下：

```python
import article_pb2_grpc
import article_pb2
import grpc
import asyncio


class ArticleServer(article_pb2_grpc.ArticleServiceServicer):
    async def ArticleList(self, request, context):
        page = request.page
        page_size = request.page_size
        print("page:", page, "page_size:", page_size)
        # 创建response对象
        response = article_pb2.ArticleListResponse()
        # 异步阻塞1s
        await asyncio.sleep(1)
        articles = [
            article_pb2.Article(id=100, title='xx', content='yy', create_time='2030-10-10'),
            article_pb2.Article(id=101, title='11', content='22', create_time='2030-10-10'),
        ]
        response.articles.extend(articles)
        print("="*10)
        return response

async def main():
    server = grpc.aio.server()
    article_pb2_grpc.add_ArticleServiceServicer_to_server(ArticleServer(), server)
    server.add_insecure_port("0.0.0.0:5000")
    await server.start()
    print('服务器已经启动！')
    await server.wait_for_termination()

if __name__ == '__main__':
    asyncio.run(main())
```

### 2.客户端
```python
import grpc
from article_pb2_grpc import ArticleServiceStub
from article_pb2 import ArticleListRequest
import asyncio

async def main():
    async with grpc.aio.insecure_channel("127.0.0.1:5000") as channel:
        stub = ArticleServiceStub(channel=channel)
        request = ArticleListRequest()
        request.page = 1
        request.page_size = 10
        ret = await stub.ArticleList(request)
        print(ret.articles)


if __name__ == '__main__':
    asyncio.run(main())
```



## 五、集成到Django项目中
有时候，我们的微服务需要集成到Django项目中。实现逻辑是，先创建一个app（比如叫做rpc），然后创建一个命令`runrpcserver`，将服务端的rpc代码放到命令中。

### 1.创建gRPC服务
首先将`article_pb2_grpc.py` 中，导入`article_pb2` 模块的代码修改为如下：

```python
import grp
import warnings

from apps.rpc.article import article_pb2 as article__pb2
...
```

在`apps/rpc/management/commands/runrpcserver.py` 中下添加以下代码：

```python
from apps.rpc.article import article_pb2_grpc, article_pb2
from apps.article.models import Article
import grpc
import concurrent.futures as futures
import asyncio
from django.core.management.base import BaseCommand

class ArticleServer(article_pb2_grpc.ArticleServiceServicer):
    async def ArticleList(self, request, context):
        """
        实现GetArticle方法
        :param request: GetArticleRequest对象
        :param context: 通过此对象可以设置调用返回的异常信息
        :return:
        """
        page = request.page
        print('page:', page)
        resp = article_pb2.ArticleListResponse()
        queryset = Article.objects.all()
        articles = []
        async for article in queryset:
            articles.append(article_pb2.Article(id=article.id, title=article.title, content=article.content, create_time=article.create_time.strftime('%Y-%m-%d %H:%M:%S')))
        resp.articles.extend(articles)
        return resp

async def main():
    # 创建一个rpc服务器
    server = grpc.aio.server()
    # 注册ArticleServer
    article_pb2_grpc.add_ArticleServiceServicer_to_server(ArticleServer(), server)
    # 监听端口
    server.add_insecure_port('127.0.0.1:5000')
    # 启动服务器
    await server.start()
    # 阻塞主线程
    await server.wait_for_termination()

class Command(BaseCommand):
    def handle(self, *args, **options):
        asyncio.run(main())
```

记得要在`settings.py` 的`INSTALLED_APPS` 中添加`apps.rpc` 。

### 2. 运行rpc服务
在项目根路径下执行以下命令即可启动微服务：

```bash
$ python manage.py runrpcserver
```

### 3.调用gRPC服务
同样，在Django项目中，可以调用gRPC服务。示例代码如下：

```python
from django.shortcuts import render
from django.views import View
import grpc
from rpc.article.article_pb2_grpc import ArticleServiceStub
from rpc.article.article_pb2 import GetArticleRequest, GetArticleResponse
from django.http.response import JsonResponse

class ArticleView(View):
    async def get(self, request):
        async with grpc.aio.insecure_channel("127.0.0.1:5000") as channel:
            # 创建调用的辅助工具对象 stub
            stub = ArticleServiceStub(channel)

            # 调用远程方法
            article_request = ArticleListRequest()
            article_request.page = 2
            article_request.page_size = 10
            ret = stub.ArticleList(article_request)
            for article in ret.articles:
                print(article.title, article.content, article.id, article.create_time)

            # 处理返回结果
            return JsonResponse({"articles": articles})
```

## 六、gRPC官方文档：
1. gRPC文档：[https://grpc.io/docs/languages/python/quickstart/](https://grpc.io/docs/languages/python/quickstart/)
2. gRPC AsyncIO：[https://grpc.github.io/grpc/python/grpc_asyncio.html](https://grpc.github.io/grpc/python/grpc_asyncio.html)



> 原文: <https://www.yuque.com/hynever/micro-service/inryvyvhgf7bwmwc>