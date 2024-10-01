# 第一节：FastAPI介绍

## <font style="color:rgb(26, 32, 41);">一、FastAPI介绍</font>
<font style="color:rgb(26, 32, 41);">FastAPI 是一个高性能的 Python Web 框架，旨在为开发人员提供快速、简洁且强大的 API 开发体验。它于 2017 年由 Sebastián Ramírez 发布，旨在解决当时市场上现有框架在处理 HTTP 请求和响应时的复杂性和维护困难问题。</font>

<font style="color:rgb(26, 32, 41);">Sebastián Ramírez 在开发 FastAPI 时，面临着构建 API 的挑战，他发现现有的框架在处理 HTTP 请求和响应时过于复杂，难以维护。因此，他决定开发一个更简洁、更易于使用的框架，以帮助其他开发人员更专注于业务逻辑，而不是处理 HTTP 请求和响应。</font>

**<font style="color:rgb(26, 32, 41);">FastAPI 支持异步编程</font>**<font style="color:rgb(26, 32, 41);">，这意味着它可以更有效地处理并发请求，提高应用程序的性能。此外，FastAPI 还支持多种数据库和数据存储后端，如 PostgreSQL、MySQL、SQLite、MongoDB 等，以及 ORM 框架，如 SQLAlchemy。这使得开发者可以轻松地将 FastAPI 与各种数据库集成，构建强大的数据驱动应用程序。</font>

<font style="color:rgb(26, 32, 41);">许多知名的公司和项目已经开始使用 FastAPI，包括 Google、Netflix、Amazon、Dropbox 等。这些公司在使用 FastAPI 时，都对其高性能和简洁性给予了高度评价。</font>

<font style="color:rgb(26, 32, 41);">FastAPI 的基准测试成绩也证明了其高性能。根据官方的基准测试结果，FastAPI 在处理并发请求时，其性能优于其他流行的 Web 框架，如 Flask 和 Django。这使得 FastAPI 成为构建高性能 Web 应用程序和 API 的理想选择。</font>

## 二、FastAPI的优点
+ **<font style="color:rgba(0, 0, 0, 0.87);">快速</font>**<font style="color:rgba(0, 0, 0, 0.87);">：可与</font><font style="color:rgba(0, 0, 0, 0.87);"> </font>**<font style="color:rgba(0, 0, 0, 0.87);">NodeJS</font>**<font style="color:rgba(0, 0, 0, 0.87);"> </font><font style="color:rgba(0, 0, 0, 0.87);">和</font><font style="color:rgba(0, 0, 0, 0.87);"> </font>**<font style="color:rgba(0, 0, 0, 0.87);">Go</font>**<font style="color:rgba(0, 0, 0, 0.87);"> </font><font style="color:rgba(0, 0, 0, 0.87);">并肩的极高性能（归功于 Starlette 和 Pydantic）。</font>[最快的 Python web 框架之一](https://fastapi.tiangolo.com/zh/#_11)<font style="color:rgba(0, 0, 0, 0.87);">。</font>
+ **<font style="color:rgba(0, 0, 0, 0.87);">高效编码</font>**<font style="color:rgba(0, 0, 0, 0.87);">：提高功能开发速度约 200％ 至 300％。</font>
+ **<font style="color:rgba(0, 0, 0, 0.87);">更少 bug</font>**<font style="color:rgba(0, 0, 0, 0.87);">：减少约 40％ 的人为（开发者）导致错误。</font>
+ **<font style="color:rgba(0, 0, 0, 0.87);">智能</font>**<font style="color:rgba(0, 0, 0, 0.87);">：极佳的编辑器支持。处处皆可</font><font style="color:rgba(0, 0, 0, 0.87);">自动补全</font><font style="color:rgba(0, 0, 0, 0.87);">，减少调试时间。</font>
+ **<font style="color:rgba(0, 0, 0, 0.87);">简单</font>**<font style="color:rgba(0, 0, 0, 0.87);">：设计的易于使用和学习，阅读文档的时间更短。</font>
+ **<font style="color:rgba(0, 0, 0, 0.87);">简短</font>**<font style="color:rgba(0, 0, 0, 0.87);">：使代码重复最小化。通过不同的参数声明实现丰富功能。bug 更少。</font>
+ **<font style="color:rgba(0, 0, 0, 0.87);">健壮</font>**<font style="color:rgba(0, 0, 0, 0.87);">：生产可用级别的代码。还有自动生成的交互式文档。</font>
+ **<font style="color:rgba(0, 0, 0, 0.87);">标准化</font>**<font style="color:rgba(0, 0, 0, 0.87);">：基于（并完全兼容）API 的相关开放标准：</font>[OpenAPI](https://github.com/OAI/OpenAPI-Specification)<font style="color:rgba(0, 0, 0, 0.87);"> (以前被称为 Swagger) 和 </font>[JSON Schema](https://json-schema.org/)<font style="color:rgba(0, 0, 0, 0.87);">。</font>

## <font style="color:rgba(0, 0, 0, 0.87);">三、安装</font>
使用以下命令即可安装：

```shell
$ pip install fastapi==0.111.0
```

## 四、一个最简单的FastAPI程序
在学习FastAPI时，我们主要采用异步版本。

### 代码
```python
from typing import Union

from fastapi import FastAPI

app = FastAPI()


@app.get("/")
async def read_root():
    return {"Hello": "World"}


@app.get("/items/{item_id}")
async def read_item(item_id: int, q: Union[str, None] = None):
    return {"item_id": item_id, "q": q}
```

### 运行
通过以下命令即可启动运行该项目：

```python
# 在开发环境
$ fastapi dev main.py

# 在生产环境（底层也是基于uvicorn）
$ fastapi run
```

然后在PostMan中即可通过：`http://127.0.0.1:8000`访问到项目。以后代码发生修改，只要按一下保存，那么就会自动重新加载代码。

### Uvicorn
uvicorn是一个高性能的异步web服务器。我们也可以使用uvicorn来直接运行FastAPI项目。首先通过以下命令安装uvicorn：

```shell
$ pip install 'uvicorn[standard]'
```

然后可以通过以下命令运行项目：

```shell
$ uvicorn main:app --reload
```

### 修改host和port
不管你是用`fastapi dev main.py`还是`uvicorn main:app --reload`，都可以通过`--host`和`--port`来指定具体的host和port。比如：

```shell
$ fastapi dev main.py --port 8001
$ uvicorn main:app --reload --port 8002
```



## 五、官方文档
官方文档：[https://fastapi.tiangolo.com/](https://fastapi.tiangolo.com/)



> 原文: <https://www.yuque.com/hynever/wms8gi/goyfki66ig3n1rpf>