# 第三节：Pydantic介绍

在其他web框架中，比如Django或Flask，都有相应的表单类用来校验前端上传的数据是否合法。在FastAPI中则是通过`Pydantic`来实现的。Pydantic有以下特点：

+ <font style="color:rgb(26, 32, 41);">由类型提示提供支持——使用Pydantic，模式验证和序列化由类型注释控制。更少的学习，更少的代码编写，以及与IDE和静态分析工具的集成。</font>
+ <font style="color:rgb(26, 32, 41);">速度——Pydantic的核心验证逻辑是用Rust编写的。因此，Pydantic是Python中最快的数据验证库之一。</font>
+ <font style="color:rgb(26, 32, 41);">JSON模式——Pydantic模型可以发出JSON模式，允许与其他工具轻松集成。</font>
+ <font style="color:rgb(26, 32, 41);">严格和宽松模式- Pydantic可以在Strict =True模式(数据不被转换)或Strict =False模式下运行，Pydantic试图在适当的地方将数据强制转换为正确的类型。</font>
+ <font style="color:rgb(26, 32, 41);">自定义——Pydantic允许自定义验证器和序列化器以许多强大的方式改变数据的处理方式。</font>
+ <font style="color:rgb(26, 32, 41);">生态系统——PyPI上大约有8000个包使用Pydantic，包括大量流行的库，如FastAPI、huggingface、Django Ninja、SQLModel和LangChain。</font>
+ <font style="color:rgb(26, 32, 41);">Battle测试——Pydantic每月下载量超过7000万次，被所有FAANG公司和纳斯达克25家最大公司中的20家使用。如果你想用Pydantic做点什么，别人可能已经做过了。</font>

Pydantic官网：[https://docs.pydantic.dev/latest/](https://docs.pydantic.dev/latest/)

## 一、安装：
通过以下命令即可安装：

```bash
$ pip install pydantic==2.7.4
```

## 二、基本使用
`Pydantic`是通过定义模型，以及在模型中指定字段来对值进行校验的。示例代码如下：

```python
from datetime import date

from pydantic import BaseModel, PositiveInt


class User(BaseModel):
    id: int  
    name: str = 'John Doe'  
    date_joined: date | None
    tastes: dict[str, PositiveInt]

external_data = {
    'id': 123,
    'date_joined': '2030-06-01',  
    'tastes': {
        'wine': 9,
        b'cheese': 7,  
        'cabbage': '1',  
    },
}

user = User(**external_data) 
print(user.id, user.name)
```

上述代码的相关解释如下：

+ 校验类必须继承自`BaseModel`，上述代码中类`User(BaseModel)`。
+ `id`字段：必须为整形，或者可以转换为整形的值，并且由于没有指定默认值，或者使用`Optional`，所以`id`字段的值不能为空。
+ `name`字段：必须为字符串类型，由于`name`指定了默认值，所以`name`字段可以为空。
+ `signup_ts`字段：为`datetime`类型或为None，即该字段可以为空。
+ `tastes`字段：为字典类型，字典的`key`为`str`类型，`value`为`PositiveInt`（正整形）。

传给`User`模型的字段的值有可能不满足要求，那么我们如何获取错误信息呢？看下面示例代码：

```python
external_data = {'id': 'not an int', 'tastes': {}}  

try:
    user = User(**external_data)  
except ValidationError as e:
    print(e.errors())
```







> 原文: <https://www.yuque.com/hynever/wms8gi/txkggvgmbrugq89g>