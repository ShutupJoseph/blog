# 第四节：gRPC进阶

本节内容都是基于以下ProtocolBuffer代码实现的：

```protobuf
// 指定版本号为3
syntax = "proto3";

/*
这是一个Article消息
*/
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

## 一、元数据
元数据，或者说是gRPC协议中的请求头，类似于HTTP协议中的header，可以让我们在客户端和服务端数据交互过程中携带一些额外的数据。

### 服务端代码实现
```python
from proto import article_pb2, article_pb2_grpc
import grpc
from concurrent import futures


class ArticleService(article_pb2_grpc.ArticleServiceServicer):
    def ArticleList(self, request, context):
        for key, value in context.invocation_metadata():
            print("客户端的元数据: key=%s value=%s" % (key, value))

        context.set_trailing_metadata(
            (
                ("server", "grpc"),
                ("system", "windows"),
            )
        )
        response = article_pb2.ArticleListResponse()
        return response


def main():
    server = grpc.server(futures.ThreadPoolExecutor(max_workers=10))
    article_pb2_grpc.add_ArticleServiceServicer_to_server(ArticleService(), server)
    server.add_insecure_port("[::]:50051")
    server.start()
    server.wait_for_termination()


if __name__ == '__main__':
    main()
```

### 客户端代码实现
```python
import grpc
from proto import article_pb2, article_pb2_grpc


def main():
    with grpc.insecure_channel("localhost:50051") as channel:
        stub = article_pb2_grpc.ArticleServiceStub(channel)
        response, call = stub.ArticleList.with_call(
            article_pb2.ArticleListRequest(page=1),
            metadata=(
                ("username", 'zhiliao'),
                ("accesstoken", "gRPC Python is great"),
            ),
        )
        print(response.articles)
        # 获取服务端返回的元数据
        for key, value in call.trailing_metadata():
            print(f"收到服务端返回的元数据：key={key}, value={value}")


if __name__ == '__main__':
    main()
```

客户端的元数据的key，不能出现大写字母以及中文。

## 二、拦截器
gRPC中，可以设置类似Django或FastAPI中的中间件，不过它不叫中间件，而叫做`Interceptor`（拦截器）。我们可以在拦截器中模仿Django的中间件一样，做一些请求服务前的拦截处理，比如校验用户信息。这里我们以拦截器的使用示例代码如下。

### 服务端：
服务端实现拦截器，需要继承自`grpc.ServerInterceptor`，在`intercept_service`方法中实现具体的拦截逻辑。然后再把写好的拦截器，传给`grpc.server`的`interceptors`参数中。

```python
from proto import article_pb2, article_pb2_grpc
import grpc
from concurrent import futures

def _unary_unary_rpc_terminator(code, details):
    def terminate(ignored_request, context):
        context.abort(code, details)

    return grpc.unary_unary_rpc_method_handler(terminate)

class MyInterceptor(grpc.ServerInterceptor):
    def intercept_service(self, continuation, handler_call_details):
        metadata = dict(handler_call_details.invocation_metadata)
        if 'authorization' in metadata and metadata['authorization'] == 'zhiliao':
            handler = continuation(handler_call_details)
            return handler
        else:
            return _unary_unary_rpc_terminator(grpc.StatusCode.UNAUTHENTICATED, '请先登录！')

class ArticleService(article_pb2_grpc.ArticleServiceServicer):
    def ArticleList(self, request, context):
        response = article_pb2.ArticleListResponse()
        articles = [
            article_pb2.Article(id=1, title='xx', content='yy', create_time='2030-10-10')
        ]
        response.articles.extend(articles)
        return response

def main():
    server = grpc.server(futures.ThreadPoolExecutor(max_workers=10), interceptors=[MyInterceptor()])
    article_pb2_grpc.add_ArticleServiceServicer_to_server(ArticleService(), server)
    server.add_insecure_port("[::]:5000")
    server.start()
    server.wait_for_termination()


if __name__ == '__main__':
    main()
```

### 客户端：
客户端的拦截器需要继承自`grpc.UnaryUnaryClientInterceptor`，然后实现`intercept_unary_unary`方法。接着再在创建`channel`时，使用`grpc.intercept_channel`接收拦截器，并将返回的`interceptor_channel`传给`Stub`对象。

```python
import grpc
from proto import article_pb2, article_pb2_grpc

class ClientInterceptor(
    grpc.UnaryUnaryClientInterceptor
):

    def intercept_unary_unary(self, continuation, client_call_details, request):
        print("开始拦截")
        response = continuation(client_call_details, request)
        print("结束拦截")
        return response

def main():
    with grpc.insecure_channel("localhost:5000") as channel:
        intercept_channel = grpc.intercept_channel(
            channel, ClientInterceptor()
        )
        stub = article_pb2_grpc.ArticleServiceStub(intercept_channel)
        request = article_pb2.ArticleListRequest(page=1)
        metadata = (
            ('authorization', 'zhiliao'),
        )
        response, call = stub.ArticleList.with_call(request, metadata=metadata)
        print(response.articles)


if __name__ == '__main__':
    main()
```

### 更多
`gRPC`自带的拦截器功能比较有限，用起来不是很方便。为了改善拦截器的用户体验，有一位大神开发了一个`grpc-interceptor`的插件，让我们能非常方便的在拦截器中使用`request`和`context`对象。首先通过以下命令安装：

```bash
$ pip install grpc-interceptor==0.15.4
```

服务端示例示例代码如下。

```python
from grpc_interceptor import ServerInterceptor

class UserInterceptor(ServerInterceptor):
    def intercept(
        self,
        method: Callable,
        request_or_iterator: Any,
        context: grpc.ServicerContext,
        method_name: str,
    ) -> Any:
        # 将user对象绑定到context上，以后在服务中就可以直接使用了
        setattr(context, 'user', self.user)
        return method(request_or_iterator, context)
    
    def intercept_service(self, continuation, handler_call_details):
        metadata = dict(handler_call_details.invocation_metadata)
        if 'authorization' in metadata and metadata['authorization'] == 'zhiliao':
            # 从数据库中获取User数据
            self.user = {"id": 1, 'username': '知了'}
        else:
            self.user = None
        return super().intercept_service(continuation, handler_call_details)
```

`grpc-interceptor` github地址：[https://github.com/d5h-foss/grpc-interceptor](https://github.com/d5h-foss/grpc-interceptor)

## 三、错误处理
`gRPC`与`http`协议类似，也可以在请求结果非正常时返回错误状态码和错误信息。服务端通过在`context`中设置状态码和错误信息即可返回错误。

### 服务端实现：
```python
class ArticleService(article_pb2_grpc.ArticleServiceServicer):
    def ArticleList(self, request, context):
        response = article_pb2.ArticleListResponse()
        context.set_code(grpc.StatusCode.UNAUTHENTICATED)
        context.set_details('请先登录！')
        return response
```

### 客户端实现：
```python
def main():
    with grpc.insecure_channel("localhost:50051") as channel:
        interceptor_channel = grpc.intercept_channel(channel, RequestInterceptor())
        stub = article_pb2_grpc.ArticleServiceStub(interceptor_channel)
        try:
            response, call = stub.ArticleList.with_call(
                article_pb2.ArticleListRequest(page=1),
            )
            print(response.articles)
        except grpc.RpcError as e:
            detail = e.details()
            code = e.code()
            print(code, detail)
```

`gRPC`关于错误状态码的网址：[https://grpc.io/docs/guides/status-codes/](https://grpc.io/docs/guides/status-codes/)

## 四、超时机制
由于微服务间是通过网络通信的，那么很有可能在微服务通信过程中出现异常，这时候如果无限制等待，那么将会影响客户体验。我们可以设置客户端在请求服务端时等待的最长时间，即通过`timeout`参数指定。示例代码如下。

### 服务端
```python
from protos.book import book_pb2, book_pb2_grpc
import asyncio
import grpc


class BookService(book_pb2_grpc.BookServiceServicer):
    async def BookList(self, request, context):
        response = book_pb2.BookListResponse()
        await asyncio.sleep(3)
        return response


async def main():
    # 1. 创建一个grpc服务器对象
    server = grpc.aio.server()
    # 2. 将文章服务，添加到server服务器中
    book_pb2_grpc.add_BookServiceServicer_to_server(BookService(), server)
    # 3. 绑定ip和端口号
    server.add_insecure_port("0.0.0.0:5000")
    # 4. 启动服务
    await server.start()
    print('gRPC服务器已经启动！监听：0.0.0.0:5000')
    # 5. 死循环，等待关闭
    await server.wait_for_termination()


if __name__ == '__main__':
    asyncio.run(main())
```

### 客户端
```python
from protos.book import book_pb2
from protos.book import book_pb2_grpc
import grpc
import asyncio


async def main():
    # 1. 使用上下文管理器创建一个channel对象
    async with grpc.aio.insecure_channel("127.0.0.1:5000") as channel:
        # 2. 要发起grpc请求，需要借助Stub对象
        stub = book_pb2_grpc.BookServiceStub(channel)
        # 3. 构建一个请求对象
        request = book_pb2.BookListRequest()
        request.page = 100
        # 4. 发起请求
        try:
            response = await stub.BookList(request, timeout=1)
            print(response.books)
        except grpc.RpcError as e:
            print(e.code())
            print(e.details())


if __name__ == '__main__':
    asyncio.run(main())
```







> 原文: <https://www.yuque.com/hynever/micro-service/zcvri783yxp6hoih>